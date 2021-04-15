# Overview

Kustomize controller 跟其他的operator一样，也是维持actual state --> desire state的一个循环。

有意思的是，这个kustomize  controller实际上，他会从Fluxcd的source controller里面去获取来源：gitrepository，然后再做一个类似kustomize build的操作，再apply到集群中。

当来源git repository发生改变，或者本身的kustomization的对象发生改变，比如postBuild的环境变量更改之类，也会被定期同步到K8S中。

# 数据结构

Spec

```go
type KustomizationSpec struct {
	// 依赖其他对象，才进行kustomize build   +optional
	DependsOn []dependency.CrossNamespaceDependencyReference `json:"dependsOn,omitempty"`

	// 将集群中的secret 在apply在集群之前先解密   +optional
	Decryption *Decryption `json:"decryption,omitempty"`
	// +required
	Interval metav1.Duration `json:"interval"`

	// +optional
	RetryInterval *metav1.Duration `json:"retryInterval,omitempty"`

	// 这个是用于在remote cluster作认证   +optional
	KubeConfig *KubeConfig `json:"kubeConfig,omitempty"`

	// kustomization.yaml 文件的路径
	// +optional
	Path string `json:"path,omitempty"`
	
	//在kustomize build之后的action, 执行一种类似overlay的替换操作  +optional
	PostBuild *PostBuild `json:"postBuild,omitempty"`

	// Prune enables garbage collection.  +required
	Prune bool `json:"prune"`
    ..
    
	// Reference of the source where the kustomization file is.
	// +required
	SourceRef CrossNamespaceSourceReference `json:"sourceRef"`

	// 这是一个enable 挂起操作的开关，当打开，也就是意味着不执行kustomimze build
	// +optional
	Suspend bool `json:"suspend,omitempty"`

	// TargetNamespace 覆盖kustomization.yaml的namespace 字段  +optional
	TargetNamespace string `json:"targetNamespace,omitempty"`
	
	// validation, apply and health checking 操作的timeout 时间  +optional
	Timeout *metav1.Duration `json:"timeout,omitempty"`

	// Validate the Kubernetes objects before applying them on the cluster.
	// The validation strategy can be 'client' (local dry-run), 'server'
	// (APIServer dry-run) or 'none'.
	// When 'Force' is 'true', validation will fallback to 'client' if set to
	// 'server' because server-side validation is not supported in this scenario.
	// +kubebuilder:validation:Enum=none;client;server
	// +optional
	Validation string `json:"validation,omitempty"`

	// Force instructs the controller to recreate resources
	// when patching fails due to an immutable field change.
	// +kubebuilder:default:=false
	// +optional
	Force bool `json:"force,omitempty"`
}

```



Status

```go
type KustomizationStatus struct {
	// 这是一个每次迭代之后增加的数字 +optional
	ObservedGeneration int64 `json:"observedGeneration,omitempty"`

	// +optional
	Conditions []metav1.Condition `json:"conditions,omitempty"`

	// 上一次apply的Revision
	LastAppliedRevision string `json:"lastAppliedRevision,omitempty"`

	// 上一次reconcicile的revision
	LastAttemptedRevision string `json:"lastAttemptedRevision,omitempty"`

	meta.ReconcileRequestStatus `json:",inline"`

	// The last successfully applied revision metadata.
	// +optional
	Snapshot *Snapshot `json:"snapshot,omitempty"`
}
```





# Reconcile

operator的一般流程，包括finalizer的检查， deletetimestamp的检查，我们就跳过。直接到我们最关心的工作流程：

1. 检查对象是否enable了suspend: kustomization.spec.suspend，如果是，那么直接返回
2. 获取对象 kustomization的来源 .spec.sourceRef （gitRepository或者Bucket），读取对象的来源
3. 尝试获取来源的artifact， 如果为空，说明source 是不ready
4. 检查对象的依赖
5. 设置对象kustomization的status，将condition跟状态设置成in progress
6. 进入真正的reconcile 调和，见下方reconcile 
7.  如果reconciliation 错误，就重新入队，重新尝试调和
8. 广播这次调和成功，然后根据interal时间重新入队

```go
func (r *KustomizationReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	....

	// 检查finalizer
	if !controllerutil.ContainsFinalizer(&kustomization, kustomizev1.KustomizationFinalizer) {
		controllerutil.AddFinalizer(&kustomization, kustomizev1.KustomizationFinalizer)
		...
	}

	// 检查对象是否被删除
	if !kustomization.ObjectMeta.DeletionTimestamp.IsZero() {
		return r.reconcileDelete(ctx, kustomization)
	}

	// 检查对象是否被挂起
	if kustomization.Spec.Suspend {
		return ctrl.Result{}, nil
	}

	// 获取对象的来源
	source, err := r.getSource(ctx, kustomization)
    
    // 尝试获取来源的artifact， 如果为空，说明source 是不ready
    // 这个GetArtifact() 其实就是查source对象的status， 例如如果来源对象是GitRepository， 就查它的.Status.Artifact
    if source.GetArtifact() == nil {
		msg := "Source is not ready, artifact not found"
		...
		return ctrl.Result{RequeueAfter: kustomization.GetRetryInterval()}, nil
	}        
    // 检查对象的依赖
	if len(kustomization.Spec.DependsOn) > 0 {
		...
	}
    
    // 设置对象kustomization的status，将condition跟状态设置成in progress
	kustomization = kustomizev1.KustomizationProgressing(kustomization)
	if err := r.patchStatus(ctx, req, kustomization.Status); err != nil {
		return ctrl.Result{Requeue: true}, err
	}

	// reconcile kustomization by applying the latest revision
	reconciledKustomization, reconcileErr := r.reconcile(ctx, *kustomization.DeepCopy(), source)
	if err := r.patchStatus(ctx, req, reconciledKustomization.Status); err != nil {
		log.Error(err, "unable to update status after reconciliation")
		return ctrl.Result{Requeue: true}, err
	}
	r.recordReadiness(ctx, reconciledKustomization)
	
    // 如果reconciliation 错误，就重新入队，重新尝试调和
	if reconcileErr != nil {
		...
		return ctrl.Result{RequeueAfter: kustomization.GetRetryInterval()}, nil
	}
	
    // 广播这次调和成功，然后根据interal时间重新入队
	log.Info(fmt.Sprintf("Reconciliation finished in %s, next run in %s",
		time.Now().Sub(reconcileStart).String(),
		kustomization.Spec.Interval.Duration.String()),
		"revision",
		source.GetArtifact().Revision,
	)
	r.event(ctx, reconciledKustomization, source.GetArtifact().Revision, events.EventSeverityInfo,
		"Update completed", map[string]string{"commit_status": "update"})
	return ctrl.Result{RequeueAfter: kustomization.Spec.Interval.Duration}, nil
}
```



## reconcile 

工作流程如下：

1. 获取对象kustomization的annotation, 设置对象的status
2. 在本地创建临时目录， 用于下载source的artifact
3.  使用http去下载 source的artifact 以及解压untar
4. 检查kustomize build的path 是否存在
5. 创建kube-client
6. 创建 kustomization.yaml 文件，以及检查manifest的checksum
7. 构建kustomization文件，并且生成GC 的snapshot

```go
func (r *KustomizationReconciler) reconcile(
	ctx context.Context,
	kustomization kustomizev1.Kustomization,
	source sourcev1.Source) (kustomizev1.Kustomization, error) {	
    // 获取对象kustomization的annotation, 设置对象的status
	if v, ok := meta.ReconcileAnnotationValue(kustomization.GetAnnotations()); ok {
		kustomization.Status.SetLastHandledReconcileRequest(v)
	}

	// 在本地创建临时目录， 用于下载source的artifact
	tmpDir, err := ioutil.TempDir("", kustomization.Name)
	if err != nil {
		...
	}
	defer os.RemoveAll(tmpDir)

	// 使用http去下载 source的artifact 以及解压
    // URL地址其实就在source对象的Status.Artifact.URL
	err = r.download(source.GetArtifact().URL, tmpDir)
	if err != nil {
		...		
	}

	// 检查kustomize build的path 是否存在
	dirPath, err := securejoin.SecureJoin(tmpDir, kustomization.Spec.Path)
	if err != nil {
		...
	}
	if _, err := os.Stat(dirPath); err != nil {
		....
	}
	
    // 创建kube-client
	impersonation := NewKustomizeImpersonation(kustomization, r.Client, r.StatusPoller, dirPath)
	kubeClient, statusPoller, err := impersonation.GetClient(ctx)
	if err != nil {
		..
	}
	
    // 创建 kustomization.yaml 文件，以及检查manifest的checksum
	checksum, err := r.generate(ctx, kubeClient, kustomization, dirPath)
	if err != nil {
		...
	}
	
    // 构建kustomization文件，并且生成GC 的snapshot
	snapshot, err := r.build(ctx, kustomization, checksum, dirPath)
	if err != nil {
		return kustomizev1.KustomizationNotReady(
			kustomization,
			source.GetArtifact().Revision,
			kustomizev1.BuildFailedReason,
			err.Error(),
		), err
	}

	// dry-run apply
	err = r.validate(ctx, kustomization, impersonation, dirPath)
	if err != nil {
		return kustomizev1.KustomizationNotReady(
			kustomization,
			source.GetArtifact().Revision,
			kustomizev1.ValidationFailedReason,
			err.Error(),
		), err
	}

	// apply
	changeSet, err := r.applyWithRetry(ctx, kustomization, impersonation, source.GetArtifact().Revision, dirPath, 5*time.Second)
	if err != nil {
		return kustomizev1.KustomizationNotReady(
			kustomization,
			source.GetArtifact().Revision,
			meta.ReconciliationFailedReason,
			err.Error(),
		), err
	}

	// prune
	err = r.prune(ctx, kubeClient, kustomization, checksum)
	if err != nil {
		return kustomizev1.KustomizationNotReady(
			kustomization,
			source.GetArtifact().Revision,
			kustomizev1.PruneFailedReason,
			err.Error(),
		), err
	}

	// health assessment
	err = r.checkHealth(ctx, statusPoller, kustomization, source.GetArtifact().Revision, changeSet != "")
	if err != nil {
		return kustomizev1.KustomizationNotReadySnapshot(
			kustomization,
			snapshot,
			source.GetArtifact().Revision,
			kustomizev1.HealthCheckFailedReason,
			err.Error(),
		), err
	}

	return kustomizev1.KustomizationReady(
		kustomization,
		snapshot,
		source.GetArtifact().Revision,
		meta.ReconciliationSucceededReason,
		"Applied revision: "+source.GetArtifact().Revision,
	), nil
}
```



### 计算checksum

```go
func (r *KustomizationReconciler) generate(ctx context.Context, kubeClient client.Client, kustomization kustomizev1.Kustomization, dirPath string) (string, error) {
	gen := NewGenerator(kustomization, kubeClient)
	return gen.WriteFile(ctx, dirPath)
}

func (kg *KustomizeGenerator) WriteFile(ctx context.Context, dirPath string) (string, error) {
	kfile := filepath.Join(dirPath, konfig.DefaultKustomizationFileName())

	checksum, err := kg.checksum(ctx, dirPath)
	if err != nil {
		return "", err
	}

	if err := kg.generateLabelTransformer(checksum, dirPath); err != nil {
		return "", err
	}

	data, err := ioutil.ReadFile(kfile)
	if err != nil {
		return "", err
	}

	kus := kustypes.Kustomization{
		TypeMeta: kustypes.TypeMeta{
			APIVersion: kustypes.KustomizationVersion,
			Kind:       kustypes.KustomizationKind,
		},
	}

	if err := yaml.Unmarshal(data, &kus); err != nil {
		return "", err
	}

	if len(kus.Transformers) == 0 {
		kus.Transformers = []string{transformerFileName}
	} else {
		var exists bool
		for _, transformer := range kus.Transformers {
			if transformer == transformerFileName {
				exists = true
				break
			}
		}
		if !exists {
			kus.Transformers = append(kus.Transformers, transformerFileName)
		}
	}

	if kg.kustomization.Spec.TargetNamespace != "" {
		kus.Namespace = kg.kustomization.Spec.TargetNamespace
	}

	for _, m := range kg.kustomization.Spec.PatchesStrategicMerge {
		kus.PatchesStrategicMerge = append(kus.PatchesStrategicMerge, kustypes.PatchStrategicMerge(m.Raw))
	}

	for _, m := range kg.kustomization.Spec.PatchesJSON6902 {
		patch, err := json.Marshal(m.Patch)
		if err != nil {
			return "", err
		}
		kus.PatchesJson6902 = append(kus.PatchesJson6902, kustypes.Patch{
			Patch:  string(patch),
			Target: adaptSelector(&m.Target),
		})
	}

	for _, image := range kg.kustomization.Spec.Images {
		newImage := kustypes.Image{
			Name:    image.Name,
			NewName: image.NewName,
			NewTag:  image.NewTag,
		}
		if exists, index := checkKustomizeImageExists(kus.Images, image.Name); exists {
			kus.Images[index] = newImage
		} else {
			kus.Images = append(kus.Images, newImage)
		}
	}

	kd, err := yaml.Marshal(kus)
	if err != nil {
		return "", err
	}

	return checksum, ioutil.WriteFile(kfile, kd, os.ModePerm)
}

```



# 附录

下方是一个gitrepository对象的输出

就可以看到status.artifact.url 下载artifact的地址

```yaml
spec:
  gitImplementation: go-git
  interval: 1m
  ref:
    branch: branch-name
  secretRef:
    name: xx-secret
  timeout: 20s
  url: ssh://git@xxx.git
status:
  artifact:
    checksum: a2f0xx
    lastUpdateTime: "2021-04-15T08:34:18Z"
    path: gitrepository/some-namespace/aaa/2d21xxx.tar.gz
    revision: branch-name/2d21xxx
    url: http://source-controller.flux-system.svc.cluster.local./gitrepository/xxx/-aaa/2d214xxxx.tar.gz
  conditions:
  - lastTransitionTime: "2021-04-15T08:34:18Z"
    message: 'Fetched revision: xxx'
    reason: GitOperationSucceed
    status: "True"
    type: Ready
  observedGeneration: 3
  url: http://source-controller.flux-system.svc.cluster.local./gitrepository/xxx/-aaa/latest.tar.gz

```

