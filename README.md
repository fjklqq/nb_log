## 1. pip install nb_log

### 解释一下什么叫pycharm下的点击自动跳转功能

![Image text](https://i.niupic.com/images/2020/07/30/8tjN.png)


```
very sharp color display,monkey patch bulitin print  
and high-performance multiprocess safe roating file handler,
other handlers includeing dintalk ,email,kafka,elastic and so on 

0) 自动转换print效果，再也不怕有人在项目中随意print，导致很难找到是从哪里冒出来的print。
只要import nb_log，项目所有地方的print自动现型并在控制台可点击几精确跳转到print的地方。

1） 兼容性
使用的是python的内置logging封装的，返回的logger对象的类型是py官方内置日志的Logger类型，兼容性强，
保证了第三方各种handlers扩展数量多和方便，和一键切换现有项目的日志。

比如logru和logbook这种三方库，完全重新写的日志，
它里面主要被用户使用的logger变量类型不是python内置Logger类型，
造成logger说拥有的属性和方法有的不存在或者不一致，这样的日志和python内置的经典日志兼容性差，
只能兼容（一键替换logger类型）一些简单的debug info warning errror等方法，。

2） 日志记录到多个地方
内置了一键入参，每个参数是独立开关，可以把日志同时记录到8个常用的地方的任意几种组合，
包括 控制台 文件 钉钉 邮件 mongo kafka es 等等 。在第8章介绍实现这种效果的观察者模式。

3） 日志命名空间独立，采用了多实例logger，按日志命名空间区分。
命名空间独立意味着每个logger单独的日志界别过滤，单独的控制要记录到哪些地方。

logger_aa = LogManager('aa').get_logger_and_add_handlers(10，log_filename='aa.log')
logger_bb = LogManager('bb').get_logger_and_add_handlers(30，is_add_stream_handler=False,
ding_talk_token='your_dingding_token')
logger_cc = LogManager('cc').get_logger_and_add_handlers(10，log_filename='cc.log')

那么logger_aa.debug('哈哈哈')  
将会同时记录到控制台和文件aa.log中，只要debug及debug以上级别都会记录。

logger_bb.warning('嘿嘿嘿')   
将只会发送到钉钉群消息，并且logger_bb的info debug级别日志不会被记录，
非常方便测试调试然后稳定了调高界别到生产。

logger_cc的日志会写在cc.log中，和logger_aa的日志是不同的文件。

4） 对内置looging包打了猴子补丁，使日志永远不会使用同种handler重复记录

例如，原生的  

from logging import getLogger,StreamHandler
logger = getLogger('hi')
getLogger('hi').addHandler(StreamHandler())
getLogger('hi').addHandler(StreamHandler())
getLogger('hi').addHandler(StreamHandler())
logger.warning('啦啦啦')

明明只warning了一次，但实际会造成 啦啦啦 在控制台打印3次。
使用nb_log，对同一命名空间的日志，可以无惧反复添加同类型handler，不会重复记录。


5）支持日志自定义，运行此包后，会自动在你的python项目根目录中生成nb_log_config.py文件，按说明修改。
```

## 2. 最简单的使用方式,这只是演示控制台日志
###### 2.0）自动拦截改变项目中所有地方的print效果。（支持配置文件自定义关闭转化print）
###### 2.1）控制台日志变成可点击，精确跳转。（支持配置文件自定义修改或增加模板，内置了7种模板，部分模板生成的日志可以在pycharm控制台点击跳转）
###### 2.2）控制台日志根据日志级别自动变色。（支持配置文件关闭彩色或者关闭背景色块）

```python
from nb_log import LogManager

logger = LogManager('lalala').get_logger_and_add_handlers()

logger.debug('绿色')
logger.info('蓝色')
logger.warn('黄色')
logger.error('紫红色')
logger.critical('血红色')
print('print样式被自动发生变化')
```

## 3 文件日志
###### 3.1）这个文件日志的自定义filehandler是python史上性能最强大的 支持多进程下日志文件按大小自动切割。

在各种filehandler实现难度上 
单进程永不切割 <  多进程按时间切割 < 单进程按大小切割  << 多进程按大小切割

因为每天日志大小很难确定，如果每天所有日志文件以及备份加起来超过40g了，硬盘就会满挂了，所以nb_log的文件日志filehandler采用的是按大小切割，不使用按时间切割。

文件日志自动使用的是多进程安全切割的自定义filehandler，
logging包的RotatingFileHandler多进程运行代码时候，如果要实现向文件写入到规定大小时候并自动备份切割，win和linux都100%报错。

支持多进程安全切片的知名的handler有ConcurrentRotatingFileHandler，
此handler能够确保win和linux切割正确不出错，此包在linux使用的是高效的fcntl文件锁，
在win上性能惨不忍睹，这个包在win的性能在三方包的英文说明注释中，作者已经提到了。

nb_log是基于自动批量聚合，从而减少写入次数（但文件日志的追加最多会有1秒的延迟），从而大幅度减少反复给文件加锁解锁，
使快速大量写入文件日志的性能大幅提高，在保证多进程安全且排列的前提下，对比这个ConcurrentRotatingFileHandler
使win的日志文件写入速度提高100倍，在linux上写入速度提高10倍。

###### 3.2）演示文件日志，并且直接演示最高实现难度的多进程安全切片文件日志

```python
from multiprocessing import Process
from nb_log import LogManager

#指定log_filename不为None 就自动写入文件了，并且默认使用的是多进程安全的切割方式的filehandler。
#默认都添加了控制台日志，如果不想要控制台日志，设置is_add_stream_handler=False
#为了保持方法入场数量少，具体的切割大小和备份文件个数有默认值，
#如果需要修改切割大小和文件数量，在当前python项目根目录自动生成的nb_log_config.py文件中指定。
logger = LogManager('ha').get_logger_and_add_handlers(is_add_stream_handler=True,
log_filename='ha.log')

def f():
    for i in range(1000000000):
        logger.debug('测试文件写入性能，在满足 1.多进程运行 2.按大小自动切割备份 3切割备份瞬间不出错'
                    '这3个条件的前提下，验证这是不是python史上文件写入速度遥遥领先 性能最强的python logging handler')
       
if __name__ == '__main__':
    [Process(target=f).start() for _ in range(10)]
```

## 4 钉钉日志
```python
from nb_log import LogManager
logger4 = LogManager('hi').get_logger_and_add_handlers(is_add_stream_handler=True,
    log_filename='hi.log',ding_talk_token='your_dingding_token')
logger4.debug('这条日志会同时出现在控制台 文件 和钉钉群消息')
```

## 5 其他handler包括kafka日志，elastic日志，邮件日志，mongodb日志

按照get_logger_and_add_handler函数的入参说明就可以了，和上面的2 3 4中的写法方式差不多，都是一参 傻瓜式，设置了，日志记录就会记载在各种地方。

## 6 日志优先默认配置

只要项目任意文件运行了，带有import nb_log的脚本，就会在项目根目录下生成nb_log_config.py配置文件。
nb_log_config.py的内容如下，默认都是用#注释了，如果放开某项配置则优先使用这里的配置，否则使用nb_log_config_default.py中的配置。

配置示例如下：
```
如果反对日志有各种彩色，可以设置 DEFAULUT_USE_COLOR_HANDLER = False
如果反对日志有块状背景彩色，可以设置 DISPLAY_BACKGROUD_COLOR_IN_CONSOLE = False
如果想屏蔽nb_log包对怎么设置pycahrm的颜色的提示，可以设置 WARNING_PYCHARM_COLOR_SETINGS = False
如果想改变日志模板，可以设置 FORMATTER_KIND 参数，只带了7种模板，可以自定义添加喜欢的模板
```

```python
import logging
ELASTIC_HOST = '127.0.0.1'
ELASTIC_PORT = 9200

KAFKA_BOOTSTRAP_SERVERS = ['192.168.199.202:9092']
ALWAYS_ADD_KAFKA_HANDLER_IN_TEST_ENVIRONENT = False

MONGO_URL = 'mongodb://myUserAdmin:mima@127.0.0.1:27016/admin'

DEFAULUT_USE_COLOR_HANDLER = True  # 是否默认使用有彩的日志。
DISPLAY_BACKGROUD_COLOR_IN_CONSOLE = True  # 在控制台是否显示彩色块状的日志。为False则不使用大块的背景颜色。
AUTO_PATCH_PRINT = True  # 是否自动打print的猴子补丁，如果打了后指不定，print自动变色和可点击跳转。
WARNING_PYCHARM_COLOR_SETINGS = True

DEFAULT_ADD_MULTIPROCESSING_SAFE_ROATING_FILE_HANDLER = False  # 是否默认同时将日志记录到记log文件记事本中。
LOG_FILE_SIZE = 100  # 单位是M,每个文件的切片大小，超过多少后就自动切割
LOG_FILE_BACKUP_COUNT = 3

LOG_LEVEL_FILTER = logging.DEBUG  # 默认日志级别，低于此级别的日志不记录了。例如设置为INFO，那么logger.debug的不会记录，只会记录logger.info以上级别的。
RUN_ENV = 'test'

FORMATTER_DICT = {
    1: logging.Formatter(
        '日志时间【%(asctime)s】 - 日志名称【%(name)s】 - 文件【%(filename)s】 - 第【%(lineno)d】行 - 日志等级【%(levelname)s】 - 日志信息【%(message)s】',
        "%Y-%m-%d %H:%M:%S"),
    2: logging.Formatter(
        '%(asctime)s - %(name)s - %(filename)s - %(funcName)s - %(lineno)d - %(levelname)s - %(message)s',
        "%Y-%m-%d %H:%M:%S"),
    3: logging.Formatter(
        '%(asctime)s - %(name)s - 【 File "%(pathname)s", line %(lineno)d, in %(funcName)s 】 - %(levelname)s - %(message)s',
        "%Y-%m-%d %H:%M:%S"),  # 一个模仿traceback异常的可跳转到打印日志地方的模板
    4: logging.Formatter(
        '%(asctime)s - %(name)s - "%(filename)s" - %(funcName)s - %(lineno)d - %(levelname)s - %(message)s -               File "%(pathname)s", line %(lineno)d ',
        "%Y-%m-%d %H:%M:%S"),  # 这个也支持日志跳转
    5: logging.Formatter(
        '%(asctime)s - %(name)s - "%(pathname)s:%(lineno)d" - %(funcName)s - %(levelname)s - %(message)s',
        "%Y-%m-%d %H:%M:%S"),  # 我认为的最好的模板,推荐
    6: logging.Formatter('%(name)s - %(asctime)-15s - %(filename)s - %(lineno)d - %(levelname)s: %(message)s',
                         "%Y-%m-%d %H:%M:%S"),
    7: logging.Formatter('%(levelname)s - %(filename)s - %(lineno)d - %(message)s',"%Y-%m-%d %H:%M:%S"), # 一个只显示简短文件名和所处行数的日志模板
}

FORMATTER_KIND = 5  # 如果get_logger_and_add_handlers不指定日志模板，则默认选择第几个模板
```

## 7. 各種日志截圖

钉钉

![Image text](https://i.niupic.com/images/2020/05/12/7OSE.png)

控制台日志模板之一

![Image text](https://i.niupic.com/images/2020/05/12/7OSF.png)

控制台日子模板之二

![Image text](https://i.niupic.com/images/2020/05/12/7OSG.png)

邮件日志

![Image text](https://i.niupic.com/images/2020/05/12/7OSH.png)

文件日志

![Image text](https://i.niupic.com/images/2020/05/12/7OSI.png)

elastic日志

![Image text](https://i.niupic.com/images/2020/05/12/7OSK.png)

mongo日志

![Image text](https://i.niupic.com/images/2020/05/12/7OSL.png)
 
 
## 8 关于日志的观察者模式
不会扩展日志记录到什么地方，主要是不懂什么叫观察者模式

```python
# 例如 日志想实现记录到 控制台、文件、钉钉群、redis、mongo、es、kafka、发邮件其中的几种的任意组合。
# low的人，会这么写，以下是伪代码，实现记录到控制台、文件、钉钉群这三种的任意几种组合。

def 记录到控制台(msg):
    """实现把msg记录到控制台"""

def 记录到文件(msg):
    """实现把msg记录到文件"""

def 记录到钉钉(msg):
    """实现把msg记录到钉钉"""

def 记录到控制台和文件(msg):
    """实现把msg记录到控制台和文件"""

def 记录到控制台和钉钉(msg):
    """实现把msg记录到控制台和钉钉"""

def 记录到文件和钉钉(msg):
    """实现把msg记录到文件和钉钉"""

def 记录到控制台和文件和钉钉(msg):
    """实现把msg记录到控制台和文件和钉钉"""

#当需要把msg记录到文件时候，调用函数 记录到文件(msg)
#当需要把msg记录到控制台时候，调用函数 记录到控制台(msg)
#当需要把msg记录到钉钉时候，调用函数 记录到钉钉(msg)
#当需要把msg记录到控制台和文件，调用函数 记录到控制台和文件(msg)
#当需要把msg记录到控制台和钉钉，调用函数 记录到控制台和钉钉(msg)
#当需要把msg记录到控制台和文件和钉钉，调用函数 记录到控制台和文件和钉钉(msg)

"""
这样会造成，仅记录到控制台 文件 钉钉这三种的任意几个，需要写6个函数，调用时候需要调用不同的函数名。
但是现在日志可以记录到8种地方，如果还这么low的写法，需要写8的阶乘个函数，调用时候根据场景需要会调用8的阶乘个函数名。
8的阶乘结果是 40320 ，如果很low不学设计模式做到灵活组合，需要多写 4万多个函数，不学设计模式会多么吓人。

"""
```

##### 观察者模式图片
![Image text](https://www.runoob.com/wp-content/uploads/2014/08/observer_pattern_uml_diagram.jpg)

菜鸟教程的观察者模式demo连接
[观察者模式demo](https://www.runoob.com/design-pattern/observer-pattern.html)

这个uml图上分为Subject 和 基类Observer，以及各种继承或者实现Observer的XxObserver类，
其中每个不同的Observer需要实现doOperation方法。

如果对应到python内置的logging日志包的实现，那么关系就是：

Logger是uml图的Subject

loging.Handler类是uml图的Observer类

StreamHandler FileHandler DingTalkHandler 是uml图的各种XxObservers类。

StreamHandler FileHandler DingTalkHandler类的 emit方法是uml图的doOperation方法

只有先学设计模式，才能知道经典固定套路达到快速看代码，能够达到秒懂源码是怎么规划设计实现的。

如果不先学习经典设计模式，每次看包的源码，需要多浪费很多时间看他怎么设计实现的，不懂设计模式，会觉得太难了看着就放弃了。

在python日志的理解和使用上，国内能和我打成平手的没有几人。


## 9.演示一个由于不好好理解观察者模式，封装的日志类在调用时候十分惨烈的例子，惨烈程度达到10级。

这个是真实发生的例子。

这个例子是为了记录10万次日志到控制台和文件，就算python性能很差，就这个例子而言，预期耗时肯定是需要10秒以内才算合格。

看起来10秒内可以运行完成，实际上1周内能运行结束这个代码，我愿意吃10斤翔。

```python
"""
演示重复，由于封装错误的类造成的。模拟一个封装严重失误错误的封装例子。

看起来10秒内可以运行完成，实际上1周内能运行结束这个代码，我愿意吃10斤翔。

这个代码惨烈程度达到10级。明明是想记录10000次日志，结果却记录了 10000 * 10001 /2 次。
如果把f函数调用100万次，那么控制台和文件将会各记录5000亿次，日志会把代码拖累死。
不好好理解观察者模式有多惨烈。因为反复添加观察者（handler）,
导致第1次调用记录1次，第二次调用时候记录2次，第10次调用时候记录10次，这成了高斯求和算法了。

这种类似的封装造成的后果可想而知，长期部署运行后，不仅项目代码性能几乎被日志占了99%，还造成磁盘被弄爆炸。
"""
import logging
import time


class LogUtil:
    def __init__(self):
        self.logger = logging.getLogger('a')
        self.logger.setLevel(logging.DEBUG)
        self._add_stream_handler()
        self._add_file_handler()

    def _add_stream_handler(self):
        sh = logging.StreamHandler()
        sh.setFormatter(logging.Formatter(fmt="%(asctime)s-%(name)s-%(levelname)s-%(message)s"))
        self.logger.addHandler(sh)

    def _add_file_handler(self):
        fh = logging.FileHandler('a.log')
        fh.setFormatter(logging.Formatter(fmt="%(asctime)s-%(name)s-%(levelname)s-%(message)s"))
        self.logger.addHandler(fh)

    def debug(self, msg):
        self.logger.debug(msg)

    def info(self, msg):
        self.logger.info(msg)


def f(x):
    log = LogUtil()  # 重点是这行，写在了函数内部。既没有做日志命名空间的handlers判断控制，封装的类本身也没写单利或者享元模式。
    log.debug(x)


t1 = time.time()
for i in range(100000):
    f(i)

print(time.time() - t1)

```


