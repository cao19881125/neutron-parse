# Paste Deploy
对于Paste Deploy的介绍，官方介绍如下
> Paste Deployment is a system for finding and configuring WSGI applications and servers. For WSGI application consumers it provides a single, simple function (loadapp) for loading a WSGI application from a configuration file or a Python Egg. For WSGI application providers it only asks for a single, simple entry point to your application, so that application users don’t need to be exposed to the implementation details of your application.

> Paste Deployment是一个用于查找和配置WSGI应用程序和服务器的系统。对于WSGI应用程序使用者，它提供了一个简单的函数（loadapp），用于从配置文件或Python Egg加载WSGI应用程序。对于WSGI应用程序提供程序，它只要求为您的应用程序提供一个简单的入口点，这样应用程序用户就不需要了解应用程序的实现细节。

上一节我们介绍了wsgi服务，其中提到了wsgi
service的两个重要步骤，在这里也有提及，为通过PasteDeploy可以做到：
- 1.配置WSGI服务
- 2.通过配置文件加载WSGI application
- 3.WSGI application只需要提供一个简单的入口点

对于以上几点，我们会在后面的讲解中进行说明

# 安装PasteDeploy

```
# sudo pip install PasteDeploy
```

# 例子
本节通过一个简单的例子来演示PasteDeploy的用法，其中wsgi服务使用上节讲解的eventlet.wsgi

## 编写配置文件

pastedeploylab.ini

```
[DEFAULT]
key1=value1
key2=value2
key3=values
[composite:pdl]
use=egg:Paste#urlmap
/:root
/calc:calc
[pipeline:root]
pipeline = logrequest showversion
[pipeline:calc]
pipeline = logrequest calculator
[filter:logrequest]
username = root
password = root123
paste.filter_factory = pastedeploylab:LogFilter.factory
[app:showversion]
version = 1.0.0
paste.app_factory = pastedeploylab:ShowVersion.factory
[app:calculator]
description = This is an "+-*/" Calculator
paste.app_factory = pastedeploylab:Calculator.factory
```
- [DEFAULT]段定义了一些全局变量
- [composite:pdl]定义了一个名为pdl的分发器
    - 直接访问ip:port的请求分发给root过滤器
    - 访问ip:port/calc的请求分发给calc过滤器
- [pipeline:root]定义了root过滤器，转发请求给logrequest和showversion
- [pipeline:calc]定义了calc过滤器，转发请求给logrequest和calculator
- [filter:logrequest]定义了logrequest过滤器，调用的代码为pastedeploylab.py中的LogFilter.factory函数
- [app:showversion]定义了一个名为showversion的app，调用代码为pastedeploylab.py中的ShowVersion.factory函数
- [app:calculator]定义了一个名为calculator的app，调用代码为pastedeploylab.py中的Calculator.factory函数


## 编写py文件

pastedeploylab.py

```python

import os
from webob import Request
from webob import Response
from paste.deploy import loadapp
from eventlet import wsgi
import eventlet
#Filter
class LogFilter():
    def __init__(self,app):
        self.app = app
        pass
    def __call__(self,environ,start_response):
        print "filter:LogFilter is called."
        return self.app(environ,start_response)
    @classmethod
    def factory(cls, global_conf, **kwargs):
        print "in LogFilter.factory", global_conf, kwargs
        return LogFilter
class ShowVersion():
    def __init__(self):
        pass
    def __call__(self,environ,start_response):
        start_response("200 OK",[("Content-type", "text/plain")])
        return ["Paste Deploy LAB: Version = 1.0.0",]
    @classmethod
    def factory(cls,global_conf,**kwargs):
        print "in ShowVersion.factory", global_conf, kwargs
        return ShowVersion()
class Calculator():
    def __init__(self):
        pass

    def __call__(self,environ,start_response):
        req = Request(environ)
        res = Response()
        res.status = "200 OK"
        res.content_type = "text/plain"
        # get operands
        operator = req.GET.get("operator", None)
        operand1 = req.GET.get("operand1", None)
        operand2 = req.GET.get("operand2", None)
        print req.GET
        opnd1 = int(operand1)
        opnd2 = int(operand2)
        if operator == u'plus':
            opnd1 = opnd1 + opnd2
        elif operator == u'minus':
            opnd1 = opnd1 - opnd2
        elif operator == u'star':
            opnd1 = opnd1 * opnd2
        elif operator == u'slash':
            opnd1 = opnd1 / opnd2
        res.body = "%s /nRESULT= %d" % (str(req.GET) , opnd1)
        return res(environ,start_response)
    @classmethod
    def factory(cls,global_conf,**kwargs):
        print "in Calculator.factory", global_conf, kwargs
        return Calculator()
if __name__ == '__main__':
    configfile="pastedeploylab.ini"
    appname="pdl"
    wsgi_app = loadapp("config:%s" % os.path.abspath(configfile), appname)
    wsgi.server(eventlet.listen(('', 8090)), wsgi_app)
```

- main函数中指定加载pastedeploylab.ini配置文件，并load pdl这个app，对应配置文件中的[composite:pdl]分发器
- 使用eventlet.wsgi启动一个0.0.0.0:8090端口的http服务
- 每个app或者filter的factory函数，global_conf参数为全局变量字典，**kwargs为ini文件中定义的此app段中的局部变量字典
- 在ShowVersion.__call__中，调用start_response返回消息头，return返回消息体
- 在Calculator.__call__中，先构造一个Response对象，在其中填充消息头和消息体，然后一起返回

## 运行

```
# python pastedeploylab.py
```

## 调用

```
# curl http://127.0.0.1:8080
Paste Deploy LAB: Version = 1.0.0
```

```
# curl "http://127.0.0.1:8090/calc?operator=plus&operand1=12&operand2=23"
GET([(u'operator', u'plus'), (u'operand1', u'12'), (u'operand2', u'23')]) /nRESULT= 35
```

---
通过本章，了解了如何通过PasteDeploy加载配置文件，将wsgi server和wsgi
application进行解耦合，并通过其中的middleware机制对请求进行中间处理，后面在讲解neutron的keystone
校验时，会着重讲解middleware在neutron中的应用