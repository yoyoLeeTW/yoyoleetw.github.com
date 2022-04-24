.. post:: Mar 20, 2022
  :tags: Python, descriptor, attribute
  :category: Python
  :location: TW
  :language: zh
  :exclude:

  descriptor class

.. _attr_class:

============================================
Python 中客製 attribute - descriptor class
============================================

前言
=====

此篇要介紹的內容實務上不像 with 或是 decorator 之類的那麼常使用，除非是製作 framework 的時候，
但是實際上他卻無所不在，因此建議可以了解，有助於了解平常使用工具的運行方式，有助於之後釐清問題。

使用方法介紹
============

最常見，也是最入門使用到此方式的為 ``@property``，舉例來說如下，就可以做到一個變數 ``name`` 只能讀
，不能寫，如果對於 ``@`` 不懂的話，可以先看看 :ref:`Python 中用 @ 讓函數賦予特型 <decorator>`
了解一下再回來。

.. code-block:: python

  class Person:
    def __init__(self, name):
      self._name = name

    @property
    def name(self):
      return self._name

  wendy = Person('wendy')
  print(wendy.name) # wendy

因此如果想要執行如 ``wendy.name = 'andy'`` 的話就會噴以下的 Error：

.. code-block:: text

  Traceback (most recent call last):
  File "a.py", line 11, in <module>
    wendy.name = 'andy'
  AttributeError: can't set attribute

能做到這樣的效果，就是因為 ``property`` 把 ``name`` 這個 method 作為一個 getter method，並且
沒有額外提供 setter 的能力，Pythone 在底層時，因為找不到 set 的能力就報 Error 了。

看過上述 ``property`` 的使用方式後，相信聰明的朋友一定有注意到只要 method 的實作方式換，可以做到
更多客製化的 getter 行為，舉例來說以下例子就做到實際取得的直，與儲存的值中間需要個轉換的情況，也可以
經由客製 getter 的方式做到。

.. code-block:: python

  class Wallet:
    @porperty
    def total_money(self):
      return self.count_of_5_coin * 5 + self.count_of_1_coin

但對於一個錢包來說，不見得裡面的硬幣數量永遠都是固定的，還是有可能增減，但需要一些檢查，如數量總不會是負的，
因此可以如下方式提供 setter，並在其中檢察職符合預期的情況才給設定：

.. code-block:: python

  class Wallet:
    @porperty
    def count_of_5_coin(self):
      return self._count_of_5_coin

    @memey_in_US.setter
    def _count_of_5_coin_setter(self, count):
      if count < 0:
        raise ValueError(f'Count({count}) cannot be negative')
      self._count_of_5_coin = count

這邊看到 ``setter`` 實際上是針對 ``count_of_5_coin`` 再多賦予存值的能力，且要存的話就使用後面的函數
，且是使用 ``@`` 的方式註冊到原 ``count_of_5_coin`` 的 property 中。


Python 的解讀方式和實際的運作方式
=================================

object 的 attribute
-------------------

這邊使用取值的時候做舉例 (也就是用 ``__get__``)，同樣還有設值的時候 (也就是用 ``__set__``)，
還有刪除值的時候 (也就是 ``__delete__``)，他們的運作方式都一樣，只有呼叫時的參數給的不一樣。

以下以固定皆回傳 ``10`` 這種最簡單的 descriptor class 做舉例：

.. code-block:: python

  class Ten:
    def __get__(self, obj, objtype=None):
      return 10

  class A
    x = 5
    y = Ten()

  a = A()
  print(a.x)  # 5
  print(a.y)  # 10

當執行 ``a.x`` 的時候，會先看看 class 中定義的 ``x`` 實際上的性質，以此狀況來說，看到了純數字 
(非 descriptor 的 object)，所以會直接取出來使用，也就會拿到 5。

當執行 ``a.y`` 的時候，會先看看 class 中定義的 ``y`` 實際上的性質，以此狀況來說，看到了
descriptor object (有 ``__get__`` 此 magic method 的 object)，就會用以下的方式解讀：

.. code-block:: python

  A.__dict__['y'].__get__(a, A)

所以實際上會由 ``class A`` 的定義中取出經 ``Ten()`` 當時建構出來的 object，也就是
``A.__dict__['y']``，之後執行 ``__get__`` 這個 method之後，代入 object 本身 ``a`` 和其
class ``A`` 做為參數。

.. note::

  ``A.__dict__['y']`` 拿到的是 ``Ten`` 建構出來的 object，所以實際上 call ``__get__`` 時，
  拿到的 self 會是此 object，因為他是放在 ``class A`` 裡面，所以這個是所有 ``A`` 建構出來的
  object 共用的，如果要依據不同的 A object 存放的內容，請存在 ``obj`` 這個參數裡面。

class 的 attribute
------------------

一般我們也會認為 class 的 attribute 也就是當時 class 定義的時候放的值，但實際上不見得，和 object
的情況要考慮 descriptor object 一樣，class 的情況它也適用，方式也是一樣的，都是會執行 ``__get__``
來取得，唯一的差別在於 ``obj`` 這個參數拿到的會是 ``None``，因此為了讓 class 的行為會符合一般我們對於
class 的認知，通常會檢查如果 ``obj`` 為 ``None`` 就會回傳 ``self`` 來符合，如下。

.. code-block:: python

  class Ten:
    def __get__(self, obj, objtyp=None):
      if obj is None:
        return self
      return 10

  class A:
    x = 5
    y = Ten()

  print(type(A.y))  # Ten


取值、設值、刪除值的例子
=========================

取值、設值、刪除值實際上的運作方式都同上面的說明，這邊為加強印象以及理解 descriptor class 的 object
與 ``__get__`` 拿到的 object 是不同的對象。

.. code-block:: python

  import logging

  class LoggedAccess
    def __set_name__(self, owner, name):
      self._public_name = name
      self._private_name = f'_{name}'

    def __get__(self, obj, objtype):
      value = getattr(obj, self._private_name)
      logging.info('Accessing %r giving %r', self._public, value)
      return value

    def __set__(self, obj, value):
      logging.info('Updating %r to %r', self._public_name, value)
      setattr(obj, self._private_name, value)

    def __delete__(self, obj)
      logging.info('Deleting %r', self._public_name)
      delattr(obj, self._private_name)

  class Person:
    name = LoggedAccess()
    age = LoggedAccess()

  wendy = Person()
  wendy.name = 'wendy'
  # Updating name to wendy
  wendy.name
  # Accessing name giving wendy
  del wendy.name
  # Deleting name

上面的例子中，除了 ``__get__``、``__set__``、``__delete__`` 是上面介紹過的，還有
``__set_name__`` 有特別的用法，他是在 descriptor class 被建構出來，且賦予到一個變數時，
會被呼叫的一個 hook，以此例子來說，也就是在 ``name = LoggedAccess()`` 的時候，且 owner
會拿到 ``Person`` 這個 class，name 會拿到 ``name`` 這個 string，也就可以得到之後使用的名字。

常見使用 descriptor class 的工具
=================================

property
----------
也是一般最容易會接觸到的方式，方法與上面介紹的差不多，``funtools.cached_property`` 也和此有一樣的
使用方式，唯一的差別在於他不會把前一次取得的值先存起來，就不需要一直重複計算。

而如果對於底層的實作方式有興趣，或是想知道為什麼可以有 ``.setter`` 動態加入 setter 的方式，之類的問題
，建議直接看 `官方的說明 <https://docs.python.org/3/howto/descriptor.html#properties>`_ 。


method
---------

其實在 Python 之中，function 和 method 有些許的不一樣，method 和 function 的差別在於 method
有多綁定所屬的 class，因此 class 中的 method 的第一個參數才會統一是 self，也就是拿到 object 自身。

為什麼有這樣的效果呢？原因在於 ``def`` 定義出來的 function，實際上有實作 ``__get__``，只是我們都
不知道而已，也因為此 ``__get__`` 的效果，所以經由 ``.`` 取得的 method 實際上是一個 callable 的
attribute，而這時，實際上已經經 ``__get__`` 綁定了 object 進去，變成了 method，所以之後執行的時候
，才會在第一個參數拿到 self。

.. code-block:: python

  class MethodType:

    def __init__(self, func, obj):
      self.__func__ = func
      self.__self__ = obj

    def __call__(self, *args, **kwargs):
      func = self.__func__
      obj = self.__self__
      return func(obj, *args, **kwargs)

  class Function:
    def __get__(self, obj, objtype=None):
      if obj is None:
        return self
      return MethodType(self, obj)

取自 `官方教學 <https://docs.python.org/3/howto/descriptor.html#functions-and-methods>`_ ，
這邊的 ``Function`` 是模仿 ``def`` 的行為寫出來的，實際上的行為定義在 Python 的原始碼之中，
但我們可以經由上面的程式碼理解實際的運作方式。

接下來我們舉個簡單的例子一起看一下實際的運作方式：

.. code-block:: python

  class Dog:
    def eat(self, food):
      pass

  dog = Dog()
  dog.eat('meat')

同上說明，因為 ``eat`` 這個 function (且他有實作 ``__get__``)，被放到了 ``class Dog`` 裡面，
所以經 ``dog.eat`` 拿到的會是 ``MethodType`` 的 object (``MethodType`` 的建構子可以知道綁定
了當時經 ``.`` 操作的 object，也就是 ``dog``)，且因為 ``MethodType`` 有 ``__call__`` 所以他是
callable 的，也就可以經由 ``()`` 的方式帶入要執行的參數，並執行 ``__call__``，這時實際代進 function
的是 ``func(obj, *args, **kwargs)``，因此 method 拿到的第一個參數才會是 self，也就是 object 本身。

類似行為的還有 ``classmethod``、``staticmethod``、``abstractmethod``，他們只是差在經由 ``@``
的方式把 function 中的 ``__get__`` 換成別的。

這邊簡單用 ``classmethod`` 來舉例
(擷取自 `官方範例 <https://docs.python.org/3/howto/descriptor.html#class-methods>`_)。

.. code-block:: python

  class ClassMethod:
    def __init__(self, f):
      self.f = f

    def __get__(self, obj, cls=None):
      if cls is None:
          cls = type(obj)
      if hasattr(type(self.f), '__get__'):
          return self.f.__get__(cls, cls)
      return MethodType(self.f, cls)

這邊可以注意到差別在於建構出 ``MethodType`` 時的參數不同，``Function`` 用的是
``MethodType(self, obj)``，而 ``ClassMethod`` 用的是 ``MethodType(self.f, cls)``，
第一個參數同樣是 function 本身，但第二個參數就由 object 本身改成了 object 的 class 了。


提供的介面有專門為 method 設計的情況
------------------------------------

其實一般來說只要是放到 class 中使用的，基本上都會使用 descriptor class，以下舉幾個例子：

functools.partialmethod
~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: python

  class Cell(object):
    def __init__(self):
      self._alive = False
    @property
    def alive(self):
      return self._alive
    def set_state(self, state):
      self._alive = bool(state)
    set_alive = partialmethod(set_state, True)
    set_dead = partialmethod(set_state, False)

  c = Cell()
  c.alive # False
  c.set_alive()
  c.alive # True

以上例子取自
`官方 <https://docs.python.org/zh-tw/3.7/library/functools.html#functools.partialmethod>`_。

一般的情況會使用 `functools.partial <https://docs.python.org/zh-tw/3.7/library/functools.html#functools.partial>`_
，但當使用的 function 放在 class 裡面供 class 用的時候，請要用 method 的版本，為什麼會有這樣的差別
就在於 descriptor class 的部分。

.. note::

  類似狀況的還有
  `functools.singledispatchmethod <https://docs.python.org/3/library/functools.html#functools.singledispatchmethod>`_。


參考
======

- `此篇 <https://openhome.cc/Gossip/Python/Descriptor.html>`_ 針對 ``__get__``
  有簡單的入門，對於原理介紹舉例的不錯。
- 整篇 `官方文件 <https://docs.python.org/3/howto/descriptor.html>`_ 取其比較常用的部分，
  如想了解更多可參考。
