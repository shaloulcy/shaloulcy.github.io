---
layout: post
title: "Docker镜像存储相关数据结构" 
categories: docker, image
---

*本分析基于docker v1.12.1*

## 1、layerStore

docker daemon在初始化过程中，会初始化一个layerStore，那么layerStore是什么呢？从名字可以看出，是用来存储layer的，docker镜像时分层的，一层称为一个layer。在之前的docker源码中，docker的镜像是由一个叫graph的数据结构进行管理的，现在换成了layerStore

```
type layerStore struct {
	store  MetadataStore
	driver graphdriver.Driver

	layerMap map[ChainID]*roLayer
	layerL   sync.Mutex

	mounts map[string]*mountedLayer
	mountL sync.Mutex
}
```

layStore主要包含四个主要的结构成员，store、driver、layerMap和mounts。

### 1.1 store

store的数据类型为MetadataStore，主要用来存储每个layer的元数据，存储的目录位于/var/lib/docker/image/{driver}/layerdb，这里的driver包括aufs、devicemapper、overlay和btrfs。

layerdb下面有三个目录，mounts、sha256和tmp，tmp目录主要存放临时性数据，因此不做介绍。主要介绍mounts和sha256两个目录。

#### 1.1.1  mounts目录

mounts主要用来存储layerStore的mounts成员的信息，每个运行的容器，其实都是若干只读的layers（也就是docker镜像），加上一层只读的init-layer，再加上一层可写的layer构成。这层可写的layer保存在mounts这个成员里面，具体的元数据则保存在/var/lib/docker/image/{driver}/layerdb/mounts下面。mounts下面的子目录名字是各个容器的uuid，在每一个子目录下面包含三个文件，即init-id，mount-id和parent。

* **mount-id** 是可写的layer的uuid
* **init-id** 是mount-id加上后缀'-init'
* **parent** 则存放可写的layer的父layer的chain-id。


#### 1.1.2 sha256目录

sha256下面则存放了每一层只读的Layer的原信息。之所以会有sha256，这是因为这些只读的layer的chain-id是使用sha256加密算法生成的，未来也支持sha384和sha512。sha256下面的子目录名称为各个只读layer的chain-id。每个子目录包含以下五个文件cache-id、diff、parent、size和tar-split.json.gz

* **cache-id**：layer的cache-id，它制定了该层layer的数据存放位置，假如driver为aufs，数据存在/var/lib/docker/aufs/diff/cache-id，假如driver为overlay，数据存放在/var/lib/docker/overlay/cache-id，假如driver为devicemapper，数据存放的device-id的信息位于/var/lib/docker/devicemapper/metadata/cache-id下

* **diff**：存放diff-id，docker下载镜像时会首先会下载一个docker image的元数据文件，在元数据文件标识了每一个layer，就是用diff-id。

* **parent**：保存了parent layer的chain-id。假如是最底层的layer,则没有Parent这个文件

* **size**:存放layer的数据大小

* **tar-split.json.gz**：存放的是关于这个layer的json信息


从上面的分析，我们可以看出一个layer它有三个id，分别为chain-id、diff-id和cache-id，很让人困惑，我们接下来的章节会进行解释

### 1.2 driver

存储layer的具体的driver，目前主要包括四种

* aufs
* overlay
* devicemapper
* btrfs

这里我们暂时不展开，后面会详细介绍

### 1.3 layerMap

layerMap本质上是一个map，map的类型为map[ChainID]*roLayer，即map的键为ChainID（字母串），值为roLayer。前面说store本质上是磁盘上保存了各个layer的元数据信息，当docker初始化时，它会利用这些元数据文件在内存中构造各个layer，每个Layer都用一个roLayer结构体表示，即只读(ro)的layer

```
type roLayer struct {
	chainID    ChainID
	diffID     DiffID
	parent     *roLayer
	cacheID    string
	size       int64
	layerStore *layerStore
	descriptor distribution.Descriptor

	referenceCount int
	references     map[Layer]struct{}
}
```
前面提到每一层layer都有三个id。chain-id、diffID、cache-id。现在我们来解释一下三个id的区别，之所以会有三个id是由于docker从v 1.10开始采用了一种叫做content addressable storage的模型

* **diff-id**：通过docker pull下载镜像时，镜像的json文件中每一个layer都有一个唯一的diff-id
* **chain-id**：chain-id是根据parent的chain-id和自身的diff-id生成的，假如没有parent，则chain-id等于diff-id，假如有parent，则chain-id等于sha256sum( "parent-chain-id diff-id")
* **cache-id**：随机生成的64个16进制数。前面提到过，cache-id标识了这个layer的数据具体存放位置

cache-id的类型的ChainID，diff-id类型为DiffID，本质上二者都是digest.Digest类型，其实都是string类型,都是sha256:uuid，uuid为64个16进制数

roLayer还有parent数据成员、referenceCount和references成员，referentces存放的是他的子layer的信息。当删除镜像时，只有roLayer的referentceCount为零时，才能够删除该layer。

### 1.4 mounts

mounts本质上是一个map，类型为map[string]*mountedLayer。前面提到过mounts存放的其实是每个容器可写的layer的信息，他们的元数据存放在/var/lib/docker/image/{driver}/layerdb/mounts目录下。而mountedLayer则是这些可写的layer在内存中的结构

```
type mountedLayer struct {
	name       string
	mountID    string
	initID     string
	parent     *roLayer
	path       string
	layerStore *layerStore

	references map[RWLayer]*referencedRWLayer
}
```

mountedLayer没有chain-id、diff-id、cached-id，只有mountID和initID，其中mountID是由随机生成的64个16进制数构成，initID等于mountID-init。initID和mountID表示了这个layer数据存放的位置，和cache-id一样。


## 2、imageStore

imageStore存放的是各个docker image的信息。imageStore的类型为image.Store，结构体为

```
type store struct {
	sync.Mutex
	ls        LayerGetReleaser
	images    map[ID]*imageMeta
	fs        StoreBackend
	digestSet *digest.Set
}
```

ls类型为LayerGetReleaser接口，初始化时将ls初始化为layerStore。fs类型为StoreBackend。

```
ifs, err := image.NewFSStoreBackend(filepath.Join(imageRoot, "imagedb"))
d.imageStore, err = image.NewImageStore(ifs, d.layerStore)
```
fs存放了image的原信息，存储的目录位于/var/lib/docker/image/{driver}/imagedb，该目录下主要包含两个目录content和metadata

* **content目录**：content下面的sha256目录下存放了每个docker image的元数据文件，除了制定了这个image由那些roLayer构成，还包含了部分配置信息，了解docker的人应该知道容器的部分配置信息是存放在image里面的，如volume、port、workdir等，这部分信息就存放在这个目录下面，docker启动时会读取镜像配置信息，反序列化出image对象

* **metadata目录**：metadata目录存放了docker image的parent信息


imageStore包含了images成员，类型为map[ID]*imageMeta，images就是每一个镜像的信息，看看imageMeta结构体

```
type imageMeta struct {
	layer    layer.Layer
	children map[ID]struct{}
}
```

imageMeta包含一个layer成员，之前说过docker image是由多个只读的roLayer构成，而这里的layer就是最上层的layer。

此外，imageStore还包含一个digestSet成员，本质上是一个set数据结构，里面存放的其实是每个docker的最上层layer的chain-id。

## 3、referenceStore

referfenceStore的类型为reference.store，这个应该是docker用户最熟悉的部分了。以一个ubunu镜像为例，ubuntu镜像的名字就叫ubuntu，一个完成的镜像还包括tag，于是就有了ubuntu:latest、ubuntu:14.04等。这部分信息其实就是存储才referenceStore中。这部分信息其实保存在/var/lib/docker/image/{driver}/repositories.json这个文件中

```json
 "Repositories": {
    "ubuntu": {
      "ubuntu@sha256:bd00486535fd3ab00463b0572d94a62715cb790e482d5419c9179cd22c74520b": "sha256:f2d8ce9fa988ed844dda693fe260b9afd393b9a65b647aa02f62d6eecdb7b635",
      "ubuntu@sha256:3235a49037919e99696d97df8d8a230717272d848ee4ddadbca8d54f97ee30cb": "sha256:45bc58500fa3d3c0d67233d4a7798134b46b486af1389ca87000c543f46c3d24",
      "ubuntu:latest": "sha256:45bc58500fa3d3c0d67233d4a7798134b46b486af1389ca87000c543f46c3d24",
      "ubuntu:14.04": "sha256:f2d8ce9fa988ed844dda693fe260b9afd393b9a65b647aa02f62d6eecdb7b635"
    },
    "busybox": {
      "busybox@sha256:a59906e33509d14c036c8678d687bd4eec81ed7c4b8ce907b888c607f6a1e0e6": "sha256:2b8fd9751c4c0f5dd266fcae00707e67a2545ef34f9a29354585f93dac906749",
      "busybox:latest": "sha256:2b8fd9751c4c0f5dd266fcae00707e67a2545ef34f9a29354585f93dac906749"
    }
  }
}
```

从这里我们可以看出，这才机器包括两个镜像，ubuntu和busybox，其中ubuntu有两个tag分别为latest和14.04，而busybox只有latest一个tag

referfenceStore其实就是从这个文件反序列化而来的

```
type store struct {
	mu sync.RWMutex
	jsonPath string
	Repositories map[string]repository
	referencesByIDCache map[image.ID]map[string]Named
}
```

从上面的json文件我们可以看出，"ubuntu:14.04"和"ubuntu@sha256:bd00486535fd3ab00463b0572d94a62715cb790e482d5419c9179cd22c74520b"指向的其实是同一个image。其实我们pull镜像时

即可以

docker pull ubuntu:14.04

也可以

docker pull ubuntu@sha256:bd00486535fd3ab00463b0572d94a62715cb790e482d5419c9179cd22c74520b


## 4、distributionMetadataStore

这个结构体没去详细了解过，它在我们下载镜像时会用到。数据存储在/var/lib/docker/image/{driver}/distribution

```
type FSMetadataStore struct {
	sync.RWMutex
	basePath string
}
```

## 5、storage driver

目前docker支持四种storage driver，aufs、devicemapper、overlay和btrfs。所有的driver必须支持以下接口

```
type Driver interface {
	ProtoDriver
	Diff(id, parent string) (archive.Archive, error)
	Changes(id, parent string) ([]archive.Change, error)
	ApplyDiff(id, parent string, diff archive.Reader) (size int64, err error)
	DiffSize(id, parent string) (size int64, err error)
}

type ProtoDriver interface {
	String() string
	CreateReadWrite(id, parent, mountLabel string, storageOpt map[string]string) error
	Create(id, parent, mountLabel string, storageOpt map[string]string) error
	Remove(id string) error
	Get(id, mountLabel string) (dir string, err error)
	Put(id string) error
	Exists(id string) bool
	Status() [][2]string
	GetMetadata(id string) (map[string]string, error)
	Cleanup() error
}
```

镜像的数据前面提过，存放在/var/lib/docker/{driver}目录下

aufs下面有三个目录，diff、layers、mnt。diff目录下面存储以各个layer的cache-id存储各个layer的数据；layers文件存储了各个layer的parent layer；mnt目录主要是用于挂在layer

devicemapper下面有三个目录，devicemapper、metadata、mnt。devicemapper目录面有两个文件,metadata和data，metadata为2G的大小，而data为100G，所有的镜像数据都存在这两个文件里面，这里主要涉及到thin-provision技术；metadata目录下面存放的是各个layer的大小，都为10G以及所对应的device_id

overlay下面直接以各个layer的cache-id存储各个layer的数据

btrfs这个驱动没有使用过，因此不了解










