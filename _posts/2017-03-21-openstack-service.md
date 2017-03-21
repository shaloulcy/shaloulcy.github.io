---
layout: post 
author: shalou
title:  "OpenStack中的Service" 
category: openstack
tag: [openstack, service]
---

*本文档基于OpenStack liberty*

OpenStack里面的Service主要分为两类，第一类是普通的Service，第二类是WSGIService，我们以nova组件为例，nova-api是wsgiservice，而其他nova组件则为普通的service

## 1. 普通Service

我们首先看看nova-conductor的启动文件

<!-- more -->

**nova/cmd/conductor.py +34**

```python
def main():
    config.parse_args(sys.argv)
    logging.setup(CONF, "nova")
    utils.monkey_patch()
    objects.register_all()

    gmr.TextGuruMeditation.setup_autorun(version)

    server = service.Service.create(binary='nova-conductor',
                                    topic=CONF.conductor.topic,
                                    manager=CONF.conductor.manager)
    workers = CONF.conductor.workers or processutils.get_worker_count()
    service.serve(server, workers=workers)
    service.wait()
```

即调用service.Service.create函数，调用的其实是class Service(service.Service)的初始化函数，目的是初始化一个Service对象，之前的文档说过，最终将调用Service的start方法启动服务，我们看看他的start方法

**nova/service.py +159**

```python
def start(self):
    verstr = version.version_string_with_package()
    LOG.info(_LI('Starting %(topic)s node (version %(version)s)'),
              {'topic': self.topic, 'version': verstr})
    self.basic_config_check()
    self.manager.init_host()
    self.model_disconnected = False
    ctxt = context.get_admin_context() 
    self.service_ref = objects.Service.get_by_host_and_binary(
        ctxt, self.host, self.binary)
    if not self.service_ref:
        try:
            self.service_ref = self._create_service_ref(ctxt)
        except (exception.ServiceTopicExists,
                exception.ServiceBinaryExists):
            # NOTE(danms): If we race to create a record with a sibling
            # worker, don't fail here.
            self.service_ref = objects.Service.get_by_host_and_binary(
                ctxt, self.host, self.binary)

    self.manager.pre_start_hook()

    if self.backdoor_port is not None:
        self.manager.backdoor_port = self.backdoor_port

    LOG.debug("Creating RPC server for service %s", self.topic)

    target = messaging.Target(topic=self.topic, server=self.host)

    endpoints = [
        self.manager,
        baserpc.BaseRPCAPI(self.manager.service_name, self.backdoor_port)
    ]       
    endpoints.extend(self.manager.additional_endpoints)

    serializer = objects_base.NovaObjectSerializer()

    self.rpcserver = rpc.get_server(target, endpoints, serializer)
    self.rpcserver.start()           

    self.manager.post_start_hook()      

    LOG.debug("Join ServiceGroup membership for this service %s",
              self.topic)
    # Add service to the ServiceGroup membership group.
    self.servicegroup_api.join(self.host, self.topic, self)
    
    if self.periodic_enable: 
        if self.periodic_fuzzy_delay:
            initial_delay = random.randint(0, self.periodic_fuzzy_delay)
        else:
            initial_delay = None

        self.tg.add_dynamic_timer(self.periodic_tasks,
                                 initial_delay=initial_delay,
                                 periodic_interval_max=
                                    self.periodic_interval_max)
```

该方法其实就是进行一些初始化操作，一个重要的操作就是初始化对消息队列的监听，假如有消息到达时，创建协程进行处理，最后将一些周期的tasks添加进来



## 2. WSGIService

首先看看nova-api

**nova/cmd/api.py +40**

```python
def main():
    config.parse_args(sys.argv)
    logging.setup(CONF, "nova")
    utils.monkey_patch()
    objects.register_all()

    gmr.TextGuruMeditation.setup_autorun(version)

    launcher = service.process_launcher()
    for api in CONF.enabled_apis:
        should_use_ssl = api in CONF.enabled_ssl_apis
        if api == 'ec2':
            server = service.WSGIService(api, use_ssl=should_use_ssl,
                                         max_url_len=16384)
        else:
            server = service.WSGIService(api, use_ssl=should_use_ssl)
        launcher.launch_service(server, workers=server.workers or 1)
    launcher.wait()
```

查看WSGIService的start方法

**nova/service.py +386**

```python
def start(self):
    if self.manager:
        self.manager.init_host()
        self.manager.pre_start_hook()
        if self.backdoor_port is not None:
            self.manager.backdoor_port = self.backdoor_port
    self.server.start()
    if self.manager:
        self.manager.post_start_hook()
```

本质上调用server.start（）方法

**nova/wsig.py +157**

```python
def start(self):   
    dup_socket = self._socket.dup()
    dup_socket.setsockopt(socket.SOL_SOCKET,
                          socket.SO_REUSEADDR, 1)
    # sockets can hang around forever without keepalive
    dup_socket.setsockopt(socket.SOL_SOCKET,
                          socket.SO_KEEPALIVE, 1)

    # This option isn't available in the OS X version of eventlet
    if hasattr(socket, 'TCP_KEEPIDLE'):
        dup_socket.setsockopt(socket.IPPROTO_TCP,
                              socket.TCP_KEEPIDLE,
                              CONF.tcp_keepidle)
    ...
    wsgi_kwargs = {
        'func': eventlet.wsgi.server,
        'sock': dup_socket,
        'site': self.app,
        'protocol': self._protocol,
        'custom_pool': self._pool,
        'log': self._logger,
        'log_format': CONF.wsgi_log_format,
        'debug': False,
        'keepalive': CONF.wsgi_keep_alive,
        'socket_timeout': self.client_socket_timeout
        }

    if self._max_url_len:
        wsgi_kwargs['url_length_limit'] = self._max_url_len

    self._server = eventlet.spawn(**wsgi_kwargs)
```
最后调用eventlet.spawn创建了一个协程，协程的function是eventlet.wsgi.server，我们可以进一步深入看看该函数

**eventlet/wsgi.py +714**

```python
def server(sock, site,
           log=None,
           environ=None,
           max_size=None,
           max_http_version=DEFAULT_MAX_HTTP_VERSION,
           protocol=HttpProtocol,
           server_event=None,
           minimum_chunk_size=None,
           log_x_forwarded_for=True,
           custom_pool=None,
           keepalive=True,
           log_output=True,
           log_format=DEFAULT_LOG_FORMAT,
           url_length_limit=MAX_REQUEST_LINE,
           debug=True,
           socket_timeout=None,
           capitalize_response_headers=True):
    serv = Server(sock, sock.getsockname(),
                  site, log,
                  environ=environ,
                  max_http_version=max_http_version,
                  protocol=protocol,
                  minimum_chunk_size=minimum_chunk_size,
                  log_x_forwarded_for=log_x_forwarded_for,
                  keepalive=keepalive,
                  log_output=log_output,
                  log_format=log_format,
                  url_length_limit=url_length_limit,
                  debug=debug,
                  socket_timeout=socket_timeout,
                  capitalize_response_headers=capitalize_response_headers,
                  )
    if server_event is not None:
        server_event.send(serv)
    if max_size is None:
        max_size = DEFAULT_MAX_SIMULTANEOUS_REQUESTS
    if custom_pool is not None:
        pool = custom_pool
    else:
        pool = greenpool.GreenPool(max_size)
    try:
        host, port = sock.getsockname()[:2]
        port = ':%s' % (port, )
        if hasattr(sock, 'do_handshake'):
            scheme = 'https'
            if port == ':443':
                port = ''
        else:
            scheme = 'http'
            if port == ':80':
                port = ''

        serv.log.info("(%s) wsgi starting up on %s://%s%s/" % (
            serv.pid, scheme, host, port))
        while is_accepting:
            try:
                client_socket = sock.accept()
                client_socket[0].settimeout(serv.socket_timeout)
                serv.log.debug("(%s) accepted %r" % (
                    serv.pid, client_socket[1]))
                try:
                    pool.spawn_n(serv.process_request, client_socket)
                except AttributeError:
                    warnings.warn("wsgi's pool should be an instance of "
                                  "eventlet.greenpool.GreenPool, is %s. Please convert your"
                                  " call site to use GreenPool instead" % type(pool),
                                  DeprecationWarning, stacklevel=2)
                    pool.execute_async(serv.process_request, client_socket)
            except ACCEPT_EXCEPTIONS as e:
                if support.get_errno(e) not in ACCEPT_ERRNO:
                    raise
            except (KeyboardInterrupt, SystemExit):
                serv.log.info("wsgi exiting")
                break
    finally:
        pool.waitall()
        serv.log.info("(%s) wsgi exited, is_accepting=%s" % (
            serv.pid, is_accepting))
        try:
            # NOTE: It's not clear whether we want this to leave the
            # socket open or close it.  Use cases like Spawning want
            # the underlying fd to remain open, but if we're going
            # that far we might as well not bother closing sock at
            # all.
            sock.close()
        except socket.error as e:
            if support.get_errno(e) not in BROKEN_SOCK:
                traceback.print_exc()
```

首先创建一个server，然后进入一个while循环，循环缩减后如下

```python
while True:
    client_socket = sock.accept()
    pool.spawn_n(serv.process_request, client_socket)
```

即接受一个请求后，创建一个协程处理该请求，原理很是很简单的


## 3. 总结
本文介绍了两种Service服务，其实深入剖析了wsigservice的实现，当深入底层时，发现其原理和教科书上相似             
