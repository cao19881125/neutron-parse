通过上一节的例子已经讲解了如果通过oslo-service构建一个resetful service

本节对其扩展用法进行一下说明

## 创建service的方式

### 创建单个服务进程
上一节的例子即创建了单个服务线程，是通过如下的方式进行创建的

```
from oslo_config import cfg
from oslo_service import service

CONF = cfg.CONF


service_launcher = service.ServiceLauncher(CONF)
service_launcher.launch_service(service.Service())
```
这样创建出来的wsgi server只有单个进程进行处理

### 创建多个服务进程

```
from oslo_config import cfg
from oslo_service import service

CONF = cfg.CONF

process_launcher = service.ProcessLauncher(CONF, wait_interval=1.0)
process_launcher.launch_service(service.Service(), workers=2)
```

创建launcher时选择ProcessLauncher并且在调用launch_service时通过workers参数配置进程数量


### 简单创建
以上两种创建方式可以再简化为如下方式

```
from oslo_config import cfg
from oslo_service import service

CONF = cfg.CONF
launcher = service.launch(CONF, service, workers=2)
```

通过workers值来选择创建单进程还是多进程服务


## 通过backdoor访问后门进行调试
oslo-service预留了调试的后门，打开后门后，可以通过远程连接进去debug

默认oslo-service内置了5个调试命令分别如下

| 命令  | 作用 | 说明|
|---|---|---|
|  fo | 查找对象 | 查找server栈中的所有对象 |
|  pgt| 打印greenthreads | oslo-service内部的线程使用的是eventlet.greenthreads,此命令打印所有greenthread的堆栈信息|
|  pnt| 打印native thread|打印python原生线程的堆栈信息|
| exit|退出|不能使用，仅仅打印一行无用信息|
| quit|退出|不能使用，仅仅打印一行无用信息|


对于其代码实现原理，可以查看后续章节的源码解析

我们同样是利用上一节的service.py的代码进行演示

### 添加一些变量和打印

```python
class ShowVersion():
    def __init__(self):
        self.name = 'lucy'
    def __call__(self,environ,start_response):
        start_response("200 OK",[("Content-type", "text/plain")])
        return ["Paste Deploy LAB: Version = 1.0.0",' name is:' + self.name]

```
- 在ShowVersion中添加一个name属性，并且在收到resetful
  api调用时返回值中加上name

### 修改配置文件开启backdoor

编辑config.conf文件

```
[DEFAULT]
backdoor_port = 1111
api_paste_config=./service.ini
```

- 添加backdoor_port属性，表示连接后门的端口

### 运行

```
# python service.py --config-file ./config.cfg
backdoor server listening on 127.0.0.1:1111
```

### 先看一下当前的name

```
# curl http://127.0.0.1:8090
Paste Deploy LAB: Version = 1.0.0 name is:lucy
```

- 打印出来开启了backdoor server，监听了127.0.0.1:1111端口

### 远端连接backdoor

```
# telnet 127.0.0.1 1111
Python 2.7.16 (default, Apr 12 2019, 15:32:40)
[GCC 4.2.1 Compatible Apple LLVM 10.0.1 (clang-1001.0.46.3)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
(InteractiveConsole)
>>>
```

### 获取变量

```
>>> from service import ShowVersion
>>> obj = fo(ShowVersion)
>>> obj
[<service.ShowVersion instance at 0x106d73ea8>]
```

- fo是内置的获取对象的函数，这里我获取所有ShowVersion类的对象
- 返回的是一个list，看到获取到了一个对象，即通过PasteDeploy创建的对象

### 打印并修改变量

```
>>> obj[0].name
'lucy'
>>> obj[0].name='kity'
```

- 打印当前值，为我们程序初始化时赋予的默认值'lucy'
- 修改这个值为'kity'

### 远程调用查看服务中的值是否真正修改
```
# curl http://127.0.0.1:8090
Paste Deploy LAB: Version = 1.0.0 name is:kity
```
- 可以看到server的name真的改成了'kity'


### 打印native thread信息
通过内置命令pnt打印native thread信息 

``` 
>>> pnt()
4581979584
  File "/usr/local/lib/python2.7/site-packages/eventlet/backdoor.py", line 58, in run
    console.interact()
  File "/usr/local/Cellar/python@2/2.7.16/Frameworks/Python.framework/Versions/2.7/lib/python2.7/code.py", line 243, in interact
    more = self.push(line)
  File "/usr/local/Cellar/python@2/2.7.16/Frameworks/Python.framework/Versions/2.7/lib/python2.7/code.py", line 265, in push
    more = self.runsource(source, self.filename)
  File "/usr/local/Cellar/python@2/2.7.16/Frameworks/Python.framework/Versions/2.7/lib/python2.7/code.py", line 87, in runsource
    self.runcode(code)
  File "/usr/local/Cellar/python@2/2.7.16/Frameworks/Python.framework/Versions/2.7/lib/python2.7/code.py", line 103, in runcode
    exec code in self.locals
  File "<console>", line 1, in <module>
  File "/usr/local/lib/python2.7/site-packages/oslo_service/eventlet_backdoor.py", line 131, in _print_nativethreads
    traceback.print_stack(stack)
```

### 打印greenthread信息
通过内置命令pgt打印eventlet greenthread信息

```
>>> pgt()
0 <greenlet.greenlet object at 0x10d2020f0>
  File "/usr/local/lib/python2.7/site-packages/eventlet/hubs/hub.py", line 346, in run
    self.wait(sleep_time)
  File "/usr/local/lib/python2.7/site-packages/eventlet/hubs/kqueue.py", line 107, in wait
    readers.get(fileno, noop).cb(fileno)
  File "/usr/local/lib/python2.7/site-packages/eventlet/backdoor.py", line 66, in switch
    greenlets.greenlet.switch(self, *args, **kw)

1 <eventlet.greenthread.GreenThread object at 0x10d2022d0>
  File "/usr/local/lib/python2.7/site-packages/eventlet/greenthread.py", line 214, in main
    result = function(*args, **kwargs)
  ...

2 <eventlet.greenthread.GreenThread object at 0x10d202370>
  File "/usr/local/lib/python2.7/site-packages/eventlet/greenthread.py", line 214, in main
    result = function(*args, **kwargs)
  ...

3 <eventlet.greenthread.GreenThread object at 0x10d202410>
  File "/usr/local/lib/python2.7/site-packages/eventlet/greenthread.py", line 214, in main
    result = function(*args, **kwargs)
  ...

4 <eventlet.backdoor.SocketConsole object at 0x10d2024b0>
  File "/usr/local/lib/python2.7/site-packages/eventlet/backdoor.py", line 58, in run
    console.interact()
  ...

5 <greenlet.greenlet object at 0x10bc18b90>
  File "service.py", line 75, in <module>
  ...
```

- 可以看到有5个greenthread创建，并且把每个thread的当前堆栈都打印了出来，由于内容太多我进行了省略

