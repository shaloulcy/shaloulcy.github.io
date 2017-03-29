---
layout: post 
author: shalou
title:  "nova-compute对数据库的访问" 
category: openstack
tag: [openstack, nova-compute]
---

nova-compute有时需要访问数据库，比如它需要定时像数据库更新自己的状态、它需要更新instance的状态。但nova-compute只不能直接访问数据库的，安装计算节点的时候，计算节点的配置文件里面也没有配置数据库的信息。

nova-compute其实是通过rpc请求nova-conductor，由nova-conductor去更新数据

/etc/nova/nova.conf有一个配置，配置如何访问数据库，当use_local为false的时候通过发送rpc请求到nova-conductor，当use_local为true时，直接更新数据库，默认情况下为false

<!-- more -->

```
[conductor]
use_local=false
```

nova-compupte是如何更新数据库呢？看看Instance这个对象的几个函数，get_by_id表示用一个uuid去找一个instance，而destroy去销毁instance。他们都有一个装饰器，分别为@base.remotable_classmethod和@base.remotable，这两个装饰器就是更新数据库的奥秘所在

```python
@base.remotable_classmethod
def get_by_id(cls, context, inst_id, expected_attrs=None):
    if expected_attrs is None:
        expected_attrs = ['info_cache', 'security_groups']
    columns_to_join = _expected_cols(expected_attrs)
    db_inst = db.instance_get(context, inst_id,
                              columns_to_join=columns_to_join)
    return cls._from_db_object(context, cls(), db_inst,
                               expected_attrs)
                               
                              
@base.remotable
def destroy(self):
    if not self.obj_attr_is_set('id'):
        raise exception.ObjectActionError(action='destroy',
                                          reason='already destroyed')
    if not self.obj_attr_is_set('uuid'):
        raise exception.ObjectActionError(action='destroy',
                                          reason='no uuid')
    if not self.obj_attr_is_set('host') or not self.host:
        # NOTE(danms): If our host is not set, avoid a race
        constraint = db.constraint(host=db.equal_any(None))
    else:
        constraint = None

    cell_type = cells_opts.get_cell_type()
    if cell_type is not None:
        stale_instance = self.obj_clone()

    try:
        db_inst = db.instance_destroy(self._context, self.uuid,
                                      constraint=constraint)
        self._from_db_object(self._context, self, db_inst)
    except exception.ConstraintNotMet:
        raise exception.ObjectActionError(action='destroy',
                                          reason='host changed')
    if cell_type == 'compute':
        cells_api = cells_rpcapi.CellsAPI()
        cells_api.instance_destroy_at_top(self._context, stale_instance)
    delattr(self, base.get_attrname('id')) 
```

这两个装饰器位于oslo\_versionedobjects这个包里面，我们以@base.remotable\_classmethod为例

```python
def remotable_classmethod(fn):
    """Decorator for remotable classmethods."""
    @six.wraps(fn)
    def wrapper(cls, context, *args, **kwargs):
        if cls.indirection_api:
            result = cls.indirection_api.object_class_action(
                context, cls.obj_name(), fn.__name__, cls.VERSION,
                args, kwargs)
        else:
            result = fn(cls, context, *args, **kwargs)
            if isinstance(result, VersionedObject):
                result._context = context
        return result

    # NOTE(danms): Make this discoverable
    wrapper.remotable = True 
    wrapper.original_fn = fn 
    return classmethod(wrapper)
```

当cls.indirection\_api为真的时候，调用cls.indirection\_api.object\_class\_action方法，其中indirection\_api其实就是conductor_api。这样就把请求发送给nova-conductor了，由他去更新数据库


假如所有数据库请求都发送给nova-conductor，nova-conductor会不会成为一个瓶颈呢？这不需要担心，因为nova-conductor其实是可以部署多个的。
