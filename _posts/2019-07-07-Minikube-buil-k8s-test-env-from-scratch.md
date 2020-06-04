---
layout: article
title: Minikube 快速搭建k8s本地环境笔记
tags: K8s environment scratch
---

# Minikube 快速搭建k8s本地环境
---

minikube 操作简易，允许快速搭建一个学习用和开发用的k8s环境

<!-- more -->

## 前提条件：

* 支持虚拟化的机器
* 安装了 Hypervisor 组件， 如 virtual box, hyper-v (windows 推荐使用virtualbox)

## 下载 minikube 和 kubectl

Linux从 googleapis 下载，这个不需要VPN
```bash
curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 \
  && chmod +x minikube
```

Windows 可以用Powershell的 chocolately 工具
```powershell
choco install minikube
```

安装完后可以查看minikube 的版本
```bash
minikube version
```

## 基本命令

```bash
# minikube 管理
minikube start
minikube stop
minikube delete
minikube logs
minikube dashboard  # 开启k8s dashboard web管理页面
```

```bash
# k8s pod 管理
kubectl get pods --all-namespaces
kubectl logs -n=<namespace> <pod-name>
kubectl describe pod -n=<namespace> <pod-name>
kubectl cluster-info [dump]
```

## 注意点

如遇到问题需要删掉当前的cluster 设置：

```bash
minikube stop
minikube delete
rm -r ~/.minikube   # 删的比较彻底， 在 minikube delete 命令执行错误时比较有用
```

minikube 使用国内镜像库， 注意 `--image-repository` 的参数需要 v1.2.0以上的 minikube
```bash
minikube start --registry-mirror=https://xxx.mirror.aliyuncs.com  --image-repository=registry.cn-hangzhou.aliyuncs.com/google_containers
```

minikube dashboard 访问503

1. vm驱动产生了冲突，保留其中一个，彻底删除其它的
2. 无法创建密钥：
   - `kubectl get pod -n=kube-system` 看到kubernetes-dashboard 的pod 状态是 `CrashLoopBackOff`, 查看pod 日志，发现报错 secrets is forbidden
   - 解决方法： `kubectl create clusterrolebinding kube-system-cluster-admin --clusterrole=cluster-admin --serviceaccount=kube-system:default`

***参考***

- [Install Minikube](https://kubernetes.io/docs/tasks/tools/install-minikube/)
- [Installing Kubernetes with Minikube](https://kubernetes.io/docs/setup/learning-environment/minikube/)
- ["Minikube windows 10 hyperv 本地实验环境"](https://yq.aliyun.com/articles/221687#modal-login)
- [minikube dashboard returns 503 error](https://stackoverflow.com/questions/52916548/minikube-dashboard-returns-503-error-on-macos)
- [CrashLoopBackOff: secrets is forbidden](https://www.devopsschool.com/blog/minikube-error-crashloopbackoff-secrets-is-forbidden-user-systemserviceaccountkube-systemdefault/)
