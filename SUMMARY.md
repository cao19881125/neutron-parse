# Summary

## 简介
* [前言](README.md)

* [序章](forward.md)

## oslo-config配置解析
* [安装](common_lib/oslo_config/config.md)
* [例子](common_lib/oslo_config/example.md)
* [变量类型](common_lib/oslo_config/value_type.md)
* [cfg.CONF实现解析](common_lib/oslo_config/cfg_conf.md)
* [neutron与cfg.CONF](common_lib/oslo_config/neutron_conf.md)

## oslo-service构建resetful server
* [简介](common_lib/oslo_service/define.md)
* [eventlet.wsgi](common_lib/oslo_service/wsgi_eventlet.md)
* [PasteDeploy](common_lib/oslo_service/pastedeploy.md)
* [oslo-service使用](common_lib/oslo_service/oslo_service.md)
* [oslo-service其他用法](common_lib/oslo_service/service_usage.md)
* [oslo-service源码解析](common_lib/oslo_service/code_parse.md)
* [neutron与oslo-service](common_lib/oslo_service/neutron_oslo_service.md)

## oslo-message搭建rpc通信框架
* [简介](common_lib/oslo_messaging/define.md)
* [kombu](common_lib/oslo_messaging/kombu.md)
* [oslo-message使用](common_lib/oslo_messaging/oslo_message_usage.md)
* [RabbitDriver解析](common_lib/oslo_messaging/rabbit_driver.md)

## oslo-db操作数据库
* [简介](common_lib/oslo_db/define.md)

## oslo-log记录日志
* [简介](common_lib/oslo_log/define.md)

## core-plugin二层业务处理
* [简介](core_plugin/define.md)

## service-plugin三层业务处理
* [简介](service_plugin/define.md)

## mechanism-driver构建sdn桥梁
* [简介](mechanism_driver/define.md)

### openvswitch架构
* [简介](openvswitch/define.md)

### ovn架构
* [简介](ovn/define.md)

### 自己实现一个mechanism-driver
* [简介](mechanism_self/define.md)