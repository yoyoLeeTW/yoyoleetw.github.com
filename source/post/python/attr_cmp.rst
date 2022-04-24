.. post:: Apr 24, 2022
  :tags: Python, attribute
  :category: Python
  :location: TW
  :language: zh
  :exclude:

  class descriptor v.s. value saved in object

============================================
Python 各種 attribute 取得方式的比較
============================================

前言
=======

:ref:`class descriptor <attr_class>`、:ref:`attribute hook <attr_hook>`
或是使用一般的 attribte 來存取，都是種取得 attribute 的方式，既然每種方式都有可以做到，當遇到都提供的時候，
實際上的運作方式會是什麼呢？也就是這篇要介紹給大家的。

一般的 attribute 行為
=======================

在理解交互作用前，需要先理解一般我們常用的 attribute 是如何運作的，才好在後面了解同時發生的狀況。

attribute 直接放在 object 裡的方式
-----------------------------------

一般最常見使用 attribute 的方式是直接在 object 存入，並直接取出使用，如下例子，原本 ``Dog`` 這個
class 裡面就沒有定義 ``_name``，而是在 ``__init__`` 裡面才直接以 ``self._name = name`` 的方式
，動態的在 object 裡建立一個 ``_name`` 的 attribute 並儲存為 ``name``。

這裡以底層精準來講是存在 object 中的 ``__dict__`` 裡，所以以 ``self._name = name`` 來說，
底層實際上是做 ``obj.__dict__['_name'] = name``，就變成了 dict 的方式，就可以很好的理解了。

同理 ``name = self._name`` 是取出 object 裡面 ``_name`` 的 attriute，底層的運作方式為
``name = self.__dict__['_name']``，所以也可以直接以 dict 的方式理解。

.. code-block:: python

  class Dog:
    def __init__(self, name):
      self._name = name

    def eat(self, food):
      name = self._name
      print(f'{name} eats {food}')

取得 class 的 attribute
-------------------------

我們知道可以在 class 宣告的時候就賦予值進 class，這樣之後即使拿到的是物件也可以取得，如下例子，
在說明之前，我們需要先理解一下 class 究竟把此值存在哪裡，以及怎麼存，以此例來說，實際的存放位置在
``Dog.__dict__['leg_num']`` 裡面，因此實際上的存取方式都和 object 的情況一樣，只是對象由 object
變成了 class 而已。

如果 class 是有繼承的話，會依 ``__mro__`` 的順序，依序去找，找到的話就回傳，就不會繼續往後找了，
如果都找不到，就跳 ``AttributeError``。

.. code-block:: python

  class Dog:
    leg_num = 4

  dog = Dog()
  print(dog.leg_num)  # 4

object 和 class 之間 attribute 關係
-------------------------------------------------

實際在取 object 的 attribute 時，會先嘗試從 object 裡面找，如果找不到的時候，才會在 class 裡面找，
因此以上面的例子，實際上 ``dog.leg_num`` 拿到的會是 4，而不會是跳 ``AttributeError``。

現在我們理解了取的狀況，換來看賦值的狀況，當執行 ``dog.leg_num = 3`` 的情況下，實際上就直接存到
object 上，也就是 ``dog.__dict__['leg_num'] = 3``，這樣才能符合針對 object 設定的時候，不影響到
class 的這個行為，因此如果確定是要改 class 的 attribute 時，請直接針對 class 本身修改，以上面例子來說
，是 ``Dog.leg_num = 3``，實際上也可以藉由 ``dog.__class__.leg_num = 3`` 達到一樣的效果。


一般 attribute 的存取方式，和 class descriptor 混用時
=========================================================

建議在閱讀後面的內容前，先讀過 :ref:`class descriptor <attr_class>`，會比較容易理解。
如果想快速了解行為就好，可以直接跳到「:ref:`取值時的檢查流程整理 <flow_summary>`」。

當 class descriptor 為 non-data descriptor 時
-----------------------------------------------

non-data descriptor 也就是只有實作 ``__get__`` 沒有實作 ``__set__`` 的 descriptor，
而其中 non-data 的原因就是因為無法 set，所以無法把此 descriptor 當作一個 data 看待
(data 需要不管存取都可以)。

但如果真的要硬做，還是可以對一個 non-data descriptor 的 attribute 做賦值的動作，這時候為了符合行為，
只能存到 object 中，且因為實際上有 set 的動作了，應該要拿到賦值之後的值，所以取值時，應該要取到
object 內的，而非 non-data descriptor 回傳。

當 class descriptor 為 data descriptor 時
-------------------------------------------

data descriptor 也就是有實作 ``__get__`` 和 ``__set__`` 的 descriptor。

因為此 descriptor 可以做到取和設，所以賦值的部分不能直接存到 object 中，因此取值的時候應該
data descriptor 會比 object 的取值優先權來得高。

.. _flow_summary:

取值時的檢查流程整理
====================

一般 attribute 的存取方式，和 class descriptor 混用時
-----------------------------------------------------

上面講了這麼多，可以簡單地總結 attribute 取值檢查的流程，以下取自
`官方教學 <https://docs.python.org/3/howto/descriptor.html#invocation-from-an-instance>`_ 。

.. code-block:: python

  def object_getattribute(obj, name):
    null = object()
    objtype = type(obj)
    cls_var = getattr(objtype, name, null)
    descr_get = getattr(type(cls_var), '__get__', null)
    if descr_get is not null:
      if (hasattr(type(cls_var), '__set__')
          or hasattr(type(cls_var), '__delete__')):
        return descr_get(cls_var, obj, objtype)       # data descriptor
    if hasattr(obj, '__dict__') and name in vars(obj):
      return vars(obj)[name]                          # instance variable
    if descr_get is not null:
      return descr_get(cls_var, obj, objtype)         # non-data descriptor
    if cls_var is not null:
      return cls_var                                  # class variable
    raise AttributeError(name)

再加上 attribte hook 混用時
------------------------------

內容在 :ref:`attribute hook <attr_hook>` 提過，但在了解
「一般 attribute 的存取方式，和 class descriptor 混用時」再來看一次會更清楚，實際上，上述的
``object_getattribute`` 也就是預設的 ``__getattribute__`` 行為，因此整體流程會事先看
``__getattribute__`` (如果沒有實作，就是用上面的 ``object_getattribute``) 有沒有找到，
如果沒有找到 (跳 ``AttributeError`` 表示沒有找到)，就看看有沒有 ``__getattr__``，
如果有就嘗試用 ``__getattr__`` 拿看看，沒有就直接跳 ``AttributeError``。

.. code-block:: python

  def getattr_hook(obj, name):
    try:
      return obj.__getattribute__(name)
    except AttributeError:
      if not hasattr(type(obj), '__getattr__'):
        raise
    return type(obj).__getattr__(obj, name)

參考
======

整篇擷取自 `官方文件 <https://docs.python.org/3/howto/descriptor.html>`_ 取其比較常用的部分，
如想了解更多可參考。
