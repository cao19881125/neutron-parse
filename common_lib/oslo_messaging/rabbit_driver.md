上两章我们详细讲解了rabbitmq库kombu以及oslo-messaging的用法，乍一看oslo-messaging与kombu没有联系，实际上，oslo-messaging是通过RabbitDriver与kombu产生联系的，这一章，我们将分析oslo-messaging的默认driver RabbitDriver，从源码角度来解析RabbitDriver是如何将oslo-messaging的概念转化为rabbitmq的概念并通过kombu进行通信的
# 前言
> The ``rabbit`` driver is the default driver used in OpenStack's integration tests.
> The driver is aliased as ``kombu`` to support upgrading existing installations with older settings.

上面这段话是openstack官方对RabbitDriver的介绍，大致意思为，RabbitDriver是openstack继承测试使用的默认driver，又名kombu


oslo-message官方源码中实现了多个driver，除了kombu还有ZmqDriver、ProtonDriver、KafkaDriver、FakeDriver、PikaDriver，详细代码路径可以参考oslo_message目录下的setup.cfg中oslo.messaging.drivers这个entry_point

由于RabbitDriver是官方默认的driver，所以本文主要分析RabbitDriver的实现


# RabbitDriver
我们在kombu这一章讲解了实现kombu的几个重要步骤，这里再复习一下，分别为
- consumer端步骤
1. 定义connection
2. 定义exchange
3. 定义queue，绑定exchange与routing_key
4. 定义consumer，绑定queue与回调函数
5. 启动服务，处理回调

- producer端步骤
1. 定义connection
2. 定义exchange
3. 定义producer
4. 通过producer向exchange发送消息

既然RabbitDriver是对kombu的封装，那我们就从这几个kombu的实现步骤入手，看一下RabbitDriver是如何封装kombu的
## consumer端
### 定义connection
加载RabbitDriver后，会调用RabbitDriver.__init__函数


```
class RabbitDriver(amqpdriver.AMQPDriverBase):
    def __init__(self, conf, url,
                 default_exchange=None,
                 allowed_remote_exmods=None):
        # ...
        connection_pool = pool.ConnectionPool(
            conf, max_size, min_size, ttl,
            url, Connection)
        super(RabbitDriver, self).__init__(
            conf, url,
            connection_pool,
            default_exchange,
            allowed_remote_exmods
        )    
```
这里的Connection为oslo_messaging/_drivers/impl_rabbit.Connection类，初始化父类AMQPDriverBase后，此connection_pool会保存在父类AMQPDriverBase的self._connection_pool属性中

```
graph TD
A(MessageHandlingServer.start)-->B(RPCServer._create_listener)
B-->C(Transport._listen)
C-->D(AMQPDriverBase.listen)
D-->E(self._get_connection)
E-->F(rpc_common.ConnectionContext.__init__)
F-->G(ConnectionPool.create)
```
MessageHandlingServer是启动Rabbitmq server定义的类，在后面讲解如何使用RabbitDriver时会讲到

```
class ConnectionPool(Pool):
    def create(self, purpose=common.PURPOSE_SEND):
        return self.connection_cls(self.conf, self.url, purpose)
```
这里的connection_cls即为上面初始化的impl_rabbit.Connection类

在其__init__函数中，创建了kombu Connection

```
class Connection(object):
    def __init__(self, conf, url, purpose):
        # ...
        self.connection = kombu.connection.Connection(
            self._url, ssl=self._fetch_ssl_params(),
            login_method=self.login_method,
            heartbeat=self.heartbeat_timeout_threshold,
            failover_strategy=self.kombu_failover_strategy,
            transport_options={
                'confirm_publish': True,
                'client_properties': {
                    'capabilities': {
                        'authentication_failure_close': True,
                        'connection.blocked': True,
                        'consumer_cancel_notify': True
                    },
                    'connection_name': self.name},
                'on_blocked': self._on_connection_blocked,
                'on_unblocked': self._on_connection_unblocked,
            },
        )
        # ...
```

### 定义exchange

```
graph TD
A(MessageHandlingServer.start)-->B(RPCServer._create_listener)
B-->C(Transport._listen)
C-->D(AMQPDriverBase.listen)
D-->E(_drivers.impl_rabbit.Connection.declare_topic_consumer)
E-->F(Consumer.__init__)
```


```
class Consumer(object):
    def __init__(self, exchange_name,...)
        # ...
        self.exchange = kombu.entity.Exchange(
            name=exchange_name,
            type=type,
            durable=self.durable,
            auto_delete=self.exchange_auto_delete)
```

注意，如果在定义Target时没有指定exchange，这里使用的是默认的exchange，即"openstack",在oslo-messaging使用这一章有详细介绍

### 定义queue
```
graph TD
A(MessageHandlingServer.start)-->B(RPCServer._create_listener)
B-->C(Transport._listen)
C-->D(AMQPDriverBase.listen)
D-->E(_drivers.impl_rabbit.Connection.declare_topic_consumer)
```


```
class AMQPDriverBase(base.BaseDriver):
        def listen(self, target, batch_size, batch_timeout):
        conn = self._get_connection(rpc_common.PURPOSE_LISTEN)

        listener = AMQPListener(self, conn)

        conn.declare_topic_consumer(exchange_name=self._get_exchange(target),
                                    topic=target.topic,
                                    callback=listener)
        conn.declare_topic_consumer(exchange_name=self._get_exchange(target),
                                    topic='%s.%s' % (target.topic,
                                                     target.server),
                                    callback=listener)
        conn.declare_fanout_consumer(target.topic, listener)

        return base.PollStyleListenerAdapter(listener, batch_size,
                                             batch_timeout)
```
- 这里创建了一个AMQPListener对象
- 创建了三个consumer:两个topic，一个fanout，其中两个topic中有一个添加了.server，这是RabbitDriver
- 将listener作为callback传入

```
def declare_topic_consumer(self, exchange_name, topic, callback=None,
                               queue_name=None):
        """Create a 'topic' consumer."""
        consumer = Consumer(exchange_name=exchange_name,
                            queue_name=queue_name or topic,
                            routing_key=topic,
                            type='topic',
                            durable=self.amqp_durable_queues,
                            exchange_auto_delete=self.amqp_auto_delete,
                            queue_auto_delete=self.amqp_auto_delete,
                            callback=callback,
                            rabbit_ha_queues=self.rabbit_ha_queues)

        self.declare_consumer(consumer)
```


```
graph TD
E(_drivers.impl_rabbit.Connection.declare_topic_consumer)-->F(self.declare_consumer)
F-->G(self.ensure)
G-->H(self.ensure.execute_method)
H-->I(self.declare_consumer._declare_consumer)
I-->J(Consumer.declare)
```


```
# oslo_messaging/_drivers/impl_rabbit.py
class Consumer(object):
    def declare(self, conn):
        self.queue = kombu.entity.Queue(
            name=self.queue_name,
            channel=conn.channel,
            exchange=self.exchange,
            durable=self.durable,
            auto_delete=self.queue_auto_delete,
            routing_key=self.routing_key,
            queue_arguments=self.queue_arguments)
        self.queue.declare()
        
        self._declared_on = conn.channel
```

### 定义consumer
在我们上一章的kombu例子中，我们是这样定义consumer的

```
consumer = Consumer(conn.channel(),queues=[q],accept=['pickle', 'json'],callbacks=[_callback])
consumer.consume()
```
定义consumer的一个目标是绑定queue与回调函数

在RabbitDriver的实现中，他并没有像我们kombu例子中一样显式的定义Consumer对象，而是通过queue.comsumer函数来绑定queue与回调的，这个具体的步骤会在下一节讲到

为了印证这一点，我们先来看一下kombu源码中对于Consumer.consume()函数的实现

```python
# kombu/messaging.py
class Consumer(object):
    ...
    def consume(self, no_ack=None):
        ...
        self._basic_consume(queue, no_ack=no_ack, nowait=True)
        ...
    
    def _basic_consume(self, queue, consumer_tag=None,
                       no_ack=no_ack, nowait=True):
        ...
        queue.consume(tag, self._receive_callback,
                          no_ack=no_ack, nowait=nowait)
        ..
```
实际上Consumer对象也是间接的调用了queue.consume()函数来绑定回调

所以在RabbitDriver中，他直接省掉了定义Consumer这一步

### 启动服务，处理回调

调用MessageHandlingServer.start启动服务
得到ListenAdapter
```
graph TD
A(MessageHandlingServer.start)-->B(RPCServer._create_listener)
B-->C(Transport._listen)
C-->D(AMQPDriverBase.listen)
```

```
def listen(self, target, batch_size, batch_timeout):
    listener = AMQPListener(self, conn)
    # ...
    return base.PollStyleListenerAdapter(listener, batch_size,
                                             batch_timeout)
```

MessageHandlingServer._create_listener()最终得到一个PollStyleListenerAdapter类型的对象,文件位置oslo_messaging/_drivers/base.py


```
class MessageHandlingServer(service.ServiceBase, _OrderedTaskRunner):
    def start(self, override_pool_size=None):
        # ...
        self.listener = self._create_listener()
        
        # ...
        self.listener.start(self._on_incoming)
```

然后PollStyleListenerAdapter.star得到调用，传入的回调函数是self._on_incoming

self._poll_style_listener类型为amqpdriver.AMQPListener
```
class PollStyleListenerAdapter(Listener):
    def __init__(self, poll_style_listener, batch_size, batch_timeout):
        # ...
        self._poll_style_listener = poll_style_listener
        self._listen_thread = threading.Thread(target=self._runner)
        # ...
    
    def start(self, on_incoming_callback):
        super(PollStyleListenerAdapter, self).start(on_incoming_callback)
        self._started = True
        self._listen_thread.start()        
```


start中启动了一个线程，线程执行体是self._runner


```
    def _runner(self):
        while self._started:
            incoming = self._poll_style_listener.poll(
                batch_size=self.batch_size, batch_timeout=self.batch_timeout)

            if incoming:
                self.on_incoming_callback(incoming)
```


self._poll_style_listener类型为amqpdriver.AMQPListener


```
class AMQPListener(base.PollStyleListener):
    def poll(self, timeout=None):
        # ... 
        while not self._shutdown.is_set():
            self._message_operations_handler.process()
            if self.incoming:
                return self.incoming.pop(0)
            
            # ...
            self.conn.consume(timeout=min(self._current_timeout, left))
        
        # ... 
        if self.incoming:
            return self.incoming.pop(0)
```

我们可以看到在poll中的操作可以分为，先从消息队列中获取消息并放入到self.incoming容器中，然后判断self.incoming中是否有消息，如果有，取一个消息并且返回


先看一下self.conn.consume这个函数

```
graph TD
A(impl_rabbit.Connection.consume)-->B(self.ensure)
B-->C(self.ensure.execute_method)
C-->D(self.consume._consume)
```



```
class Connection(object):
    def consume(self, timeout=None):
        # ...
        def _consume():
            # ...
            # 
            while self._new_tags:
                for consumer, tag in self._consumers.items():
                    if tag in self._new_tags:
                        consumer.consume(self, tag=tag)
                        self._new_tags.remove(tag)
            
            while True:
                # ...
                self.connection.drain_events(timeout=poll_timeout)
                # ...
```

consumer.consume这一步是声明回调函数，通过_new_tags标志来控制只声明一次

```
class Consumer(object):
    def consume(self, conn, tag):
        # ...
        self.queue.consume(callback=self._callback,
                           consumer_tag=six.text_type(tag),
                           nowait=self.nowait)
        # ...                   
```
self._callback是在创建queue时传进来的AMQPListener对象


self.connection.drain_events是告诉kombu connection去队列里拿消息，收到消息后，回调函数得到调用，即AMQPListener对象得到调用

这里用到了python的可调用对象特性，即将对象当做函数进行调用时，对象的__call__方法响应


```python
# oslo_messaging/_drivers/amqpdriver.py
class AMQPListener(base.PollStyleListener):
    def __call__(self, message):
        # ...
        self.incoming.append(AMQPIncomingMessage(
            self,
            ctxt.to_dict(),
            message,
            unique_id,
            ctxt.msg_id,
            ctxt.reply_q,
            self._obsolete_reply_queues,
            self._message_operations_handler)
```


self.incoming中有了数据后，调用堆栈在回到PollStyleListenerAdapter._runner

```
    def _runner(self):
        while self._started:
            incoming = self._poll_style_listener.poll(
                batch_size=self.batch_size, batch_timeout=self.batch_timeout)

            if incoming:
                self.on_incoming_callback(incoming)
```
self.on_incoming_callback是MessageHandlingServer.start时传入的回调函数self._on_incoming

```
self.listener.start(self._on_incoming)
```

看一下self._on_incoming函数

```
def _on_incoming(self, incoming):
    self._work_executor.submit(self._process_incoming, incoming)
```

self._work_executor是利用stevedore加载的执行插件，初始化在init函数


```
class MessageHandlingServer(service.ServiceBase, _OrderedTaskRunner):
    def __init__(self, transport, dispatcher, executor='blocking'):
        # ...
        mgr = driver.DriverManager('oslo.messaging.executors',
                                       self.executor_type)
        self._executor_cls = mgr.driver
        # ...
```
_executor_cls默认是用的executor='blocking'

在setup.cfg中的定义如下

```
oslo.messaging.executors =
    blocking = futurist:SynchronousExecutor
    eventlet = futurist:GreenThreadPoolExecutor
    threading = futurist:ThreadPoolExecutor
```

可以看到blocking的类为futurist:SynchronousExecutor

初始化worker的位置在MessageHandlingServer.start函数

```
class MessageHandlingServer(service.ServiceBase, _OrderedTaskRunner):
    def start(self, override_pool_size=None):
        # ...
        self._work_executor = self._executor_cls(**executor_opts)
        # ...
```

SynchronousExecutor是一个同步执行的工具，调用submit时传入函数执行体和参数

则self._process_incoming得到调用


对象实例是oslo_messaging/rpc/server.RPCServer
```
class RPCServer(msg_server.MessageHandlingServer):
    def _process_incoming(self, incoming):
        message = incoming[0]
        # ...
        res = self.dispatcher.dispatch(message)
        # ...
```

调用dispatch分发数据


```
# oslo_messaging/rpc/dispatcher.py
class RPCDispatcher(dispatcher.DispatcherBase):
    def dispatch(self, incoming):
        message = incoming.message
        ctxt = incoming.ctxt

        method = message.get('method')
        args = message.get('args', {})
        namespace = message.get('namespace')
        version = message.get('version', '1.0')
        
        for endpoint in self.endpoints:
            target = getattr(endpoint, 'target', None)
            if not target:
                target = self._default_target

            if not (self._is_namespace(target, namespace) and
                    self._is_compatible(target, version)):
                continue

            if hasattr(endpoint, method):
                if self.access_policy.is_allowed(endpoint, method):
                    return self._do_dispatch(endpoint, method, ctxt, args)

```
- 获取client消息中的method函数名
- 获取client消息中的函数参数args
- 获取client消息中的namespace和version
- 循环保存的self.endpoints
    - 检查endpoint是否定义了namespace和version，如果没有定义，则用default值
    - 与client传过来namespace和version进行比对
    - 获取endpoint中的method方法并执行
    
## producer端
RabbitDriver在producer端关于kombu的实现，可以集中在AMQPDriverBase.send这个函数中，我们下面就来着重分析这个函数

oslo_messaging/_drivers/amqpdriver.py


```
graph TD
A(AMQPDriverBase.send)-->B(AMQPDriverBase._send)
```

```
def _send(self, target, ctxt, message,
          wait_for_reply=None, timeout=None,
          envelope=True, notify=False, retry=None):
    ...
    with self._get_connection(rpc_common.PURPOSE_SEND) as conn:
        if notify:
            exchange = self._get_exchange(target)
            conn.notify_send(exchange, target.topic, msg, retry=retry)
        elif target.fanout:
            conn.fanout_send(target.topic, msg, retry=retry)
        else:
            topic = target.topic
            exchange = self._get_exchange(target)
            if target.server:
                topic = '%s.%s' % (target.topic, target.server)
            conn.topic_send(exchange_name=exchange, topic=topic,
                                    msg=msg, timeout=timeout, retry=retry)
        ...
```
分析这段函数
- 使用self._get_connection获取连接，在上面讲解consumer时注重介绍了这个函数，这里不再做说明
- 根据notify，fanout，topic的定义顺序确定是使用哪种模式进行发送

#### notify
这是RabbitDriver实现的notification功能，我们本章只分析rpc相关，暂时不分析notification

#### fanout

```
    def fanout_send(self, topic, msg, retry=None):
        """Send a 'fanout' message."""
        exchange = kombu.entity.Exchange(name='%s_fanout' % topic,
                                         type='fanout',
                                         durable=False,
                                         auto_delete=True)

        self._ensure_publishing(self._publish, exchange, msg, retry=retry)
```
- 定义了一个fanout类型的exchange


```
def _publish(self, exchange, msg, routing_key=None, timeout=None):
    ...
    with self._transport_socket_timeout(timeout):
    self._producer.publish(msg,
                           exchange=exchange,
                           routing_key=routing_key,
                           expiration=timeout,
                           compression=self.kombu_compression)
```
- 使用通过producer.publish发送出去

#### topic
topic这里与直接用kombu不同的是，如果设置了server属性，则topic需要加上server组成kombu的topic，相当于是RabbitDriver添加的一个server概念


```
    def topic_send(self, exchange_name, topic, msg, timeout=None, retry=None):
        """Send a 'topic' message."""
        exchange = kombu.entity.Exchange(
            name=exchange_name,
            type='topic',
            durable=self.amqp_durable_queues,
            auto_delete=self.amqp_auto_delete)

        self._ensure_publishing(self._publish, exchange, msg,
                                routing_key=topic, timeout=timeout,
                                retry=retry)
```

- 定义一个topic类型的exchange，并通过publish函数发出
