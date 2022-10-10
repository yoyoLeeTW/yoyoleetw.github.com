.. post:: Oct 9, 2022
  :tags: Python, class
  :category: Python
  :location: TW
  :language: zh
  :exclude:

  class

.. _class:

========================================
用例子學 class
========================================

前言
======

此篇會用舉例的方式解釋 class 的用法，並多少說明中間的原理，聽不懂原理也沒關係，只要可以依據範例知道中間個關係，
並且以模仿的方式使用，基本上就可以處理大部分的狀況了，等已經熟練之後，如果有遇到特別的狀況，可以再回來尋找原理的部分，再來閱讀其他進階的文章，
希望以這樣的方式可以讓新手在剛接觸時可以少點銜接障礙。

class 有資訊整理的能力
========================

.. code-block:: python

  class Apple:
    color = 'red'
    shape = 'ball'

  print(Apple.color)  # red
  print(Apple.shape)  # ball

class 像是個 section 一樣，可以把相關的資訊整理在一起，之後要取得資訊時，
可以用 section 的名字取得其內容，比較好管理。
以此例子來說，`Apple` 這個 class 就像是個 section 一樣，
section 裡面放了 color 和 shape 兩個 attribute，attribute 可以使用 `Apple.color` 的方式取出其內容，甚至可以修改其內容。

但切記並非每個東西放到 class 裡都會是此行為，像是 function 就不是，後面會特別說明，詳細原因是因為 function 的 `__get__` 性質，
有興趣的可以看 :ref:`另一篇<attr_class>` 的教學。

.. code-block:: python

  class Fruit:
    class Apple:
      color = 'red'

    class Banana:
      color = 'yellow'

如同 section 可以多層一樣，class 也可以有多層，來做進一步的分類。
以此例子來說，就可以使用 `Fruit.Apple.color` 的方式來取得蘋果的顏色。

class 與 object 之間的關係
============================

而在 python 裡面，class 實際上也就是一個物件，所以上面看到一些 `Apple.color` 都是物件可以有的操作，
但是不覺得我要一個水果的行為，就需要寫一個 class 覺得很累嗎？因此我們可以把共通的地方抽象出來，這就是要用到 class 的特性。

class 與 instance 之間的關係
------------------------------

在認識這些之前，需要先理解 class 和 instance 之間的關係：
class 經歷實例化的過程 (也可以說是 new 的過程) 會得到一個 instance，此 instance 的行為是由 class 決定的，
所以可以想像，對於一個 instance 而言，其 class 就像是他的 spec。

舉例來說，可以定義出一個叫 Person 的 class，之後經過 new 的過程，可以得到一個叫 edward 的 instance，而此 edward instance 的行為皆由 Person 這個 class 決定，
如他會有自己的名字 (e.g. `edward.name`) 或是他可以有吃的行為 (e.g. `edward.eat(apple)`)，而其中的 `name` 和 `eat` 就是由 class 定義來的。

藉由上面的例子，會知道想要做出想要的 instance 行為，需要知道此行為的 class 要怎麼寫，因此後面會介紹一些對應的行為。

實例化
--------

.. code-block:: python

  class Person:
    def __init__(self):
      self.name = 'edward'

  a = Person()
  print(a.name) # edward

在前面提到 class 可以經過實例化 (或是稱 new) 得到一個 instance，在此例中，就是由 Person 這個 class 經過實例化得到 a 這個 instance，
由此可知在 Python 裡面，對一個 class 做 call 這個動作，就會得到對應的 instance，而 call 的這個行為，其實就是在後面加上 `()`，就像上例的 `Person()`

有參數的實例化
~~~~~~~~~~~~~~~~

在看過上一個例子後，一定會覺得每個 Person 都叫 edward 會顯得彈性有點低，因此應該要把一些客製化 (不屬於 class 要的 spec) 的部分獨立出來，並提供客製化的方式，
最常見的方式就是在建構的時候給予客製化的參數，如下例，就是把 Person 的名字使用參數的方式交給製作 instance 的時候客製，而這克制的行為，就會是每個 instance 不一樣的部分即是他們的 class 一樣。

.. note::

  仔細分析可以知道一個 Person 的物件有 `name` 這個 attribute 屬於 class 的行為，但內容叫什麼則是屬於 instance 客製化後的行為，這中間有些微的差異。

.. code-block:: python

  class Person:
    def __init__(self, name):
      self.name = name

  a = Person('edward')
  print(a.name) # edward

實例化的流程 - self 是什麼
~~~~~~~~~~~~~~~~~~~~~~~~~~~

實例化的過程有很多的細節，主要有兩個 `__new__`、`__init__`，`__new__` 比較複雜這邊就不詳細說明，只需要知道一般來說經過 `__new__` 的過程，會得到一個新的 instance，
而此 instance 會被當作 self 這個 call `__init__` 這個 function，因此可以當作一般狀況時，實例化的過程都會執行到 `__init__`，你可以在此 function 初步的對此 instance 設定，
就像上面的例子，就是在 `__init__` 裡面定義了此 instance 有 name 這個 `attribute`，且值為 `Person(name)` 裡面的 name，也就是 `'edward'`。

再由前面的例子以及前面的說明，可以知道 self 就是新的一個 instacne，用來放所有此 instance 相關的東西，就像是 `self.name`。
且 class 實例化後的 instance 會直接作為 return value 的方式提供給執行實例化 (也就是 call Class 名) 的人，在上例中也就是 `a` 的這個變數。
由這兩點，我們可以推論 self 也就是 `a`，也就因此為什麼後面執行 `a.name` 的時候會拿到 `'edward'` 了，就是因為在 `__init__` 的過程中有把 `self.name` 設為了 `'edward'` 的原因。

instance 如何使用 class 的東西
--------------------------------

以前面的例子，我們知道 instance 的建構流程是由 class 的 `__init__` 決定的，所以一些 instance 上的 attribute 也會間接的建立，但事實上 instance 會使用到 class 的部分可不止如此。
先由最簡單的一般變數來說，以下例來說，除了前面介紹設定於 instance 上的 `name` attribute 外，還在 class 裡設定了 `leg_num` 這個 attribute，因此當對 `leg_num` 這個 attribute 取得時，
會先找找 instance 裡面有沒有，沒有的話去找找他的 class 有沒有，因為在 class 裡面有找到 `leg_num = 2`，因此 `a.leg_num` 才會得到 2。

.. code-block:: python

  class Person:
    leg_num = 2
    def __init__(self, name):
      self.name = name

  a = Person('edward')
  print(a.name)     # edward
  print(a.leg_num)  # 2

.. note::

  讀取的時候會由 instance 一路往上找，但是寫的時候，就會直接寫到此 instance 上，所以如果此 Person 如果斷腿的時候，可以直接藉由 `a.leg_num = 1` 的方式修改，
  也不會讓其他 Person 也變成斷腿的，因為只改到了此 Person 的 instance 上，並沒有改到 class 上，如果真的想要每個 Person 都斷腿，就需要 `Person.leg_num` 這樣改。

  如果想瞭解更多細節，可參考 :ref:`此 <attr_cmp>` 。

def 定義出來的 function 用於 class 的特別之處
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

前面我們看到一般狀況上，instance 如何使用到 class 的性質了，但一般會寫 class 的時候，最常要共用的是 function 的行為，如一個 Person 可能會不同的名字，但是多數的行為都會是同樣的，
如都會需要有 `eat('Apple')` 之類的行為，現在知道對於 class 而言 function 佔了多重要的角色了，但在 python 裡面 function 的行為卻是最不值覺得，中間有太多 :ref:`原理 <desc_class>` 的細節，
這裡簡單用栗子的方式給各位這中間的感覺。

.. code-block:: python

  class Person:
    def __init__(self, name):
      self.name = name

    def eat(self, food):
      print(f'{self.name} eats {food}.')

  a = Person('edward')
  a.eat('apple')          # edward eats apple.
  Person.eat(a, 'apple')  # edward eats apple.

由上例可知，由 instance 執行的 class function 會自動把第一個參數帶入 instance 給 self，所以 `a.eat('apple')` 時，實際上 eat function 拿到的參數是 a 和 'apple'，
也就相當於是執行 `Person.eat(a, 'apple')`。

.. note::

  這之間會有這樣的原因，是因為用 def 宣告出來的 function 其 `__get__` 的能力，因此在 instance 上取得 class 的東西時，會經特別的處理，會把原本的 function 綁定 instance 為第一個參數，
  而此這種東西的型態稱為 bound method，也就是為什麼常見習慣都會稱 class 裡面的 function 為 method，method 就是已經有綁定 instance 的 function 的意思。

  也因此如果想要 instance 上取得的 function，第一個參數改為 cls 自己，則會用 classmethod 修飾，如果不會多傳第一個參數，則會用 staticmethod 修飾，實際上就是要改掉預設的行為。

常見的使用方式
===============

property
--------

在 class 中，`@property` 修飾後的 function，會變成取得此 attribute 時，實際是由執行此 function 來得到其 attribute 的，
以此例來說，`name` 這個 function 經 `@property` 修飾後，`a.name` 或是 `self.name` 的方式要取得 name 這個 attribute 時，實際上都是執行 name 這個 function 來取得，
因此才會拿到 `f'{self.first_name} {self.last_name}'` 的結果，也就是 `Edward Yan`。

一般遇到值可以由別的 instance 取得，或是想做到某個 attribute 不能被修改時，經常會使用此方式。

.. code-block:: python

  class Person:
    def __init__(self, first_name, last_name):
      self.first_name = first_name
      self.last_name = last_name

    @property
    def name(self):
      return f'{self.first_name} {self.last_name}'

  a = Person('Edward', 'Yan')
  print(a.name) # Edward Yan

classmethod
-----------

一般 class 中的 function，第一個參數會是 self，也就是拿到 instance 本人，但是如果想要拿到的是 class 而非 instance，就可以用 `@classmethod` 來修飾此 function，
如此例的 `get_apple`，這樣第一個參數就會改成拿到 `cls`，也就是 Fruit 這個 class，因此就可以使用 `cls('Apple')` 的方式初始。

常見的使用方式是提供一些常見實例化此 class 的 function，如此例就是認為實例化 Apple 常會用到，所以特別包出來。

.. code-block:: python

  class Fruit:
    def __init__(self, name):
      self.name = name

    @classmethod
    def get_apple(cls):
      return cls('Apple')

  a = Fruit.get_apple()
  print(a.name) # Apple

staticmethod
------------

如果只是想要 class 當作 section 看待，那 section 中想要放 function 就會被都塞第一個參數，而導致並非只是整理相關 function 了，這時候可以使用 `@staticmethod` 的方式修飾 function，
這樣此 function 即使在 class 中，也不會被偷帶第一個參數。如此例來說，只是想要整理各種 Sport 到一個 class 中，但並想要表明不需要先實例化 Sport 才能使用，就會使用 `@staticmethod` 修飾。

.. code-block:: python

  class Sport:
    @staticmethod
    def run(person):
      print(f'{person.name} runs.')

    @staticmethod
    def swim(person):
      print(f'{person.name} swims.')

  a = Person('edward')
  Sport.run(a)    # edward runs.
  Sport.swim(a)   # edward swims.
