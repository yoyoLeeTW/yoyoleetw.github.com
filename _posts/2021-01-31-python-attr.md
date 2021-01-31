---
title: Python 中的 attribute
tags: python, attribute, __get__, __set__, __del__, __getattribute__, __getattr__, __setattr__, descriptor, bound method
pageview: true
---

## Abstract

此篇將會介紹 python 是如何使用動態的方式使用，並做到其他語言的效果

在了解實作方式後，甚至可以利用其特性動態的做出想要的 class，而不用像編譯語言使用複雜的 marco 否則會需要寫出很多內容上差不多的結構

> 實際上也可以藉由物件的繼承盡可能地減少重複，但 python 可以不需設計到如此複雜

## 快速入門

依循 python 的語法撰寫，可以做到像其他語言的使用方式 (建構子) new 出 object 操作

而 python 動態語言的特性也可以動態的新增屬性

``` python
class A:
    def __init__(self):
        self.attr1 = 1

# __init__ 會幫我們初始 object拿到的內容，new object 的職後都會 call
# __init__ 中看到的 self 實際上等於 new 出來的 object (如下利的 a)
a = A()
print(a.attr1)
# 1

# 如果想要動態的加入新的 attr 也可以
a.attr2 = 2
print(a.attr2)
# 2

```

同樣也可以做到繼承的效果，使用的概念也和其他語言大同小異

``` python
class A:
    # 可以設定一些變數給各個 A new 出來的 object 共用
    # >>> a1 = A()
    # >>> a2 = A()
    # >>> print(a1.a_attr_in_cls)
    # 1
    # >>> print(a2.a_attr_in_cls)
    # 1

    # 且在單一物件上修改不影響其他 object
    # >>> a1.a_attr_in_cls = 2
    # >>> print(a1.a_attr_in_cls)
    # 2
    # >>> print(a2.a_attr_in_cls)
    # 1

    a_attr_in_cls = 1
    def __init__(self):
        # 可以設定一些變數只給 A new 出來的 object 使用
        # >>> a = A()
        # >>> print(a.a_attr_in_obj)
        # 2

        # 且可以藉由 object 存取
        # >>> a.a_attr_in_obj = 3
        # >>> print(a.a_attr_in_obj)
        # 3

        self.a_attr_in_obj = 2

# 可以從 parent class 那邊繼承到 class 中的屬性 (e.g. a_attr_in_cls)
# >>> b = B()
# >>> print(a.a_attr_in_cls)
# 1

class B(A):
    b_attr_in_cls = 3
    def __init__(self):
        # 同理，可以由 parent 的 __init__ 設定 object 的變數 (e.g. a_attr_in_obj)，原因請看 case 1, 2
        # case 1: before call __init__
        # >>> b = B()
        # >>> print(b.a_attr__in_obj)
        # AttributeError: 'B' object has no attribute 'a_attr_in_obj'
        super().__init__()

        # case 2: after call __init__
        # >>> b = B()
        # >>> print(b.a_attr__in_obj)
        # 2
        self.b_attr_in_obj = 4
```

## 原理

### 原理 Abstract

在 Python 中，基本上東西都是物件 (連 class 也不例外)，
而物件存放東西的方式基本上就是以 `dict` 的方式存放，get 或 set 的時候再依當時的關係作用於特定 object 上

因此會一以下順序介紹，先了解基本的概念，再把複雜的能力摻進來看

- object 和 class 如何存取一般的變數 (e.g. int)
- object 與 class 間的互動
- 繼承實際上是在指定的 class object 中找是否有符合的 attribute 名
- 使用 `__getattr__`、`__setattr__` 做到函數式的取得與儲存
- 使用 `__getattribute__` 強制覆蓋舊有機制
- desc object 可以做出特化的 get, set 流程
- class 中的 function 實際上被 desc 包裝過
- 如同時擁有多個設定 (上述的各種方式) 符合，使用的優先權
  - get 有 `__getattribute__`、desc class、`__get__`
  - set 有 desc class、`__set__`
  - del 有 desc class、`__del__`

### object 實際上存放的方式

object 有兩種方式：

- 預設下，使用 `__dict__`
- 在其 class 設定 `__slots__` 改變其存放方式

class 只有使用 `__dict__` 的方式

以下使用 object 的角度來看，後續會介紹 class 如何對應到 object 上來看

#### 預設下，使用 `__dict__`

``` python
class A:
    def __init__(self):
        # >>> print(self.__dict__)
        # {}
        self.init_attr = 1

a = A()

print(a.__dict__)
# {'init_attr': 1}

a.new_attr = 2
print(a.__dict__)
# {'init_attr': 1, 'new_attr', 2}

# 沒有此 attribute 時為 AttributeError 和 dict 找不到 key 時的 KeyError 不一樣
print(a.non_attr)
# AttributeError: 'A' object has no attribute 'non_attr'
```

##### `__dict__` 特性與說明

- object 的 attribute 實際上以 dict 的方式放在 `__dict__` 裡面
  - `__dict__` 的構成為：key 為 string 型態的 attibute name；value 為其存放的值
  - `__init__` 中的 self 與 new 出來的 object 實際為同個東西
- 找不到此 attribute 時為 AttributeError

#### class 設定 `__slots__` 時

``` python
class B:
    # 表示使用 B new 出來的 object 只能有 list 中的 attribute
    # 沒有定義的時候，表示 new 出來的 object 使用 __dict__ 記錄其 attribute
    __slots__ = ['a']

b = B()

# 沒有 assign 前，如果取得會有 AttributeError
print(b.a)
# AttributeError: a

# assign 後
b.a = 1
print(b.a)
# 1

b.c = 1
# AttributeError: 'B' object has no attribute 'c'
print(b.c)
# AttributeError: 'B' object has no attribute 'c'

class C(B):
    pass

# __slots__ 的繼承 tree 中有任一個 class 未定義使用 __slots__ 則會失去效果，退回使用 __dict__ 的模式
c = C()
print(c.__dict__)
# {}
```

> 眼利的可以注意到 `__dict__` 和 `__slots__` 在找不到 attribute 時，雖同樣為 AttributeError，
> 但為什麼 `__dict__` 的方式會找到 class A 去了，後續 class 的部分將會揭曉

##### `__slots__` 特性與說明

- `__slots__` 是定義在 class 中的 str list，表示此 class new 出來的 object 只能有 list 中定義的 attribute
  - parent class 中定義的會繼承
- class 的繼承 tree 中，如有任一個 class 未設定 `__slots__` 將會失去其效果，new 出來的 object 退回使用 `__dict__`
- 使用 `__slots__` 的情況與優缺
  - 優點：`__slots__` 使用 list 的方式搜尋 attribute，原生的使用 dict 的方式，因此搜尋上的效率較好
  - 缺點：不能動態新增 attribute
  - 使用情境：attribute 少，需要高效能

#### object 與 class 不同方式 attribute 的方式

一般我們都是以此方式來設定 class 的 attribute 的

``` python
class A:
    a = 1
    def func(self):
        pass
```

如果強迫使用和 object 一樣的方式，則會變成下面這樣

``` python
class A:
    pass

A.a = 1

def func(self):
    pass
A.func = func
```

### object 與 class 之間的互動

#### 取得變數 (get)

object 在取 attribute 時，是有個搜尋順序的，沒有找到就換下一個，最後都找不到就 raise AttributeError：

1.找 object 中的 `__dict__`
2. 找 class 的 `__dict__`
3. 依繼承的關係往 parent class 的 `__dict__` 找

``` python
class A:
    a = 1

class B(A):
    b = 2
    def __init__(self):
        self.c = 3

b = B()

# 1. 現在來找 attribute: c
# 經由前面的經驗，看 b.__dict__ 中有找到，就直接取得
# >>> print(b.__dict__)
# {'c': 3}
print(b.c)
# 3

# 2. 現在來找 attribute: b
# b.__dict__ 中找不到，但 B.__dict__ 中有找到
# >>> print(B.__dict__)
# {'b': 2}
print(b.c)
# 2

# 3. 現在來找 attribute: a
# b.__dict__, B.__dict__ 都找不到，但 A.__dict__ 中有找到
# >>> print(A.__dict__)
# {'a': 1}
print(b.a)
# 1
```

> 也因為此特性，在 python 中，只要 attribute 或是 function 的名字與 parent 相同時，都是 override parent 的

#### 設定變數 (set)

set attribute 皆是存到 `object.__dict__` 中，搭配 get 的特性可以做出以下效果

> 也藉此熟悉一下 python 的邏輯

``` python
class A:
    a = 1

a1 = A()
a2 = A()
print(a1.a)
# 1

# class 的 attribute 不會因為其中一個 object 被改掉而影響其他 object
a1.a = 2
print(a1.a)
# 2
print(a2.a)
# 1
```

``` python
class A:
    def __init__(self):
        self.x = 1
        self.y = 3

class B(A):
    x = 2
    def __init__(self):
        super().__init__()
        self.y = 4

b = B()

# 因為在 A 的 __init__ 中以放到 self 中，所以 object 裡面會有
print(b.x)
# 1

# 因為是先執行 A 的 __init__ 才做 self.y = 4，所以被換掉了
print(b.y)
# 4
```

## 工具

- property
- staticmethod
- classmethod
- functools.cached_property

## 參考資料

- [官方](https://docs.python.org/3/howto/descriptor.html)
- [__slots__ 教學](https://openhome.cc/Gossip/Python/Slots.html)
- [get, set, del 時，個種 attribute 的處理順序](https://openhome.cc/Gossip/Python/AttrAttribute.html)
