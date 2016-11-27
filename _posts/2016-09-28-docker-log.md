---
layout: post
author: shalou
title: "docker容器日志" 
categories: docker, log
---

*本分析基于docker v1.12.1*

随着版本的迭代，docker的日志也变得越来越复杂。docker容器的日志，是指docker容器进程的日志，一般输出在标准输出上面。早期的docker会将docker容器日志记录到一个jsonfile，这个文件一般位于/var/lib/docker/containers/uuid/uuid-json.log。

到了v1.12.1，docker的日志更加丰富，它支持多种log driver，如jsonfile、journald、syslog、flunentd、awslog、splunk等。


## 1、多种log driver的初始化

这部分代码位于docker/daemon/logger目录下，每一种驱动都在该目录下面有一个子目录，它们必须实现Logger接口

```
type Logger interface {
	Log(*Message) error
	Name() string
	Close() error
}
```

其中Log(*Message)函数，描述了具体如何记录日志。

我们以jsonfile这个log driver为例。首先是log driver的注册，这部分工作在docker daemon初始化前就完成了

**docker/daemon/logger/jsonfilelog/jsonfilelog.go +32**

```
func init() {
	if err := logger.RegisterLogDriver(Name, New); err != nil {
		logrus.Fatal(err)
	}
	if err := logger.RegisterLogOptValidator(Name, ValidateLogOpt); err != nil {
		logrus.Fatal(err)
	}
}
```
RegisterLogDriver将log driver注册到一个map里面，Name是driver的名字，作为map的键，而New是driver的初始化函数，作为map的值

## 2、 docker daemon启动日志记录

我们在启动容器的时候，可以指定log driver，主要通过参数--log-driver，假如没有指定log driver，则使用默认的log driver。默认的log driver是在docker daemon启动时指定的。


在docker v1.12.1中，整个docker被拆分成docker-client、docker-daemon、containerd、containrd-shim和runC。记录日志的工作主要由docker daemon负责。记录日志的工作主要在启动容器的时候开始的。



在这里先讲一下创建容器的流程，首先docker client会根据用户的输入生成Config和HostConfig，然后发送给docker daemon，docker首先container配置信息后，在内存中生成一个container对象，然后docker daemon会发送给containerd，由containerd调用runC去运行容器。

启动容器日志记录就发生在docker daemon将容器配置信息发送给containerd以后。

**docker/libcontainerd/client_linux.go +136**

```
func (clnt *client) Create(containerID string, checkpoint string, checkpointDir string, spec Spec, options ...CreateOption) (err error) {
    ...
	container := clnt.newContainer(filepath.Join(dir, containerID), options...)
    ...
	return container.start(checkpoint, checkpointDir)
}
```

首先创建了一个container对象，但这时候container对象还没有发送给containerd。接着调用container.start函数

```
func (ctr *container) start(checkpoint string, checkpointDir string) error {
	...
	iopipe, err := ctr.openFifos(spec.Process.Terminal)
	....
	r := &containerd.CreateContainerRequest{
		Id:            ctr.containerID,
		BundlePath:    ctr.dir,
		Stdin:         ctr.fifo(syscall.Stdin),
		Stdout:        ctr.fifo(syscall.Stdout),
		Stderr:        ctr.fifo(syscall.Stderr),
		Checkpoint:    checkpoint,
		CheckpointDir: checkpointDir,
		// check to see if we are running in ramdisk to disable pivot root
		NoPivotRoot: os.Getenv("DOCKER_RAMDISK") != "",
		Runtime:     ctr.runtime,
		RuntimeArgs: ctr.runtimeArgs,
	}
	ctr.client.appendContainer(ctr)

	if err := ctr.client.backend.AttachStreams(ctr.containerID, *iopipe); err != nil {
		close(createChan)
		ctr.closeFifos(iopipe)
		return err
	}

	resp, err := ctr.client.remote.apiClient.CreateContainer(context.Background(), r)
	...
}

```

这个函数首先会构建一个CreateContainerRequest，然后调用ctr.client.backend.AttachStreams(ctr.containerID, *iopipe)，这个就是启动日志记录的关键，最后将request发送给containerd。AttachStreams其实就是daemon.AttachStreams

**docker/daemon/monitor.go +122**

```
func (daemon *Daemon) AttachStreams(id string, iop libcontainerd.IOPipe) error {
	var (
		s  *runconfig.StreamConfig
		ec *exec.Config
	)

	c := daemon.containers.Get(id)
	if c == nil {
		...
	} else {
		s = c.StreamConfig
		if err := daemon.StartLogging(c); err != nil {
			c.Reset(false)
			return err
		}
	}

	copyFunc := func(w io.Writer, r io.Reader) {
		s.Add(1)
		go func() {
			if _, err := io.Copy(w, r); err != nil {
				logrus.Errorf("%v stream copy error: %v", id, err)
			}
			s.Done()
		}()
	}

	if iop.Stdout != nil {
		copyFunc(s.Stdout(), iop.Stdout)
	}
	if iop.Stderr != nil {
		copyFunc(s.Stderr(), iop.Stderr)
	}
    ...
}
```

StartLogging是问题的关键

**docker/daemon/logs.go +125**

```
func (daemon *Daemon) StartLogging(container *container.Container) error {
	if container.HostConfig.LogConfig.Type == "none" {
		return nil // do not start logging routines
	}

	l, err := container.StartLogger(container.HostConfig.LogConfig)
	if err != nil {
		return fmt.Errorf("Failed to initialize logging driver: %v", err)
	}

	copier := logger.NewCopier(map[string]io.Reader{"stdout": container.StdoutPipe(), "stderr": container.StderrPipe()}, l)
	container.LogCopier = copier
	copier.Run()
	container.LogDriver = l
    ...
}
```

container.StartLogger(container.HostConfig.LogConfig)函数传入了一个LogConfig，LogConfig包含了log driver的类型，这个函数会返回一个logger对象。接着将这个logger传入logger.NewCopier返回一个copier，然后copier运行起来

**docker/daemon/logger/copier.go +34**

```
func (c *Copier) Run() {
	for src, w := range c.srcs {
		c.copyJobs.Add(1)
		go c.copySrc(src, w)
	}
}

func (c *Copier) copySrc(name string, src io.Reader) {
	defer c.copyJobs.Done()
	reader := bufio.NewReader(src)

	for {
		select {
		case <-c.closed:
			return
		default:
			line, err := reader.ReadBytes('\n')
			line = bytes.TrimSuffix(line, []byte{'\n'})

			if err == nil || len(line) > 0 {
				if logErr := c.dst.Log(&Message{Line: line, Source: name, Timestamp: time.Now().UTC()}); logErr != nil {
					logrus.Errorf("Failed to log msg %q for logger %s: %s", line, c.dst.Name(), logErr)
				}
			}

			if err != nil {
				if err != io.EOF {
					logrus.Errorf("Error scanning log stream: %s", err)
				}
				return
			}
		}
	}
}
```

看到这个函数就豁然开朗了，copySrc里面有两个变量，一个src，另一个dst，这两个变量其实是我们在调用logger.NewCopier(map[string]io.Reader{"stdout": container.StdoutPipe(), "stderr": container.StderrPipe()}, l)传入进去的，其中src代表日志来源，而des就是logger。在copySrc中首先调用src.Read(buf[n:upto])读取日志，然后调用c.dst.Log(msg)写日志。这样日志就被记录下来了

src的日志从哪里来呢？logger.NewCopier(map[string]io.Reader{"stdout": container.StdoutPipe(), "stderr": container.StderrPipe()}, l)，首先要弄清楚container.StdoutPipe()是什么，其实它调用的是StreamConfig.StdoutPipe()，StreamConfig是Container对象的匿名成员。

回到前面的函数AttachStreams


```
func (daemon *Daemon) AttachStreams(id string, iop libcontainerd.IOPipe) error {
	var (
		s  *runconfig.StreamConfig
		ec *exec.Config
	)

	c := daemon.containers.Get(id)
	if c == nil {
		...
	} else {
		s = c.StreamConfig
		if err := daemon.StartLogging(c); err != nil {
			c.Reset(false)
			return err
		}
	}

	copyFunc := func(w io.Writer, r io.Reader) {
		s.Add(1)
		go func() {
			if _, err := io.Copy(w, r); err != nil {
				logrus.Errorf("%v stream copy error: %v", id, err)
			}
			s.Done()
		}()
	}

	if iop.Stdout != nil {
		copyFunc(s.Stdout(), iop.Stdout)
	}
	if iop.Stderr != nil {
		copyFunc(s.Stderr(), iop.Stderr)
	}
    ...
}
```

首先s = c.StreamConfig，最后copyFunc(s.Stdout(), iop.Stdout)和copyFunc(s.Stderr(), iop.Stderr)将iop的输出流复制s的输出流上面。显然iop是关键。

继续回溯到start函数

```
func (ctr *container) start(checkpoint string, checkpointDir string) error {
	...
	iopipe, err := ctr.openFifos(spec.Process.Terminal)
	....
	r := &containerd.CreateContainerRequest{
		Id:            ctr.containerID,
		BundlePath:    ctr.dir,
		Stdin:         ctr.fifo(syscall.Stdin),
		Stdout:        ctr.fifo(syscall.Stdout),
		Stderr:        ctr.fifo(syscall.Stderr),
		Checkpoint:    checkpoint,
		CheckpointDir: checkpointDir,
		// check to see if we are running in ramdisk to disable pivot root
		NoPivotRoot: os.Getenv("DOCKER_RAMDISK") != "",
		Runtime:     ctr.runtime,
		RuntimeArgs: ctr.runtimeArgs,
	}
	ctr.client.appendContainer(ctr)

	if err := ctr.client.backend.AttachStreams(ctr.containerID, *iopipe); err != nil {
		close(createChan)
		ctr.closeFifos(iopipe)
		return err
	}

	resp, err := ctr.client.remote.apiClient.CreateContainer(context.Background(), r)
	...
}

```

iop就是iopipe。管道？ctr.openFifos(spec.Process.Terminal)这个函数创建的

**docker/libcontainerd/process_linux.go +29**

```
func (p *process) openFifos(terminal bool) (*IOPipe, error) {
	bundleDir := p.dir
	if err := os.MkdirAll(bundleDir, 0700); err != nil {
		return nil, err
	}

	for i := 0; i < 3; i++ {
		f := p.fifo(i)
		if err := syscall.Mkfifo(f, 0700); err != nil && !os.IsExist(err) {
			return nil, fmt.Errorf("mkfifo: %s %v", f, err)
		}
	}

	io := &IOPipe{}
	stdinf, err := os.OpenFile(p.fifo(syscall.Stdin), syscall.O_RDWR, 0)
	if err != nil {
		return nil, err
	}

	io.Stdout = openReaderFromFifo(p.fifo(syscall.Stdout))
	if !terminal {
		io.Stderr = openReaderFromFifo(p.fifo(syscall.Stderr))
	} else {
		io.Stderr = emptyReader{}
	}
    ...
	return io, nil
}
```

看到函数syscall.Mkfifo函数没？Mkfifo用来构造命名管道。管道可以分为命名管道和未命名管道，他们通常用于两个进程之间交换数据。其中命名管道用于两个毫无关联的进程之间交换数据，而未命名管道用于父进程和子进程(fork)交换数据。

在这里面我们使用了命名管道，创建了三个命名管道：init-stdin、init-stdout和init-stderr。这三个管道文件位于/var/run/docker/libcontainerd/uuid目录下面。

为什么要使用命名管道呢？docker daemon和容器进程没有关联？其实docker daemon和容器之间的确没有关联。容器都是由containerd调用runC创建的。而docker daemon和contained之间是通过gRPC通信的。

显然容器将日志写入init-stdout、init-stderr这两个管道文件，而docker daemon则负责从这两个管道文件读取日志，然后调用具体的log driver去记录日志。

## 3、runC如何写入日志

runC如何写入日志，我们在这里就不具体展开了。其实runC写入日志的关键是，必须知道这几个管道文件具体位于哪里。知道了管道文件的位置，打开管道文件，从管道中读取或写入数据即可。

前面提到到docker daemon会构建一个CreateContainerRequest，然后传递给containerd。如下所示，Request里面包含了Stdio、Stdout、Stderr，这三个字符串就是三个管道文件的全路径。

```
r := &containerd.CreateContainerRequest{
		Id:            ctr.containerID,
		BundlePath:    ctr.dir,
		Stdin:         ctr.fifo(syscall.Stdin),
		Stdout:        ctr.fifo(syscall.Stdout),
		Stderr:        ctr.fifo(syscall.Stderr),
		Checkpoint:    checkpoint,
		CheckpointDir: checkpointDir,
		// check to see if we are running in ramdisk to disable pivot root
		NoPivotRoot: os.Getenv("DOCKER_RAMDISK") != "",
		Runtime:     ctr.runtime,
		RuntimeArgs: ctr.runtimeArgs,
	}
```

进一步查看runC的代码，你会发现，其实runC会将容器的标准输出写入到管道文件、从管道文件中读取标准输入。其实容器的日志就是其标准输出


## 4、docker exec的秘密

当我们执行docker exec 时，docker会往容器中增加一个进程。相应的在/var/run/docker/libcontainerd/uuid下面会多出三个管道文件出来，pid-uuid-stdin、pid-uuid-stdout、piduuid-stderr。这是的piduuid是指新加入的进程的uuid。为啥生成这个三个命名管道，而不是复用原有的init-stdin、init-stdout、init-stderr。这个就涉及到新加入的进程与容器原有进程的关系？其实他们是相互独立的，新加入的进程并不是原有进程fork出来的，虽然原有的第一个进程进程号为1，这与linux的init进程不同。

因此docker exec加入的进程，其标准输出并不会输出到日志中去。每次用docker exec加入一个进程，就会在/var/run/docker/libcontainerd/uuid下面新生成三个命名管道。











