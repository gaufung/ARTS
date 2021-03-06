---
date: 2018-08-25
status: public
title: Tornado异步单元测试
---

# 1 前言
## 1.1 单元测试
关于单元测试的重要性不言而喻： `好的代码需要单元测试` -> `单元测试要求好的设计` -> `好的设计意味着解耦` -> `接口有助于解耦` -> `解耦有助于编写单元测试` -> `单元测试有助于编写好的代码`。
换一句话来讲，没有单元测试的代码是不值得信任的，而且编写单元测试也有助于他人理解代码，这个对团队开发是非常重要的。
## 1.2 `unittest` 包
`python`内置了`unittest`包，包含了通用的单元测试所需要的基本功能， 使用方法也非常简单：
```python
import unittest
class TestStringMethods(unittest.TestCase):
    def test_upper(self):
        self.assertEqual('foo'.upper(), 'FOO')
    def test_isupper(self):
        self.assertTrue('FOO'.isupper())
        self.assertFalse('Foo'.isupper())
    def test_split(self):
        s = 'hello world'
        self.assertEqual(s.split(), ['hello', 'world'])
        # check that s.split fails when the separator is not a string
        with self.assertRaises(TypeError):
            s.split(2)
if __name__ == '__main__':
    unittest.main()
```
每一个单元测试类继承`unittest.TestCase`，每一个测试方法以`test_`开头，`self.assert__`方法为单元测试判断比较。`unittest.main()`启动全部测试。在子类中可以重载`setUp`和`tearDown`方法，在这里可以做一些测试初始化和清理功能，比如数据路连接和关闭等等。一旦`assert`相关方法失败，则抛出异常。
## 1.3 代码覆盖率
单元测试还有一个很重要的目的是测试代码的覆盖率，`coverage`包提供了检查代码覆盖率的功能，而且能够输出网页版的查看工具。
```python
import unittest
import coverage
COV = None
COV = coverage.coverage(branch=True, include="./*", omit=["ENV/*", "_run.py", "test/*", "pep8/*", "*/__init__.py"])
COV.start()
def _test():
    tests = unittest.TestLoader().discover("test")
    unittest.TextTestRunner(verbosity=2).run(tests)
    COV.stop()
    COV.save()
    COV.report()
    basedir = os.path.abspath(os.path.dirname(__file__))
    covdir = os.path.join(basedir, "tmp/coverage")
    COV.html_report(directory=covdir)
    COV.erase()
```
`coverage`函数中`include`参数给出检查单元覆盖率的文件夹，`omit`参数给出忽略的文件夹。`html_report`将单元测试覆盖率输出到文件到指定文件夹中。在启动`_test`方法后，可以查看每一行代码是否被单元测试覆盖到。
# 2 异步方法测试
一般的方法编写测试非常简单，但是如果使用`tornado`编写的异步方法和函数该如何编写单元测试呢？没关系，`tornado`提供了`tornado.testing`包，包含了`AsyncTestCase`和`AsyncHTTPTestCase`类， 该类继承`unittest.TestCase`, 因此也包含了之前的`assert`相关的方法。`gen_test`装饰器用来异步的单元测试中，将`yield`的异步操作变成同步操作，该装饰器`timeout`参数用来指明这个异步单元测试方法的时间界限，超出时间将会引发单元测试失败异常。
```python
# func.py
from tornado.httpclient import HTTPRequest, AsyncHTTPClient
from tornado.gen import coroutine, Return
@coroutine
def run():
    request = HTTPRequest("https://gaufung.com")
    http_client = AsyncHTTPClient()
    response = yield http_client.fetch(request)
    raise Return(response.code)
    
# test_func.py
import unittest
from tornado.testing import AsyncTestCase, gen_test
from func import run
class TestFunc(AsyncTestCase):
    @gen_test(timeout=5)
    def test_run():
        code = yield run()
        self.assertEqual(code, 200)
```
每一个单元测试只能有一个`IOLoop`实例，如果在单元测试中使用`IOLoop`，则必须要重载`setUp`方法，否则将无法使用`self.io_loop`。
```python
class TestFunc(AsyncTestCase):
    def setUp(self):
        super(TestFunc, self).setUp()
        pass
```
# 3 Tornado Web应用API测试
除了对函数和方法的单元测试，还需要对`Web`接口进行单元测试，单元测试类继承`AsyncHTTPTestCase`, 并且重载`get_app`方法才可以进行单元测试。
```python
# app.py
import tornado.ioloop
import tornado.web
from tornado.gen import coroutine, Return
from tornado.httpclient import AsyncHTTPClient, HTTPRequest
@coroutine
def blog():
    request = HTTPRequest("https://gaufung.com")
    http_client = AsyncHTTPClient(ioloop=IOLoop.current())
    response = yield http_client.fetch(request)
    raise Return(response.body)

class MainHandler(tornado.web.RequestHandler):
    @coroutine
    def get(self):
        body = yield blog()
        self.write(body)

def make_app():
    return tornado.web.Application([
        (r"/gaufung", MainHandler),
    ])
    
# app_test.py
import unittest
from tornado.testing import gen_test, AsyncHTTPTestCase
import app
class TestApp(AsyncHTTPTestCase):
    def get_app(self):
        return app.make_app()
    
    @gen_test(timeout=5)
    def test_gaufung(self):
        response = yield self.http_client.fetch(self.get_url("/gaufung"))
        self.assertNotNone(response)
```
`make_app`方法返回`app.py`中创建的`Application`类，`self.get_url`获取特定接口完整的`url`。

# 4 异步方法 Mock
在测试中往往需要依赖外部内容，比如向外部服务器发送HTTP请求来获取相应的状态。但是搭建外部服务器通常费时费力，而且不符合单元测试的依赖性要求。因此`Python`提供了`Mock`库，可以模拟外部请求并且返回内容。普通的请求`Mock`使用可以查看标准库内容，而且使用非常简单。但是对于异步的请求需要如下的`Mock`方法：
```python
import tornado.ioloop
import tornado.web
from tornado.gen import coroutine, Return
from tornado.httpclient import AsyncHTTPClient, HTTPRequest

@coroutine
def blog():
    request = HTTPRequest("https://gaufung.com")
    http_client = AsyncHTTPClient(ioloop=IOLoop.current())
    response = yield http_client.fetch(request)
    raise Return(response.body)

class MainHandler(tornado.web.RequestHandler):
    @coroutine
    def get(self):
        body = yield blog()
        self.write(body)

def make_app():
    return tornado.web.Application([
        (r"/gaufung", MainHandler),
    ])

# app_test.py
import app
import mock
import unittest
from tornado.testing import gen_test, AsyncHTTPTestCase
from tornado.httpclient import AsyncHTTPClient
from tornado.concurrent import Future
class TestApp(AsyncHTTPTestCase):
    def get_app(self):
        return app.make_app()

    @mock.patch("app.blog")
    @gen_test
    def test_get(self, blog):
        blog_future = Future()
        blog_future.set_result("gaufung's blog")
        blog.return_value = blog_future
        response = yield self.http_client.fetch(self.get_url("/gaufung")
        self.assertEqual(response.code, 200)
        self.assertEqal(response.body, "gaufung's blog")
```
首先同样引入`mock`包，然后在单元测试方法使用装饰器`mock.patch`，装饰器参数为需要`mock`函数为全名。如果想要`mock`类实例的方法，参数为`"<package_name>.<class_name><instance_method_name>"`。注意现在单元测试方法增加了`blog`参数，该参数就是被`mock`掉的内容。在方法内部定义了一个`Future`对象，并且将需要的返回值作为`set_result`的参数。如果在单元测试中需要`mock`多个内容，则`mock.patch`装饰器可以叠加，相应的在单元测试方法增加参数，但是注意注意参数顺序是相反的，也就是说装饰器从上到下，相应的参数是从右到左。
**注意**
笔者在使用的过程中发现，如果单元测试中需要`mock`的包含普通方法和实例方法，通常普通方法无效，因此建议项目在设计的过程中将普通方法封装成实例方法。