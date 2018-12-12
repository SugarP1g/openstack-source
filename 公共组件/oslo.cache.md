# oslo.cache

作者：Bill\_Xiang\_

原文：https://blog.csdn.net/bill_xiang_/article/details/78701612

在 OpenStack 中除了使用数据库对云平台所产生的数据进行持久化外，还需要对一些常用的数据或状态进行缓存。而 oslo.cache 便通过 dogpile.cache 库实现了一个缓存机制为 OpenStack 其他组件提供缓存。目前，oslo.cache 支持多种缓存机制，包括 Memcache、etcd 3.x、MongoDB、dictionary 等。本文将详细介绍 oslo.cache 提供的缓存机制与常用的使用方法。

## 1. dogpile.cache 库

dogpile.cache 是一个缓存API，它为各种类型的缓存后端提供了一个通用的接口；另外，它还提供了API钩子，可以将这些不同的缓存后端与dogpile库提供的锁机制结合使用。由于本文重点介绍 oslo.cache，所以不对 dogpile.cache 库做深入展开，有兴趣的同学可以参考 dogpile.cache 文档。本文只对 dogpile.cache 中提供的通用接口进行介绍。

首先，dogpile.cache 封装了 CacheValue 类用来保存一个缓存数据，该类中包含两个属性：

- payload 属性，载荷，即缓存保存的数据；
- metadata 属性，即 dogpile.cache 的元数据。

所有通过 dogpile.cache 库进行缓存的数据都会被封装成 CacheValue 类的实例化对象。

**CacheBackend 类**是 dogpile.cache 为不同缓存后端提供的一个通用的缓存接口，该接口为不同类型的缓存后端，如 Memcache 等提供了统一的接口，程序员在使用时只需要为该类添加实现即可实现读写缓存等操作。该接口主要提供了一下几个属性和方法：

- key_mangler ：表示一个 key 的压缩函数，可能是空，也可能声明为一个普通的实例方法。
- set(key, value) ：缓存一个值，key 表示这个值的关键字，value 代表一个具体的 CacheValue 对象。
- set_multi(mapping) ：缓存多个值，mapping 是一个字典类型的值。
- get(key) ：从缓存中获取一个值，返回一个 CacheValue 对象，如果指定的 key 找不到对应的值，则返回一个 NoValue 类型的对象，表示空。
- get_multi(keys) ：从缓存中获取多个值。
- get_mutex(key) ：为给定的键返回一个可选的互斥锁对象，该对象需要提供两个方法：加锁 acquire() 和释放锁 release() 。
- delete(key) ：从缓存中删除一个值。
- delete_multi(keys) ：从缓存中删除多个值。

## 2. oslo.cache 支持的后端缓存机制

目前，oslo.cache 实现了四种后端缓存机制的支持，包括 Memcache、etcd 3.x、MongoDB、dictionary 等。这些实现都保存在 oslo_cache/backend 目录下。

- oslo.cache.backend.memcache_pool ：该模块提供了 Memcache 缓存池支持，首先实现了 Memcache 缓存连接池 ConnectionPool ，然后实现了 PooledMemcachedBackend 类对 Memcache 缓存连接池进行读写等操作。
- oslo_cache.backend.etcd3gw ：该模块提供了 etcd 3.x 版本的缓存操作，实现了 Etcd3gwCacheBackend类。
- oslo_cache.backend.mongo ：该模块通过 MongoCacheBackend 类实现了使用 MongoDB 进行缓存的操作。
- oslo_cache.backed.dictionary ：该模块 DictCacheBackend 类实现了通过字典进行缓存的操作机制。

上述这些实现缓存的类，包括 PooledMemcachedBackend、Etcd3gwCacheBackend、MongoCacheBackend、DictCacheBackend ，都是 dogpile.cache 中 CacheBackend 类的实现。其通过具体的后端缓存机制实现了对缓存的增删查等操作。

oslo.cache 除了支持自身实现的四种缓存机制外，还支持 dogpile.cache 库本身实现的各类缓存机制，包括 Redis、dbm、memory、pylibmc 等。

## 3. oslo.cache 缓存机制的实现

oslo.cache 缓存机制的核心实现都定义在 oslo_cache.core 模块中，而缓存机制的实现主要依赖于以下几个方法：

- `create_region(function=function_key_generator)` ：创建缓存区，该方法主要调用了 dogpile.cache 模块的 `make_region(function_key_generator=function)` 方法创建了一个 CacheRegion 对象。该对象通过配置文件找到对应的后端缓存实现机制创建缓存区，该对象通过具体的后端缓存机制实现了缓存数据的增删改操作。该方法调用了 oslo.cache 自己定义的 key 键生成方法。
- `configure_cache_region(conf, region)` ：该方法通过配置文件中缓存的相关配置以及 CacheRegion 对象提供的配置方法配置缓存区。
- `get_memoization_decorator(conf, region, group, expiration_group=None)` ：这是一个根据 CacheRegion 对象中 cache_on_arguments() 装饰器定义的 oslo.cache 的一个装饰器，其会根据 group 或 expiration_group 确定是否允许缓存以及缓存的时间。而 CacheRegion 对象中的 cache_on_arguments() 方法则提供了对一个或多个值的缓存、获取等操作方法。

## 4. oslo.cache 的使用

oslo.cache 的使用方式也非常简单，首先在使用 oslo.cache 时需要在对应 OpenStack 服务中添加相关的配置信息。这些配置信息包括是否允许使用缓存 enabled 、后端缓存机制 backend 以及缓存的保存时间 cache_time 等。

```
[cache]
enabled = true
backend = dogpile.cache.memory

[feature-name]
caching = True
cache_time = 7200
```

接下来，你可以直接使用 oslo.cache 中封装的方法进行缓存操作。首先根据配置文件创建一个 CacheRegion 对象，然后使用 oslo.cache 中的 get_memoization_decorator 装饰器进行缓存操作。

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

当然，你也可以对 oslo.cache 的功能进行扩展，使其符合项目的自身需求。在此，以 nova 组件为例介绍对 oslo.cache 的扩展方法。 nova 在 nova.cache_utils 模块中实现了对 oslo.cache 的扩展。首先， nova 实现了两种创建 CacheRegion 对象的方式：

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

接着， nova 组件实现了一个 CacheClient 类，封装了对数据的缓存操作。该类包含一个 region 属性保存 CacheRegion 对象，而对数据的缓存、获取、删除等操作具体是通过 CacheRegion 对象来实现的。

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

最后， nova 组件在 cache_utils 中实现了两个创建 CacheClient 对象的方法，这两个方法可以在使用中快速创建所需要的 CacheClient 对象。 get_memcached_client() 方法创建了一个后端缓存为 Memcache 的 CacheClient 对象， get_client() 方法则创建了一个后端缓存为 dictionary 的 CacheClient 对象。其中，都会使用 _warn_if_null_backend() 方法检查后端缓存 backend 是否为空。

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

在使用时，首先需要调用上述方法创建 CacheClient 对象，然后通过该对象进行具体的缓存操作。

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

在上述示例中， nova 首先创建了一个 memoize 装饰器，在该装饰器中首先调用 get_client() 获取了一个 CacheClient 对象，然后调用该对象的 get() 方法获取指定 key 的值，如果查不到则将该值保存到缓存中，如 id_to_glance_id(context, image_id) 方法中 image_id 的值和 glance_id_to_id(context, glance_id) 中的 glance_id 的值等。