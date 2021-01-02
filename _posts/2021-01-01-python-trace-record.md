---
title: Python trace 心路歷程
tags: python
---
###### tags: python, trace

## 讓 module 的介面有預設的使用，卻又有彈性

當需要個功能的時候，常會直覺地把 function 寫出來，比如說下面的程式
但此情況下是沒有主控權的彈性的，也就是說對於每個使用的人，都是一樣的立場

```python
def debug(msg):
  pass
```

要如何處理，常見的做法是把相關的使用一個 class 包裝起來
而在 module 一開始載入的時候就配置預設的
讓沒有想客製化的有同上共用的介面
但也提供想客製化的有 class 的介面的選擇

```python
class _Logger:
  def debug(self, msg):
    pass

logger = _Logger(core=_Core(),
  exception=None)
```

``` python
def get_cls_(cls_name):
    cls_map = {
        cls.__name__: cls
        for cls in A.__subclasses__()
    }
    return cls_map[cls_name]

class A:
    pass

class B(A):
    pass
```

e.g. [logging](https://docs.python.org/3/library/logging.config.html)

``` python
handlers:
  console:
    class : logging.StreamHandler
```

e.g. [yaml](http://pyyaml.org/wiki/PyYAMLDocumentation)

- 共用的資訊可以用參數傳進來，沒有就預設使用同一個，可以方便使用

```=python
pool = Pool()
def get_cleint(client_name, pool=pool):
    pass
```



