---
layout: post 
author: shalou
title:  "k8s资源的级联删除(cascading deletion)" 
category: 容器技术
tag: [k8s,cascading deletion, kubectl]
---

*本文档基于kubernetes v1.5.2*

## 1. 什么是级联删除？
当我们创建一个ReplicationController时，k8s会为我们创建相应备份数的pod。而当我们删除这个ReplicationController时，相应的Pod也会被删除，这就成为级联删除(**cascading deletion**)。但有时，我们只希望相应的ReplicationController被删除，而相应的Pod被保留，这时就需要禁止级联删除

<!-- more -->

## 2. 级联删除的两种方式

调用k8s api有两种方式：

* 通过kubectl这个二进制文件
* 直接像kube-apiserver发送http请求

### 2.1 kubectl的级联删除

首先考虑一下，哪些资源需要级联删除，包括ReplicationController、Deployment、ReplicaSet、Job以及StatefulSet。

kubectl delete提供了一个--cascade选项，这个选项默认的值为true，也就是默认情况下会开起级联删除，当你删除ReplicationController时，会删除相应的Pod。

假如你不想删除相应的Pod，只需要执行

```go
kubectl delete --cascade=false replicationcontroller xxx
```

### 2.2 通过http实现级联删除

http的级联删除是一个比较新的特性，它在k8s v1.3版本才被引入。目前只有ReplicationController和ReplicaSet两类资源实现了相关功能。

只需在删除选项中加入orphanDependents。

```go
curl -X DELETE http://127.0.0.1:8080/api/namespaces/default/replicationcontrollers/test?orphanDependents=false
```

默认情况下orphanDependents为true，想要级联删除需要传递为false。orphanDependents的字面意思就是**孤儿依赖**，孤儿相当于Pod，当为true的时候，保留依赖的pod,为false的时候，不保留，即删除

## 3. 级联删除的实现方式

以上两种删除方式，它们的实现机制完全不同，kubectl是完全将实现逻辑放在客户端实现，而http请求的方式则完全将实现放在服务端(kube-apiserver和kube-controller-manager)。

### 3.1 kubectl实现方式

实现了级联删除的资源类型都实现了Reaper接口

**pkg/kubectl/cmd/delete.go +135**

```go
func RunDelete(f cmdutil.Factory, out, errOut io.Writer, cmd *cobra.Command, args []string, options *resource.FilenameOptions) error {
	...
	r := resource.NewBuilder(mapper, typer, resource.ClientMapperFunc(f.UnstructuredClientForMapping), runtime.UnstructuredJSONScheme).
		ContinueOnError().
		NamespaceParam(cmdNamespace).DefaultNamespace().
		FilenameParam(enforceNamespace, options).
		SelectorParam(cmdutil.GetFlagString(cmd, "selector")).
		SelectAllParam(deleteAll).
		ResourceTypeOrNameArgs(false, args...).RequireObject(false).
		Flatten().
		Do()
	...
	gracePeriod := cmdutil.GetFlagInt(cmd, "grace-period")
	force := cmdutil.GetFlagBool(cmd, "force")
	if cmdutil.GetFlagBool(cmd, "now") {
		if gracePeriod != -1 {
			return fmt.Errorf("--now and --grace-period cannot be specified together")
		}
		gracePeriod = 1
	}
	wait := false
	if gracePeriod == 0 {
		if force {
			fmt.Fprintf(errOut, "warning: Immediate deletion does not wait for confirmation that the running resource has been terminated. The resource may continue to run on the cluster indefinitely.\n")
		} else {
			wait = true
			gracePeriod = 1
		}
	}

	shortOutput := cmdutil.GetFlagString(cmd, "output") == "name"
	// By default use a reaper to delete all related resources.
	if cmdutil.GetFlagBool(cmd, "cascade") {
		return ReapResult(r, f, out, cmdutil.GetFlagBool(cmd, "cascade"), ignoreNotFound, cmdutil.GetFlagDuration(cmd, "timeout"), gracePeriod, wait, shortOutput, mapper, false)
	}
	return DeleteResult(r, out, ignoreNotFound, shortOutput, mapper)
}
```

从代码可以看出，当cascade为true的时候执行ReapResult，否则执行DeleteResult，我们比较关注ReapResult，继续查看即源代码

**pkg/kubectl/cmd/delete.go +201**

```go
func ReapResult(r *resource.Result, f cmdutil.Factory, out io.Writer, isDefaultDelete, ignoreNotFound bool, timeout time.Duration, gracePeriod int, waitForDeletion, shortOutput bool, mapper meta.RESTMapper, quiet bool) error {
    ...
	err := r.Visit(func(info *resource.Info, err error) error {
       ...
		reaper, err := f.Reaper(info.Mapping)
		if err != nil {
			if kubectl.IsNoSuchReaperError(err) && isDefaultDelete {
				return deleteResource(info, out, shortOutput, mapper)
			}
			return cmdutil.AddSourceToErr("reaping", info.Source, err)
		}
		var options *api.DeleteOptions
		if gracePeriod >= 0 {
			options = api.NewDeleteOptions(int64(gracePeriod))
		}
		if err := reaper.Stop(info.Namespace, info.Name, timeout, options); err != nil {
			return cmdutil.AddSourceToErr("stopping", info.Source, err)
		}
       ...
	})
	...
}
```

首先调用f.Reaper构建一个reaper，这里我们就不继续展开了，只有ReplicationController、ReplicaSet、DaemonSet、Pod、Service、Job、StatefulSet和Deployment实现了reaper接口，假如是其他的资源则会返回NoSuchReaperError错误

假如返回错误为NoSuchReaperError，则直接调用deleteResource删除资源，否则调用reaper.Stop，我们这里查看ReplicationController reaper的stop方法

**pkg/kubectl/stop.go +160**

```go
func (reaper *ReplicationControllerReaper) Stop(namespace, name string, timeout time.Duration, gracePeriod *api.DeleteOptions) error {
	rc := reaper.client.ReplicationControllers(namespace)
	scaler := &ReplicationControllerScaler{reaper.client}
	ctrl, err := rc.Get(name)
	...
	overlappingCtrls, err := getOverlappingControllers(rc, ctrl)
	if err != nil {
		return fmt.Errorf("error getting replication controllers: %v", err)
	}
	exactMatchRCs := []api.ReplicationController{}
	overlapRCs := []string{}
	for _, overlappingRC := range overlappingCtrls {
		if len(overlappingRC.Spec.Selector) == len(ctrl.Spec.Selector) {
			exactMatchRCs = append(exactMatchRCs, overlappingRC)
		} else {
			overlapRCs = append(overlapRCs, overlappingRC.Name)
		}
	}
   ...
	if len(exactMatchRCs) == 1 {
		// No overlapping controllers.
		retry := NewRetryParams(reaper.pollInterval, reaper.timeout)
		waitForReplicas := NewRetryParams(reaper.pollInterval, timeout)
		if err = scaler.Scale(namespace, name, 0, nil, retry, waitForReplicas); err != nil {
			return err
		}
	}
	falseVar := false
	deleteOptions := &api.DeleteOptions{OrphanDependents: &falseVar}
	return rc.Delete(name, deleteOptions)
}
```

该函数首先检查是否有其他重叠的ReplicationController，即他们的label相同。假如只有一个重叠，即自身，则调用scaler.Scale将备份数设置为0（这样相应的Pod就会被删除），然后再调用rc.Delete删除ReplicationController。假如发生了重叠，则只删除自身。

其他资源的Reap，读者自己可以查看源代码。

k8s的命名还是蛮有意思的，reap即收割的意思，收割意味着大批量的将所有依赖都delete。而delete则是针对单个资源

从kubectl的实现方式看，它完全依赖于客户端的实现

### 3.2 http实现方式

http实现级联方式主要依赖于kuberntes的GC（garbage connection），它的实现很晚，在k8s v1.3版本才引入，现阶段只有ReplicationController和ReplicaSet实现了。后面我们会专门介绍GC的实现。这是官方给的文档，也是PR
[https://github.com/kubernetes/community/blob/master/contributors/design-proposals/garbage-collection.md](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/garbage-collection.md)


## 4. 总结

本文我们介绍了k8s中的级联删除，他有两种实现方式，分别放在客户端和服务器端实现，从官方的意思看，最后会将所有的资源回收都统一放在服务器端实现，利用GC实现，而kubelet的实现会慢慢取消掉，这也是合理的。