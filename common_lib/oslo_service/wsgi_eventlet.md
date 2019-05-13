# 什么是WSGI
Web服务器网关接口（Web Server Gateway Interface，缩写为WSGI）是为Python语言定义的Web服务器和Web应用程序或框架之间的一种简单而通用的接口。

WSGI不是服务器，python
模块，框架，API或者任何软件，只是一种规范，描述web server如何与web application通信的规范。server和application的规范在PEP 3333中有具体描述。要实现WSGI协议，必须同时实现web server和web application

WSGI协议主要包括server和application两部分：
- WSGI
  server：负责从客户端接收请求，将request转发给application，将application返回的response返回给客户端
- WSGI application：接收由server转发的request，处理请求，并将处理结果返回给server。application中可以包括多个栈式的中间件(middlewares)，这些中间件需要同时实现server与application，因此可以在WSGI服务器与WSGI应用之间起调节作用：对服务器来说，中间件扮演应用程序，对应用程序来说，中间件扮演服务器

# oslo-service与eventlet

oslo-service用到了eventlet中的wsgi模块，相当于oslo-service是对eventlet.wsgi的一个封装

eventlet 的 wsgi 模块提供了一种启动事件驱动的WSGI服务器的简洁手段，可以将其作为某个应用的嵌入web服务器，或作为成熟的web服务器



# eventlet.wsgi
## 例子
本节通过一个简单的例子来演示eventlet.wsgi

要启动一个 wsgi 服务器，只需要创建一个套接字，然后用它调用
eventlet.wsgi.server() 就可以

```python
from eventlet import wsgi
import eventlet

def hello_world(env, start_response):
    start_response('200 OK', [('Content-Type', 'text/plain')])
    return ['Hello, World!\r\n']

sock = eventlet.listen(('', 8090))
wsgi.server(sock=sock, site=hello_world)
```
这个简单的例子包含了WSGI协议的两个部分，分别为
- 创建了一个socket，监听8090端口
- 通过调用wsgi.server创建了一个wsgi service
- 注册了一个WSGI application，处理函数为hello_world

运行一下

```
# python eventlst_wsgi.py
(56124) wsgi starting up on http://0.0.0.0:8090
```

通过客户端访问

```
# curl http://127.0.0.1:8090
Hello, World!
```

访问成功并且通过hello_world函数处理后返回了结果

---
本节旨在通过简单的例子来演示一下eventlet.wsgi的用法，在后面讲解oslo-service时对eventlet.wsgi有一个概念，关于eventlet.wsgi更详细的用法请大家通过网络自行学习，由于不是本书主要介绍的内容，暂不进行更深入的讲解
