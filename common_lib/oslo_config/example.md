## 例子
以下两个例子演示了通过两种方式使用oslo-config模块的方法，分别为通过命令行传递参数和通过配置文件传递参数
### cli传递参数
> oslo-config可以通过读取命令行来获取参数，例子代码如下

- 编写osloclitest.py

```
import sys

from oslo_config import cfg

cli_opts = [cfg.StrOpt('option1'),
            cfg.IntOpt('option2', default=42)]

cfg.CONF.register_cli_opts(cli_opts)

cfg.CONF(sys.argv[1:])

print("The value of option1 is %s" % cfg.CONF.option1)
print("The value of option2 is %d" % cfg.CONF.option2)
```

- 运行

```
# python osloclitest.py --option1 100
The value of option1 is 100
The value of option2 is 42
```

- 可以看到通过命令行指定了option1为100，没有指定option2，option2使用了默认值42


### 解析
- cli_opts是参数列表，类型是字典，每个参数是又cfg中定义的类型
- 这里定义了两个参数，其中option1是string类型，option2为int类型，并且通过default属性给option2指定了默认值
- 通过register_cli_opts将cli_opts注册为命令行参数
- sys.argv\[1:\]表示所有的启动参数，传入cfg.CONF模块
- 在代码中直接通过cfg.CONF.变量名的形式进行变量访问
- cli命令中，如果不通过命令行指定参数:
  - 如果参数有默认值，如option2，则使用默认值
  - 如果参数没有默认值，则值为None
- cli命令中，指定了参数的话，则使用命令行的值，无论是否有默认值


### 通过配置文件传递参数

官网给出了一个简单的例子

- 编写oslocfgtest.py文件

```
import sys

from oslo_config import cfg

grp = cfg.OptGroup('mygroup')

opts = [cfg.StrOpt('option1'),
        cfg.IntOpt('option2', default=42)]

cfg.CONF.register_group(grp)

cfg.CONF.register_opts(opts, group=grp)
cfg.CONF.register_opt(cfg.BoolOpt('option3'))

cfg.CONF(sys.argv[1:])

print("The value of option1 is %s" % cfg.CONF.mygroup.option1)
print("The value of option2 is %d" % cfg.CONF.mygroup.option2)
print("The value of option3 is %s" % str(cfg.CONF.option3))
```

- 编写oslocfgtest.conf配置文件 

```
[DEFAULT]
option3 = true

[mygroup]
option1 = foo
# Comment out option2 to test the default value
# option2 = 123
```

- run

```
python oslocfgtest.py --config-file oslocfgtest.conf
```

- output

```
The value of option1 is foo
The value of option2 is 42
```

- 创建了一个名为oslocfgtest.conf的配置文件，并且在配置文件中配置了两个组，DEFAULT组合mygroup组
  -   DEFAULT组下定义了一个参数为option3
  -   mygroup组下面有一个参数option1并且值设为了foo
- 在代码中先分别定义了一个mygroup组和option1，option2两个变量，option2变量指定了默认值
- 通过register_opts函数注册了mygroup组和两个变量，并且将变量与mygroup组进行了绑定
- 通过register_opt函数注册了一个DEFAULT组的变量option3
- 通过sys.argv\[1:\]取到了命令行参数，传给cfg模块
- 直接用cfg.CONF.组名.变量名进行变量访问
- 如果不加组名，则默认是DEFAULT组下的变量
- 如果指定了默认值，则在文件中没有进行重新定义变量的情况下，使用默认值，否则使用文件中定义的值