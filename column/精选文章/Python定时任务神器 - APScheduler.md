# Python定时任务神器 - APScheduler
> 定时任务在很多的开发场景中都会使用到，在Python中也提供很多的定时任务库。比如：
- sched
- schedule
- celery

但是这些定时任务库都只是提供了简答的，或者只支持静态的定时任务。而对于需要复杂定时功能，或者动态注册定时任务的场景，则无法满足。

而今天介绍的主角 - APScheduler，则会完美的解决这个问题。
- 多种的定时任务类型支持
- 静态、动态定时任务支持

## 简单说明
> 不管你使用哪种APScheduler的定时任务，你都需要先了解APScheduler的简单机制。即：job、executors、jobstores、trigger、scheduler等

### job
即需要被执行的具体任务，主要对应Python中的函数或方法。在APScheduler中即可提前配置，也可以动态添加job。

### executors
即执行job的对象。通常可以是多线程、多进程、协程等对象。

### jobstores
即存储job元数据的地方。可以是memory、sqllite、mysql等。

### trigger
即决定任务的触发模式。通常有指定时间、指定时间间隔、指定周期策略等。

### scheduler
用于调度和管理上述提到的所有对象。

任意一个APScheduler的实例启动的时候都需要配置这些初始参数，如果没有指定则会使用默认的值。

## 使用方式
首先你得安装apscheduler，方式如下：
```bash
pip install apscheduler
```

### 静态配置任务
```python
import time
from apscheduler.schedulers.blocking import BlockingScheduler
 
sched = BlockingScheduler()
 
@sched.scheduled_job('interval', seconds=5)     # 以5s为间隔来触发
def my_job():
    print(time.strftime('%Y-%m-%d %H:%M:%S', time.localtime(time.time())))
 
sched.start()
```

### 动态添加任务
```python
import time
from apscheduler.schedulers.blocking import BlockingScheduler
 
def my_job():
    print(time.strftime('%Y-%m-%d %H:%M:%S', time.localtime(time.time())))
 
sched = BlockingScheduler()
# 在11,12月份的第三个星期五的00:00,01:00,02:00,03:00触发
sched.add_job(my_job, 'cron', month='11-12', day='3rd fri', hour='0-3')

sched.start()
```

### 异步任务
上面的任务都是阻塞任务，即任务执行过程中需要持续等待结果；如果你不希望等待结果的话，则可以使用非阻塞的异步任务。

```python
import time
import datetime
from pytz import timezone
from apscheduler.schedulers.background import BackgroundScheduler
from apscheduler.jobstores.sqlalchemy import SQLAlchemyJobStore
from apscheduler.executors.pool import ThreadPoolExecutor

def my_job(name):
    print('%s: %s' % (time.strftime('%Y-%m-%d %H:%M:%S', time.localtime(time.time())), name))
    
jobstores = {
    'default': SQLAlchemyJobStore(url='sqlite:////tmp/test.db')
}
executors = {
    'default': ThreadPoolExecutor(20)
}
job_defaults = {
    'coalesce': False,
    'max_instances': 3
}
task_scheduler = BackgroundScheduler(jobstores=jobstores, executors=executors, job_defaults=job_defaults,
                                     timezone=timezone('Asia/Shanghai'))
task_scheduler.start()
# 指定特定的日期触发任务，这类任务只会触发一次
job = task_scheduler.add_job(my_job, 'date', run_date=datetime(2019, 3, 6, 9, 47, 5), args=('date',))
```

### 其它使用API
除了注册和添加任务之外，`apscheduler`还提供了其它比较友好的API接口。比如：
- 暂停任务
- 启动任务
- 删除任务

```python
job = scheduler.add_job(myfunc, 'interval', minutes=2)
job.pause()
job.resume()
job.remove()
```

# 完整代码
```python
from pytz import timezone
from datetime import datetime
import os
import time
from apscheduler.schedulers.background import BackgroundScheduler
from apscheduler.jobstores.sqlalchemy import SQLAlchemyJobStore
from apscheduler.executors.pool import ThreadPoolExecutor
from apscheduler.triggers.cron import CronTrigger

from config import CONFIG


db_str = CONFIG.get(os.environ.get('FLASK_SQLALCHEMY_DATABASE_URI', 'Default')).SQLALCHEMY_DATABASE_URI
jobstores = {
    'default': SQLAlchemyJobStore(url=db_str)
}
executors = {
    'default': ThreadPoolExecutor(20)
}
job_defaults = {
    'coalesce': False,
    'max_instances': 3
}
task_scheduler = BackgroundScheduler(jobstores=jobstores, executors=executors, job_defaults=job_defaults,
                                     timezone=timezone('Asia/Shanghai'))
task_scheduler.start()
# task_scheduler.remove_all_jobs()

if __name__ == '__main__':
    def func(x):
        print(x)

    # trigger   date|interval|cron
    # job = task_scheduler.add_job(func, 'date', run_date=datetime(2019, 3, 6, 9, 47, 5), args=('date',))
    task_scheduler.add_job(func, 'interval', minutes=2, args=('interval',))
    # task_scheduler.add_job(func, 'cron', hour=3, minute=30, args=('cron',))
    # task_scheduler.add_job(func, CronTrigger.from_crontab('0 0 1-15 may-aug *'), args=('cron2',))
    # task_scheduler.start()
    # task_scheduler.remove_job()

    while True:
        # task_scheduler.remove_all_jobs()
        print(len(task_scheduler.get_jobs()))
        time.sleep(1)
```

# 新书推荐
![Python Web自动化测试设计与实现](https://img-blog.csdnimg.cn/20190117100818307.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ZpdmUz,size_16,color_FFFFFF,t_70)

获取更多关于Python和自动化测试的文章，请扫描如下二维码！
![关注二维码](https://img-blog.csdnimg.cn/20190117103222240.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ZpdmUz,size_16,color_FFFFFF,t_70)
