## reconcile

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

