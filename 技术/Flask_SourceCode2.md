---
date:  2017-9-2
title: Flask源码分析(2)
status: public
---
# 1 代码准备
在[Flask源码分析(1)]()中使用了`Flask 0.1`版本，虽然只有`flask.py`文件，只有600多行的代码。但是为了更加方便的阅读源代码，需要将其切换到第一次提交的版本，使用`git checkout 33850c0`， 整个项目只有个`flask.py`一个文件，并且包含注释只有357行。为了使用程度断点调试工程，使用IDE `Pychame`, 虚拟Python运行环境 :

+ pip install virtualenv
+ virtualenv env
+ source env/bin/activate
+ pip install  werkzeug
+ pip install jinja2

# 2 _request_ctx_stack全局变量 
在`flask.py`文件中，最后有四个公共的全局变量
```python
_request_ctx_stack = LocalStack()
current_app = LocalProxy(lambda: _request_ctx_stack.top.app)
request = LocalProxy(lambda: _request_ctx_stack.top.request)
session = LocalProxy(lambda: _request_ctx_stack.top.session)
g = LocalProxy(lambda: _request_ctx_stack.top.g)
```
在使用`Flask`过程中，我们可以直接使用`from flask import request`可以表示当前的请求对象。一般来讲，Web服务器都是使用多线程或者多协程的方式来进行多用户访问请求。那么单单一个`request`如何做到针对不同用户请求赋给不同的Request请求呢？
## 2.1 LocakStack
`werkzeug`库中的`local.py`文件中包含了`LocalStack`类，具体的定义如下
```python
class LocalStack(object):
    # ...
    def __init__(self):
        self._local=Local()
    #...
    def push(self, obj):
        rv = getattr(self._local, 'stack', None)
        if rv is None:
            self._local.stack = rv = []
        rv.append(obj)
        return rv
    def pop(self):
        stack = getattr(self._local, 'stack', None)
        if stack is None:
            return None
        elif len(stack) == 1:
            release_local(self._local)
            return stack[-1]
        else:
            return stack.pop()
    @property
    def top(self):
        """The topmost item on the stack.  If the stack is empty,
        `None` is returned.
        """
        try:
            return self._local.stack[-1]
        except (AttributeError, IndexError):
            return None
```
包含了一般栈的`push, pop, top`基本操作，但是还是不能解释如何处理多线程或者多协程的的问题。其中的秘密就在`_local`对象中，与Python标准库的中的`threading.local`类似，保存在local对象中的值再每个线程中互不影响。不过`werkzeug`重新实现了一遍。
## 2.2 Local
```python
class Local(object):
    # ...
    def __init__(self):
        obj.__setattr__(self, '__storage__',{})
        obj.__setattr__(self, '__ident_func__', get_ident)
    #...
    def __getattr__(self, name):
        try:
            return self.__storage__[self.__ident_func__()][name]
        except KeyError:
            raise AttributeError(name)
    def __setattr__(self, name, value):
        ident = self.__ident_func__()
        storage = self.__storage__
        try:
            storage[ident][name] = value
        except KeyError:
            storage[ident] = {name: value}
```
`get_ident`函数可以获取当前的线程或者协程的id值，并且将该值作为一个Key，存放到`__storage__`字典中，因此在flask在运行过程中，真个内部分配情况应该为
```json
{
    '16921':[_RequestContext1, _RequestContext2,...]
    '63293':[_RequestContext3, _RequestContext4]
    ...
}
```
## 2.3 RequestContext
栈中保存着每个线程的上下文，每个上下文包含的内容有
```python
class _RequestContext(object):
    """The request context contains all request relevant information.  It is
    created at the beginning of the request and pushed to the
    `_request_ctx_stack` and removed at the end of it.  It will create the
    URL adapter and request object for the WSGI environment provided.
    """
    def __init__(self, app, environ):
        self.app = app
        self.url_adapter = app.url_map.bind_to_environ(environ)
        self.request = app.request_class(environ)
        self.session = app.open_session(self.request)
        self.g = _RequestGlobals()
        self.flashes = None
```
请求上下文包含了

+ app： 全局变量
+ url_adapter: url匹配
+ request:  当前request 请求
+ session： 当前请求上下文
+ g：全局变量
+ flash: 消息