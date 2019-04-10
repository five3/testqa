@[TOC]

# 前言
每个语言都会有自己的日志模块，Python也不例外。通常情况下当需要使用到日志的时候，
一般都是匆匆查找下资料，按照步骤进行下配置就是完事了，不太会去总结日志模块的使用方式。
经常过一段时间新项目需要用的时候，还是需要去网上搜索下配置方式。所以今天就为了日后的使用方便而进行的内容整理。

# 使用默认配置记录日志
Python的日志模块是logging，属于Python的标准发行库。如果你只是用于调试程序，又不想用`print`函数的情况下。
倒是可以直接使用logging的默认配置来进行调试信息的打印。比如：
```python
import logging

logging.debug('debug')
logging.info('info')
logging.warning('warning')
```
这段代码的运行结果如下：（日志格式为：日志等级：Logger名称：用户信息）
```bash
WARNING:root:warning
```
可以看出，默认情况下logging的日志级别为`warning`，并不是`info`。当然你也可以通过如下设置来更改日志级别。
```python
import logging
logging.basicConfig(level=logging.INFO,
                    format="[%(asctime)s][%(name)s][%(levelname)s] => %(message)s",
                    datefmt='%Y-%m-%d  %H:%M:%S %a' 
                    )
logging.info('info')                  
```
这段代码不仅设置了默认的日志等级，同时还设置了日志的输出格式和日期格式。该段代码的运行结果如下：
```bash
[2019-03-23  17:57:31 Sat][root][INFO] => info
```
关于更多的日志格式属性说明，请参见[官方说明文档](https://docs.python.org/3/library/logging.html#logrecord-attributes)

## 写入到日志文件
前面都是直接在命令行输出日志信息，有时候你可能希望输入到日志文件中。那么你也可以通过basicConfig方法来进行设置。比如：
```python
import logging
logging.basicConfig(level=logging.INFO,
                    format="[%(asctime)s][%(name)s][%(levelname)s] => %(message)s",
                    datefmt='%Y-%m-%d  %H:%M:%S %a',
                    filename='my.log',
                    filemode='a'
                    )
logging.info('info')
```
该段代码执行后，不会在命令行有任何信息的输出，但是会在代码执行目录生成一个`mylog`的日志文件。里面的内容为：
```bash
[2019-03-23  17:58:27 Sat][root][INFO] => info
```
另外，这个设置中还规定了文件日志的写入模式为`a`，即追加写入模式。重新再次执行该代码时会在原日志内容后追加新日志。
而如果你希望每次执行都是覆盖原来的日志内容，则可以把写入模式该为`w`，即写入模式。

## 日志信息格式化
前面作为介绍方便，打印的信息都是比较简单的内容；而有时候我们还是会打印一些比较复杂的、需要拼接的字符内容。
此时就需要使用格式化的功能来完成了，除了我们提前把日志信息格式化好，Logging的日志方法也提供了格式化的调用。
```python
import logging
logging.warning('this is %s', 'warning')
```

> 需要注意的是，`logging.basicConfig`方法只能配置一次，配置多次时也只有第一次会生效。

# 通过代码配置日志
上面是直接使用logging模块的方法来记录日志信息的，这时用到的是Logger是顶级的Logger，名字为`root`，是个单例对象。
其实除了这个顶级的Logger之外，还可以生成其它的Logger对象。下面就来做个介绍。

首先，想要获取Logger对象，可以通过logging.getLogger方法来实现。比如：
```python
import logging

logging.basicConfig(level=logging.INFO,
                    format="[%(asctime)s][%(name)s][%(levelname)s] => %(message)s",
                    datefmt='%Y-%m-%d  %H:%M:%S %a' 
                    )
Logger = logging.getLogger()    # 获取的是名为root的Logger
Logger.warning('this is warning')

Logger = logging.getLogger(__name__)    # 获取的是名为__main__的Logger
Logger.warning('this is warning')
```
这段代码在命令行打印的信息如下：
```bash
[2019-03-23  18:52:39 Sat][root][WARNING] => this is warning
[2019-03-23  18:52:39 Sat][__main__][WARNING] => this is warning
```
可以看到`%(name)s`占位符的内容分别是root和__main__。当然你也可以获取指定名字的Logger。
比如：`logging.getLogger('test')`则会获取一个名为test的Logger。

## 不同Logger进行不同设置
通过上面的代码，我们已经可以获取多个Logger对象了，那么就有需求对不同的Logger进行日志设置。不同Logger分别不同设置的操作如下：
```python
import logging


formatter = logging.Formatter(
    fmt="[%(asctime)s][%(name)s][%(levelname)s] => %(message)s",
    datefmt="%Y-%m-%d  %H:%M:%S %a"
)

Logger = logging.getLogger('test')
Logger.setLevel(logging.INFO)
FH = logging.FileHandler("my.log", encoding="utf-8")    # 文件处理器，日志会记录到文件
FH.setFormatter(formatter)
Logger.addHandler(FH)
Logger.info('this is info')

Logger2 = logging.getLogger(__name__)
Logger2.setLevel(logging.WARNING)
CH = logging.StreamHandler()    # 流处理器，日志默认显示在命令行
CH.setFormatter(formatter)
Logger2.addHandler(CH)
Logger2.warning('this is warning')
```
这段代码执行后，info的日志会写入到my.log文件，而warning的日志则会在命令行执行打印出来。
此外，还可以对日志文件按照指定规则进行分割，这时就需要使用特定的Handler来完成这个任务了。
比如：RotatingFileHandler可以根据文件大小来分割，TimedRotatingFileHandler可以按照指定的时间间隔来分割日志。
```python
import logging.handlers

formatter = logging.Formatter(
    fmt="[%(asctime)s][%(name)s][%(levelname)s] => %(message)s",
    datefmt="%Y-%m-%d  %H:%M:%S %a"
)

Logger = logging.getLogger('test')
Logger.setLevel(logging.INFO)
RFH = logging.handlers.RotatingFileHandler("test.log", maxBytes=1024 * 1024 * 5, backupCount=5)
RFH.setFormatter(formatter)
Logger.addHandler(RFH)
Logger.info('this is info')

Logger2 = logging.getLogger('test2')
Logger2.setLevel(logging.WARNING)
TRFH = logging.handlers.TimedRotatingFileHandler("test2.log", when='D', interval=2, backupCount=3)
TRFH.setFormatter(formatter)
Logger2.addHandler(TRFH)
Logger2.warning('this is warning')
```
这段代码中`RotatingFileHandler`的配置是日志文件按照5M大小来分割，最多最多保留5个日志文件。
而`TimedRotatingFileHandler`的配置是按照每2天分割一次日志文件，保留3个日志文件。

> 关于更多的Handler类，可以查看[官方文档](https://docs.python.org/3/library/logging.handlers.html)

# 通过文件配置日志
上面介绍的日志配置都是通过代码的方式配置的，其实还可以通过日志配置文件来配置。下面就来看看配置文件的格式，首先是字典配置的格式：
```python
import logging.config

LOGGING = {
        "version": 1,
        "disable_existing_loggers": False,
        "formatters": {
            "simple": {
                'format': '%(asctime)s [%(name)s:%(lineno)d] [%(levelname)s]- %(message)s'
            },
            'standard': {
                'format': '%(asctime)s [%(threadName)s:%(thread)d] [%(name)s:%(lineno)d] [%(levelname)s]- %(message)s'
            },
        },

        "handlers": {
            "console": {
                "class": "logging.StreamHandler",
                "level": "DEBUG",
                "formatter": "simple",
                "stream": "ext://sys.stdout"
            },

            "default": {
                "class": "logging.handlers.RotatingFileHandler",
                "level": "INFO",
                "formatter": "simple",
                "filename": 'test.log',
                'mode': 'a',
                "maxBytes": 1024*1024*5,  # 5 MB
                "backupCount": 5,
                "encoding": "utf8"
            },
        },

        "loggers": {
            "test": {
                "level": "WARNING",
                "handlers": ["console"],
                "propagate": "no"
            }
        },

        "root": {
            'handlers': ['default'],
            'level': "INFO",
            'propagate': False
        }
    }

logging.config.dictConfig(LOGGING)
Logger = logging.getLogger('test')
Logger.warning('this is warning')
```
接着是文件配置的格式，先看日志配置文件：
```bash
[loggers]
keys=root,logger01

[logger_root]
level=DEBUG
handlers=hand01

[logger_logger01]
handlers=hand02,hand03
qualname=logger01
propagate=0

[handlers]
keys=hand01,hand02,hand03

[handler_hand01]
class=StreamHandler
level=INFO
formatter=form02
args=(sys.stderr,)

[handler_hand02]
class=FileHandler
level=DEBUG
formatter=form01
args=('test.log', 'a')

[handler_hand03]
class=handlers.RotatingFileHandler
level=INFO
formatter=form02
args=('test.log', 'a', 10*1024*1024, 5)

[formatters]
keys=form01,form02

[formatter_form01]
format=%(asctime)s %(filename)s[line:%(lineno)d] %(levelname)s %(message)s
datefmt=%a, %d %b %Y %H:%M:%S

[formatter_form02]
format=%(name)-12s: %(levelname)-8s %(message)s
```
python代码就更简单了。
```python
import logging.config

logging.config.fileConfig('logging.conf')
Logger = logging.getLogger()    # root Logger
Logger.warning('this is warning')

Logger2 = logging.getLogger('logger01')     # logger01 Logger
Logger2.info('this is info')
```

# 总结
到这里关于Python的logging模块的介绍就结束了。现在回过头来再总结下，logging模块其实有很多的子模块，
不同的子模块有不同的作用，具体而言可以通过一张图来理解。

![logging](https://github.com/five3/testqa/blob/master/images/logging.png?raw=true)

从图中可以看出logging模块的主要子模块有：Logger，Handler，Filter, Formatter等。
其中Logger下面调用Handler，Handler下面调用Filter和Formatter来进行日志处理。

# 新书推荐
![Python Web自动化测试设计与实现](https://img-blog.csdnimg.cn/20190117100818307.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ZpdmUz,size_16,color_FFFFFF,t_70)

获取更多关于Python和自动化测试的文章，请扫描如下二维码！
![关注二维码](https://img-blog.csdnimg.cn/20190117103222240.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ZpdmUz,size_16,color_FFFFFF,t_70)
