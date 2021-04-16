# Overview

Kustomize controller 跟其他的operator一样，也是维持actual state --> desire state的一个循环。

有意思的是，这个kustomize  controller实际上，他会从Fluxcd的source controller里面去获取来源：gitrepository/bucket，然后再做一个类似kustomize build的操作，再apply到集群中。

当来源git repository发生改变，或者本身的kustomization的对象发生改变，比如postBuild的环境变量更改之类，也会被定期同步到K8S中。





# 问题

我在看源码之前，想了一下，有两个问题是我比较关心的：

1. 是否无脑，只要git repo来源有改变，都kubectl apply到集群中？ 
2. 如果上一次被fluxcd 部署的pod， 然后被人为删除， fluxcd会去监听所有资源的informer吗？还是压根不关心呢？
3. 如何从git repository / bucket来源去获取manifest? 这两个微服务是如何做数据交换？
4. 如何去判断上一次git repo的来源跟这一次git repo的来源去做资源清理？

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
8. 尝试使用 kubectl 命令去dry-run apply测试一下
9. 正式apply进入集群(我压根没有想到，居然真的是用shell call kubectl apply ...)
10. 资源删除（GC）
11. 健康检查

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
		...
	}

	// 尝试使用 kubectl 命令去dry-run apply测试一下
	err = r.validate(ctx, kustomization, impersonation, dirPath)
	if err != nil {
		...
	}

	// 正式apply进入集群
	changeSet, err := r.applyWithRetry(ctx, kustomization, impersonation, source.GetArtifact().Revision, dirPath, 5*time.Second)
	if err != nil {
		...
	}

	// 资源删除（GC）
	err = r.prune(ctx, kubeClient, kustomization, checksum)
	if err != nil {
		...
	}

	// 健康检查
	err = r.checkHealth(ctx, statusPoller, kustomization, source.GetArtifact().Revision, changeSet != "")
	if err != nil {
		...
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

工作流程如下：

1. 找到目录和kustomization.yaml 文件，拼接
2. 在目录执行kustomize build， 如果kustomization对象的.Spec.PostBuild 不为空，就获取PostBuild的key value，然后对kustomize build之后的结果做变量替换， 然后再把结果做sha1.Sum 计算checksum
3. 获取kustomization对象的.Spec.Prune， 如果值是true （也就是开启GC 功能），那么在目录里面产生 GC的文件：kustomization-gc-labels.yaml

```go
func (r *KustomizationReconciler) generate(ctx context.Context, kubeClient client.Client, kustomization kustomizev1.Kustomization, dirPath string) (string, error) {
	gen := NewGenerator(kustomization, kubeClient)
	return gen.WriteFile(ctx, dirPath)
}

func (kg *KustomizeGenerator) WriteFile(ctx context.Context, dirPath string) (string, error) {
    // 找到目录和kustomization.yaml 文件，拼接
	kfile := filepath.Join(dirPath, konfig.DefaultKustomizationFileName())
	// 执行kustomize build， 如果kustomization对象的.Spec.PostBuild 不为空，就获取PostBuild的key value，然后对kustomize build之后的结果做变量替换， 然后再把结果做sha1.Sum 计算checksum
	checksum, err := kg.checksum(ctx, dirPath)
	if err != nil {
		return "", err
	}
	// 获取kustomization对象的.Spec.Prune， 如果值是true （也就是开启GC 功能），那么在目录里面产生 GC的文件：kustomization-gc-labels.yaml
	if err := kg.generateLabelTransformer(checksum, dirPath); err != nil {
		return "", err
	}
	// 读取目录的kustomization.yaml的数据
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
	// 反序列化
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
	// 设置namespace 
	if kg.kustomization.Spec.TargetNamespace != "" {
		kus.Namespace = kg.kustomization.Spec.TargetNamespace
	}
	// 获取kustomization对象的.Spec.PatchesStrategicMerge
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
	// 获取kustomization对象的.Spec.Images 然后等一下用来替换
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
	// 解析成yaml 返回
	kd, err := yaml.Marshal(kus)
	if err != nil {
		return "", err
	}

	return checksum, ioutil.WriteFile(kfile, kd, os.ModePerm)
}

```



### 资源清理

工作流程如下：

判断对象.status.Snapshot 是否为空，如果不为空的情况下：

轮询namespace scope和clusterscope 的资源 ，然后使用unstructure ，尝试去get到资源，然后再删除

```go
func (kgc *KustomizeGarbageCollector) Prune(timeout time.Duration, name string, namespace string) (string, bool) {
	changeSet := ""
	outErr := ""

	ctx, cancel := context.WithTimeout(context.Background(), timeout+time.Second)
	defer cancel()

	for ns, gvks := range kgc.snapshot.NamespacedKinds() {
		...
	}

	for _, gvk := range kgc.snapshot.NonNamespacedKinds() {
		...
	}

	if outErr != "" {
		return outErr, false
	}
	return changeSet, true
}
```



# 总结





1. 是否无脑，只要git repo来源有改变，都kubectl apply到集群中？ 

   回答： 其实是定期就kubectl apply过去

2. 如果上一次被fluxcd 部署的pod， 然后被人为删除， fluxcd会去监听所有资源的informer吗？还是压根不关心呢？

   回答：压根不关心

3. 如何从git repository / bucket来源去获取manifest? 这两个微服务是如何做数据交换？

   回答： 会读source例如gitrepository对象的status.artifact.url 使用http去下载artifact 去做数据交换

4. 如何去判断上一次git repo的来源跟这一次git repo的来源去做资源清理？

   回答：其实是根据kustomization

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

