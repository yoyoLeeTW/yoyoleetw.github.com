---
title: Python 使用 with / decorator 讓區間賦予特性
tags: python, context, with, contextlib
---

## Abstract

當使用資源時，不管執行的成功與否，都會需要做釋放，也就是我們熟知的 with，
但 with 在 python 實際上要做的是賦予 with 區間特別的功能 (如: open 就是只有區間中能讀寫檔案)，
此篇整理常見的情境、原理、使用方式

## 使用情境

### 資源管理

- 情境
  - 此區間中間才有資源的使用權，也因此在區間結束後，不管成功或是失敗，都需要釋放資源 (資源取得是常見例子)
- 例子
  - 開檔
  - 開啟遠端連線
  - pytest 中內建的 tmpdir fixture，讓指定 function 中有獨立的 tmp 資料夾

### 特化對於 Exception 的處理

- unittest 中的 `assertRaises` 收集區間中是否有發生指定 raise
- contextlib 中的 `suppress`，可以指定部分的 Exception 被忽略

### 收集區間中的資訊

- 例子
  - unittest 中的 `assertLogs`，可以收集 `logging` 所印出的 log
  - contextlib 中 redirect_stdout, redirect_stderr，可以將區間中的 stdout(也就是 print) 和 stderr，導到指定的 steam 中
    - 檔案也是種 stream 的概念
  - 在區間前後的前處理與後處理，hook 的概念
    - 最簡單的用法就是在前後插 log
    - 但在 function 方面會無法方便地拿到參數，但是可以拿到 Exception，和 decorator 擅長的不一樣
      - TODO: link 到 wrap

## 概念

### with 的概念

``` python
with A() as a:
  a.f()
```

- 使用條件
  - 實作 `__enter__`, `__exit__`，供執行時使用
- 執行順序
  - `Ａ()` 後 new 出實體物件 `x`
  - 執行 `x.__enter__`() 得到回傳值 `a`
  - `a.f()`
  - 離開此 with 區間時執行，可能是正常走完，也可能是因 exception 跳出
    - 執行 `x.__exit__(self, exc_type, exc_value, tb)`
      - 如果發生 raise 的時候，否則為 None
        - `exc_type` 為 exception 的 type
        - `exc_value` 為 exception 的 object
        - `tb` 為發生當下的 call stack
- as 的來源
  - 為 `__enter__` 時，藉由 return 提供
    - 如 new 出來的物件提供使用介面，則 return self

### with v.s. decorator

- 共通的部分
  - 實際上達到的目的都是針對某區域
- 相異的部分
  - with: 做用到的區域專指因 with 縮排的區間
  - decorator: 修飾 function 的整個區間

## 使用情況

### 不需要資源使用介面的情況

- 例子

``` python
with contextlib.suppress(ValueError):
  raise ValueError
  print('not run')   # 將不被執行
```

### 需要 context 相關資訊的介面

- 例子

``` python
import logging
import unittest

class TestSomething(unittest.TestCase):
  with self.assertLogs('a.b', level='INFO') as cm:
    # cm 為取得區間中 `logging` 相關資訊的介面
    logging.debug('some happened')
  # 以此例來說，區間的概念實際只有需要收集的區間，cm 的生命週期不限於 with 中，與 open 使用 with 的情況略顯不同
  print(cm.outputs)
```

## 提供方式

### contextlib.contextmanager (asynccontextmanger)

- 優勢: 因為其結構性與方便性
- 推薦使用

#### contextmanager 的使用概念

- 使用說明

``` python
@contextlib.contextmanger
def context():
  # 原預期在 __enter__ 會做的事

  # 此處 yield 的效果相當於 __enter__ 中的 return
  # 因此如果有需要提供外部介面時，在 yield 後傳出
  yield
  # 原預期在 __exit__ 會做的事
```

- 例子

``` python
@contextlib.contextmanger
def context():
  try:
    resource = open_resource()
    # 即使已經 yield 出去了，，但實際還沒有離開 tru-cache 的區間，所以 with 的區間如同在 yield 的位置
    yield resource
  except:
    pass
  finally:
    resource.close()
```

- 優點: 相比原生的 `__enter__`, `__exit__` 更直覺
- 例子

``` python
@contextlib.contextmanger
def open_connect_by_config(host_name):
  host, user = get_host_info_by_config(host_name)
  with asyncssh.connect(host, username=user) as conn:
    yield conn
```

### 原生 magic method

- magic method: `__enter__`, `__exit__`

``` python
class Context:
  def __enter__(self):
    # do something
    return self

with Context() as c:    # 此處的 c 即為 __enter__ 中的 return
  ｃ.do_something()
```

## 進階用法

### 統一管理多個區間

- 問題: 多個，或動態數量個 context 時
- 解法: contextlib 中的 `ExitStack`
  - 使用方式: call function 式的達到 `__enter__` 的效果，並在 ExitStack 區間離開時統一執行 `__exit__`
- 使用情況例子

``` python
with open(file1) as f1:
  with open(file2) as f2:
    copy(f1, f2)
```

- 改寫後

``` python
with ExitStack() as stack:
  files = [stack.enter_context(open(fname)) for fname in [f1, f2]]
  copy(files[1], files[2])
```
