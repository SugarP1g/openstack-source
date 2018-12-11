作者：Bill\_Xiang\_

原文：https://blog.csdn.net/bill_xiang_/article/details/78701612

在OpenStack中除了使用数据库对云平台所产生的数据进行持久化外，还需要对一些常用的数据或状态进行缓存。而oslo.cache便通过dogpile.cache库实现了一个缓存机制为OpenStack其他组件提供缓存。目前，oslo.cache支持多种缓存机制，包括Memcache、etcd 3.x、MongoDB、dictionary等。本文将详细介绍oslo.cache提供的缓存机制与常用的使用方法。

## 1. dogpile.cache库

dogpile.cache是一个缓存API，它为各种类型的缓存后端提供了一个通用的接口；另外，它还提供了API钩子，可以将这些不同的缓存后端与dogpile库提供的锁机制结合使用。由于本文重点介绍oslo.cache，所以不对dogpile.cache库做深入展开，有兴趣的同学可以参考dogpile.cache文档。本文只对dogpile.cache中提供的通用接口进行介绍。

首先，dogpile.cache封装了CacheValue类用来保存一个缓存数据，该类中包含两个属性：

- payload属性，载荷，即缓存保存的数据；
- metadata属性，即dogpile.cache的元数据。

所有通过dogpile.cache库进行缓存的数据都会被封装成CacheValue类的实例化对象。

**CacheBackend类**是dogpile.cache为不同缓存后端提供的一个通用的缓存接口，该接口为不同类型的缓存后端，如Memcache等提供了统一的接口，程序员在使用时只需要为该类添加实现即可实现读写缓存等操作。该接口主要提供了一下几个属性和方法：

- key_mangler属性：表示一个key的压缩函数，可能是空，也可能声明为一个普通的实例方法。
- set(key, value)：缓存一个值，key表示这个值的关键字，value代表一个具体的CacheValue对象。
- set_multi(mapping)：缓存多个值，mapping是一个字典类型的值。
- get(key)：从缓存中获取一个值，返回一个CacheValue对象，如果指定的key找不到对应的值，则返回一个NoValue类型的对象，表示空。
- get_multi(keys)：从缓存中获取多个值。
- get_mutex(key)：为给定的键返回一个可选的互斥锁对象，该对象需要提供两个方法：加锁acquire()和释放锁release()。
- delete(key)：从缓存中删除一个值。
- delete_multi(keys)：从缓存中删除多个值。

## 2. oslo.cache支持的后端缓存机制

目前，oslo.cache实现了四种后端缓存机制的支持，包括Memcache、etcd 3.x、MongoDB、dictionary等。这些实现都保存在oslo_cache/backend目录下。

- oslo.cache.backend.memcache_pool：该模块提供了Memcache缓存池支持，首先实现了Memcache缓存连接池ConnectionPool，然后实现了PooledMemcachedBackend类对Memcache缓存连接池进行读写等操作。
- oslo_cache.backend.etcd3gw：该模块提供了etcd 3.x版本的缓存操作，实现了Etcd3gwCacheBackend类。
- oslo_cache.backend.mongo：该模块通过MongoCacheBackend类实现了使用MongoDB进行缓存的操作。
- oslo_cache.backed.dictionary：该模块DictCacheBackend类实现了通过字典进行缓存的操作机制。

上述这些实现缓存的类，包括PooledMemcachedBackend、Etcd3gwCacheBackend、MongoCacheBackend、DictCacheBackend，都是dogpile.cache中CacheBackend类的实现。其通过具体的后端缓存机制实现了对缓存的增删查等操作。

oslo.cache除了支持自身实现的四种缓存机制外，还支持dogpile.cache库本身实现的各类缓存机制，包括Redis、dbm、memory、pylibmc等。

# 3. oslo.cache缓存机制的实现

oslo.cache缓存机制的核心实现都定义在oslo_cache.core模块中，而缓存机制的实现主要依赖于以下几个方法：

- `create_region(function=function_key_generator)`：创建缓存区，该方法主要调用了dogpile.cache模块的 `make_region(function_key_generator=function)` 方法创建了一个CacheRegion对象。该对象通过配置文件找到对应的后端缓存实现机制创建缓存区，该对象通过具体的后端缓存机制实现了缓存数据的增删改操作。该方法调用了oslo.cache自己定义的key键生成方法。
- `configure_cache_region(conf, region)`：该方法通过配置文件中缓存的相关配置以及CacheRegion对象提供的配置方法配置缓存区。
- `get_memoization_decorator(conf, region, group, expiration_group=None)`：这是一个根据CacheRegion对象中cache_on_arguments()装饰器定义的oslo.cache的一个装饰器，其会根据group或expiration_group确定是否允许缓存以及缓存的时间。而CacheRegion对象中的cache_on_arguments()方法则提供了对一个或多个值的缓存、获取等操作方法。

## 4. oslo.cache的使用
oslo.cache的使用方式也非常简单，首先在使用oslo.cache时需要在对应OpenStack服务中添加相关的配置信息。这些配置信息包括是否允许使用缓存enabled、后端缓存机制backend以及缓存的保存时间cache_time等。

```
[cache]
enabled = true
backend = dogpile.cache.memory

[feature-name]
caching = True
cache_time = 7200
```

接下来，你可以直接使用oslo.cache中封装的方法进行缓存操作。首先根据配置文件创建一个CacheRegion对象，然后使用oslo.cache中的get_memoization_decorator装饰器进行缓存操作。

```python
from oslo_cache import core as cache
from oslo_config import cfg

CONF = cfg.CONF

caching = cfg.BoolOpt('caching', default=True)
cache_time = cfg.IntOpt('cache_time', default=3600)
CONF.register_opts([caching, cache_time], "feature-name")

cache.configure(CONF)
example_cache_region = cache.create_region()
MEMOIZE = cache.get_memoization_decorator(
CONF, example_cache_region, "feature-name")

# Load config file here

cache.configure_cache_region(CONF, example_cache_region)

@MEMOIZE
def f(x):
    print x
    return x
```

当然，你也可以对oslo.cache的功能进行扩展，使其符合项目的自身需求。在此，以nova组件为例介绍对oslo.cache的扩展方法。nova在nova.cache_utils模块中实现了对oslo.cache的扩展。首先，nova实现了两种创建CacheRegion对象的方式：

- `_get_default_cache_region(expiration_time)` 方法使用默认的后端缓存实现
- `_get_custom_cache_region(expiration_time=WEEK, backend=None, url=None)` 方法可以自己指定后端缓存的实现。

```python
def _get_default_cache_region(expiration_time):
    region = cache.create_region()
    if expiration_time != 0:
        CONF.cache.expiration_time = expiration_time
    cache.configure_cache_region(CONF, region)
    return region


def _get_custom_cache_region(expiration_time=WEEK,
                             backend=None,
                             url=None):
    """Create instance of oslo_cache client.
    For backends you can pass specific parameters by kwargs.
    For 'dogpile.cache.memcached' backend 'url' parameter must be specified.
    :param backend: backend name
    :param expiration_time: interval in seconds to indicate maximum
        time-to-live value for each key
    :param url: memcached url(s)
    """

    region = cache.create_region()
    region_params = {}
    if expiration_time != 0:
        region_params['expiration_time'] = expiration_time
     
    if backend == 'oslo_cache.dict':
        region_params['arguments'] = {'expiration_time': expiration_time}
    elif backend == 'dogpile.cache.memcached':
        region_params['arguments'] = {'url': url}
    else:
        raise RuntimeError(_('old style configuration can use '
                             'only dictionary or memcached backends'))
     
    region.configure(backend, **region_params)
    return region
```

接着，nova组件实现了一个CacheClient类，封装了对数据的缓存操作。该类包含一个region属性保存CacheRegion对象，而对数据的缓存、获取、删除等操作具体是通过CacheRegion对象来实现的。

```python
class CacheClient(object):
    """Replicates a tiny subset of memcached client interface."""

    def __init__(self, region):
        self.region = region
     
    def get(self, key):
        value = self.region.get(key)
        if value == cache.NO_VALUE:
            return None
        return value
     
    def get_or_create(self, key, creator):
        return self.region.get_or_create(key, creator)
     
    def set(self, key, value):
        return self.region.set(key, value)
     
    def add(self, key, value):
        return self.region.get_or_create(key, lambda: value)
     
    def delete(self, key):
        return self.region.delete(key)
     
    def get_multi(self, keys):
        values = self.region.get_multi(keys)
        return [None if value is cache.NO_VALUE else value for value in
                values]
     
    def delete_multi(self, keys):
        return self.region.delete_multi(keys)
```

最后，nova组件在cache_utils中实现了两个创建CacheClient对象的方法，这两个方法可以在使用中快速创建所需要的CacheClient对象。get_memcached_client()方法创建了一个后端缓存为Memcache的CacheClient对象，get_client()方法则创建了一个后端缓存为dictionary的CacheClient对象。其中，都会使用_warn_if_null_backend()方法检查后端缓存backend是否为空。

```python
def _warn_if_null_backend():
    if CONF.cache.backend == 'dogpile.cache.null':
        LOG.warning("Cache enabled with backend dogpile.cache.null.")


def get_memcached_client(expiration_time=0):
    """Used ONLY when memcached is explicitly needed."""
    # If the operator has [cache]/enabled flag on then we let oslo_cache
    # configure the region from the configuration settings
    if CONF.cache.enabled and CONF.cache.memcache_servers:
        _warn_if_null_backend()
        return CacheClient(_get_default_cache_region(expiration_time=expiration_time))


def get_client(expiration_time=0):
    """Used to get a caching client."""
    # If the operator has [cache]/enabled flag on then we let oslo_cache
    # configure the region from configuration settings.
    if CONF.cache.enabled:
        _warn_if_null_backend()
        return CacheClient(
            _get_default_cache_region(expiration_time=expiration_time))
    # If [cache]/enabled flag is off, we use the dictionary backend
    return CacheClient(
        _get_custom_cache_region(expiration_time=expiration_time,
            backend='oslo_cache.dict'))
```

在使用时，首先需要调用上述方法创建CacheClient对象，然后通过该对象进行具体的缓存操作。

```python
from nova import cache_utils


def memoize(func):
    @functools.wraps(func)
    def memoizer(context, reqid):
        global _CACHE
        if not _CACHE:
            _CACHE = cache_utils.get_client(expiration_time=_CACHE_TIME)
        key = "%s:%s" % (func.__name__, reqid)
        key = str(key)
        value = _CACHE.get(key)
        if value is None:
            value = func(context, reqid)
            _CACHE.set(key, value)
        return value
        return memoizer


@memoize
def id_to_glance_id(context, image_id):
    """Convert an internal (db) id to a glance id."""
    return objects.S3ImageMapping.get_by_id(context, image_id).uuid


@memoize
def glance_id_to_id(context, glance_id):
    """Convert a glance id to an internal (db) id."""
    if not glance_id:
        return
    try:
        return objects.S3ImageMapping.get_by_uuid(context, glance_id).id
    except exception.NotFound:
        s3imap = objects.S3ImageMapping(context, uuid=glance_id)
        s3imap.create()
        return s3imap.id
```

在上述示例中，nova首先创建了一个memoize装饰器，在该装饰器中首先调用get_client()获取了一个CacheClient对象，然后调用该对象的get()方法获取指定key的值，如果查不到则将该值保存到缓存中，如id_to_glance_id(context, image_id)方法中image_id的值和glance_id_to_id(context, glance_id)中的glance_id的值等。