---
layout: post
author: shalou
title:  "docker rpm包制作"
categories: docker,rpm
---

由于国内网络的原因，直接使用docker官方的教程制作docker的rpm包往往会失败，或需要很长的时间。本文将docker中制作rpm的核心思路抽取出来，以docker 1.12.1为例，总结如下.

## 1、下载源码
```go
git clone https://github.com/docker/docker.git docker.git
cd docker.git
git checkout v1.12.1
```

## 2、生成build rpm镜像

rpm制作的过程其实是在一个镜像中完成的，docker源码中包含了这个镜像的Dockerfile文件，位于目录contrib/builder/rpm/amd64/下面，包括centos、fedora、opensuse、oraclelinux等，我们以centos-7为例

```
docker build -t dockercore/builder-rpm:centos-7 contrib/builder/rpm/amd64/centos-7
```

这一步主要生成了dockercore/builder-rpm:centos-7镜像

## 3、生成manpages
这一步主要生成docker的帮助文档

```
make manpages
```

也是创建一个容器完成，过程比较慢，它会在man目录下生成man1、man5和man8三个文件夹，里面包含docker的帮助文档

## 4、创建Dockerfile.build

按照官方给的教程，这一步其实是由脚本完成的，本文的主要工作就是讲这部分东西抽出来，手动创建Dockerfile.build，其内容如下

```
FROM dockercore/builder-rpm:centos-7

COPY . /usr/src/docker-engine

RUN mkdir -p /go/src/github.com/docker && mkdir -p /go/src/github.com/opencontainers

ENV RUNC_COMMIT cc29e3dded8e27ba8f65738f40d251c885030a28
ENV CONTAINERD_COMMIT 0ac3cd1be170d180b2baed755e8f0da547ceb267

# Install runc
RUN git clone https://github.com/opencontainers/runc.git "/go/src/github.com/opencontainers/runc" \
        && cd "/go/src/github.com/opencontainers/runc" \
        && git checkout -q "$RUNC_COMMIT"
RUN set -x && export GOPATH="/go" && cd "/go/src/github.com/opencontainers/runc" \
        && make BUILDTAGS="$RUNC_BUILDTAGS" && make install

# Install containerd
RUN git clone https://github.com/docker/containerd.git "/go/src/github.com/docker/containerd" \
        && cd "/go/src/github.com/docker/containerd" \
        && git checkout -q "$CONTAINERD_COMMIT"
RUN set -x && export GOPATH="/go" && cd "/go/src/github.com/docker/containerd" && make && make install

RUN mkdir -p /root/rpmbuild/SOURCES \
        && echo '%_topdir /root/rpmbuild' > /root/.rpmmacros

WORKDIR /root/rpmbuild

RUN ln -sfv /usr/src/docker-engine/hack/make/.build-rpm SPECS

WORKDIR /root/rpmbuild/SPECS

RUN tar --exclude .git -r -C /usr/src -f /root/rpmbuild/SOURCES/docker-engine.tar docker-engine
RUN tar --exclude .git -r -C /go/src/github.com/docker -f /root/rpmbuild/SOURCES/docker-engine.tar containerd
RUN tar --exclude .git -r -C /go/src/github.com/opencontainers -f /root/rpmbuild/SOURCES/docker-engine.tar runc
RUN gzip /root/rpmbuild/SOURCES/docker-engine.tar
#RUN { cat /usr/src/docker-engine/contrib/builder/rpm/amd64/changelog; } >> docker-engine.spec && tail >&2 docker-engine.spec

RUN rpmbuild -ba \
        --define '_gitcommit 23cf638' \
        --define '_release 1' \
        --define '_version 1.12.1' \
        --define '_origversion 1.12.1' \
        --define '_experimental 0' \
        docker-engine.spec

RUN tar -cz -C /usr/src/docker-engine/contrib/selinux -f /root/rpmbuild/SOURCES/docker-engine-selinux.tar.gz docker-engine-selinux
RUN rpmbuild -ba \
        --define '_gitcommit 23cf638' \
        --define '_release 1' \
        --define '_version 1.12.1' \
        --define '_origversion 1.12.1' \
        docker-engine-selinux.spec
```

将这个文档放在bundles/1.12.1/build-rpm目录下，先创建该目录。然后执行


```
docker build -t docker-temp/build-rpm:centos-7 -f bundles/1.12.1/build-rpm/Dockerfile.build .
```

当这一步完成以后，其实rpm包已经生成在镜像里面了，通过以下方法拷贝出来


```
docker run --rm docker-temp/build-rpm:centos-7 bash -c 'cd /root/rpmbuild && tar -c *RPMS' | tar -xvC /root/rpm
```

这时会在/root/rpm下面生成两个目录，RPMS和SRPMS。其中RPMS是docker的二进制包，而SRPMS是源码包。

## 5、制作yum源
生成的rpm包最好不要直接安装，因为他依赖其他的软件包，最好通过yum安装，又yum解决软件依赖。本文简单介绍一下制作yum源的步骤

```
yum install httpd createrepo
systemectl enable httpd
mkdir /var/www/html/updocker  //将生成的rpm包拷贝到该目录下
cd /var/www/html
createrepo updocker
```


/etc/yum.repos.d/updocker.repo内容如下

```
[updocker]
name=updocker
baseurl=http://ip/updocker  //ip为yum源所在主机IP
gpgcheck=0
enabled=1
```

```
yum makecache
yum install docker-engine
```

