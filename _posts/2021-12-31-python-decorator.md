---
title: Python 中用 @ 讓函數賦予新特性
tags: python, decorator, wrap
---

## 摘要

## 使用方法與原理介紹
### 快速入門
這邊以一個簡單的例子來帶入，如果想要紀錄 function 執行的所花時間，此時最常見的作法可能是
``` python
import time
import logging

def do_something():
	start = time.perf_counter()

	time.sleep(1)	# do something e.g. just sleep

	end = time.perf_counter()

	logging.info('do_something ran in %d', end - start)

do_something()
```

這樣就可以得知 `do_something` 這個 function 花了多久的時間執行，但這只限於如果要計算所花時間的 function 很少的情況下，當數量亦多起來相信多數人都會覺得很困擾。
且更嚴重的問題而是計時這件事與 `do_something` 原本要做的事一點關係都沒有，卻在他的 function 裏面寫了完全無關的邏輯，這會對程式碼的維護造成負擔。
因此會推薦使用 decorator 的方式把計時這個功能額外抽出來，在 python 裏面可以使用 `@` 的關鍵字做到比較漂亮的呈現。

``` python
import functools

def logging_time(original_func):
	@functools.wraps(original_func)	# 後面會繼續介紹
	def decoratored_func(*args, **kwargs):
		start = time.perf_counter()

		ret = original_func(*args, **kwargs)

		end = time.perf_counter()

		logging.info('%s ran in %d', original_func.__name__, end - start)
		return ret

	return decoratored_func

@logging_time
def do_something():
	time.sleep(1)

do_something()
```

`logging_time` 做到的效果是把 `do_something` 這個 function 換成 `decoratored_func`，只是名字還是叫 `do_something`，且 `decoratored_func` 除了想要多做的事之外，還是有做到 `do_something` 要做的事，所以對 `do_something` 的原訴求還是有被達成。

### @ 實際上幫我們做的事

`@` 實際上也只是個語法糖，只是換一個方式寫而已，且感覺上更有將 func 修飾的感覺，以上述的例子，實際上的運作方式如下：
``` python
def do_something():
	time.sleep(1)

new_func = logging_time(do_something)
do_something = new_func	# 把原本的 do_something 換成了 logging_time return 回來的 function，而非原本的
```

### 使 decorator 的修飾更完整 (使用 library)

為什麼說更完整呢？要回答這個問題需要先了解實際上「function 不只代表著要執行的內容，還包含了 function 的名字和說明」，而上面看到 `functools.wraps(original_func)`，實際就是幫我們把這類 function 的屬性給與新的 function。

以上述的例子來說，就是拿 `original_func` 的 function 資訊，給 `decoratored_func` 這個 function，因此達到的效果如下，才不會因為 decorator 後遺失了原本該有的資訊。

舉例來說原 function 如下：
``` python
def work():
	'''do something'''
	time.sleep(1)

print(work.__name__)	# work
print(work.__doc__)	# do something
```

有正常修飾的話，結果如下：
``` python
@logging_time
def work():
	'''do something'''
	time.sleep(1)

print(work.__name__)	# work
print(work.__doc__)	# do something
```

如果 `logging_time` 中的 `decoratored_func` 沒有經 `functools.wraps` 修飾，結果會如下：
``` python
@logging_time	# without using functools.wraps on decoratored_func
def work():
	'''do something'''
	time.sleep(1)

print(work.__name__)	# decoratored_func
print(work.__doc__)	# None
```

### decorator 給參數
上面的例子是每次都使用 logging 的方式印出來，但如果想在修飾的時候給定不同的 logger 的話目前的做法就會做不到，會需要可以有參數代入 logger，實際上可以由以下的方式做到。

``` python
def logger_time(logger):
	def logging_time(original_func):
		@functools.wraps(original_func)
		def decoratored_func(*args, **kwargs):
			start = time.perf_counter()

			ret = original_func(*args, **kwargs)

			end = time.perf_counter()

			logger.info('%s ran in %d', original_func.__name__, end - start)
			return ret

		return decoratored_func

	return logging_time

my_logger = logging.getLogger('my')
@logger_time(my_logger)
def do_something():
	time.sleep(1)
```

由上面的例子可以看出來，基本上 `logging_time` 這個 function 內容基本上都一樣，只有 call `info` 的時候改用 logger，而此就是 call `logger_time` 時代進來的參數，所以同樣的來看 `@` 拿掉的情況下，實際的執行流程：

``` python
def do_something():
	time.sleep(1)

logging_time = logger_time(my_logger)
do_something = logging_time(do_something)
```

所以可以簡單總結一下，拿參數的 function (這裡是 `logger_time`) 會先被執行，之後會得到一個用來修飾 function 的 function (這裡是 `logging_time`)，而原本的目標 function (這裡是純粹只會 sleep 的 do_something)，再被修飾之後得到一個新的被修飾的 function(這裡是會記錄執行多久的 do_something)。

## 用法舉例
在知道了基本的概念與運作方式之後，再來看看一些別人怎麼使用的例子，藉此來熟悉用法與更了解可以做到什麼樣的效果。

### 賦予 function 不同面向的能力

上面的舉例的 logging_time 就是這種類型，另外有名的還有 functools.cache, functools.lru_cache，這裡再說明一下 functools.cache 的功能，再多看看這類的功能：
``` python
@cache
def factorial(n):
    return n * factorial(n-1) if n else 1

factorial(10)	# 3628800
factorial(5)	# 120
```

費波納西數列，如果使用遞迴的方式寫會花非常多的時間重複計算相同的值，所以實際上要使用動態規劃的方式解，也就是需要把之前算過得值快取下來，供之後需要的時候不用重複計算，而 `cache` 的 decorator 就是讓此 function 賦予相同的 key 進來會直接拿之前算過得快取使用，就可以用非常簡單的方式就做到快取的效果。
類似的東西還有 contextlib.contextmanager, pytest.mark.timeout，有興趣可以去看看相關的用法。
> 可以參考「Python 使用 with 讓區間賦予特性」看看 contextlib.contextmanager 的部分。

### 修改 function 的 `__get__` 行為
最常見的就是 `property` 詳細的原理牽扯到 `__get__` 的效果，會在之後的文章在進行詳細說明，但實際上要達到的效果是讓改變一個 function 在 class 中的行為，舉例來說：

``` python
class Dog:
	def __init__(self, name):
		self._name = name

	def get_name(self):
		return self._name

	@property
	def name(self):
		return self._name

dog = Dog('wendy')
assert dog.name == dog.get_name()	# wendy
```

從例子可以看出來 `.name` 的時候不需要像 `get_name` 一樣需要有 call (也就是 `()`) 這個操作，就會直接得到 return 的值，這邊就是因為 `property` 改變了 name 這個 function 的行為。

類似的東西還有 staticmethod, classmethod, abc.abstractmethod, functools.cached_property，有興趣可以去看看相關的用法。

### 修飾 class
了解 decorator 的中心思想後，會知道實際上做到的就是把某個 object 換成另一個行為類似的，因此不只是 function 可以被修改，class 也是可以做到類似的效果。而 dataclasses.dataclass 就是很常見且好用的例子，下面來看一下使用方式：
``` python
import dataclasses

@dataclasses.dataclass(frozen=True)
class Coordinate
	x: int
	y: int

point_a = Coordinate(1, 2)
```

經由 `dataclass` 就可以有很簡單的方式提供一個專門放資料的 class，以上述的例子，就自動的實作 `__init__`，並把相關的資料記下來，且 frozen 表示一旦這個物件被初始過後，就不能再被修改 (也就是執行 `point_a = 2` 之類的行為會發生 exception)。

類似的東西還有 functools.total_ordering，有興趣可以去看看相關的用法。

> 這裡可以先當使用 class 的方式定義出一個 data 物件有哪些性質 (e.g. x, y)，之後只是把這樣的介面提供給 dataclasses.dataclass 這樣的介面當作參數，之後 dataclass 就會做出一個全新的 class 賦予了上述的功能，不需要講究中間怎麼做到。

### 註冊到其他管理介面
#### flask
flask 是一個 python 寫網頁的 framework，他使用 app.route 的方式去註冊給 framework 說如果有個 client 訪問 `http://ip/` 都要經下面的函數處理，把最後函數 return 的結果作為網頁回傳給 client，所以以下面的例子來說，client 就會看到一個顯示 hello world 的網頁。
``` python
@app.route('/')
def index():
	return 'hello world'
```

#### pytest
pytest 為 python 的 test framework，而其中 fixture (`@pytest.fixture`) 的功能非常好用  可以想像成 setup 和 teardown 的包裝，進而做出客製的 setup 和 teardown 的功能，再註冊給 framework，之後就可以使用此 fixture 做測試，細節有點複雜，直接看個例子更有感覺：

``` python
@pytest.fixture
def db():
	db = DB()
	# connect to DB
	try:
		yield db
	finally:
		# close db

def test_something(db):
	# query some db data by db object

```
在一堆測試的項目中，可能會有大量的測試需要基於 Database 的內容，或是行為做測試，但是每次連線到 database 可能都會需要固定的流程，且不管成功或失敗都應該正確地把 database 關閉，因此可以使用類似 `@contextlib.contextmanager` 的方式寫一個資源取得與釋放的 context manager，並將 `db` 這個名字註冊到 framework，之後要測試的 function 只要使用 `db` 做為參數，framework 就會在測試前先做 `db` 裡面 yield 前的操作，作為 setup，測試結束後做 `db` 裏面 yield 後的的操作，作為 teardown。

#### 其他官方提供的 fixture
經由上面的例子，應該可以想像其實很多功能都需要類似 db 的處理，都需要有環境的準備，測試玩恢復環境等功能，最直覺的就是需要暫存的檔案，測試完後會自動的回收，官方就提供了非常多類似的功能可以選擇，如 `testdir`、`tmpdir`，想知道更多好官方提供的 fixrue 請直接前往官方的[說明文件](https://docs.pytest.org/en/latest/reference/fixtures.html#built-in-fixtures)。

看過上述的說明後，是不是覺得 fixture 很適合因應不同的功能做出不同的 fixture 呢？如果各種功能都有 library 可以用其不是超級方便的，舉例來說 docker 相關的操作也很適合，這樣就可以使用 docker 的能力來做測試了，官方稱此為 plugin，官方已經整理出一些好用的 [plugin](https://docs.pytest.org/en/latest/reference/plugin_list.html)，之後有類似需求的時候先別急著自己寫，可以先上去看看說不定早有人提供了，即使上面沒有也可以先自己找找看，可能只是官方沒有收錄而已。

#### click.command()

click 為用 python 要做 command line 工具非常有名的 framework，可以把 function 註冊給 framework 說某個 command 的時候執行，讓我們直接由[官方的例子](https://click.palletsprojects.com/en/8.0.x/)了解：

``` python
import click

@click.command()
@click.option('--count', default=1, help='Number of greetings.')
@click.option('--name', prompt='Your name',
              help='The person to greet.')
def hello(count, name):
    """Simple program that greets NAME for a total of COUNT times."""
    for x in range(count):
        click.echo(f"Hello {name}!")

if __name__ == '__main__':
    hello()
```

執行後會看到：
``` bash
$ python hello.py --count=3
Your name: John
Hello John!
Hello John!
Hello John!
```
上述例子就是將 hello 這個 function 註冊給 framework 説執行的時候 call 這個 function，且這個 function 可以有兩個參數，用法分別使用 `@click.option` 的方式設定參數的類型，舉例來說 `count` 就是可以提供要重複幾次的選項，而 command line 輸數的時候就可以使用 `--count` 的方式提供想要的的值。

## 進階用法與細節

### 多個 decorator 修飾時的順序

整理以下條件是固定的
- 有參數的 decorator 實際是提供一個沒有參數的 decorator，wrap 是在函數生成的時候就會執行，並非 call 的時候。
- 同樣是沒有參數的 decorator 會是愈接近 function 的會先執行，可以理解為 fun1(func2(raw_func))，會是 func2 2的先執行，func1 才把 func2 的結果當參數處理。
- call 的時候，會是裡面的 function 執行完才執行外面的。

實際最快的方式直接執行以下程式理解更有感覺
``` python
import logging

logging.basicConfig(level=logging.INFO)

def wrap(name):
	logging.info('%s start to new wrap', name)
	def new_wrap(func):
		logging.info('%s start wrap', name)
		def new_func(*args, **kwargs):
			logging.info('%s enter', name)
			ret = func(*args, **kwargs)
			logging.info('%s exit', name)
			return ret
		logging.info('%s end to wrap', name)
		return new_func
	logging.info('%s end to new wrap', name)
	return new_wrap

@wrap('B')
@wrap('A')
def do_something():
	print('do something')

print('call function:')
do_something()
```

以 python 3.8 來說，輸出結果如下
```
INFO:root:B start to new wrap
INFO:root:B end to new wrap
INFO:root:A start to new wrap
INFO:root:A end to new wrap
INFO:root:A start wrap
INFO:root:A end to wrap
INFO:root:B start wrap
INFO:root:B end to wrap
call function:
INFO:root:B enter
INFO:root:A enter
do something
INFO:root:A exit
INFO:root:B exit
```

### 使用 class `__call__` 的方式做到類似效果

在談論 decorator 的用法前，先理解一下為什麼前面使用 `def` 的方式可以，`def` 的語法，實際上就是建構出符合我們預期，且有 callable 的 object (有 `__call__` 性質)，來取代要 decorator 的 function，因此同理也可以寫出一個有 `__call__` 的 class，再 new 出來取代 callable 的 object，來達到類似的效果。

常見使用 `__class__` 取代 `def` 的 decorator 的情況有
1. 單使用 function 會顯得很腐雜的情況。
2. 想避免有參數的 decorator 會寫出兩個 wrap function，造成理解不易。
3. 想使用 class 的概念來強化使用的想法。

下面以 [click](https://click.palletsprojects.com/en/8.0.x/) 的 `@command` 的實作，只簡化重點的方式來介紹
``` python
class Command:
	def __init__(self, name, f):
		pass

	def __call__(self, *args, **kwargs):
		pass

def command(name, cls=None):
	if cls is None:
		cls = Command

	def decorator(f):
		cmd = cls(name, f)
		cmd.__doc__ = f.__doc__
		return cmd
	return decorator
```
由上例，因為使用了 class 的方式來包裝，因此達到了以下幾點的優點
1. 因爲實際上 Command 要的行為可以很複雜，此例只舉了 name 的部分，但實際上還有很多，如：需要的參數有什麼 (型態為何)、help 的說明。
2. 整體的實作性複雜，不需要以 `def` 的方式包出如此龐大的架構。
3. 把 function 包裝的就像是一個 command 一般，且此 command 的功能正是以 class 來表示，更可以使用 class 的繼承等特型來加以變化，甚至可以使用客製的 class 做為參數來使用。

#### 差異上需注意的部分

在 class 只實作 `__call__` 未實作 `__get__` 的情況下，預設的行為與使用 `def` 定義出來的 function，在作為一個 class 的 method 的時候行為不一樣，以下說明其原因：

在介紹前需要先簡單介紹一下 `__get__` 的能力，相信有寫過 python class 的人都知道，class 中的 method 的第一個參數是 self，也就是拿到 new 出來的 object 本身，有更深了解的話會知道可以使用 `@classmethod` 或 `@staticmethod` 的方式來修改期性質，而實際上怎麼做到的是因為 `__get__` 的能力。

經過上面的說明，會知道直接寫 `def` 包裝出來的 function 放到 class 下的話，第一個參數拿到的是 self，但如果寫 class 的方式，因為沒有定義 `__get__` 的性質，所以預設的行為會和 `@staticmethod` 包裝後的比較相近，因此不會有第一個參數傳進來，因此在傳進 function 時的參數會不一樣。

### 有參數的 decorator 可以有預設的參數
下面以 [loguru](https://github.com/Delgan/loguru) 的 `@catch` 的實作，只簡化重點的方式來介紹：

在介紹前，我們先以官方的範例先理解使用情況，如下例，可以像 my_function1 任何 Exception 的時候都處理，或是像 my_function2 只針對 ZeroDivisionError 發生的時候才處理
``` python
@logger.catch
def my_function1(x, y, z):
    return 1 / (x + y + z)

@logger.catch(ZeroDivisionError)
def my_function2(x, y, z):
    return 1 / (x + y + z)
```

在經由前面的說明過後，都會認為有參數的 decorator 和沒有參數的 deorator 其寫法甚至是目的都不太一樣，要由一個 function 皆支援是很困難的，因此讓我們直接來欣賞前人是如何做到的：(以下有修改其行為，以簡化出這次主題的部分)

``` python
def catch(exception=Exeption, *, message='...'):
	if callable(exception) and (
		not isclass(exception) or not issubclass(exception, BaseException)
	):
		return catch()(exception)

	class Catcher:
		def __call__(self):
			pass

	return Catcher()
```

重點在於 if 的那個判斷，與 if 成立時的處理，進而處理了沒有帶參數進來的情況：
1. callable(exception)：這邊要處理的狀況是沒有帶入參數的情況，所以第一個參數應該要是預修飾的 function(這邊只是為了符合有參數的情況才取名 exception 的)，因此要是 callable 的。
2. not isclass(...) or not is subclass(...)：這個要以預期 exception 要符合才可以當 exception 用，因此會輸入不符合的應該就是不適當 exception 傳，也就是沒有帶入參數的情況，這邊就只是判斷不符合而已，

## 參考資料

[functools](https://docs.python.org/3/library/functools.html)
[Factory](https://medium.com/@geoffreykoh/implementing-the-factory-pattern-via-dynamic-registry-and-python-decorators-479fc1537bbe)
