.. post:: Nov 15, 2021
  :tags: Python, with, context manager
  :category: Python
  :location: TW
  :language: zh
  :exclude:

  with

===================================
Python 使用 with 讓區間賦予特性
===================================

摘要
======

我們知道取得資源後，不管執行的成功與否，都需要對資源做釋放，否則會有 leak 發生，
在 Python 中為達到此目的，常見的且推薦的做法是使用 with，
但 with 在 python 實際上要做的是賦予 with 區間特別的功能 (如: open 就是只有區間中能讀寫檔案)。

內容綱要

- **運作方式與原理**: 理解怎麼使用 ``with``，以及其運作原理
  - 使用以例子帶特性的方式介紹特性，因此如想快速理解的可以參考目錄跳過例子的部分
- **如何提供 with 可以使用的 API**: 理解如何實作出 ``with`` 的 API 供其他人使用
- **使用情境**: 理解常見的使用情境，並舉出一些情境下的例子

.. _quickstart:

快速入門
==========

情境：假設我們有一個檔案 ``input.txt``，需要將檔案的內容讀出來，並顯示在螢幕上

.. code-block:: python
  :linenos:
  
  f = open('input.txt', 'r')
  print(f.read())
  f.close()

未使用 with：此時都會使用 try-finally 的方式來保證有做到資源回收

.. code-block:: python
  :linenos:

  f = open('input.txt', 'r')
  try:
      # 可能是 f 的相關操作可能發生錯誤 (exception)，甚至是其他毫不相關的操作發生錯誤
      print(f.read())
  finally:
      f.close()

使用 with：可以簡單做到相容的效果，且更具可讀性

.. code-block:: python
  :linenos:

  with open('input.txt, 'r') as f:
      print(f.read())


with 關鍵字背後的運作方式與原理
================================

這邊將會將 with 的實際運作流程，改以 try-except-finally 的方式來呈現，已了解 with 實際的運作方式

p.s. 往後如果遇到一個情境不懂的用法是，可以自己試著換換看，以熟悉運作流程

.. note::
  如果還未了解 try-except-finally 的用法建議先理解後再回來繼續

進入區間時執行 ``__enter__``、離開區間時執行 ``__exit__``
---------------------------------------------------------

此段以常見的開檔讀取作為例子

使用 with 時
^^^^^^^^^^^^^

.. code-block:: python
  :linenos:

  with open('input.txt', 'r') as f:
      # Block start
      print(f.read())
      # Block end

改使用 try-except-finally 時
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
.. code-block:: python
  :linenos:

  import sys

  # with open('input.txt', 'r') as f:
  obj = open('input.txt', 'r')
  f = obj.__enter__()

  exc = True  # a flag to determine if process trap into the except block
  try:
      # Block start
      print(f.read())
      # Block end
  except:
      # go to here if there is an exception happened inside the block
      # set flag as False to avoid double-free
      exc = False
      # collect exception information
      exc_type, exc_val, exc_tb = sys.exit_info()

      if not obj.__exit__(exc_type, exc_val, exc_tb):
          raise
  finally:
      if exc:
          # in this way there is no exception
          exc_type = exc_val = exc_tb = None
          # f.close() is implemented in __exit__
          obj.__exit__(exc_type, exc_val, exc_tb)

``with`` 的特性與說明
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

藉由觀察與理解上面的例子，以及後續會陸續介少的例子，我們可以歸納以下幾點：

- with 後面接的物件，需要 **有實作 ``__enter__`` 和 ``__exit__``** 兩個函數
- ``__enter__`` 的 return 會 assign 給 ``with`` 敘述中的 ``as``，作為操作資源的媒介
  - 如果沒有接 ``as`` 的解法，也就視為不取得 ``__enter__`` 回傳的內容
  - 需要的如 assertLogs
  - 不需要的如 contextlib.suppress、contextlib.redirect_stdout
- 不管有沒有 exception 發生 **``__exit__`` 都會被執行**
  - 如果有 exception 發生，會略過 (不執行) 區間中 exception 發生點後的內容，並執行 ``__exit__``
  - 如果沒有 exception 發生，會在離開區間的時候執行 ``__exit__``
- 當有錯誤發生時，會收集當下的錯誤的原因與環境，並做為參數帶入 ``__exit__``，反之皆為 ``None``
  - ``exc_type`` 為發生錯誤的型態
  - ``exc_val`` 為發生錯誤時，raise 拋出來的 object
  - ``exc_tb`` 為發生錯誤當下 stack 的資訊，也就是平常發生錯誤時看到的 call trace 的前生
  - 如果也想獲取上面資訊，也可以使用 ``sys.exit_info()`` 的方式獲得
- ``__exit__`` 的回傳職為 True 的時候，表示 with 區間中發生的 exception 將會被 catch，不繼續向上拋出
  - ``contextlib.suppress`` 的例子

``__aenter__``、 ``__aexit__``
---------------------------------

分別為 ``__enter__``、 ``__exit__`` 的 async 版本

.. note::

  如不知道什麼是 async function，建議先了解後再回來繼續

.. _aenter:

``__aenter__``、 ``__aexit__`` 的例子
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

此例子使用 `asyncssh <https://asyncssh.readthedocs.io/en/latest/>`_ 的功能作為範例

此例子中，因為 `connect` 為一個需要維護的資源，需要正常的關閉連線，否則會讓 server 誤以為 client 還在而發生 leak

.. note::

  實際上網路相關的實作都會考慮到 client 不正常斷線的例外處理，因此都會有機制在時一段時間後回收 leak 的資源，但能做好的時候能第一時間的預防

因為 `coonect` 此行為是有需多的時間在等待連線的完成，適合使用 async 的方式實作，以減少浪費等待時間，而 asyncssh 正是使用此方式

但一般的 ``__enter__`` 並非 async function，因此無法在 function 中使用 await，因此改使用 async 版的 ``__aenter__`` 實作

.. code-block:: python
  :linenos:

  import asyncio, asyncssh, sys

  async def run_client():
      async with asyncssh.connect('localhost') as conn:
          result = await conn.run('ls abc', check=True)
          print(result.stdout, end='')

  try:
      asyncio.get_event_loop().run_until_complete(run_client())
  except (OSError, asyncssh.Error) as exc:
      sys.exit('SSH connection failed: ' + str(exc))

``run_client`` 的 function 中，使用 ssh 連練到 localhost，並在連線成功後執行 ``ls abc``，並印出結果

在離開 `async with` 的區間後關閉 ssh 的連線

.. note::

  其餘只是為了能在非 async function 下執行 async function 常見的方式

``async with`` 的特性與說明
^^^^^^^^^^^^^^^^^^^^^^^^^^^

經上述例子，可以在整理以下 async 版的特性：

- ``__aenter__``、 ``__aexit__`` 分別為 ``__enter__``、__exit__`` 的 async 版本
  - 當 ``__enter__`` 或 ``__exit__`` 的內容需要 ``await`` 時使用 async 版本
- ``__enter__`` 使用時使用 ``with``，``__aenter__`` 使用時使用 ``async with``
  - ``async with`` 需要在 async function 中執行

如何提供 ``with`` 可以使用的 API
=================================

推薦使用 `contextlib <https://docs.python.org/3/library/contextlib.html>`_，提供相關的工具，在此先介紹實作介面相關的功能

實作 ``__enter__``、 ``__exit__``
----------------------------------

依據前面介紹的原理，with 實際上就是在某些時間點執行 ``__enter__``、 ``__exit__`` 做到的，因此只需要實作即可

.. code-block:: python
  :linenos:

  class Context(contextlib.AbstractContextManager):
    # 如果是要使用 __aenter__、__aexit__ 的情況，改使用 AbstractAsyncContextManager
    def __enter__(self):
      # accquire something
      return self

    def __exit__(self,  exc_type, exc_val, exc_tb):
      # release something

  with Context() as c:    # 此處的 c 即為 __enter__ 中的 return
    ｃ.do_something()

.. note::

  雖然不加 contextlib.AbstractContextManager 也可以，只是有加的情況比較好的 pylint 之類的檢查暗示，比較能提供檢查

但觀察上面的使用方式，當有部分的的資訊在 ``__enter__``、 ``__exit__`` 要共用時，就會顯得很麻煩，因此推薦使用 contextlib 提供的 contextmanager、asynccontextmanager

使用 contextlib.contextmanager
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

依據上例 (AbstractContextManager) 中所使用的做法，改使用

.. code-block:: python
  :linenos:

  @contextlib.contextmanger
  def context():
    # 原預期在 __enter__ 會做的事

    # 此處 yield 的效果相當於 __enter__ 中的 return
    # 因此如果有需要提供外部介面時，在 yield 後傳出
    yield
    # 原預期在 __exit__ 會做的事

- 例子

.. code-block:: python
  :linenos:

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

- 優點: 相比原生的 `__enter__`, `__exit__` 更直覺
- 例子

.. code-block:: python
  :linenos:

  @contextlib.contextmanger
  def open_connect_by_config(host_name):
    host, user = get_host_info_by_config(host_name)
    with asyncssh.connect(host, username=user) as conn:
      yield conn

使用情境
===========

現在我們知道基本的運作方式了，以這樣的特性，整理有以下幾種常見的情境，適合使用 `with` (或 `async with`) 的情境，並舉幾個常見的例子

資源管理
----------

此區間中間才有資源的使用權，也因此在區間結束後，不管成功或是失敗，都需要釋放資源 (資源取得是常見例子)

開檔
^^^^^^^^^

如 :ref:`quickstart` 範例

開啟遠端連線
^^^^^^^^^^^^^^

如 :ref:`aenter`

pytest 中的  yield function 做的 fixture (已內建的 tmpdir 為例)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

使用 pytest fixture 中的 `tmpdir` 功能，可以為每個測試 function 提供獨立的 dir 做測試，並在測試結束後移除此 dir

.. code-block:: python
  :linenos:

  import pytest, os, pathlib

  def test_touch_file(tmpdir):
      pathlib.Path(os.path.join(tmpdir)).touch()

特化對於 Exception 的處理
-------------------------

unittest 中的 ``assertRaises``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

可以收集區間中是否有發生指定 raise

在寫 unittest 的時候，會需要測試錯誤的情況，是否有如預期拋出 exception，此時可以使用 unittest 的 assertRaises 來驗證區間中確實有拋出符合預期的 exception

.. code-block:: python
  :linenos:

  import unittest

  class TestAssert(unittest.TestCase):
      def test_asert_division_zero(self):
          with self.assertRaises(ZeroDivisionError):
              # 此區間中必須拋出 `ZerDivisionError` 的 exception，沒有拋出或拋出別的 exception 都視為測試失敗
              print(1/0)

.. note::

  pytest.raises 也是相同的道理

contextlib 中的 `suppress`
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

可以指定部分的 exception 被忽略

.. code-block:: python
  :linenos:

  with contextlib.suppress(ZeroDivisionError):
    # 因為有 suppress ZeroDivisionError 所以執行 1/0 將不會拋出 exception，或者說會被 with 接住
    print(1 / 0)

以此例子執行後將不會看到 ZeroDivisionError 的 exception，因為被 suppress 掉了

收集區間中的資訊
------------------

unittest 中的 `assertLogs`
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

可以收集 `logging` 所印出的 log

.. code-block:: python
  :linenos:

  import unittest

  class TestLogging
    def test_log_debug(self):
      with self.assertLogs('main', level='DEBUG') as cm:
        # 這裡面的 logging 如果是 main 的會被 filter 出來，並存在 cm.outputs 中
        logging.debug('this will be catched by cm')
      self.assertEqual(cm.outputs, ['this will be catched by cm'])

contextlib 中 redirect_stdout, redirect_stderr
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

可以將區間中的 stdout(也就是 print) 和 stderr，導到指定的 steam 中 (檔案也是種 stream 的概念)

.. code-block:: python
  :linenos:

  import contextlib

  with contextlib.redirect_stdout('output.txt'):
    print('this will be write to output.txt')

在區間前後的前處理與後處理 (hook 的概念)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

最簡單的用法就是在前後插 log，下面舉個前後印訊息的簡單範例

.. code-block:: python
  :linenos:

  import contextlib
  import logging

  @contextlib.contextmanger
  def log_cmd(cmd):
    logging.debug('Start cmd: %s', cmd)

    # 想要傳遞資訊給 with 的時候使用很方便
    yield

    logging.debug('Finish cmd %s:', cmd)

  def run_cmd(cmd):
    with log(cmd):
      # create_subprocess_shell 這個 function 可以執行如 shell 一般的指令
      ret = subprocessing.create_subprocess_shell(cmd)
      # 此處的 ret 無法方便的傳到 log，供 __exit__ 的流程使用

      return ret

但也如上方說明，如果想要印的還有執行的結果，也就需要將 ret 傳給 log 供後續的流程使用，但以此情況不容易做到，
因此以此情況，建議使用 decorator (會在其他篇介紹)，擅長的也不一樣

.. note::

  如果其中也有需要 with 的部分可以搭配使用
  TODO: link 到 wrap

參考資料
=============

- `Context Management in Python <https://stephlin.github.io/python/2021/01/02/context-manager-in-python.html>`_
