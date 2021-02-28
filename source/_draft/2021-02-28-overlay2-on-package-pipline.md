---
title: registry + overlay2 在容器云平台发打包发布中的应用
date: 2021-02-10
updated: 2021-02-11
slug:
categories: 技术
tag:
  - registry
  - images
  - OCI
copyright: true
comment: true
---

## 背景

自从去年五月份入职后一直在负责公司 toB 产品：Compass 容器云平台和 Clever 人工智能云平台的发布及部署工作。性质上有点类似于 Kubernetes SIGs 社区的 Release 团队。使用期期间的主要工作就是优化我们先有的打包发布流程。在这期间对打包发布流水线优化了很多，其中最突出的是镜像同步的优化，将镜像同步的速度提升了 5 到 15 倍。大大缩短了整个产品的发布耗时，也得到了同事们的一致好评。于是今天就想着把这项优化和背后的原理分享出来。因为这些代码是开源的，所以也不必担心泄露公司机密什么的。

我们的产品打包时会有一个镜像列表，并根据这个镜像列表在 CI/CD 的流水线镜像仓库里将镜像同步到一个发布归档的镜像仓库和一个打包的镜像仓库。最终会将打包的镜像仓库的 registry 存储目录打包一个未经 gzip 压缩的 tar 包。最终在客户环境部署的时候将这个 tar 包解压到部署的镜像仓库存储目录中，供集群部署和组件部署使用。至于部署的时候为什么可以这样做，其中的原理可以参考我之前写过的文章 [docker registry 迁移至 harbor](https://blog.k8s.li/docker-registry-to-harbor.html)。

在打包的过程中镜像同步会进行两次，每次都会根据一个 images.list 列表将镜像同步到不同的镜像仓库中，同步的方式使用的是  `docker pull –> docker tag –> docker push`。第一次是从CI/CD 流水线镜像仓库（cicd.registry.local）中拉取镜像并 push 到发布归档的镜像仓库(archive.registry.local)中，其目的是归档并备份我们已经发布的镜像。

第二次将镜像从发布归档的镜像仓库 (archive.registry.local) 同步镜像到打包镜像仓库（package.registry.local）中。不同于第一次的镜像同步，这次同步镜像的时候会对镜像仓库做清理的操作，首先清理打包镜像仓库的存储目录，然后容器 registry 容器让 registry 重新提取镜像的元数据信息到内存中。其目的是清理旧数据，防止历史的镜像带入本次发布版本的安装包中。镜像同步完成之后会将整个打包镜像仓库的存储目录打包成一个 tar 包，并放到产品安装包中。

## 问题

我刚入职的时候，我们的产品发布耗时最久的就是镜像同步阶段， 最长的时候耗时 `2h30min`。耗时这么久的主要原因如下：

### docker 性能问题

因为在做镜像同步的时候使用的是  `docker pull –> docker tag –> docker push` 的方式。而在 docker pull 和 docker push 的过程中 docker 守护进程都会对镜像的 layer 做解压缩的操作。这是及其耗时和浪费 CPU 资源的。又因为我们的内网机器性能十分地烂，有时甚至连 USB 2.0 的速度都不如！那慢的程度可想而知。

### 无法复用旧数据

在第二次镜像同步时会对打包镜像仓库做清理的操作，导致无法复用历史的镜像。其实每次发布的时候，变更和新增的镜像很少，平均为原来的 1/10 左右，增量同步的镜像也就那么一丢丢而已。因为要保证这次打包发布的镜像仓库中只能包好这个需要的镜像，不能包含与本次无关的镜像，因此每次都需要清理打包镜像仓库，这无法避免。

## 优化

根据上面提到的两个问题，经过反复的研究和测试终于都完美地解决了。

### skopeo 替代 docker

针对  `docker pull –> docker tag –> docker push`  的性能问题，当时第一个方案想到的就是使用 skopeo 来替代它。使用 `skopeo copy` 直接将镜像从一个 registry 复制到另一个 registry 中。关于 skopeo 的使用和其背后的原理可以参考我之前的博客 [镜像搬运工 skopeo 初体验](https://blog.k8s.li/skopeo.html) 。使用 skopeo 之后镜像同步比之前快了很多，平均快了 5 倍左右。

###  overlay2 复用旧数据

解决了 docker 的性能问题，剩下的就是无法复用旧数据的问题了。在如何保留历史镜像的问题上可煞费苦心。当时也不知道就想到了 overlay2 的特性：`写时复制`。就好比如 docker run 启动一个容器，在容器内进行修改和删除文件的操作，这些操作并不会影响到镜像本身，因为 docker 使用 overlay2 联合挂载的方式将镜像的每一层挂载为一个 merged 的层。在容器内看到的就是这个 merged 的层，在容器内对 merged 层文件的修改和删除操作是通过 overlay2 的 upper 层完成的，并不会影响到镜像本身。

从 docker 官方文档 [Use the OverlayFS storage driver](https://docs.docker.com/storage/storagedriver/overlayfs-driver/) 里偷来的一张图片

![img](img/overlay_constructs.jpg)

关于上图中这些 Dir 的作用，下面是一段从 [StackOverflow](https://stackoverflow.com/questions/56550890/docker-image-merged-diff-work-lowerdir-components-of-graphdriver) 上搬运过来的解释。

>   **LowerDir**: these are the read-only layers of an overlay filesystem. For docker, these are the image layers assembled in order.
>
>   **UpperDir**: this is the read-write layer of an overlay filesystem. For docker, that is the equivalent of the container specific layer that contains changes made by that container.
>
>   **WorkDir**: this is a required directory for overlay, it needs an empty directory for internal use.
>
>   **MergedDir**: this is the result of the overlay filesystem. Docker effectively chroot’s into this directory when running the container.

如果想对 overlayfs 文件系统有详细的了解，可以参考 Linux 内核官网上的这篇文档 [overlayfs.txt](https://www.kernel.org/doc/Documentation/filesystems/overlayfs.txt) 。

总之 overlay2 大法好！既然找到了方法，就看看怎么实现了。

## overlay2

### registry 存储结构

再一次搬出这张 registry 存储目录结构的图片😂

![](https://p.k8s.li/registry-storage.jpeg)

z

### 套娃：镜像里塞镜像？

第一个想到的方案就是将历史的镜像仓库存储目录复制到一个 registry 的镜像里，然后用这个镜像来启动打包镜像仓库的 registry 容器。

-   Dockerfile

```dockerfile
FROM registry:latest

ADD docker.tar /var/lib/registry/
RUN find /var/lib/registry/docker/registry/v2/repositories -type d -name "_manifests" -exec rm -rf {} \;
```

### 容器挂载 merged 目录



## 结果

