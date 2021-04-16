# Overview

kustomize controller 是一个典型的Kubernetes Operator， 专门为使用Kubernetes定义并使用Kustomize组装的manifest 运行连续交付管道。

# 功能描述

1. check:  检查是否满足依赖条件

2. Fetch: 从源控制器(Git存储库或S3存储桶)获取manifest

3. generate: 如果需要，生成定制kustomization

4. build: 使用kustomization 构建清单

5. decrypt: 使用Mozilla SOPS解密Kubernetes的secret

6. Validate: 验证结果对象

7. impersonate: 模仿Kubernetes帐户

8. apply: 部署到集群

9. prune: 从源文件中删除对象

10. verify: 验证部署状态

11.  alert: 如果出了什么问题就发出警报

12. notify: 如果集群状态改变，通知

# 例子