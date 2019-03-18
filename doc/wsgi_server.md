这次我们了解一下 flask 内置服务器的原理，也就是调用 Flask.run() 方法启动的服务器

Flask.run() 方法调用了 werkzeug 模块中的 run_simple() 函数，run_simple() 函数则调用了同文件中的 make_server() 函数

```python
def make_server(host=None, port=None, app=None, threaded=False, processes=1,
                request_handler=None, passthrough_errors=False,
                ssl_context=None, fd=None):
    if threaded and processes > 1:
        raise ValueError("cannot have a multithreaded and "
                         "multi process server.")
    elif threaded:
        return ThreadedWSGIServer(host, port, app, request_handler,
                                  passthrough_errors, ssl_context, fd=fd)
    elif processes > 1:
        return ForkingWSGIServer(host, port, app, processes, request_handler,
                                 passthrough_errors, ssl_context, fd=fd)
    else:
        return BaseWSGIServer(host, port, app, request_handler,
                              passthrough_errors, ssl_context, fd=fd)
```

可以看到，运行的服务器分为多线程、多进程和单线程三种方式

我们先来讨论单线程的 BaseWSGIServer 类。

首先在 Flask.\_\_call_\_() 方法上打上断点，使用 PyCharm  的 debug 模式运行，下图为请求到来后的函数调用流程图

其中，黑色的实线表示函数调用

如 server_forever 指向 _handle_request_noblock 表示 server_forever() 函数中调用了 _handle_request_noblock() 函数

![调用流程图]()

接下来则是 ThreadedWSGIServer 类的函数调用流程图

![调用流程图]()

最后则是 ForkingWSGIServer 类的函数调用流程图才怪，由于我的是 Windows 系统，不支持 fork 调用，因此无法使用 flask 的多进程 WSGI 服务器。(┬＿┬)

