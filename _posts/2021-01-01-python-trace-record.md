---
title: Python trace 心路歷程
tags: python
---
###### tags: python, trace

## 某個區間希望有不同的特性

常見用法： with, warpper (lib: contextlib)

### 使用情境

#### 此區間中間才有資源的使用權，也因此在區間結束後，不管成功或是失敗，都需要釋放資源 (資源取得是常見例子)

- 開檔
- 開啟遠端連線

##### 混合用法

contextlib 中的 ExitStack，可以收集許多擁有 context 性質的東西，統一恢復資源

舉例來說，以下情況可能需要開多個檔案，就可以使用 ExitStack 改寫，尤其在需要管理的資源數量不定時特別好用

``` =python
with open(file1) as f1:
  with open(file2) as f2:
    copy(f1, f2)
```

改寫後

``` =python
with ExitStack() as stack:
  files = [stack.enter_context(open(fname)) for fname in [f1, f2]]
  copy(files[1], files[2])
```

每次的 enter_context 都是將 with 的後處理交給 stack 離開區間後再統一執行

#### 對於 Exception 可以有特別的處理方式

- unittest 中的 assertRaises 收集區間中是否有發生指定 raise
- contextlib 中的 suppress，可以指定部分的 Exception 被忽略

#### 收集區間中的資訊或顯示

- unittest 中的 assertLogs，可以收集 logging 所印出的 log
- contextlib 中 redirect_stdout, redirect_stderr，可以將區間中的 stdout(也就是 print) 和 stderr，導到指定的 steam 中
  - 檔案也是種 stream 的概念
- 在區間前後的前處理與後處理，hook 的概念
  - 最簡單的用法就是在前後插 log
  - 但在 function 方面會無法方便地拿到參數，但是可以拿到 Exception，和 decorator 擅長的不一樣
    - TODO: link 到 wrap

### 概念與好用工具

#### 原生概念

``` =python
with A() as a:
  a.f()
```

實際上執行的順序如下

- Ａ() 後 new 出實體物件 x
- 執行 x.__enter__() 得到回傳值 a
- a.f()
- 離開此 with 區間時執行，可能是正常走完，也可能是因 exception 跳出
  - 執行 x.__exit__(self, exc_type, exc_value, tb)
    - 如果發生 raise 的時候
      - exc_type 為 exception 的 type
      - exc_value 為 exception 的 object
      - tb 為發生當下的 call stack

因此要為一個 class 賦予使用 with 使用的能力，需要實作 __enter__, __exit__ 兩個 function，as 的對象由 __enter__ 的 return 提供

#### 常見使用方式

##### 不需要資源使用介面的情況

以下面的情況來說，在區間內 ＶalueError 不會跳出，但發生時，不會繼續往後走，因此 not run 不會印出來

``` =python
with contextlib.suppress(ValueError):
  raise ValueError
  print('not run)
```

##### 需要 context 相關資訊的介面

cm 為取得區間中 logging 相關資訊的介面，以此例子來說，區間的概念實際只有需要收集的區間，但區間外還可取得區間中輸出的 logging 資訊，與 open 超 with 區間後略顯不同

``` =python
import logging
import unittest

class TestSomething(unittest.TestCase):
  with self.assertLogs('a.b', level='INFO') as cm:
    logging.debug('some happened)
  print(cm.outputs)
```

#### 常見提供方式

##### 使用 functools.contextmanager(asynccontextmanger)

因為其結構性與方便性，為最常使用的一種方式

###### 概念

``` =python
@functools.contextmanger
def context():
  # 原預期在 __enter__ 會做的事

  # 此處 yield 的效果相當於 __enter__ 中的 return
  # 因此如果有需要提供外部介面時，在 yield 後傳出
  yield
  # 原預期在 __exit__ 會做的事
```

###### 常見例子

原本提供 try-catch 介面，甚至有些需要 finally，如資源使用

``` =python
@functools.contextmanger
def context():
  try:
    resource = open_resource()
    yield resource
  except:
    pass
  finially:
    resource.close()
```

原本提供 with 介面，以此情況更需要使用 functool.contextmanger，因為使用原生的 __enter__, __exit__ 會顯得有些彆扭，且不直覺

``` =python
@functools.contextmanger
def open_connect_by_config(host_name):
  host, user = get_host_info_by_config(host_name)
  with asyncssh.connect(host, username=user) as conn:
    yield conn
```

##### 使用原生 __enter__, __exit__

以下以 資源存取介面即為當初 new 出來的物件“ 作為舉例，實際概念與使用 functools.contextmanager 相仿
以下面的狀況來說，Context new 出來的物件，即為執行 do_something 的物件

``` =python
class Context:
  def __enter__(self):
    # do something
    return self

with Context() as c:
  ｃ.do_something()
```
