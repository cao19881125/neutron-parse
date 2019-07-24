
上一章介绍了rabbitmq的概念，本章介绍一个python实现的amqp库kumbo

之所以要介绍kumbo，是因为在oslo-messaging中，kombu是openstack官方实现的amqp
driver，其他的driver还有ZeroMQ,kafka,pika等

通过本章的学习，你将知道如何使用kumbo来连接rabbitmq，并在rabbitmq上创建exchange与queue，生产消息以及消费消息


## 几个重要步骤
我们在上一章讲解rabbitmq时，讲解了一些重要的概念，其中包括connection,exchange,queue,producer,consumer等

本章通过代码实践，将这些概念穿插起来，让读者能够通过真实的例子了解到这些概念的作用

通过总结，consumer端和producer端的步骤如下

### consumer端步骤
1. 定义connection
2. 定义exchange
3. 定义queue，绑定exchange与routing_key
4. 定义consumer，绑定queue与回调函数
5. 启动服务，处理回调

### producer端步骤
1. 定义connection
2. 定义exchange
3. 定义producer
4. 通过producer向exchange发送消息

# 代码实例
本节通过代码实例来讲解kombu的用法，代码实例按照三种exchange分别举一个例子


## direct exchange
direct exchange根据确定的routing_key进行消息路由

### consumer

recver.py

```python
import sys
from kombu import Exchange, Queue,Consumer
from kombu import Connection


conn = Connection('amqp://guest:guest@192.168.184.128:5672//')

task_exchange = Exchange('kombu_test', type='direct')

task_queues = []

severity = sys.argv[1]

if not severity:
    sys.stderr.write("Usage: %s [info] [warning] [error]\n" % sys.argv[0])
    sys.exit(1)


q = Queue(severity, task_exchange, channel=conn.channel(),routing_key=severity)
q.declare()

def _callback( body, message):
    print(" [x] %r:%r" % (message.delivery_info['routing_key'], body))
    message.ack()

consumer = Consumer(conn.channel(),queues=[q],accept=['pickle', 'json'],callbacks=[_callback])

consumer.consume()

while True:
    conn.drain_events()
```

### producer
sender.py
```
import sys
from kombu.pools import producers
from kombu import Exchange, Queue
from kombu import Connection


task_exchange = Exchange('kombu_test', type='direct')

connection = Connection('amqp://guest:guest@192.168.184.128:5672//')

severity = sys.argv[1] if len(sys.argv) > 1 else 'info'
message = ' '.join(sys.argv[2:]) or 'Hello World!'

with producers[connection].acquire(block=True) as producer:
    producer.publish(message,
                     serializer='pickle',
                     compression='bzip2',
                     exchange=task_exchange,
                     declare=[task_exchange],
                     routing_key=severity)

print(" [x] Sent %r:%r" % (severity, message))
```

### run
- consumer 
```
python recver.py key1
```

- producer 
```
python sender.py key1 hello world!
```

- consumer收到消息

```
python recver.py key1
 [x] u'key1':'hello world!'
```

## fanout exchange
### consumer
recver.py
```
from kombu import Exchange, Queue
from kombu import Connection
from kombu.mixins import ConsumerMixin
import uuid


task_exchange = Exchange('kombu_test_fanout', type='fanout')

task_queues = [Queue(str(uuid.uuid4()), task_exchange)]


class Worker(ConsumerMixin):

    def __init__(self, connection):
        self.connection = connection

    def get_consumers(self, Consumer, channel):
        return [Consumer(queues=task_queues,
                         accept=['pickle', 'json'],
                         callbacks=[self.process_task])]

    def process_task(self, body, message):
        print(" [x] %r:%r" % (message.delivery_info['routing_key'], body))
        message.ack()


with Connection('amqp://guest:guest@192.168.184.128:5672//') as conn:
    try:
        worker = Worker(conn)
        worker.run()
    except KeyboardInterrupt:
        print('bye bye')
```

### producer
sender.py
```
import sys
from kombu.pools import producers
from kombu import Exchange, Queue
from kombu import Connection

task_exchange = Exchange('kombu_test_fanout', type='fanout')

connection = Connection('amqp://guest:guest@192.168.184.128:5672//')

message = ' '.join(sys.argv[1:]) or 'Hello World!'

with producers[connection].acquire(block=True) as producer:
    producer.publish(message,
                     serializer='pickle',
                     compression='bzip2',
                     exchange=task_exchange,
                     declare=[task_exchange])

print(" [x] Sent %r" % ( message))
```


### run

启动两个consumer

```
python recver.py
```

启动一个producer

```
python sender.py how are you
 [x] Sent 'how are you'
```

两个consumer均收到数据

```
# python recver.py
 [x] u'key1':'how are you'
```

```
# python recver.py
 [x] u'key1':'how are you'
```


## topic exchange
### consumer
recver.py
```
import sys
from kombu import Exchange, Queue
from kombu import Connection
from kombu.mixins import ConsumerMixin



task_exchange = Exchange('kombu_test_topic', type='topic')

task_queues = []

severities = sys.argv[1:]

if not severities:
    sys.stderr.write("Usage: %s [info] [warning] [error]\n" % sys.argv[0])
    sys.exit(1)

for severity in severities:
    task_queues.append(Queue(severity, task_exchange, routing_key=severity))

class Worker(ConsumerMixin):

    def __init__(self, connection):
        self.connection = connection

    def get_consumers(self, Consumer, channel):
        return [Consumer(queues=task_queues,
                         accept=['pickle', 'json'],
                         callbacks=[self.process_task])]

    def process_task(self, body, message):
        print(" [x] %r:%r" % (message.delivery_info['routing_key'], body))
        message.ack()


with Connection('amqp://guest:guest@192.168.184.128:5672//') as conn:
    try:
        worker = Worker(conn)
        worker.run()
    except KeyboardInterrupt:
        print('bye bye')
```

### producer
sender.py
```
import sys
from kombu.pools import producers
from kombu import Exchange, Queue
from kombu import Connection


task_exchange = Exchange('kombu_test_topic', type='topic')

connection = Connection('amqp://guest:guest@192.168.184.128:5672//')

severity = sys.argv[1] if len(sys.argv) > 1 else 'info'
message = ' '.join(sys.argv[2:]) or 'Hello World!'

with producers[connection].acquire(block=True) as producer:
    producer.publish(message,
                     serializer='pickle',
                     compression='bzip2',
                     exchange=task_exchange,
                     declare=[task_exchange],
                     routing_key=severity)

print(" [x] Sent %r:%r" % (severity, message))
```

### run
recver1监听green.*


```
# python recver.py "green.*"
```

recver2监听#.dog

```
# python recver.py "#.dog"
```

sender发送
1. 以green.dog为key发送

```
python sender.py green.dog this is tom
 [x] Sent 'green.dog':'this is tom'
```
两个都能收到

2. 以blue.dog为key发送

只有recver2能收到

3.以old.green.dog发送

只有recver2能收到