# Overview

source-controller 是Kubernetes的一个Operator， 其实是定义了外部来源包括Git, Helm Repository 和S3 bucket。

实际上，source-controller 在 Fluxcd是作为一个生产者而存在的，他提供给其他controller artifact的来源。并且所有的认证（username/password， SSH等）是在source-controller来做控制。

![](../images/source-controller-overview.png)



# 功能

1. 对源进行身份验证(SSH、用户/密码、API令牌)

2. 验证源真实性(PGP)

3. 基于更新策略(semver)检测源更改

4. 按需和按时间表获取资源

5. 将获取的资源打包成众所周知的格式(tar.gz, yaml)

6. 使artifacts可以i通过sha或者version来寻址

7. 使相关第三方可以在集群中使用工件

8. 通知感兴趣的第三方源的更改和可用性(status conditions、events、hooks)

9. 响应Git push和Helm chart upload事件(通过notification-controller)

# 数据结构

## Artifact

所有类型的source都需要使用到的一个数据结构。 这个数据是跟其他controller里面helm controller 或者 kustomize controller交换所使用的。

gitrepository 和 helmrepository 以及bucket 都会在Status使用Artifact 来记录artifact信息，以供其他controller下载。

```go
type Artifact struct {
	// Path是该工件的相对文件路径 +required
	Path string `json:"path"`
	
	// artifact 的HTTP 地址 +required
	URL string `json:"url"`

	// 它可以是一个Git commit SHA，一个Git tag，一个Helm index timestamp，一个Helm chart版本，等等。 +optional
	Revision string `json:"revision"`

	// +optional
	Checksum string `json:"checksum"`

	// 上一次更新这个artifact的timestamp  +required
	LastUpdateTime metav1.Time `json:"lastUpdateTime,omitempty"`
}
```



### 接口

所有类型：gitrepository/ helmrepository/ bucket 都要实现Source的接口，去让其他controller 通过GetArtifact/ GetInterval 

```go
type Source interface {
	// GetArtifact returns the latest artifact from the source if present in the
	// status sub-resource.
	GetArtifact() *Artifact
	// GetInterval returns the interval at which the source is updated.
	GetInterval() metav1.Duration
}
```



## gitrepository 

值得注意如下事情：

1. Spec.URL 必须是`^(http|https|ssh)://`  pattern 
2. SecretRef 是k8s 的secret name， 如果是git ssh认证的secret 必须要包含`identity`, `identity.pub ` 和 `known_hosts`
3. Interval 是定期调和时间
4. Timeout 是git 操作的超时时间
5. Reference 是git repo的特定版本：可以是branch 或者Tag 或者Commit

```go
// GitRepository is the Schema for the gitrepositories API
type GitRepository struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Spec   GitRepositorySpec   `json:"spec,omitempty"`
	Status GitRepositoryStatus `json:"status,omitempty"`
}

// GitRepositorySpec defines the desired state of a Git repository.
type GitRepositorySpec struct {
	// The repository URL, can be a HTTP/S or SSH address.
	// +kubebuilder:validation:Pattern="^(http|https|ssh)://"
	// +required
	URL string `json:"url"`

	// The secret name containing the Git credentials.
	// For HTTPS repositories the secret must contain username and password
	// fields.
	// For SSH repositories the secret must contain identity, identity.pub and
	// known_hosts fields.
	// +optional
	SecretRef *meta.LocalObjectReference `json:"secretRef,omitempty"`

	// +required
	Interval metav1.Duration `json:"interval"`

	Timeout *metav1.Duration `json:"timeout,omitempty"`
	
    // git的reference， 可以是branch 或者tag 或者Commit  +optional
	Reference *GitRepositoryRef `json:"ref,omitempty"`

	// Verify OpenPGP signature for the Git commit HEAD points to.
	// +optional
	Verification *GitRepositoryVerification `json:"verify,omitempty"`
	// 默认，任何.git  ,jpg之类的文件扩展都是默认被execude的  
	Ignore *string `json:"ignore,omitempty"`

	Suspend bool `json:"suspend,omitempty"`

	// Determines which git client library to use.
	// Defaults to go-git, valid values are ('go-git', 'libgit2').
	// +kubebuilder:validation:Enum=go-git;libgit2
	// +kubebuilder:default:=go-git
	// +optional
	GitImplementation string `json:"gitImplementation,omitempty"`
}
```



## helmrepository

值得注意的是Helm repository 认证的secret,  如果是HTTP/S的认证就必须要包含username 跟password的 field 。 如果是TLS的secret 必须要包含certFile和keyFile，caCert是optional的

```go
// HelmRepository is the Schema for the helmrepositories API
type HelmRepository struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Spec   HelmRepositorySpec   `json:"spec,omitempty"`
	Status HelmRepositoryStatus `json:"status,omitempty"`
}

// HelmRepositorySpec defines the reference to a Helm repository.
type HelmRepositorySpec struct {
	// The Helm repository URL, a valid URL contains at least a protocol and host.
	// +required
	URL string `json:"url"`
	
	// Helm repository 认证的secret,  如果是HTTP/S的认证就必须要包含username 跟password的 field 。 如果是TLS的secret 必须要包含certFile和keyFile，caCert是optional的  +optional
	SecretRef *meta.LocalObjectReference `json:"secretRef,omitempty"`

	Interval metav1.Duration `json:"interval"`

	Timeout *metav1.Duration `json:"timeout,omitempty"`

	Suspend bool `json:"suspend,omitempty"`
}

```



## bucket

先跳过

```go

```



# Reconcile

## gitrepository reconcile

工作流程如下（忽略Finalizer和Deletetimestamp的流程）：

1. 新建/data/repository.Name 目录，用于保存git clone下来的代码
2. 如果`repository.Spec.SecretRef` 非空，则先获取该git repo的认证方式，然后确保在该namespace下能获取到同名的secret，否则就返回NotReady报错。也就是说，如果我们使用的git repo是需要验证的，那么需要在该namespace下先创建secret
3. 尝试`checkout`操作，并且获取git commit
4. 实例化`artifact`对象，也就是设置该artifact的目录，revision。 如果该artifact有revision那么设置Ready状态
5. 如果传入的GitRepository对象有要求设置校验PGP（也就是.Spec.Verification不为空）， 那么尝试校验
6. 创建artifact目录
7. 在artifact目录下创建一个跟artifact path同名的.lock文件
8. 创建/更新最新的软链接，实际上是获取artifact的目录，在该目录的上一级下创建一个latest.tar.gz，然后链接到
9. 尝试删除artifact path的.lock文件
10. 删除/data/repository.Name 目录

```go
func (r *GitRepositoryReconciler) reconcile(ctx context.Context, repository sourcev1.GitRepository) (sourcev1.GitRepository, error) {
	// create tmp dir for the Git clone
	tmpGit, err := ioutil.TempDir("", repository.Name)
	if err != nil {
		err = fmt.Errorf("tmp dir error: %w", err)
		return sourcev1.GitRepositoryNotReady(repository, sourcev1.StorageOperationFailedReason, err.Error()), err
	}
	defer os.RemoveAll(tmpGit)

	// determine auth method
	auth := &git.Auth{}
	if repository.Spec.SecretRef != nil {
		authStrategy, err := strategy.AuthSecretStrategyForURL(repository.Spec.URL, repository.Spec.GitImplementation)
		if err != nil {
			return sourcev1.GitRepositoryNotReady(repository, sourcev1.AuthenticationFailedReason, err.Error()), err
		}

		name := types.NamespacedName{
			Namespace: repository.GetNamespace(),
			Name:      repository.Spec.SecretRef.Name,
		}

		var secret corev1.Secret
		err = r.Client.Get(ctx, name, &secret)
		if err != nil {
			err = fmt.Errorf("auth secret error: %w", err)
			return sourcev1.GitRepositoryNotReady(repository, sourcev1.AuthenticationFailedReason, err.Error()), err
		}

		auth, err = authStrategy.Method(secret)
		if err != nil {
			err = fmt.Errorf("auth error: %w", err)
			return sourcev1.GitRepositoryNotReady(repository, sourcev1.AuthenticationFailedReason, err.Error()), err
		}
	}

	checkoutStrategy, err := strategy.CheckoutStrategyForRef(repository.Spec.Reference, repository.Spec.GitImplementation)
	if err != nil {
		return sourcev1.GitRepositoryNotReady(repository, sourcev1.GitOperationFailedReason, err.Error()), err
	}
	commit, revision, err := checkoutStrategy.Checkout(ctx, tmpGit, repository.Spec.URL, auth)
	if err != nil {
		return sourcev1.GitRepositoryNotReady(repository, sourcev1.GitOperationFailedReason, err.Error()), err
	}

	// return early on unchanged revision
	artifact := r.Storage.NewArtifactFor(repository.Kind, repository.GetObjectMeta(), revision, fmt.Sprintf("%s.tar.gz", commit.Hash()))
	if apimeta.IsStatusConditionTrue(repository.Status.Conditions, meta.ReadyCondition) && repository.GetArtifact().HasRevision(artifact.Revision) {
		if artifact.URL != repository.GetArtifact().URL {
			r.Storage.SetArtifactURL(repository.GetArtifact())
			repository.Status.URL = r.Storage.SetHostname(repository.Status.URL)
		}
		return repository, nil
	}

	// verify PGP signature
	if repository.Spec.Verification != nil {
		publicKeySecret := types.NamespacedName{
			Namespace: repository.Namespace,
			Name:      repository.Spec.Verification.SecretRef.Name,
		}
		var secret corev1.Secret
		if err := r.Client.Get(ctx, publicKeySecret, &secret); err != nil {
			err = fmt.Errorf("PGP public keys secret error: %w", err)
			return sourcev1.GitRepositoryNotReady(repository, sourcev1.VerificationFailedReason, err.Error()), err
		}

		err := commit.Verify(secret)
		if err != nil {
			return sourcev1.GitRepositoryNotReady(repository, sourcev1.VerificationFailedReason, err.Error()), err
		}
	}

	// create artifact dir
	err = r.Storage.MkdirAll(artifact)
	if err != nil {
		err = fmt.Errorf("mkdir dir error: %w", err)
		return sourcev1.GitRepositoryNotReady(repository, sourcev1.StorageOperationFailedReason, err.Error()), err
	}

	// acquire lock
	unlock, err := r.Storage.Lock(artifact)
	if err != nil {
		err = fmt.Errorf("unable to acquire lock: %w", err)
		return sourcev1.GitRepositoryNotReady(repository, sourcev1.StorageOperationFailedReason, err.Error()), err
	}
	defer unlock()

	// archive artifact and check integrity
	if err := r.Storage.Archive(&artifact, tmpGit, repository.Spec.Ignore); err != nil {
		err = fmt.Errorf("storage archive error: %w", err)
		return sourcev1.GitRepositoryNotReady(repository, sourcev1.StorageOperationFailedReason, err.Error()), err
	}

	// update latest symlink
	url, err := r.Storage.Symlink(artifact, "latest.tar.gz")
	if err != nil {
		err = fmt.Errorf("storage symlink error: %w", err)
		return sourcev1.GitRepositoryNotReady(repository, sourcev1.StorageOperationFailedReason, err.Error()), err
	}

	message := fmt.Sprintf("Fetched revision: %s", artifact.Revision)
	return sourcev1.GitRepositoryReady(repository, artifact, url, sourcev1.GitOperationSucceedReason, message), nil
}

```

