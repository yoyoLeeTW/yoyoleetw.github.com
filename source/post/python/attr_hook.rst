.. post:: Apr 23, 2022
  :tags: Python, attribute
  :category: Python
  :location: TW
  :language: zh
  :exclude:

  attribute hook

.. _attr_hook:

=========================================================
Python 中為定義的 attribute 統一操作 -- attriubte hook
=========================================================

前言
=======

我們都知道如果想要讓一個 class 有某個 attribute，只需要 assign 需要的變數名即可，如 ``a.x = 1``，
就可以讓 a 這個 object 有 x 這個 attribute，且為 1，再進階一點可以使用
:ref:`class descriptor <attr_class>` 的方式客製化，但如果想要處理的並非固定的 attribute 名
而已呢？這時候就需要今天要介紹的功能來實作了。

快速入門
===========

如果想要有個可以使用 attribute 的存取方式來使用的 dict，就很適合使用這種方式來實作：

.. code-block:: python

  class AttrDict(dict):
    def __getattr__(self, key):
      try:
        return self.__getitem__(key)
      except KeyError:
        raise AttributeError(key)

    def __setattr__(self, key, value):
      return self.__setitem__(key, value)

    def __dir__(self):
      names = dir({})
      names.extend(self.keys())
      return names

  obj = AttrDict()
  obj.x = 1
  assert obj.x == obj['x']

.. note::

  雖然實際上也有 `套件 <https://pypi.org/project/attrdict/>`_ 提供，但目前看下來許多的
  open source 需要的時候也都是自己實作，實際上也是沒有什麼難度。


我們先從取得的方式看起，Python 裡面提供了 ``__getattr__`` 的方式可以客製化，用法上是如果使用
``obj.x`` 來取得 ``x`` 這個 attribute，因為沒有 ``x`` 這個 attribute 在原本 class 的定義裡
(先知道有此性質就好，後面會詳細解釋)，這時候就會交給 ``__getattr__`` 這個 magic method 來處理，
其中的 ``key`` 拿到的就會是 attribute 的名字，在此例中也就是 ``x``。

在理解了 ``__getattr__`` 的用法後，我們再回到原本想實作可以使用 attribute 取得的 dict 吧，
經由上面的說明可以知道要提供 class 中多個值的 attribute，可以使用 ``__getattr__`` 的方式來提供，
且因為 ``AttrDict`` 繼承了 dict，因此只需要在 ``__getattr__`` 裡面使用 dict 的方式取得需要的
key 就可以了，舉例來說，想要取得 ``obj.x`` 時，會執行 ``__getattr__`` 的 method，且 key 帶入
``x``，而只需要轉拿 ``obj['x']`` 就可以了，這邊有個要注意的是，如果沒有此 attribute 時，會有
``AttributeError``，和 dict 裡面的 key 沒有時，跳的 ``KeyError`` 不一樣，因此這邊需要特別針
對此狀況處理，所以才會使用 try-except 的方式接 ``KeyError`` 改跳 ``AttributeError``。

再來換賦值，Python 裡面提供了 ``__setattr__`` 的方式可以客製化，用法上與 ``__getattr__`` 差不多，
差別在於是 ``obj.x = 1`` 之類的賦值行為時會執行，其中 ``key`` 會是 ``x``，``value`` 會是 ``1``。


例子中還有看到實作了 ``__dir__`` 這個 method，原因在於 ``dir()`` 這個 function 會取得一個 object
有多少 attribute 可以提供，底層也就是去執行此 object 的 ``__dir__``，因此在 ``AttrDict`` 中，所
有 item 的內容也屬於 attribute，因此在符合 ``dir()`` 介面的需求下，實際上還要提供 item 的 key。

``__getattr__`` v.s. ``__getattribute__``
=========================================

相信第一此在查的時候很容易查到 ``__getattr__`` 和 ``__getattribute__`` 兩種，且不容易分出
兩者的差別，因此這裡直接引用 `官方的說明 <https://docs.python.org/3/howto/descriptor.html>`_
，相信可以又更好地理解。

.. code-block:: python

  def getattr_hook(obj, name):
    try:
      return obj.__getattribute__(name)
    except AttributeError:
      if not hasattr(type(obj), '__getattr__'):
        raise
    return type(obj).__getattr__(obj, name)

此 ``getattr_hook`` 可以理解為 ``obj.x`` 取得時，會直接轉成執行 ``getattr_hook(obj, 'x')``，
因此可以看出來， ``__getattr__`` 算是如果 ``__getattribute__`` 處理到的話，又發現有
``__getattr__`` 可以處理，就會嘗試看看。

由此可知，不管是什麼 attribute 的取得都會經 ``__getattribute__``，其中連 class method 的取得都是，
因此在使用 ``__getattribute__`` 要小心使用，如果使用不當，可能會連 class method 這種預期照
原本行為取得即可的操作都會被覆蓋掉，因此建議一般來說 ``__getattr__`` 就可以處理的情況，不會使用到
``__getattribute__``。

.. note::

  實際上原本的 ``__getattribute__`` 會做很多事，詳細下一篇會介紹 ``__getattribute__`` 的實際行為。
