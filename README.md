# fastapi-cache

![pypi](https://img.shields.io/pypi/v/fastapi-cache2.svg?style=flat)
![license](https://img.shields.io/github/license/long2ice/fastapi-cache)
![workflows](https://github.com/long2ice/fastapi-cache/workflows/pypi/badge.svg)
![workflows](https://github.com/long2ice/fastapi-cache/workflows/ci/badge.svg)

## Introduction

`fastapi-cache` is a tool to cache fastapi response and function result, with backends support `redis`, `memcache`, and `dynamodb`.

## Features

- Support `redis` and `memcache` and `in-memory` backends.
- Easily integrate with `fastapi`.
- Support http cache like `ETag` and `Cache-Control`.
- Event handlers for when a new key is added and existing key called

## Requirements

- `asyncio` environment.
- `redis` if use `RedisBackend`.
- `memcache` if use `MemcacheBackend`.
- `aiobotocore` if use `DynamoBackend`.

## Install

```shell
> pip install fastapi-cache2
```

or

```shell
> pip install "fastapi-cache2[redis]"
```

or

```shell
> pip install "fastapi-cache2[memcache]"
```

or

```shell
> pip install "fastapi-cache2[dynamodb]"
```

## Usage

### Quick Start

```python
import aioredis
from fastapi import FastAPI
from starlette.requests import Request
from starlette.responses import Response

from fastapi_cache import FastAPICache
from fastapi_cache.backends.redis import RedisBackend
from fastapi_cache.decorator import cache

app = FastAPI()


@cache()
async def get_cache():
    return 1


@app.get("/")
@cache(expire=60)
async def index(request: Request, response: Response):
    return dict(hello="world")


@app.on_event("startup")
async def startup():
    redis =  aioredis.from_url("redis://localhost", encoding="utf8", decode_responses=True)
    FastAPICache.init(RedisBackend(redis), prefix="fastapi-cache")

```

### Initialization

Firstly you must call `FastAPICache.init` on startup event of `fastapi`, there are some global config you can pass in.

### Use `cache` decorator

If you want cache `fastapi` response transparently, you can use `cache` as decorator between router decorator and view function and must pass `request` as param of view function.

Parameter | type, description
------------ | -------------
expire | int, states a caching time in seconds
namespace | str, namespace to use to store certain cache items
coder | which coder to use, e.g. JsonCoder
key_builder | which key builder to use, default to builtin


And if you want use `ETag` and `Cache-Control` features, you must pass `response` param also.

You can also use `cache` as decorator like other cache tools to cache common function result.

### Custom coder

By default use `JsonCoder`, you can write custom coder to encode and decode cache result, just need inherit `fastapi_cache.coder.Coder`.

```python
@app.get("/")
@cache(expire=60,coder=JsonCoder)
async def index(request: Request, response: Response):
    return dict(hello="world")
```

### Custom key builder

By default use builtin key builder, if you need, you can override this and pass in `cache` or `FastAPICache.init` to take effect globally.

```python
def my_key_builder(
    func,
    namespace: Optional[str] = "",
    request: Request = None,
    response: Response = None,
    *args,
    **kwargs,
):
    prefix = FastAPICache.get_prefix()
    cache_key = f"{prefix}:{namespace}:{func.__module__}:{func.__name__}:{args}:{kwargs}"
    return cache_key

@app.get("/")
@cache(expire=60,coder=JsonCoder,key_builder=my_key_builder)
async def index(request: Request, response: Response):
    return dict(hello="world")
```

### InMemoryBackend

`InMemoryBackend` store cache data in memory and use lazy delete, which mean if you don't access it after cached, it will not delete automatically.

### Event handling

For the events `new_key` and `existing_key` you can pass in a custom handler. By default there are no handlers.

*Via a decorator:*

```python
@FastAPICache.on_event("existing_key")
def exists_in_cache(func, *args, **kwargs):
    print("Existing key")
    return None

@FastAPICache.on_event("new_key")
def new_in_cache(func, *args, **kwargs):
    print("New key set")
    return None
```

*Via a class method:*

```python
def exists_in_cache(func, *args, **kwargs):
    print("Existing key")
    return None

def new_in_cache(func, *args, **kwargs):
    print("New key set")
    return None

FastAPICache.set_on_existing_key(exists_in_cache)
FastAPICache.set_on_new_key(new_in_cache)
```

## License

This project is licensed under the [Apache-2.0](https://github.com/long2ice/fastapi-cache/blob/master/LICENSE) License.
