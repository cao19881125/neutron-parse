# neutron源码解析

本书主要从neutron代码层面入手，通过分析neutron代码的架构设计、模块/插件设计、对象设计、业务流程、函数属性等，将neutron以及背后的sdn技术进行剖析

### 本书面向的读者为：
- 对openstack有一定的部署、操作经验，希望了解其内部架构的
- 对neutron设计到的sdn技术，如openvswitch,linuxbridge,ovn等感兴趣，希望进行学习的
- 需要对neutron代码进行开发改造的

### 通过本书你将学习到如下内容
- neutron用到的openstack公共模块，如:oslo-config,oslo-db,oslo-log,oslo-messing,oslo-service等
- neutron的模块及插件设计，包括:core-plugin,service-plugin,mechanism-driver 等，以及他们的内部架构
- neutron包含的sdn技术，以及这些技术相对的业务架构，如linux-bridge，openvswitch,ovn等
- 如何自己开发一款mechanism-driver插件


---

- 作者：曹云涛，Github [@cao](https://github.com/cao19881125)
- 本书github地址: [neutron-parse](https://github.com/cao19881125/neutron-parse)

