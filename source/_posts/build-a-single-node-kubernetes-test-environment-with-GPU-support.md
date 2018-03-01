---
title: 搭建支持GPU的单节点Kubernetes测试环境
date: 2018-03-01 15:00:10
tags: ['Kubernetes', 'GPU', '单节点']
---

本文介绍了如何在不使用Minikube的情况下，在一台机器上搭建支持GPU的k8s测试环境。为什么要这么干呢——因为Minikube对GPU的支持还不完善，而手头正好在做的是一个使用GPU的机器学习项目。为了踩使用k8s编排机器学习微服务的坑，需要搭建一个本地的单机测试环境。此外，在写本篇博客时，即使是Google英文站也很少能搜索到不使用Minikube进行单机测试环境部署的文章，因此，在摸爬滚打一阵后，我决定写下此文……

<!--more-->

# 0x00 准备工作
本文假设：
* 已经安装好了Nvidia Driver（这篇文章重点是使用GPU，如果不用GPU，那么使用Minikube就好了）
* 已经安装好了Docker
* 已经安装好了Golang的开发环境（书写本文时使用的是 go1.9.4）

# 0x01 安装nvidia-docker
nvidia-docker是一个能让Docker容器内部的程序访问显卡的Docker组件，由Nvidia开发和维护，其项目地址为[https://github.com/NVIDIA/nvidia-docker](https://github.com/NVIDIA/nvidia-docker)

> nvidia-docker项目的首页有详细的安装方法，直接阅读并按照文档进行安装即可。需要说明的是，这个组件对Docker的版本比较挑剔，在安装时如果出现版本冲突的情况，需要升级或者降级Docker的版本，以解决冲突。

安装完成后，需要向`/etc/docker/daemon.json`这个文件中添加一行`"default-runtime": "nvidia"`,将nvidia-docker设置为默认的runtime组件，修改后的`daemon.json`应该是这个样子的：
```json
{
    "default-runtime": "nvidia",
    "runtimes": {
        "nvidia": {
            "path": "/usr/bin/nvidia-container-runtime",
            "runtimeArgs": []
        }
    }
}
```

在安装完成后，可以执行下面的命令，检测容器内运行的nvidia-smi是否可以正确检测到显卡信息,如果一切正常，就可以进行下一步了。
```shell
# 注意，按照nvidia-docker文档里的说明，应该执行下面的这一行，通过--runtime=nvidia来手动指定runtime
# docker run --runtime=nvidia --rm nvidia/cuda nvidia-smi
# 但是，后续k8s在编排容器时，需要nvidia作为默认的runtime, 这也是我们修改daemon.json的目的，所以，我们再测试的时候，不要手工指定runtime,看默认值是否已经被修改
docker run --rm nvidia/cuda nvidia-smi
```



# 0x02 安装etcd
从etcd的github仓库[https://github.com/coreos/etcd](https://github.com/coreos/etcd)下载并安装etcd的二进制程序.
感谢Golang的静态编译，把下载后得到的二进制文件移动到/usr/bin下，加上执行权限就行了，so easy.

# 0x03 获取k8s的源码并启动集群
为了简单而快速的启动集群，我们需要使用k8s项目源码里面的一个名为`local-up-cluster.sh`的脚本，获取源码的方式在k8s仓库首页[https://github.com/kubernetes/kubernetes](https://github.com/kubernetes/kubernetes)有详细说明，作为搬运工搬运如下：
```
go get -d k8s.io/kubernetes
cd $GOPATH/src/k8s.io/kubernetes
```
执行该命令后，k8s的项目代码被拉取到Golang的GOPATH里面，进入源码目录后，需要做两处调整。

>注意：以下两处调整可能会随着k8s的版本迭代而发生改变，如不能正常操作，请自行修改调试。如有BUG，欢迎在https://github.com/myrfy001/blog.ideawand.com 发起Issue讨论。

* 将拉取的代码回退到v1.9.0版，即在源码目录下执行
```
git checkout -b v1.9.0 v1.9.0
```
  这么做的目的是为了和后面将使用的k8s-device-plugin插件保持版本一致。如果日后k8s-device-plugin的版本已经升级，则可以使用新的k8s源码和与之版本对应的k8s-device-plugin。相关问题可以参考我在github上提出的issue: https://github.com/NVIDIA/k8s-device-plugin/issues/32

* 开启k8s处于测试阶段的device-plugin特性
  k8s的`kubelet`的用于打通容器和外部硬件连接的device-plugin特性还在测试中，在写本文时默认是关闭的，为使用该功能，需要在启动`kubelet`组件时增加一个开关参数。
  修改`$GOPATH/src/k8s.io/kubernetes/hack/local-up-cluster.sh`,在其文件最开头加入如下一行代码，以定义开关参数：
  ```
  FEATURE_GATES="DevicePlugins=true"
  ```

做完上述修改后，使用下列命令启动本地集群
```
sudo ./hack/local-up-cluster.sh
```

# 0x04 启动k8s-device-plugin组件
`k8s-device-plugin`是一个由Nvidia开发并维护的工具，用来协助k8s在容器中运行使用GPU的应用。前文提到的更改docker默认runtime、开启kubelet的device-plugin特性等步骤，均是为了配合该插件运行。相关信息可以在其项目文档https://github.com/NVIDIA/k8s-device-plugin 中找到

为k8s添加这一工具很简单，运行下列命令即可，该工具会以Daemonset的形式运行：
```shell
# 注意，此处拉取的是1.9版本，与前文提到的k8s源码版本吻合，如有需要，请根据实际情况修改
kubectl create -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v1.9/nvidia-device-plugin.yml
```


至此，大功告成！

接下来就可以按照k8s-device-plugin的[github项目页面](https://github.com/NVIDIA/k8s-device-plugin)上提供的模板来启动Pod并测试。

# 0x05 一些调试信息
* 如果遇到Pod无法启动的情况，可以尝试在Docker中手动运行k8s-device-plugin组件，这样可以得到比较详细的错误日志
```
docker run -it -v /var/lib/kubelet/device-plugins:/var/lib/kubelet/device-plugins nvidia/k8s-device-plugin:1.9
```

* 比较老的显卡可能不支持容器……



