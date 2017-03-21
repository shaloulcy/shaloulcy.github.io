---
layout: post 
author: shalou
title:  "Openstack中的Launcher" 
category: openstack
tag: [openstack, launcher]
---

*本分析基于Openstack liberty*

Openstack中有一个叫Launcher的概念，即专门用来启动服务的，这个类被放在了oslo_service这个包里面，Launcher分为两种，一种是ServiceLauncher，另一种为ProcessLauncher。ServiceLauncher用来启动单进程的服务，如nova-compute、nova-scheduler,而ProcessLauncher用来启动有多个worker子进程的服务，如各类api服务(nova-api、glance-api)以及nova-conductor等

<!-- more -->

## 1. ServiceLauncher

ServiceLauncher继承自Launcher，启动服务的一个重要成员就是launcher\_service，ServiceLauncher的该成员就是继承于Launcher

**oslo_service/service.py +178**

```python

def launch_service(self, service):
    """Load and start the given service.

    :param service: The service you would like to start, must be an
                    instance of :class:`oslo_service.service.ServiceBase`
    :returns: None

    """
    _check_service_base(service)
    service.backdoor_port = self.backdoor_port
    self.services.add(service)
```

**laucher_service**就是将服务添加到self.services成员里面，services成员的类型是class Services，看看它的add方法

**oslo_service/service.py +594**

```python
class Services(object):

    def __init__(self):
        self.services = []
        self.tg = threadgroup.ThreadGroup()
        self.done = event.Event()

    def add(self, service):
        self.services.append(service)
        self.tg.add_thread(self.run_service, service, self.done)
        
    def run_service(service, done):
        service.start()
        done.wait()
```

Services这个类的初始化很简单，即创建一个ThreadGroup，ThreadGroup其实是eventlet的GreenPool，Openstack利用eventlet实现并发，add方法，将self.run\_service这个方法放入pool中，而service就是它的参数。run\_service方法很简单，就是调用service的start方法，这样就完成了服务的启动

## 2. ProcessLauncher

ProcessLauncher直接继承于Object，同样也有launch_service方法

**oslo_service/service.py +445**

```python
def launch_service(self, service, workers=1):
    _check_service_base(service)
    wrap = ServiceWrapper(service, workers)

    LOG.info(_LI('Starting %d workers'), wrap.workers)
    while self.running and len(wrap.children) < wrap.workers:
       self._start_child(wrap)
```
lauch\_service除了接受service意外，还需要接受一个workers参数，即子进程的个数，然后调用\_start\_child启动多个子进程

**oslo_service/service.py +410**

```python
def _start_child(self, wrap):
    if len(wrap.forktimes) > wrap.workers:
        if time.time() - wrap.forktimes[0] < wrap.workers:
            LOG.info(_LI('Forking too fast, sleeping'))
            time.sleep(1)

        wrap.forktimes.pop(0)

    wrap.forktimes.append(time.time())

    pid = os.fork()
    if pid == 0:
        self.launcher = self._child_process(wrap.service)
        while True:
            self._child_process_handle_signal()
            status, signo = self._child_wait_for_exit_or_signal(
                self.launcher)
            if not _is_sighup_and_daemon(signo):
                self.launcher.wait()
                break
            self.launcher.restart()

        os._exit(status)

    LOG.info(_LI('Started child %d'), pid)

    wrap.children.add(pid)
    self.children[pid] = wrap
```

看见熟悉的fork没有，只是简单的调用了一个os.fork()，然后子进程开始运行，子进程调用\_child\_process

**oslo_service/service.py +391**

```python
def _child_process(self, service):
    self._child_process_handle_signal()
    
    # Reopen the eventlet hub to make sure we don't share an epoll
    # fd with parent and/or siblings, which would be bad
    eventlet.hubs.use_hub()
        
    # Close write to ensure only parent has it open
    os.close(self.writepipe)
    # Create greenthread to watch for parent to close pipe
    eventlet.spawn_n(self._pipe_watcher)
        
    # Reseed random number generator
    random.seed()
            
    launcher = Launcher(self.conf)
    launcher.launch_service(service)
    return launcher
```

\_child\_process其实很简单，创建一个Launcher，调用Laucher.launch\_service方法，前面介绍过其实ServiceLauncher继承自Launcher，也是调用的launcher\_service方法，将服务启动，因此接下来的步骤可以参考前面，最终都将调用service.start方法启动服务

