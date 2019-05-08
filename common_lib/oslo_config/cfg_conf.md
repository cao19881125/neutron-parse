# cfg.CONF解析
本章对cfg.CONF的一些关键函数进行了分析，目标是通过本章的学习，能够了解到一些cfg.CONF关键部分的代码细节

## cfg.CONF与arpparse
首先进行说明一下，cfg.CONF是通过argparse来实现的，后面的讲解将围绕如何通过argparse来实现命令行与文件解析，这里先简单介绍一下argparse

argparse 是 Python 内置的一个用于命令项选项与参数解析的模块，通过在程序中定义好我们需要的参数，argparse 将会从 sys.argv 中解析出这些参数，并自动生成帮助和使用信息。

### 简单示例

```
import argparse

parser = argparse.ArgumentParser()
parser.add_argument('integer', type=int, help='display an integer')
args = parser.parse_args()

print args.integer
```

使用argparse主要有四个步骤

1. 创建 ArgumentParser() 对象
2. 调用 add_argument() 方法添加参数
3. 使用 parse_args() 解析参数
4. 通过返回值args.变量名来获取变量


## cfg.CONF的实现
上一节我们讲解了argparse的四个步骤，本节我们将分析cfg.CONF的处理方式，并结合上节的四个关键步骤来分析cfg.CONF是如何实现的

### 对象定义
cfg.CONF的定义在oslo_config/cfg.py中，如下 

```
CONF = ConfigOpts()
```

在cfg.py文件的最后一行，定义了CONF这个全局对象，初始化的类为ConfigOpts

### 注册变量
我们通过cfg.CONF.register_opt来注册单个变量，代码如下

#### DEFAULT group的处理
创建默认属性时，group不填或者填None，即表示创建在DEFAULT组
```python
# oslo_config/cfg.py

class ConfigOpts(collections.Mapping):
    disallow_names = ('project', 'prog', 'version',
                      'usage', 'default_config_files', 'default_config_dirs')
    def register_opt(self, opt, group=None, cli=False):
        # ...
        if group is None:
            if opt.name in self.disallow_names:
                raise ValueError('Name %s was reserved for oslo.config.'
                                 % opt.name)   
        # ...
        self._opts[opt.dest] = {'opt': opt, 'cli': cli}
        # ...                              
```

- 先跟disallow_names比较一下看一下变量名是不是合法的
- opt.dest 获取的是变量名，有一个小处理，即将变量名中的'-'替换成'_'，如下

```python
class Opt(object):
    def __init__(self, name):
        self.name = name
        self.dest = self.name.replace('-', '_')
        # ...
```

- 将opt.dest作为key保存到self._opts字典中，value为opt对象


#### 非DEFAULT group处理
调用register_opt时传入group属性，则将此变量和对应的group进行关联

```python
# oslo_config/cfg.py
class ConfigOpts(collections.Mapping):
    def register_opt(self, opt, group=None, cli=False):
        if group is not None:
            group = self._get_group(group, autocreate=True)
            if cli:
                self._add_cli_opt(opt, group)
            return group._register_opt(opt, cli)
        # ...
        
    def _get_group(self, group_or_name, autocreate=False):
        group = group_or_name 
        group_name = group.name 
        
        if group_name not in self._groups:
            self.register_group(group or OptGroup(name=group_name))
        return self._groups[group_name]    
        
    def register_group(self, group):
        if group.name in self._groups:
            return

        self._groups[group.name] = copy.copy(group)                       
```

```python
class OptGroup(object):
    def _register_opt(self, opt, cli=False):
        self._opts[opt.dest] = {'opt': opt, 'cli': cli}

        return True        
```

- register_opt的group为OptGroup类型
- 通过register_group函数创建group，保存到self._groups中
- 并通过OptGroup._register_opt将变量注册到self._opts中

#### 注册变量总结
- ConfigOpts类有两个字典：
  - self._opts字典保存DEFAULT组的变量，key为变量名，value为变量对象
  - self._groups字典保存所有的组，key为组名，value为组对应的OptGroup对象
- OptGroup类有一个字典，self._opts保存这个组的变量，字典结构同ConfigOpts._opts
- 所有的注册变量，都会按照以上的结构保存在cfg.CONF这个对象中

### 配置命令行参数

命令行参数是通过如下方式传递给ConfigOpts的

```
cfg.CONF(sys.argv[1:])
```

这里用到了python的可调用对象特性，即我们可以通过在对象后加()的方式，像调用函数一样调用对象

当如此调用对象时，对象的\_\_call\_\_方法得到调用

从这里开始，cfg.CONF将开始与argparse产生关联

#### 第一步：自定义ArgumentParser 和 Action
```
class _CachedArgumentParser(argparse.ArgumentParser)
```
- 扩展了ArgumentParser实现了_CachedArgumentParser

- \_\_call\_\_ -> self._pre_setup 中进行了初始化

```
self._oparser = _CachedArgumentParser()
```

- 同时自定义了两个Action
  分别为ConfigFileAction和ConfigDirAction，顾名思义，这两个Action是分别对文件和目录进行处理的
```
class ConfigFileAction(argparse.Action)

class ConfigDirAction(argparse.Action)
```
argparser内部实现了很多action，默认的action为store，即保存参数值

cfg扩展实现了Action，实现了两个action，分别为ConfigFileAction和ConfigDirAction，分别对应命令行中的--config-file和--config-dir命令

#### 第二步：添加注册命令行参数

##### 初始化_ConfigFileOpt和_ConfigDirOpt
这两个Option分别是处理文件和目录的，他们关联了上节两个action，后面将讲解如果通过这两个action解析文件和目录
```
self._config_opts = self._make_config_options(default_config_files,
                                                      default_config_dirs)
                                               
def _make_config_options(default_config_files, default_config_dirs):
    return [
        _ConfigFileOpt('config-file',
                       default=default_config_files,
                       metavar='PATH',
                       help=('Path to ...')),
        _ConfigDirOpt('config-dir',
                      metavar='DIR',
                      default=default_config_dirs,
                      help='Path to ...'),
    ]                                                      
```

###### 构造action

\_\_call\_\_ -> self._parse_cli_opts(args)


```python
self._args = args
for opt, group in self._all_cli_opts():
    opt._add_to_cli(self._oparser, group)
```
- 在neutron启动命令中，cli opts中只有--config-file ，我们先假设只有--config-file而没有其他的opt

- 那么opt的类型就是_ConfigFileOpt，调用到的是其父类Opt的_add_to_cli函数


```
class Opt(object):
    def _add_to_cli(self, parser, group=None):
        container = self._get_argparse_container(parser, group)
        kwargs = self._get_argparse_kwargs(group)
        # ...
        self._add_to_argparse(parser, container, self.name, self.short,
                              kwargs, prefix,
                              self.positional, deprecated_names)
```
- container在DEFAULT组情况下，获取到的还是_CachedArgumentParser
- self._get_argparse_kwargs这里，调用到的是_ConfigFileOpt多态实现的_get_argparse_kwargs函数

```
class _ConfigFileOpt(Opt):
    def _get_argparse_kwargs(self, group, **kwargs):
    kwargs = super(_ConfigFileOpt, self)._get_argparse_kwargs(group)
    kwargs['action'] = self.ConfigFileAction
    return kwargs
```
- 可以看到这里创建了action为ConfigFileAction并通过字典返回

- 继续通过self._add_to_argparse -> _CachedArgumentParser.add_parser_argument

```
class _ConfigFileOpt(Opt):
    def _add_to_argparse(self, parser, container, name,...)
        # ...
        args = [hyphen('--') + prefix + name]
        # ...
        parser.add_parser_argument(container, *args, **kwargs)

class _CachedArgumentParser(argparse.ArgumentParser):
    def add_parser_argument(self, container, *args, **kwargs):
        values = []
        values.append({'args': args, 'kwargs': kwargs})
        self._args_cache[container] = values
```
- self.name为'config-file'，在前面加上--，符合argparser的用法
- 到这里，args和ConfigFileAction保存在了self._args_cache中

##### 通过add_argument注册参数
通过以上几步构造出来了调用argparser.add_argument的几个关键参数，下面就要对其进行注册

还是回到ConfigOpts.__call__函数，继续向下走

__call__ -> self._parse_cli_opts(args) -> self._parse_config_files() -> self._oparser.parse_args(self._args, namespace) -> _CachedArgumentParser.parse_args(self, args=None, namespace=None) -> _CachedArgumentParser.initialize_parser_arguments(self)

```
class _CachedArgumentParser(argparse.ArgumentParser):
    def initialize_parser_arguments(self):
        for container, values in self._args_cache.items():
            # ...
            container.add_argument(*argument['args'],**argument['kwargs'])
            # ...
```

- 因为container就是_CachedArgumentParser，其父类是argparse.ArgumentParser，这个函数相当于是在argparser中注册了一个命令行参数，并且通过kwargs中的action注册了action为ConfigFileAction

#### 第三步：解析参数和文件
通过上面的解释，已经知道了如何通过扩展的argparse.ArgumentParser将命令行参数'--config-file'和自定义的ConfigFileAction注册进去，本节讲解如何解析文件

在argparse中，参数只会通过命令行传递，只需要解析命令行参数即可

在cfg.CONF的实现中，是通过命令行传递的配置文件路径，cfg.CONF解析出来路径后，通过自定义的ConfigFileAction回调解析对应的配置文件，获取参数


在这里，因为是通过_CachedArgumentParser扩展了ArgumentParser，所以要找到如何通过_CachedArgumentParser调用到ArgumentParser.parse_args()

还是顺着__call__函数进行梳理

\_\_call\_\_ -> self._parse_cli_opts(args) -> self._parse_config_files() -> self._oparser.parse_args(self._args, namespace) -> _CachedArgumentParser.parse_args(self, args=None, namespace=None) 

```
class _CachedArgumentParser(argparse.ArgumentParser):
    def parse_args(self, args=None, namespace=None):
        self.initialize_parser_arguments()
        return super(_CachedArgumentParser, self).parse_args(args, namespace)
```

- 可以看到，initialize_parser_arguments是注册变量的最后一步，后面紧接着调用了父类，即ArgumentParser的parse_args()函数

- 这个函数的调用会进入到argparser的栈中，我们暂不分析

- argparser经过一系列处理，最终会调用到自定义action的__call__方法，我们上面分析了，在这里的自定义action为ConfigFileAction

```
class ConfigFileAction(argparse.Action):
    def __call__(self, parser, namespace, values, option_string=None):
        ...
        ConfigParser._parse_file(values, namespace)
```

- 这里调用到了ConfigParser进行文件解析

```
class ConfigParser(iniparser.BaseParser):
    @classmethod
    def _parse_file(cls, config_file, namespace):
        ...
        parser = cls(config_file, sections)
        arser.parse()
        ...
        namespace._add_parsed_config_file(config_file, sections, normalized)
        
```
- 这里构造一个ConfigParser的实例，并调用parse解析文件
- parse的代码就不贴了，大概的意思是通过iniparser.BaseParser这个python库将文件进行读取分析，并保存到self.sections中，解析后的结果如下所示

```
{'DEFAULT': {'option3': ['true']}, 'mygroup': {'option1': ['foo']}}
```
- 即字典的key为组名，value为值的字典，value的字典以变量名为key，值为value
- 然后调用namespace的_add_parsed_config_file函数

```
class _Namespace(argparse.Namespace):
    def _add_parsed_config_file(self, filename, sections, normalized):
        for s in sections:
            self._sections_to_file[s] = filename
        self._parsed.insert(0, sections)
        self._normalized.insert(0, normalized)   
```
- 通过这个函数，sections被保存到了self._parsed中

#### 第四步：获取变量
通过以上的解析，已经知道了cfg.CONF如何通过命令行中的'--config-file'解析配置文件参数的，本节介绍如何获取参数，以如下的形式

```
cfg.CONF.mygroup.option1
cfg.CONF.mygroup.option2
cfg.CONF.option3
```

当直接以.来访问变量时，python会调用内置函数__getattr__，则ConfigOpts.__getattr__得到响应

```
class ConfigOpts(collections.Mapping):
    def __getattr__(self, name):
        return self._get(name)
    
    def _get(self, name, group=None, namespace=None):
        if isinstance(group, OptGroup):
            key = (group.name, name)
        else:
            key = (group, name)
            
        value, loc = self._do_get(name, group, namespace)
        return value
    
    def _do_get(self, name, group=None, namespace=None):
        # ...
        val, alt_loc = opt._get_from_namespace(namespace, group_name)
        return (convert(val), alt_loc)
        # ...
    
        
```
- 通过__getattr__一路调用到了Opt._get_from_namespace
- 次数的Opt是name所对应类型的Opt扩展类，如option3即BoolOpt

```
class Opt(object):
    def _get_from_namespace(self, namespace, group_name):
        # ...
        value, loc = namespace._get_value(
            names, multi=self.multi,
            positional=self.positional, current_name=current_name)
        # ...
        return (value, loc)
```
- 调用到了namespace的_get_value方法

```
class _Namespace(argparse.Namespace):
    def _get_value(self, names, multi=False, positional=False,
                   current_name=None, normalized=True):
        
        # ...
        values, loc = self._get_file_value(
                file_names, multi=multi, normalized=normalized,
                current_name=current_name)
        
        # ...
        return (values if multi else values[-1], loc)
    
    def _get_file_value(self, names):
        # ...
        for sections in (self._normalized if normalized else self._parsed):
            for section, name in names:
                # ...
                val = sections[section][name]
                # ...
        
        if multi and rvalue != []:
            return (rvalue, loc)        
```
- 从namespace中的self._parsed中，取到对应group和name的值

- 在文件解析章节，最终文件解析的结果是将文件中的值保存到了self._parsed中，此处即从self._parsed中再取出来

# 总结
通过以上的分析，梳理了cfg.CONF从'--config-file'参数获取文件名，到解析文件中的值，到取值的过程，本例只分析--config-file的解析过程，--config-dir的解析过程基本一致，区别仅为通过搜索目录下的.conf文件来定位config-file，本例不再进行分析

