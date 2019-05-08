# neutron源码解析

本书主要从neutron代码层面入手，通过分析neutron代码的架构设计、模块/插件设计、对象设计、业务流程、函数属性等，将neutron以及背后的sdn技术进行剖析

### 本书面向的读者为：
- 对openstack neutron有一定的部署、操作经验，希望了解其内部架构的
- 对neutron设计到的sdn技术，如openvswitch,linuxbridge,ovn等感兴趣，希望进行学习的
- 需要对neutron代码进行开发改造的

### 通过本书你将学习到如下内容
- neutron用到的openstack公共模块，如:oslo-config,oslo-db,oslo-log,oslo-messing,oslo-service等
- neutron的模块及插件设计，包括:core-plugin,service-plugin,mechanism-driver 等，以及他们的内部架构
- neutron包含的sdn技术，以及这些技术相对的业务架构，如linux-bridge，openvswitch,ovn等
- 如何自己开发一款mechanism-driver插件


### 备注：
本书使用的代码为openstack ocata版本

笔者学习的思路一般为不管三七二十一先写测试代码，把测试代码跑起来，然后再反过头来分析，这样的好处是能先有一些成就感，然后更有动力去学习其中的原理

所以本书也按照这个思路，会放一些例子代码上来，并且附上测试运行的过程，然后再来分析其中的原理

由于某些函数的代码量比较大不便于阅读，所以我会精简代码，包括函数定义中去掉不相关的参数，函数实现中去掉不相关的异常处理、不相关的逻辑处理，将类或者函数重新排版等，只将重要的部分纳入进来

---

- 作者：曹云涛，Github [@cao](https://github.com/cao19881125)
- 邮件：caoyuntao1125@gmail.com
- 本书github地址: [neutron-parse](https://github.com/cao19881125/neutron-parse)

