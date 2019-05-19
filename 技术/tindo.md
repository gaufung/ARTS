---
title: Web应用程序框架: Tindo
status: public
date: 2017-09-23
---
Tindo 是一个 Python Web 应用程序框架，按照`WSGI`规范实现了应用程序部分接口，[源代码](https://github.com/gaufung/tindo)托管在GitHub上。

# 1 路由选择
由于装饰器(Decorator)是Python语法的一部分，因此许多Web应用程序框架选择使用装饰器作为路由选择，比如`flask`、`web.py`等。但是装饰器核心思想是不能影响被装饰函数的原本含义，也就是剥离装饰器的作用，原函数也能完整工作。因此 `flask`框架被装饰的路由函数，往往返回一个被渲染后`Jinja2`模板对象，对代码的入侵性较强，因此`Tindo`选择耦合性更小的路由选择模式。

## 1.1 路由装饰器
```Python
def route(path, methods=None):
    if methods is None:
        methods = ['GET']
    def _decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kw):
            return func(*args, **kw)
        wrapper.__web_route__ = path
        wrapper.__web_method__ = methods
        return wrapper
    return _decorator
```
借鉴了`flask`路由装饰器，将`GET`和`POST`的方法封装到同一个路由装饰器中，并且默认为`GET`操作，使用`functools.wraps`装饰器，更新路由函数的方法签名。

## 1.2 视图装饰器
`Tindo`采用`Jinja2`作为视图层面，`Jinja2`接受一个字典类型对象来渲染模板。
```Python
def view(path):
    def _decorator(func):
        @functools.wraps(func)
        def _wrapper(*args, **kw):
            r = func(*args, **kw)
            if isinstance(r, dict):
                return Template(path, **r)
            raise ValueError('Expect return a dict when using @view() decorator.')
        return _wrapper
    return _decorator
```

##  1.3 查询路由
许多网站`URL`为`www.example.com/users/gaufung`，代表了`users`名为`gaufung`，当用户量非常大，不可能编写为这些`URL`编写每一个路由，因此需要从URL中匹配相关的参数。在Python的正则表达式可以为匹配分组命名`(?P<name>)`，因此 `Tindo`选择`/user/<username>`格式作为参数捕获。
```Python
_re_route = re.compile(r'<([a-zA-Z_]\w*)>')
def _re_char(ch):
    s = ''
    if '0' <= ch <= '9':
        s = s + ch
    elif 'a' <= ch <= 'z':
        s = s + ch
    elif '0' <= ch <= '9':
        s = s + ch
    else:
        s = s + '\\' + ch
    return s
def _build_regex(path):
    """
    build path regex pattern
    '/users/<username>' => '^/user/(?P<username>[^\/]+)$'
    :param path: the path
    :return: regex pattern
    """
    re_list = ['^']
    i = 0
    while i < len(path):
        sub_path = path[i:]
        m = _re_route.match(sub_path)
        if m:
            re_list.append(r'(?P<%s>[^\/]+)' % m.group(1))
            i = i + m.end()
        else:
            re_list.append(_re_char(path[i]))
            i = i+1
    re_list.append('$')
    return ''.join(re_list)
```
## 1.4 实例
```Python
@view('name.html')
@route('/user/<username>')
def user(name):
    return dict(name=name)
```

# 2 Request
每一个HTTP请求都可以都可以封装成一个`Request`对象，每一个HTTP请求都包含请求头(Header)和请求主体(Body)，根据WSGI协议，这些内容都被Web服务器封装成一个`envrion`字典类型对象，下面列出几个常用的关键信息

+ REQUEST_METHOD:  Request的类型，GET、POST等等
+ PATH_INFO：Request的路径
+ QUERY_STRING：形如?的查询字符串
+ HTTP_COOKIE: cookie
+ ...

对于 `POST`操作，表单的内容一般在`wgsi.input`字段中，并且表单项中每一个也都是key-value形式保存。
```Python
    def _parse_input(self):
        def _convert(item):
            if isinstance(item, list):
                return [to_unicode(i.value) for i in item]
            if item.filename:
                return MultipartFile(item)
            return to_unicode(item.value)
        fs = cgi.FieldStorage(fp=self._environ['wsgi.input'], environ=self._environ, keep_blank_values=True)
        inputs = dict()
        for key in fs:
            inputs[key] = _convert(fs[key])
        return inputs
```
# 3 Response
根据WSGI中的协议，Response部分主要包含两个部分， 一个为当前Response的状态，另一个为tuple类型的列表，包含了每个相应头的对象键值。
+ Successful
    + 200 : 'OK'
    + 201 : 'Created'
    + ...
+ Redirected
    + 300: 'Multiple Choices'
    + 301: 'Moved Permanently'
    + ...
+ Client Error
    + 400: 'Bad Request'
    + 404: 'Not Found'
    + ...
+ Server Error
    + 500: 'Internal Server Error'
    + 501: 'Not Implemented'
    + ...
在 `Tindo`中，默认每个Response初始化为`200 OK`, 如果在路由处理函数中，出现异常或者重定向过程中，抛出相关的异常，捕获相关异常，并且重新设置相关状态和响应头。

#  4 Cookie和Session
HTTP协议是幂等的，也就是HTTP不保存客户端的状态的。但是现实中，当我们登录到某个网站，在站内跳转的时候，返回的结果都是个人信息。因此`Cookie`起到了很大的作用，在`Request`和`Response`中Header部分都保存`cookie`项。
在Response中保存cookie
```python
    def set_cookie(self, name, value, max_age=None,
                   expires=None, path='/', domain=None,
                   secure=False, http_only=True):
        if not hasattr(self, '_cookies'):
            self._cookies = {}
        cookies = ['%s=%s' % (quote(name), quote(value))]
        if expires is not None:
            if isinstance(expires, (float, int, long)):
                cookies.append('Expires=%s' % datetime.datetime.fromtimestamp(expires, UTC_0)
                         .strftime('%a, %d-%b-%Y %H:%M:%S GMT'))
            if isinstance(expires, (datetime.date, datetime.datetime)):
                cookies.append('Expires=%s' % expires.astimezone(UTC_0).strftime('%a, %d-%b-%Y %H:%M:%S GMT'))
        elif isinstance(max_age, (int, long)):
            cookies.append('Max-Age=%d' % max_age)
        cookies.append('Path=%s' % path)
        if domain:
            cookies.append('Domain=%s' % domain)
        if secure:
            cookies.append('Secure')
        if http_only:
            cookies.append('HttpOnly')
        self._cookies[name] = '; '.join(cookies)
```
在Request中获取cookie比较简单，只要根据相关的键值即可。
有时web应用程序需要保存用户当前连接状态，也就是`session`, 只要将当前的SessionId作为Cookie保存下来，对于每一个Request，在服务端将当前的`session`恢复出来即可，每个session对象为字典对象。
```Python
_SESSIONS_WAREHOUSE = {}
_session_lock = Lock()
def _get_session(session_id):
    with _session_lock:
        if session_id not in _SESSIONS_WAREHOUSE:
            _SESSIONS_WAREHOUSE[session_id] = Dict()
        return _SESSIONS_WAREHOUSE[session_id]
class Request(object):
    ...
    @property
    def session(self):
        sessionid = self.cookie('sessionid')
        if sessionid is not None:
            return _get_session(sessionid)
    ....
class Reponse(object):
    ...
     @property
     def session(self):
        if ctx.request.session is None:
            sessionid = str(uuid1())
            self.set_cookie('sessionid', sessionid)
        else:
            sessionid = ctx.request.cookie('sessionid')
            self.set_cookie('sessionid', sessionid)
        return _get_session(sessionid)
        ...
```
# 5 多线程处理
web应用程序都是多线程处理每一个Request请求，因此每一个Response也要相对应与线程内的Request，对此，借鉴`flask`框架关于多线程处理方式，建立一个全局字典，每个线程的ID作为一个key，将相应的Resquest和Response作为值存放进去。
```python
from thread import get_ident
class Local(object):
    __slots__ = ('__storage__', '__ident_func__')
    def __init__(self):
        object.__setattr__(self, '__storage__', {})
        object.__setattr__(self, '__ident_func__', get_ident)

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

    def __delattr__(self, name):
        try:
            del self.__storage__[self.__ident_func__()][name]
        except KeyError:
            raise AttributeError(name)
```
在web应用程序过程中
```Python
ctx = Local()
def _wsgi_app(self, environ, start_response, debug=True):
        ctx.application = _application
        ctx.request = Request(environ)
        response = ctx.response = Response()
        try:
            ...
        except: 
            ....
        finally:
            del ctx.application
            del ctx.request
            del ctx.response
```