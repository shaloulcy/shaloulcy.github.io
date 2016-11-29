---
layout: post 
author: shalou
title:  "kube-proxy源码解析"
category: 容器技术
tag: [k8s, kubelet]
---

*本代码分析基于k8s v1.3.6*

kube-proxy应该是k8s所有服务里面最简单的服务，它的功能很单一，主要用来管理service，包括service的负载均衡。每个service都有一个cluster ip，service依靠selector label，对应后台的Pod，这个工作主要有kube-controller-manager的endpoint-controller完成，这个controller会生成与service相对应的endpoint。

kube-proxy运行于每一个node节点，它的主要工作就是对service、endpoint进行watch，然后在每一个节点建立相关的iptables表项，这样我们就可以在任意一个node上访问service，同时，假如一个service包含多个endpoint，它还起着负载均衡的作用，流量将平均分配到一个endpoint。

<!-- more -->

## 一、服务函数入口

和其他服务一致，kube-proxy的主要位于kubernetes/cmd/kube-proxy。首先options.NewProxyConfig()创建一个配置文件，然后利用配置文件创建服务app.NewProxyServerDefault(config)，最后运行这个服务。

**kubernetes/cmd/kube-proxy/proxy.go +38**

```go
func main() {
	config := options.NewProxyConfig()
	config.AddFlags(pflag.CommandLine)
    ...
	s, err := app.NewProxyServerDefault(config)
	...
	if err = s.Run(); err != nil {
		fmt.Fprintf(os.Stderr, "%v\n", err)
		os.Exit(1)
	}
}
```

## 二、创建服务

### 1、proxier和endpointsHandler的创建

kube-proxy的主要代码就位于创建服务，我们现在解析app.NewProxyServerDefault(config)这个函数。

**kubernetes/cmd/kube-proxy/app/server.go +126**

```go
func NewProxyServerDefault(config *options.ProxyServerConfig) (*ProxyServer, error) {
	...
	// Create a iptables utils.
	execer := exec.New()
	dbus := utildbus.New()
	iptInterface := utiliptables.New(execer, dbus, protocol)
    ...
	var proxier proxy.ProxyProvider
	var endpointsHandler proxyconfig.EndpointsConfigHandler

	proxyMode := getProxyMode(string(config.Mode), client.Nodes(), hostname, iptInterface, iptables.LinuxKernelCompatTester{})
	if proxyMode == proxyModeIptables {
		glog.V(0).Info("Using iptables Proxier.")
		...
		proxierIptables, err := iptables.NewProxier(iptInterface, execer, config.IPTablesSyncPeriod.Duration, config.MasqueradeAll, int(*config.IPTablesMasqueradeBit), config.ClusterCIDR, hostname, getNodeIP(client, hostname))
		if err != nil {
			glog.Fatalf("Unable to create proxier: %v", err)
		}
		proxier = proxierIptables
		endpointsHandler = proxierIptables
		...
	} else {
		glog.V(0).Info("Using userspace Proxier.")
		...
	}
    ...
	serviceConfig := proxyconfig.NewServiceConfig()
	serviceConfig.RegisterHandler(proxier)

	endpointsConfig := proxyconfig.NewEndpointsConfig()
	endpointsConfig.RegisterHandler(endpointsHandler)

	proxyconfig.NewSourceAPI(
		client,
		config.ConfigSyncPeriod,
		serviceConfig.Channel("api"),
		endpointsConfig.Channel("api"),
	)
	...
	return NewProxyServer(client, config, iptInterface, proxier, eventBroadcaster, recorder, conntracker, proxyMode)
}
```

kube-proxy有两种运行模式，一种是基于用户态proxy，另一种是基于内核态的iptables。基于iptables，效率更高。这里我们主要讨论基于iptables的工作模式。

进入 if proxyMode == proxyModeIptables 函数，这个函数内部主要构建了两个对象，一个是proxier，另一个是endpointsHandler，他们都是proxierIptables。

### 2、serviceconfig创建

NewServiceConfig首先创建了一个channel，然后创建一个serviceStore，这个serviceStore主要用于存储从api-service watch的service，当然之前还有一些过滤操作。然后创建了一个mux和bcaster，然后启动一个goroutine,运行函数watchForUpdates，假如updates可读，则立即执行bcaster.Notify(accessor.MergedState())。其实这里利用了[broadcaster](https://github.com/kubernetes/kubernetes/blob/v1.3.6/pkg/util/config/config.go)这一套框架，其实主要是我们可以向broadcaster注册listeners，broadcaster有更新时，调用notify通知所有的listener。

serviceConfig.RegisterHandler(proxier)其实就是注册了一个listener，到收到通知时，它会执行handler.OnServiceUpdate(instance.([]api.Service))函数。

只有updates可读时，才会触发bcaster.Notify，进而促发handler.OnServiceUpdate(instance.([]api.Service)),那么谁会往updates这个channel里面写入东西呢？

**kubernetes/cmd/kube-proxy/app/server.go +246**

```go
	serviceConfig := proxyconfig.NewServiceConfig()
	serviceConfig.RegisterHandler(proxier)
```

**kubernetes/pkg/proxy/config/config.go +193**

```go
func NewServiceConfig() *ServiceConfig {
	updates := make(chan struct{}, 1)
	store := &serviceStore{updates: updates, services: make(map[string]map[types.NamespacedName]api.Service)}
	mux := config.NewMux(store)
	bcaster := config.NewBroadcaster()
	go watchForUpdates(bcaster, store, updates)
	return &ServiceConfig{mux, bcaster, store}
}

func (c *ServiceConfig) RegisterHandler(handler ServiceConfigHandler) {
	c.bcaster.Add(config.ListenerFunc(func(instance interface{}) {
		glog.V(3).Infof("Calling handler.OnServiceUpdate()")
		handler.OnServiceUpdate(instance.([]api.Service))
	}))
}
...
func watchForUpdates(bcaster *config.Broadcaster, accessor config.Accessor, updates <-chan struct{}) {
	for true {
		<-updates
		bcaster.Notify(accessor.MergedState())
	}
}
```

### 3、endpointsConfig的创建

逻辑和serviceConfig的创建完全一样，只是将service换成了endpoint。

**kubernetes/cmd/kube-proxy/app/server.go +249**

```go
	endpointsConfig := proxyconfig.NewEndpointsConfig()
	endpointsConfig.RegisterHandler(endpointsHandler)
```

**kubernetes/pkg/proxy/config/config.go +84**

```go
func NewEndpointsConfig() *EndpointsConfig {
	updates := make(chan struct{}, 1)
	store := &endpointsStore{updates: updates, endpoints: make(map[string]map[types.NamespacedName]api.Endpoints)}
	mux := config.NewMux(store)
	bcaster := config.NewBroadcaster()
	go watchForUpdates(bcaster, store, updates)
	return &EndpointsConfig{mux, bcaster, store}
}

func (c *EndpointsConfig) RegisterHandler(handler EndpointsConfigHandler) {
	c.bcaster.Add(config.ListenerFunc(func(instance interface{}) {
		glog.V(3).Infof("Calling handler.OnEndpointsUpdate()")
		handler.OnEndpointsUpdate(instance.([]api.Endpoints))
	}))
}
```

###4、 service和endpoint的真实来源

service、endpoint肯定来源于apiserver，kube-proxy如何获取呢？获取的关键代码是proxyconfig.NewSourceAPI这个函数。我们看看这个函数传入的几个参数，主要看后两个，serviceConfig.Channel("api")和endpointsConfig.Channel("api")，这里以service为例。

**kubernetes/cmd/kube-proxy/app/server.go +252**

```go
	proxyconfig.NewSourceAPI(
		client,
		config.ConfigSyncPeriod,
		serviceConfig.Channel("api"),
		endpointsConfig.Channel("api"),
	)
```

首先调用c.mux.Channel构建创建了ch，ch本质上是一个channel，然后又创建了一个serviceCh。启动一个goroutine，这个goroutine的作用就是从serviceCh读取东西，然后写入ch。最后返回这个serviceCh

**kubernetes/pkg/proxy/config/config.go +243**

```go
func (c *ServiceConfig) Channel(source string) chan ServiceUpdate {
	ch := c.mux.Channel(source)
	serviceCh := make(chan ServiceUpdate)
	go func() {
		for update := range serviceCh {
			ch <- update
		}
		close(ch)
	}()
	return serviceCh
}
```

返回的serviceCh被传入proxyconfig.NewSourceAPI函数。我们看一下这个函数，其实主要创建了两个reflector，用来从kube-apiserver同步service和endpoint信息，同步的信息会写入serviceCh和endpointCh。这些信息进而会写入ch这个channel。那么谁会从ch读取信息呢。问题的关键是ch := c.mux.Channel(source)

**kubernetes/pkg/proxy/config/api.go +28**

```go
func NewSourceAPI(c cache.Getter, period time.Duration, servicesChan chan<- ServiceUpdate, endpointsChan chan<- EndpointsUpdate) {
	servicesLW := cache.NewListWatchFromClient(c, "services", api.NamespaceAll, fields.Everything())
	cache.NewReflector(servicesLW, &api.Service{}, NewServiceStore(nil, servicesChan), period).Run()

	endpointsLW := cache.NewListWatchFromClient(c, "endpoints", api.NamespaceAll, fields.Everything())
	cache.NewReflector(endpointsLW, &api.Endpoints{}, NewEndpointsStore(nil, endpointsChan), period).Run()
}

func NewServiceStore(store cache.Store, ch chan<- ServiceUpdate) cache.Store {
	fn := func(objs []interface{}) {
		var services []api.Service
		for _, o := range objs {
			services = append(services, *(o.(*api.Service)))
		}
		ch <- ServiceUpdate{Op: SET, Services: services}
	}
	if store == nil {
		store = cache.NewStore(cache.MetaNamespaceKeyFunc)
	}
	return &cache.UndeltaStore{
		Store:    store,
		PushFunc: fn,
	}
}

func NewEndpointsStore(store cache.Store, ch chan<- EndpointsUpdate) cache.Store {
	fn := func(objs []interface{}) {
		var endpoints []api.Endpoints
		for _, o := range objs {
			endpoints = append(endpoints, *(o.(*api.Endpoints)))
		}
		ch <- EndpointsUpdate{Op: SET, Endpoints: endpoints}
	}
	if store == nil {
		store = cache.NewStore(cache.MetaNamespaceKeyFunc)
	}
	return &cache.UndeltaStore{
		Store:    store,
		PushFunc: fn,
	}
}
```

### 5、Mux框架

Mux框架代码的位于[kubernetes/pkg/util/config/config.go](kubernetes/pkg/util/config/config.go)。它的主要作用是合并多个数据源，每一次调用Channel(source)都会注册一个数据源，同时返回一个channel，我们可以往返回的channel中写入数据，一旦写入数据就会促发Merge函数。上面的ch这个channel，就是调用Channel返回的。我们看看service的Merge函数，endpoint类似

**kubernetes/pkg/proxy/config/config.go +235**

```go
func (s *serviceStore) Merge(source string, change interface{}) error {
	s.serviceLock.Lock()
	services := s.services[source]
	if services == nil {
		services = make(map[types.NamespacedName]api.Service)
	}
	update := change.(ServiceUpdate)
	switch update.Op {
	case ADD:
		glog.V(5).Infof("Adding new service from source %s : %s", source, spew.Sdump(update.Services))
		for _, value := range update.Services {
			name := types.NamespacedName{Namespace: value.Namespace, Name: value.Name}
			services[name] = value
		}
	case REMOVE:
		glog.V(5).Infof("Removing a service %s", spew.Sdump(update))
		for _, value := range update.Services {
			name := types.NamespacedName{Namespace: value.Namespace, Name: value.Name}
			delete(services, name)
		}
	case SET:
		glog.V(5).Infof("Setting services %s", spew.Sdump(update))
		// Clear the old map entries by just creating a new map
		services = make(map[types.NamespacedName]api.Service)
		for _, value := range update.Services {
			name := types.NamespacedName{Namespace: value.Namespace, Name: value.Name}
			services[name] = value
		}
	default:
		glog.V(4).Infof("Received invalid update type: %s", spew.Sdump(update))
	}
	s.services[source] = services
	s.serviceLock.Unlock()
	if s.updates != nil {
		// Since we record the snapshot before sending this signal, it's
		// possible that the consumer ends up performing an extra update.
		select {
		case s.updates <- struct{}{}:
		default:
			glog.V(4).Infof("Service handler already has a pending interrupt.")
		}
	}
	return nil
}
```
这个函数的作用写将获取的service写到serviceStore.services，然后往updates写入一个空的结构体，updates变成可读的，这样就将前面的东西串联起来了。

### 6、OnServiceUpdate和OnEndpointsUpdate

以OnServiceUpdate函数为例

```go
func (proxier *Proxier) OnServiceUpdate(allServices []api.Service) {
	start := time.Now()
	defer func() {
		glog.V(4).Infof("OnServiceUpdate took %v for %d services", time.Since(start), len(allServices))
	}()
	proxier.mu.Lock()
	defer proxier.mu.Unlock()
	proxier.haveReceivedServiceUpdate = true

	activeServices := make(map[proxy.ServicePortName]bool) // use a map as a set

	for i := range allServices {
		service := &allServices[i]
		svcName := types.NamespacedName{
			Namespace: service.Namespace,
			Name:      service.Name,
		}

		// if ClusterIP is "None" or empty, skip proxying
		if !api.IsServiceIPSet(service) {
			glog.V(3).Infof("Skipping service %s due to clusterIP = %q", svcName, service.Spec.ClusterIP)
			continue
		}

		for i := range service.Spec.Ports {
			servicePort := &service.Spec.Ports[i]

			serviceName := proxy.ServicePortName{
				NamespacedName: svcName,
				Port:           servicePort.Name,
			}
			activeServices[serviceName] = true
			info, exists := proxier.serviceMap[serviceName]
			if exists && proxier.sameConfig(info, service, servicePort) {
				// Nothing changed.
				continue
			}
			if exists {
				// Something changed.
				glog.V(3).Infof("Something changed for service %q: removing it", serviceName)
				delete(proxier.serviceMap, serviceName)
			}
			serviceIP := net.ParseIP(service.Spec.ClusterIP)
			glog.V(1).Infof("Adding new service %q at %s:%d/%s", serviceName, serviceIP, servicePort.Port, servicePort.Protocol)
			info = newServiceInfo(serviceName)
			info.clusterIP = serviceIP
			info.port = int(servicePort.Port)
			info.protocol = servicePort.Protocol
			info.nodePort = int(servicePort.NodePort)
			info.externalIPs = service.Spec.ExternalIPs
			// Deep-copy in case the service instance changes
			info.loadBalancerStatus = *api.LoadBalancerStatusDeepCopy(&service.Status.LoadBalancer)
			info.sessionAffinityType = service.Spec.SessionAffinity
			info.loadBalancerSourceRanges = service.Spec.LoadBalancerSourceRanges
			proxier.serviceMap[serviceName] = info

			glog.V(4).Infof("added serviceInfo(%s): %s", serviceName, spew.Sdump(info))
		}
	}

	staleUDPServices := sets.NewString()
	// Remove services missing from the update.
	for name := range proxier.serviceMap {
		if !activeServices[name] {
			glog.V(1).Infof("Removing service %q", name)
			if proxier.serviceMap[name].protocol == api.ProtocolUDP {
				staleUDPServices.Insert(proxier.serviceMap[name].clusterIP.String())
			}
			delete(proxier.serviceMap, name)
		}
	}

	proxier.syncProxyRules()
	proxier.deleteServiceConnections(staleUDPServices.List())

}
```

代码的核心是根据获取的service信息，构建proxier.serviceMap，然后调用proxier.syncProxyRules()去同步iptables信息。

从上述的分析可知，一旦service、endpoint有变化，相应的iptables规则就会得到更新。

## 三、运行服务

**kubernetes/cmd/kube-proxy/app/server.go +272**

```go
func (s *ProxyServer) Run() error {
	// remove iptables rules and exit
	if s.Config.CleanupAndExit {
		encounteredError := userspace.CleanupLeftovers(s.IptInterface)
		encounteredError = iptables.CleanupLeftovers(s.IptInterface) || encounteredError
		if encounteredError {
			return errors.New("Encountered an error while tearing down rules.")
		}
		return nil
	}

	s.Broadcaster.StartRecordingToSink(s.Client.Events(""))

	// Start up a webserver if requested
	if s.Config.HealthzPort > 0 {
		http.HandleFunc("/proxyMode", func(w http.ResponseWriter, r *http.Request) {
			fmt.Fprintf(w, "%s", s.ProxyMode)
		})
		configz.InstallHandler(http.DefaultServeMux)
		go wait.Until(func() {
			err := http.ListenAndServe(s.Config.HealthzBindAddress+":"+strconv.Itoa(int(s.Config.HealthzPort)), nil)
			if err != nil {
				glog.Errorf("Starting health server failed: %v", err)
			}
		}, 5*time.Second, wait.NeverStop)
	}

	// Tune conntrack, if requested
	if s.Conntracker != nil {
		max, err := getConntrackMax(s.Config)
		if err != nil {
			return err
		}
		if max > 0 {
			err := s.Conntracker.SetMax(max)
			if err != nil {
				if err != readOnlySysFSError {
					return err
				}
				const message = "DOCKER RESTART NEEDED (docker issue #24000): /sys is read-only: can't raise conntrack limits, problems may arise later."
				s.Recorder.Eventf(s.Config.NodeRef, api.EventTypeWarning, err.Error(), message)
			}
		}
		if s.Config.ConntrackTCPEstablishedTimeout.Duration > 0 {
			if err := s.Conntracker.SetTCPEstablishedTimeout(int(s.Config.ConntrackTCPEstablishedTimeout.Duration / time.Second)); err != nil {
				return err
			}
		}
	}

	// Birth Cry after the birth is successful
	s.birthCry()

	// Just loop forever for now...
	s.Proxier.SyncLoop()
	return nil
}

```

**kubernetes/pkg/proxy/iptables/proxier.go +385**

```go
func (proxier *Proxier) SyncLoop() {
	t := time.NewTicker(proxier.syncPeriod)
	defer t.Stop()
	for {
		<-t.C
		glog.V(6).Infof("Periodic sync")
		proxier.Sync()
	}
}

func (proxier *Proxier) Sync() {
	proxier.mu.Lock()
	defer proxier.mu.Unlock()
	proxier.syncProxyRules()
}
```

Run()的作用主要是每隔proxier.syncPeriod，会调用一次proxier.Sync()，进而调用proxier.syncProxyRules()。proxier.syncPeriod的默认值为30秒

至于怎么更新iptables，查看synxProxyRules即可。


## 四、总结

kube-proxy的功能较为单一，其核心思想是同kube-apiserver同步service和endpoint信息，然后更新到iptables。
