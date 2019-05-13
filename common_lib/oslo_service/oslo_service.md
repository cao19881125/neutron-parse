# oslo service
上两节讲解了oslo-service用到的两项关键性的技术，分别为wsgi和PasteDeploy，oslo-service可以理解为将对他们进行的封装，使用oslo-service，你可以快速搭建出来一套通过配置文件将server和application解耦合的restful
api框架

本节将着重讲解oslo-service的使用

# 安装

```
# pip install oslo.service
```

# 例子代码
我们同样使用pastedeploy章节的例子来讲解，只不过创建wsgi
server的部分由eventlet.wsgi换成oslo-service

## 创建oslo-config配置文件
oslo-service需要用到oslo-config来读取配置文件，我们先要创建一个配置文件

config.cfg 

```
[DEFAULT]
api_paste_config=./service.ini
```

- 这里只定义了一个参数，为pastedeploy.ini的路径

## 编写pastedeploy配置文件

service.ini

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
paste.filter_factory = service:LogFilter.factory
[app:showversion]
version = 1.0.0
paste.app_factory = service:ShowVersion.factory
[app:calculator]
description = This is an "+-*/" Calculator
paste.app_factory = service:Calculator.factory
```
在上一节已经对参数进行了讲解，本节不在进行讲解，详见上一节

## 编写py文件
service.py

```
from oslo_config import cfg
import oslo_service.wsgi
from webob import Request
from webob import Response
import sys
from oslo_service import service


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
    cfg.CONF(args=sys.argv[1:])
    loader = oslo_service.wsgi.Loader(cfg.CONF)
    app = loader.load_app('pdl')
    
    server = oslo_service.wsgi.Server(conf=cfg.CONF,name="pdl",app=app,host='127.0.0.1',port=8090)
    
    service_launcher = service.ServiceLauncher(cfg.CONF)
    service_launcher.launch_service(server)
    service_launcher.wait()

```

与pastedeploy章节的例子比起来，这里只有main函数做了改动，改动如下

- application通过oslo_service.wsgi.Loader来创建
- 通过oslo_service.wsgi.Server创建服务
- 通过service.ServiceLauncher启动服务


## run

```
# python service.py --config-file ./config.cfg
```

## 客户端访问

```
# curl http://127.0.0.1:8090
Paste Deploy LAB: Version = 1.0.0
```