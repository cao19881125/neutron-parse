上面的章节我们讲解了rpc的概念，以及如何使用rabbitmq来实现rpc的，rabbitmq通过一些列的抽象，定义出来了如exchange，queue，routing_key等模型来承载rpc消息，以及通过exchange的不同类型来定义了消息分发的模式

在本章中，我们来学习openstack封装的oslo-message，oslo-message是一个rpc远程调用库，它定义了一套新的概念来描述rpc调用，并通过插件的形式来加载后端驱动，如默认的RabbitDriver，还有其他的驱动如zmp、amqp、kafka、pika等driver，这些driver的作用即将oslo-message定义的概念与后端通信框架相耦合，启动起承转合的作用，如默认的RabbitDriver驱动，即通过kombu来进行amqp通信，如kafka driver是通过kafka进行通信

oslo-message相比amqp更简洁，它抛弃掉了amqp许多复杂的概念，使得使用oslo-message变得更简单，同时oslo-message定义的endpoint概念，将message调用封装为对象调用，使用者可以像定义类一样来定义msg调用，在从面向对象思想来看，他比纯rabbitmq的传递字符串模式又进步了不少

## oslo-message安装

```
pip install oslo-message
```


## oslo-message例子
我们以一个get_version的例子来开启oslo-message的学习

### config文件

config文件采用的同样是openstack家的oslo-config
```
# vim config.cfg

[DEFAULT]
transport_url = rabbit://guest:guest@192.168.184.128:5672
```

这个config文件定义了transport的信息

格式为

```
驱动类型://用户名:密码@服务端IP:端口
```

这里默认使用rabbit驱动，rabbit驱动的后端实现为kombu

guest:guest@192.168.184.128:5672是rabbitmq服务端的信息

### server端

```python
# vim server.py

import sys
from oslo_config import cfg
import oslo_messaging
import time

class GetVersion(object):
    def get_version(self,ctx,arg):
        print "------ GetVersion.get_version arg:%s--------"%arg
        return "version 1.0.0"

class TestEndpoint(object):
    def test(self, ctx, arg):
        print "------ TestEndpoint.test --------"
        return arg

cfg.CONF(sys.argv[1:])

transport = oslo_messaging.get_transport(cfg.CONF)

target = oslo_messaging.Target(topic='test', server='server1')

endpoints= [
    GetVersion(),
    TestEndpoint(),
]

server = oslo_messaging.get_rpc_server(transport, target, endpoints,
                                      executor='blocking')

try:
    server.start()
    while True:
        time.sleep(1)
except KeyboardInterrupt:
   print("Stopping server")

server.stop()
server.wait()
```

这里为了体现endpoints的定义规则，定义了两个函数，一个get_version一个test

### client端


```
# vim client.py

import sys
from oslo_config import cfg
import oslo_messaging as messaging

cfg.CONF(sys.argv[1:])

transport= messaging.get_transport(cfg.CONF)
target = messaging.Target(topic='test')
client = messaging.RPCClient(transport, target)
ret= client.call(ctxt = {},
                  method = 'get_version',arg = 'hello world')

print ret
```


### run

- server
```
# python server.py --config-file ./config.cfg
```

- client

```
# python client.py --config-file ./config.cfg
version 1.0.0
```

- server 显示

```
# python server.py --config-file ./config.cfg
------ GetVersion.get_version arg:hello world-------
```


## 概念详解
### Transport
> This is a mostly opaque handle for an underlying messaging transport driver.

> RPCs and Notifications may use separate messaging systems that utilize
    different drivers, access permissions, message delivery, etc. To ensure
    the correct messaging functionality, the corresponding method should be
    used to construct a Transport object from transport configuration
    gleaned from the user's configuration and, optionally, a transport URL

Transport是一个连接底层驱动的句柄，他通过读取config文件中transport_url这个参数，获取驱动url，并通过解析url加载传输数据所使用的驱动

在上面一节提到了config文件，如以下的url

```
# vim config.cfg

[DEFAULT]
transport_url = rabbit://guest:guest@192.168.184.128:5672
```
即为加载RabbitDriver的驱动，最前面的rabbit代表了驱动的名称，驱动程序的具体位置在setup.cfg文件中，oslo.messaging.drivers这一个属性来定义，如官方实现的驱动如下

```
oslo.messaging.drivers =
    rabbit = oslo_messaging._drivers.impl_rabbit:RabbitDriver
    zmq = oslo_messaging._drivers.impl_zmq:ZmqDriver
    amqp = oslo_messaging._drivers.impl_amqp1:ProtonDriver

    # This driver is supporting for only notification usage
    kafka = oslo_messaging._drivers.impl_kafka:KafkaDriver

    # To avoid confusion
    kombu = oslo_messaging._drivers.impl_rabbit:RabbitDriver

    # This is just for internal testing
    fake = oslo_messaging._drivers.impl_fake:FakeDriver
    pika = oslo_messaging._drivers.impl_pika:PikaDriver
```
其中rabbit和kombu都会加载RabbitDriver，只是取了两个不同的名字而已

如果想使用其他驱动，只需修改config文件中transport_url参数，指向其他的驱动就可以了

### Target
> Identifies the destination of messages

> A Target encapsulates all the information to identify where a message should be sent or what messages a server is listening for.

Target是表示消息投递的目标用的，他就像写信的时的收件人地址一样，Target明确表示了这条消息应该投递到哪里，在server端，Target用来声明地址，在client端，用来表示目的地

Target类的参数

```python
class Target(object):
    def __init__(self, exchange=None, topic=None, namespace=None,
                 version=None, server=None, fanout=None,
                 legacy_namespaces=None):
```



#### exchange
与amqp中的exchange一样，用来定义一个交换机，可以指定一个独一无二的名字，也可以为空

为空的话，默认为"openstack"，这个默认值是通过oslo-config的默认参数来定义的，代码如下

```
    cfg.StrOpt('control_exchange',
               default='openstack',
               help='The default exchange under which topics are scoped. May '
                    'be overridden by an exchange name specified in the '
                    'transport_url option.'),
```



如果想修改这个默认值的话，只需要在config文件中定义control_exchange就可以了，如下:

```
[DEFAULT]
transport_url = rabbit://guest:guest@192.168.184.128:5672
control_exchange = my_default_exchange
```

#### topic
这里的topic与amqp中的topic概念不同，amqp中的topic是指的exchange的一种类型，而这里的topic，就是指的一个主题，或者是一个独一无二的routing_key，在oslo-message中，这也是路由消息必须配置的一个参数，而且是唯一一个必须配置的参数

在amqp的模型中，一个消息是通过两个参数来路由的，exchange name + routing_key

在oslo-message中，只需要定义一个topic就可以了，他将rpc简化为了只用一个topic路由的简单模型

#### namespace和version
在server端，Target也分为两类:server target和endpoint target

server target 就是我们上面所说的，定义一个topic并接收消息

endpoint target是定义这个endpoint所属的范围，一个endpoint代表一个可调用对象，如我们上面例子中的GetVersion object就是一个endpoint，默认情况下，所有的endpoint都属于default namespace，为了将endpoint归类，引入了endpoint target概念,即一个endpoint可以归属于某个namespace，和其对应的version，书写方式如下

我们还是以上面例子中的GetVersion为例子

```
class GetVersion(object):
    target = oslo_messaging.Target(namespace='ns1',
                                    version='2.0')

    def get_version(self,ctx,arg):
        print "------ GetVersion.get_version arg:%s--------"%arg
        return "version 1.0.0"
```

给GetVersion类添加类属性target后，表示此endpoint属于namespace ns1，同时version为2.0

则client端在定义Target时，需要这样写

```
target = messaging.Target(topic='test',namespace='ns1',version='2.0')
```


#### server
server参数是在server端填写，是必填参数，用来唯一表示一个server

client端可以填写server参数，也可以不填写，如果填写server，则按照指定的server进行投递

如果client端不填写server且有多个不同的server端监听了同一个topic的话，则按照server端注册的顺序，client对于这一topic的消息投递给第一个注册的server，后面注册的server将收不到投递的消息

#### fanout

True/False

表示这个消息是否广播，默认为False，如果设置为True，则将此消息广播到每一个监听了此topic的server

注意，一旦设置为True，则server参数将失效，即使client端显示定义了server，也将不起作用，调用如下

```
import sys
from oslo_config import cfg
import oslo_messaging as messaging

cfg.CONF(sys.argv[1:])

transport= messaging.get_transport(cfg.CONF)
target = messaging.Target(topic='test',fanout=True)
client = messaging.RPCClient(transport, target)

ret= client.cast(ctxt = {},
                  method = 'get_version',arg = 'hello world')
```

### endpoint
endpoint即rpc的方法实现体，可以定义为任意的python类中的函数，调用时使用函数名进行调用，无需对应类名，一个类可以定义一个rpc函数也可以定义多个rpc函数

需要注意的是，在创建server时会声明函数过滤器，这个过滤器会过滤可调用的函数名，默认情况下，只有以 _ 开头的函数会被过滤掉，其他函数均可以解析为rpc函数

### get_rpc_server
get_rpc_server函数，根据定义的transport、target、endpoints，创建rpc server handler

用一句话概括，就是使用配置的消息传递driver进行消息传递工具来监听target定义的topic并将收到的消息路由到endpoints进行处理

#### 参数executor
可选参数executor，这个参数默认值是blocking，这个参数有三个值可以选择:blocking,eventlet和threading

这几个参数的区别如下:
- blocking: 处理函数单线程执行，即一次只能处理一个调用
- eventlet: 处理函数用eventlet green thread pool来执行，可以同时处理多个消息
- threading: 处理函数用pyhton thread pool执行，多线程可以同时处理多个消息


oslo-message同样是使用stevedore来加载不同的执行体，详见setup.cfg


```
oslo.messaging.executors =
    blocking = futurist:SynchronousExecutor
    eventlet = futurist:GreenThreadPoolExecutor
    threading = futurist:ThreadPoolExecutor
```

#### 参数access_policy
可选参数access_policy，可以解释为它是一个函数过滤器，这个参数指向了一个函数过滤器，负责执行endpoint可调用函数的过滤，如果不设置或者设置为None，则默认使用DefaultRPCAccessPolicy进行过滤


```
class DefaultRPCAccessPolicy(RPCAccessPolicyBase):
    def is_allowed(self, endpoint, method):
        return not method.startswith('_')
```
可以看到这个过滤器，只会阻止 _ 开头的函数，其他函数均可通过过滤

除了DefaultRPCAccessPolicy，还可以使用ExplicitRPCAccessPolicy

```
class ExplicitRPCAccessPolicy(RPCAccessPolicyBase):
    """Policy which requires decorated endpoint methods to allow dispatch"""

    def is_allowed(self, endpoint, method):
        if hasattr(endpoint, method):
            return hasattr(getattr(endpoint, method), 'exposed')
        return False
```
这个过滤器需要函数体拥有exposed属性，由于类函数是bound函数，不能设置属性，所以可以通过如下方式定义endpoint来使用ExplicitRPCAccessPolicy


还是通过GetVersion来举例


```
class GetVersion(object):

    def _get_version(self):
        def real_fun(ctx,arg):
            print "------ GetVersion.get_version arg:%s--------"%arg
            return "version 1.0.0"
        real_fun.exposed = True
        return real_fun

    def __getattr__(self, item):
        if item == "get_version":
            return self._get_version()
```
通过__getattr__将get_version调用转换为unbound函数，即可设置exposed属性

获取rpc server时这样获取

```
server = oslo_messaging.get_rpc_server(transport, target, endpoints,
                                      executor='blocking',access_policy=oslo_messaging.rpc.ExplicitRPCAccessPolicy)
```


如果想自己实现过滤器的话，编写子类继承oslo_messaging.rpc.RPCAccessPolicyBase并实现is_allowed函数即可


### RPCClient
RPCClient是client端进行消息发送的类

与rpc_server一样，RPCClient也是通过transport和target确定消息应该发送到何方

他有两个发送函数，分别是call和cast

#### call

```
def call(self, ctxt, method, **kwargs):
```

call()方法用于调用远端rpc方法，并获取到一个返回值，因此调用call方法时，不能讲target的fanout属性设置为True

call方法是一个同步方法，它会阻塞住当前线程，直到调用返回，或者是通过timeout参数设定超时时间

#### cast

```
def cast(self, ctxt, method, **kwargs):
```

cast()方法用于调用远端rpc方法，不获取返回值，这个方法常用于Target设置了fanout=True的广播调用

cast方法仅仅会阻塞线程到transport即driver层获取到此rpc调用，但是这个方法不保证rpc调用被远端server响应