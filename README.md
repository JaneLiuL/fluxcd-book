# Overview

GitOps 越来越重要， Fluxcd作为GitOps的在CD的一种实现也变得越来越受欢迎。

# 大纲

在工作中经常有人询问我，为什么我使用FluxCD可以安装成功监控github，却没有办法让FluxCD监控公司内部的私库？ FluxCD比Fluxv1的版本优点在哪里？ 等等问题。此书从小白入门开始，从如何安装，功能使用，架构方面，部分源码理解以及测试。

1. 安装
   1. 使用terraform 安装fluxcd
   2. 使用flux cli 安装fluxcd
2. 功能讲解
   1. [入门] 以git repo 作为来源，某个dev 目录作为kustomization path,  如何让fluxcd 监控此来源并部署进K8S
   2. [入门] 以git repo作为来源，如何监控并部署helm chart
   3. [经验] 多环境多集群，如何最大利用使用FluxCD来部署应用
   4. [经验]  
3. 架构
4. 源码理解
   1.    [source-controller](/source-code-understanding/source-controller.md)
   2.    [helm-controller](/source-code-understanding/helm-controller.md)
   3.    [kustomize-controller](/source-code-understanding/kustomize-controller.md)
   4. notification-controller
5. 测试
   1. 功能测试
   2. 压力测试