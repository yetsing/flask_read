- app 的路径

我们知道，在创建 app 实例时，我们需要传入 \__name__ 变量，flask 通过这个变量的值获取当前 app 的路径，从而得到模板文件和静态文件的路径。

```python
class Flask(_PackageBoundObject):
    ...
    def __init__(
            self,
            import_name,
            static_url_path=None,
            static_folder='static',
            static_host=None,
            host_matching=False,
            subdomain_matching=False,
            template_folder='templates',
            instance_path=None,
            instance_relative_config=False,
            root_path=None
    ):
        _PackageBoundObject.__init__(
            self,
            import_name,
            template_folder=template_folder,
            root_path=root_path
        )
        ...

class _PackageBoundObject(object):

    def __init__(self, import_name, template_folder=None, root_path=None):
        self.import_name = import_name
        self.template_folder = template_folder

        if root_path is None:
            root_path = get_root_path(self.import_name)

        self.root_path = root_path  # app 的路径
        self._static_folder = None
        self._static_url_path = None
```

从上面的源码可以看出，flask 实现了 _PackageBoundObject 类专门处理这个问题，它在这里调用了 get_root_path() 函数

```python
def get_root_path(import_name):
    """返回一个包的路径；如果没有找到，则返回当前工作目录(cwd)
    """
    # Module already imported and has a file attribute.  Use that first.
    mod = sys.modules.get(import_name)
    if mod is not None and hasattr(mod, '__file__'):
        return os.path.dirname(os.path.abspath(mod.__file__))

    # Next attempt: check the loader.
    loader = pkgutil.get_loader(import_name)

    # Loader does not exist or we're referring to an unloaded main module
    # or a main module without path (interactive sessions), go with the
    # current working directory.
    if loader is None or import_name == '__main__':
        return os.getcwd()

    # For .egg, zipimporter does not have get_filename until Python 2.7.
    # Some other loaders might exhibit the same behavior.
    if hasattr(loader, 'get_filename'):
        filepath = loader.get_filename(import_name)
    else:
        # Fall back to imports.
        __import__(import_name)
        mod = sys.modules[import_name]
        filepath = getattr(mod, '__file__', None)

        # If we don't have a filepath it might be because we are a
        # namespace package.  In this case we pick the root path from the
        # first module that is contained in our package.
        if filepath is None:
            raise RuntimeError('No root path can be found for the provided '
                               'module "%s".  This can happen because the '
                               'module came from an import hook that does '
                               'not provide file name information or because '
                               'it\'s a namespace package.  In this case '
                               'the root path needs to be explicitly '
                               'provided.' % import_name)

    # filepath is import_name.py for a module, or __init__.py for a package.
    return os.path.dirname(os.path.abspath(filepath))
```

sys.modules 是保存了当前加载所有模块的字典，键值是模块的名字。

整个函数大概可以分为四个过程，想法设法地拿到路径

1、获取 sys.modules 中的模块 \__file__ 属性储存的模块路径

2、返回当前工作目录

3、使用 pkgutil.get_loader() 函数得到 loader ，再调用 loader.get_filename(import_name) 得到路径

4、怀疑 import_name 模块并未成功导入，重新导入一遍，使用 sys.modules 获取路径

5、抛异常（臣妾做不到）

下面是 sys.modules 和 \__name__ 的一些实验

从结果可以看出， \__name__ 和 sys.modules 已经能够满足我们的日常所需

```python
# 实验一
# path.py
import sys
import os

path = sys.modules[__name__].__file__
print('{}: {}'.format(__name__, path))
abspath = os.path.abspath(path)
print('abspath: {}'.format(abspath))
print('dirname: {}'.format(os.path.dirname(abspath)))

# 直接执行 path.py
# PyCharm 使用绝对路径执行，所以第一行是绝对路径
# 如果你在命令行中用 python path.py 命令执行，第一行则是 __main__: path.py
__main__: C:/Users/name/PycharmProjects/flask_learn/path.py
abspath: C:\Users\name\PycharmProjects\flask_learn\path.py
dirname: C:\Users\name\PycharmProjects\flask_learn
```

```python
# 实验二
# test.py
import path

# 结果
# PyCharm 和命令行执行结果一样
path: C:\Users\name\PycharmProjects\flask_learn\path.py
abspath: C:\Users\name\PycharmProjects\flask_learn\path.py
dirname: C:\Users\name\PycharmProjects\flask_learn
```

```python
# 实验三
# 文件结构如下
# - test.py
# - models
# |-- path.py
# test.py
from models.path import *

# 结果
# PyCharm 和命令行执行结果一样
models.path: C:\Users\name\PycharmProjects\flask_learn\models\path.py
abspath: C:\Users\name\PycharmProjects\flask_learn\models\path.py
dirname: C:\Users\name\PycharmProjects\flask_learn\models
```

- 模板文件和静态文件路径

这两个路径都依赖于 root_path(app 的路径)

```python
class _PackageBoundObject(object):
    
    def _get_static_folder(self):
        if self._static_folder is not None:
            return os.path.join(self.root_path, self._static_folder)

    def _set_static_folder(self, value):
        self._static_folder = value

    static_folder = property(
        _get_static_folder, _set_static_folder,
        doc='The absolute path to the configured static folder.'
    )
    
    @locked_cached_property
    def jinja_loader(self):
        if self.template_folder is not None:
            return FileSystemLoader(os.path.join(self.root_path,
                                                 self.template_folder))
```