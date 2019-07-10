# python 使用 memcached

## 1.memcache 服务器搭建

采用简单的docker搭建，方便快捷。

```
# 拉取镜像
docker pull memcached
# 运行
docker run --name my-memcache -p 11211:11211 -d memcached
```

## 2.python 使用

### 可能遇到错误如下

> 1.RuntimeError: no memcache module found
>
> 2.MemcachedKeyCharacterError: Control/space characters not allowed
>
> 3.AttributeError: 'int' object has no attribute 'encode'

1. 安装 python-memcached

   ```
   pip install python-memcached
   pip install python3-memcached
   ```

2. 取消 key里面的空格

3. key不能是int类型 ，转为str

## 3.基本使用

1.python

```python
import memcache
mc = memcache.Client(['127.0.0.1:11211'], debug=0)
mc.set("some_key", "Some value")
value = mc.get("some_key")
mc.set("another_key", 3)
mc.delete("another_key")
mc.set("key", "1")   # note that the key used for incr/decr must be a string.
mc.incr("key")
mc.decr("key")
```

2.flask werkzeug 

```python
from werkzeug.contrib.cache import MemcachedCache
cache = MemcachedCache(["192.168.5.141:11211"])
```

3.集合

```python
import json
from functools import wraps

from flask_jwt_extended import current_user

from config import cache


def get_cache_data(cache_key):
    """
    获取缓存
    :param cache_key: 缓存主键
    :return:
    """
    data_dict = cache.get(str(cache_key).replace(" ",""))
    if data_dict:
        return data_dict
    return None


def set_cache_data(cache_key, result, timeout=60 * 5):
    """
    设置缓存
    :param cache_key: 缓存主键
    :param result: 缓存内容
    :param timeout:300秒
    :return:
    """
    cache.set(str(cache_key).replace(" ",""), result, timeout=timeout)


def cache_decorator(fn):
    """
    缓存内容 get 请求
    :param fn:
    :return:
    """

    @wraps(fn)
    def _decorator(*args, **kwargs):
        parms = dict(username=current_user.admin_name, **kwargs)
        if kwargs.get("method", "get") == "get":
            cache_key = json.dumps(parms)
            dataDict = get_cache_data(cache_key)
            if dataDict:
                return True, dataDict
            status, dataDict = fn(*args, **kwargs)
            if status:
                set_cache_data(cache_key, dataDict)
            return status, dataDict
        return fn(*args, **kwargs)

    return _decorator

```



[参考](https://www.cnblogs.com/jaxu/p/5196811.html)



