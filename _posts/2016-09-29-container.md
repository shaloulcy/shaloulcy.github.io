---
layout: post
author: shalou
title:  "如何用100行go代码构建容器"
category：容器技术
tag: [container, namespace, PivotRoot]
---


使用说明：

* 编译源代码为二进制文件container
* 准备一个rootfs文件系统，里面包含正常的/bin、/var、/lib等目录，最简单的做法是直接利用docker镜像，解压，我们只用busybox这个镜像。将镜像的所有文件至于一个rootfs文件夹
* 执行 ./container run command，command为用户自定义的指令

<!-- more -->

```go
package main

import (
	"fmt"
	"os"
	"os/exec"
	"syscall"
)

func main() {
	switch os.Args[1] {
	case "run":
		parent()
	case "child":
		child()
	default:
		panic("wat should I do")
	}
}

func parent() {
	cmd := exec.Command("/proc/self/exe", append([]string{"child"}, os.Args[2:]...)...)
	cmd.SysProcAttr = &syscall.SysProcAttr{
		Cloneflags: syscall.CLONE_NEWUTS | syscall.CLONE_NEWPID | syscall.CLONE_NEWNS,
	}
	cmd.Stdin = os.Stdin
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr

	if err := cmd.Run(); err != nil {
		fmt.Println("ERROR", err)
		os.Exit(1)
	}
}

func child() {
	must(syscall.Mount("rootfs", "rootfs", "", syscall.MS_BIND, ""))
	must(os.MkdirAll("rootfs/oldrootfs", 0700))
    must(os.MkdirAll("rootfs/proc", 0700))
    must(syscall.Mount("proc", "rootfs/proc", "proc", syscall.MS_NODEV | syscall.MS_NOEXEC | syscall.MS_NOSUID, "" ))
	must(syscall.PivotRoot("rootfs", "rootfs/oldrootfs"))
	must(os.Chdir("/"))

	cmd := exec.Command(os.Args[2], os.Args[3:]...)
	cmd.Stdin = os.Stdin
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr

	if err := cmd.Run(); err != nil {
		fmt.Println("ERROR", err)
		os.Exit(1)
	}
}

func must(err error) {
	if err != nil {
		panic(err)
	}
}
```
