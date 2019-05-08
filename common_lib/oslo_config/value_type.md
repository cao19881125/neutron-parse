## 参数类型
oslo-config中支持多种类型，所有支持的类型列举如下

### boolean value (BoolOpt)
Enables or disables an option. The allowed values are true and false.
```
# Enable the experimental use of database reconnect on
# connection lost (boolean value)
use_db_reconnect = false
```

### floating point value (FloatOpt)
A floating point number like 0.25 or 1000.

```
# Sleep time in seconds for polling an ongoing async task
# (floating point value)
task_poll_interval = 0.5
```

### integer value (IntOpt)
An integer number is a number without fractional components, like 0 or 42.
```
# The port which the OpenStack Compute service listens on.
# (integer value)
compute_port = 8774
```

### IP address (IPOpt)
An IPv4 or IPv6 address. 
``` 
# Address to bind the server. Useful when selecting a particular network
# interface. (ip address value)
bind_host = 0.0.0.0
```

### key-value pairs (DictOpt)
A key-value pairs, also known as a dictionary. The key value pairs are separated by commas and a colon is used to separate key and value. Example: key1:value1,key2:value2.

```
# Parameter for l2_l3 workflow setup. (dict value)
l2_l3_setup_params = data_ip_address:192.168.200.99, \
   data_ip_mask:255.255.255.0,data_port:1,gateway:192.168.200.1,ha_port:2
```

### list value (ListOpt)
Represents values of other types, separated by commas. As an example, the following sets allowed_rpc_exception_modules to a list containing the four elements oslo.messaging.exceptions, nova.exception, cinder.exception, and exceptions:

```
# Modules of exceptions that are permitted to be recreated
# upon receiving exception data from an rpc call. (list value)
allowed_rpc_exception_modules = oslo.messaging.exceptions,nova.exception
```

### multi valued (MultiStrOpt)
A multi-valued option is a string value and can be given more than once, all values will be used.

```
# Driver or drivers to handle sending notifications. (multi valued)
notification_driver = nova.openstack.common.notifier.rpc_notifier
notification_driver = ceilometer.compute.nova_notifier
```

### port value (PortOpt)
A TCP/IP port number. Ports can range from 1 to 65535.

```
# Port to which the UDP socket is bound. (port value)
# Minimum value: 1
# Maximum value: 65535
udp_port = 4952
```

### string value (StrOpt)
Strings can be optionally enclosed with single or double quotes.

```
# The format for an instance that is passed with the log message.
# (string value)
instance_format = "[instance: %(uuid)s] "
```