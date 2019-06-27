这次我们主要了解 flask 路由的注册和匹配的实现方法

- 路由注册

我们平时都是使用 Flask.route() 方法注册路由，代码如下

```python
    def route(self, rule, **options):
        def decorator(f):
            endpoint = options.pop('endpoint', None)
            self.add_url_rule(rule, endpoint, f, **options)
            return f

        return decorator
```

我们可以很清楚地看到， Flask.route() 方法主要是调用 Flask.add_url_rule() 方法完成路由注册

```python
    def add_url_rule(self, rule, endpoint=None, view_func=None,
                     provide_automatic_options=None, **options):
        ...

        rule = self.url_rule_class(rule, methods=methods, **options)
        rule.provide_automatic_options = provide_automatic_options

        self.url_map.add(rule)
        if view_func is not None:
            old_func = self.view_functions.get(endpoint)
            if old_func is not None and old_func != view_func:
                raise AssertionError('Vieselfw function mapping is overwriting an '
                                     'existing endpoint function: %s' % endpoint)
            self.view_functions[endpoint] = view_func
```

代码中的省略部分是处理端点和方法，然后便是注册路由规则和视图函数。可以看到，整个过程并不是很复杂。

接下来，我们看看 `self.url_map.add(rule)` 这一段代码发生了什么。

其中， rule 是 Rule 类的实例， url_map 是 Map 类的实例。

```python
class Map(object):
    def add(self, rulefactory):
        for rule in rulefactory.get_rules(self):
            rule.bind(self)
            self._rules.append(rule)
            self._rules_by_endpoint.setdefault(rule.endpoint, []).append(rule)
        self._remap = True

class Rule(RuleFactory):
    def get_rules(self, map):
        yield self
```

Rule.get_rules() 方法返回了一个包含自身实例的生成器

Map.add() 方法遍历生成器并将 rule 添加进 _rules 和 _rules_by_endpoint 两个实例属性中

再来看 Rule.bind() 方法这边

```python
    def bind(self, map, rebind=False):
        """在 map 上绑定 url ，并且创建基于实例自身信息和 map 默认值的正则表达式
        """
        if self.map is not None and not rebind:
            raise RuntimeError('url rule %r already bound to map %r' %
                               (self, self.map))
        self.map = map
        if self.strict_slashes is None:
            self.strict_slashes = map.strict_slashes
        if self.subdomain is None:
            self.subdomain = map.default_subdomain
        self.compile()
```

保存当前 Map 实例，更新一些默认值，调用 Rule.compile() 创建正则表达式

这个 Rule.compile() 方法没看懂 : ( ，直接实践看一下会产生怎样的模式字符串

首先在源码里 Rule.compile() 方法的源码最后插入一行：self.regex = regex

```python
from werkzeug.routing import Map, Rule


def main():
    url_map = Map()
    methods = ('GET',)
    while True:
        r = input('输入：')
        rule = Rule(r, methods=methods)
        url_map.add(rule)
        print('输出：{}'.format(rule.regex))


if __name__ == '__main__':
    main()
```

```
# 结果
输入：/name
输出：^\|\/name$
输入：/name/
输出：^\|\/name(?<!/)(?P<__suffix__>/?)$
输入：/<id>
输出：^\|\/(?P<id>[^/]{1,})$
输入：/<int:id>
输出：^\|\/(?P<id>\d+)$
```

可以明显地看到，当 url 路径以 / 结尾时，会添加这么一串 `(?<!/)(?P<__suffix__>/?)` ，没弄懂这串什么意

> 这段关于正则的真是搞得我各种抓狂，现在先就这样，以后有空再研究

- 路由匹配

>  其实通过上面的了解，我们可以猜到 flask 使用正则表达式进行路由匹配，头疼O__O "…

flask 的路由匹配是在创建请求上下文的过程中完成的

```python
class RequestContext(object):

    def __init__(self, app, environ, request=None):
        ...
        self.url_adapter = app.create_url_adapter(self.request)
        ...
        self.match_request()

    def match_request(self):
        try:
            url_rule, self.request.view_args = self.url_adapter.match(return_rule=True)
            self.request.url_rule = url_rule
        except HTTPException as e:
            self.request.routing_exception = e
```

实例属性 url_adapter ，是由 Flask.create_url_adapter() 方法生成

```python
    def create_url_adapter(self, request):
        """根据给定的请求创建 URL adapter
        """
        if request is not None:
            subdomain = ((self.url_map.default_subdomain or None)
                         if not self.subdomain_matching else None)
            return self.url_map.bind_to_environ(
                request.environ,
                server_name=self.config['SERVER_NAME'],
                subdomain=subdomain)

        if self.config['SERVER_NAME'] is not None:
            return self.url_map.bind(
                self.config['SERVER_NAME'],
                script_name=self.config['APPLICATION_ROOT'],
                url_scheme=self.config['PREFERRED_URL_SCHEME'])
```

第一个 if 块对应的是正常的情况；第二个 if 则是对应测试的情况，比如当我们调用 Flask.test_request_context() 方法手动推送请求上下文时，请求对象就是个 None。

通过查看 Map.bind_to_environ() 和 Map.bind() 方法的源码，我们发现最终其返回了一个 MapAdapter 类的实例

从 RequestContext.match_request() 方法中可以看到，路由匹配的重点是下面这一行：

url_rule, self.request.view_args = self.url_adapter.match(return_rule=True)

```python
    # 我只抽取了代码的主要逻辑
    def match(self, path_info=None, method=None, return_rule=False,
              query_args=None):
        ...
        for rule in self.map._rules:
            try:
                rv = rule.match(path, method)
            ...

            if return_rule:
                return rule, rv
            else:
                return rule.endpoint, rv

        ...
```

这里的匹配是通过调用 Rule.match() 方法实现的，Rule.match() 方法会返回一个字典（如果 rule 为动态路由，字典包含参数；否则为空字典），可以看一下作者在注释中举的例子

```python
>>> m = Map([
...     Rule('/', endpoint='index'),
...     Rule('/downloads/', endpoint='downloads/index'),
...     Rule('/downloads/<int:id>', endpoint='downloads/show')
... ])
>>> urls = m.bind("example.com", "/")
>>> urls.match("/", "GET")
('index', {})
>>> urls.match("/downloads/42")
('downloads/show', {'id': 42})
```

- 其他

##### endpoint

如果你阅读了 flask 之前的版本(<0.8)，你就会发现之前的 flask 中注册路由使用的端点(endpoint)不能指定，只能是视图函数的名字。一直到 0.8 版本才允许指定端点(endpoint)，这样可以让 flask 更灵活

我猜测应该是为了防止某些情况下，两个不同路由的端点出现冲突，比如下面这样：

```python
from flask import Flask

app = Flask(__name__)


def hello(f):
    def decorator(*args, **kwargs):
        print('hello')
        return f(*args, **kwargs)

    return decorator


@app.route('/')
@hello
def index():
    return 'index'


@app.route('/what')
@hello
def what():
    return 'what'
```

上面这种情况，一运行便会报错

AssertionError: Vieselfw function mapping is overwriting an existing endpoint function: decorator

index() 和 what() 这两个函数的端点(endpoint)都是 decorator ，于是出现了冲突。

当然除了直接指定端点名，我们还能用另一种方法解决。

使用 functools 模块的 wraps() 函数改变函数的一些属性，其中就包括函数的 \__name_\_ 属性

```python
from functools import wraps


def hello(f):
    @wraps(f)
    def decorator(*args, **kwargs):
        print('hello')
        return f(*args, **kwargs)

    return decorator
```

##### 一个简单的路由匹配

```python
def index():
    return 'index'

def what():
    return 'what'

route_dict = {
    '/': index,
    '/what': what,
}
```

上面的代码用字典实现了一个简单的路由匹配功能。当然用这种方式无法实现 flask 的动态路由。
