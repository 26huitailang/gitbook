[TOC]

# Flask中的使用

## flask_celery.py

在flask app/flask_vises/flask_celery.py

```python
#!/usr/bin/env python
# coding=utf-8


from celery import Celery


def create_celery(name='flask'):
    return Celery(name)


def configure_celery(app, celery):
    # set broker url and result backend from app config
    celery.conf.broker_url = app.config['CELERY_BROKER_URL']
    celery.conf.result_backend = app.config['CELERY_RESULT_BACKEND']
    celery.conf.timezone = app.config['CELERY_TIMEZONE']
    celery.conf.task_send_sent_event = app.config['CELERY_TASK_SEND_SENT_EVENT']

    # subclass task base for app context
    # http://flask.pocoo.org/docs/dev/patterns/celery/
    class ContextTask(celery.Task):
        def __call__(self, *args, **kwargs):
            with app.app_context():
                return self.run(*args, **kwargs)

    celery.Task = ContextTask

    # run finalize to process decorated tasks
    # celery.finalize()  # 不能有!!
    return celery

```

## celery_worker.py

在flask app的目录下

```python
#!/usr/bin/env python
# coding=utf-8


from celery.schedules import crontab
from flask_vises.flask_celery import configure_celery
from flask_vises.deploy import DeployLevel

from config import Config
from flask_celery import celery
from application import create_app
from tasks.cron_tasks.ss_agent_detection import ss_agent_detection_beat_task
from tasks.cron_tasks.backup_db import backup_db_to_cos
from tasks.cron_tasks.find_expired_device import (
    find_and_notify_expiring_device,
    find_and_close_expired_device,
)

app = create_app()
configure_celery(app, celery)


@celery.on_after_configure.connect
def register_periodic_tasks(sender, **kwargs):
    # Calls reverse_messages every 10 seconds.
    # sender.add_periodic_task(10.0, demo_test, name='reverse every 10')
    # 每周星期天，凌晨2点0分，数据库备份，并上传至cos
    sender.add_periodic_task(crontab(day_of_week=0, hour=2, minute=0), backup_db_to_cos)
    # 每周一，查看最近30天要过期的设备，并发送slack通知
    sender.add_periodic_task(crontab(day_of_week=1, hour=1, minute=0), find_and_notify_expiring_device)
    # 每天凌晨一点，检测要过期的设备，并且关闭
    sender.add_periodic_task(crontab(hour=1, minute=0), find_and_close_expired_device)
    # 每 10分钟检测线路能否工作，只在生产环境工作
    if Config.DEPLOY_LEVEL == DeployLevel.release:
        sender.add_periodic_task(crontab(minute='*/10'), ss_agent_detection_beat_task)

    return

```

# 问题

## cannot add item to database

`celery beat raised exception <class '_dbm.error'>: error('cannot add item to database',）错误解决方法`

2018年04月11日 15:35:49

阅读数：26

在一次重启python中的celery-beat任务中，发现celery-beat启动不起来了。报错信息如下：

```
celery beat v4.1.0 (latentcall) is starting.
__    -    ... __   -        _
LocalTime -> 2018-04-11 15:04:12
Configuration ->
    . broker -> redis://127.0.0.1:9736/4
    . loader -> celery.loaders.app.AppLoader
    . scheduler -> celery.beat.PersistentScheduler
    . db -> celerybeat-schedule
    . logfile -> [stderr]@%WARNING
    . maxinterval -> 5.00 minutes (300s)
[2018-04-11 15:04:12,478: CRITICAL/MainProcess] beat raised exception <class '_dbm.error'>: error('cannot add item to database',)
Traceback (most recent call last):
  File "/usr/local/lib/python3.6/site-packages/kombu/utils/objects.py", line 42, in __get__
    return obj.__dict__[self.__name__]
KeyError: 'scheduler'

During handling of the above exception, another exception occurred:

Traceback (most recent call last):
  File "/usr/local/lib/python3.6/shelve.py", line 111, in __getitem__
    value = self.cache[key]
KeyError: 'entries'

During handling of the above exception, another exception occurred:

Traceback (most recent call last):
  File "/usr/local/lib/python3.6/site-packages/celery/beat.py", line 485, in _create_schedule
    self._store[str('entries')]
  File "/usr/local/lib/python3.6/shelve.py", line 113, in __getitem__
    f = BytesIO(self.dict[key.encode(self.keyencoding)])
KeyError: b'entries'

During handling of the above exception, another exception occurred:

Traceback (most recent call last):
  File "/usr/local/lib/python3.6/site-packages/celery/apps/beat.py", line 107, in start_scheduler
    service.start()
  File "/usr/local/lib/python3.6/site-packages/celery/beat.py", line 549, in start
    humanize_seconds(self.scheduler.max_interval))
  File "/usr/local/lib/python3.6/site-packages/kombu/utils/objects.py", line 44, in __get__
    value = obj.__dict__[self.__name__] = self.__get(obj)
  File "/usr/local/lib/python3.6/site-packages/celery/beat.py", line 593, in scheduler
    return self.get_scheduler()
  File "/usr/local/lib/python3.6/site-packages/celery/beat.py", line 588, in get_scheduler
    lazy=lazy,
  File "/usr/local/lib/python3.6/site-packages/celery/beat.py", line 428, in __init__
    Scheduler.__init__(self, *args, **kwargs)
  File "/usr/local/lib/python3.6/site-packages/celery/beat.py", line 206, in __init__
    self.setup_schedule()
  File "/usr/local/lib/python3.6/site-packages/celery/beat.py", line 456, in setup_schedule
    self._create_schedule()
  File "/usr/local/lib/python3.6/site-packages/celery/beat.py", line 489, in _create_schedule
    self._store[str('entries')] = {}
  File "/usr/local/lib/python3.6/shelve.py", line 125, in __setitem__
    self.dict[key.encode(self.keyencoding)] = f.getvalue()
_dbm.error: cannot add item to database
```

解决办法：

1.删除启动目录中的celerybeat-schedule.db（我的项目地址为/data/www/master/project，在这目录下你发现有个celerybeat-schedule.db文件，删除即可）

2.再重新启动 (celery beat -A project.celery_job )

原因：

当celerybeat启动的时候会带启动目录中生成 celerybeat-schedule.db作为数据存储的文件，当celerybeat异常关闭时，此文件没有被删除，在下一下启动的时候，被占用，导致_dbm.error: cannot add item to database错误。所以把文件删除就可以了

## django中解决beat发送重复任务
利用一个装饰器，在任务进入worker执行前，检查该任务是否是激活状态，如果有则直接return不执行后面的实际任务代码。
```python
#!/usr/bin/env python
# coding=utf-8


from functools import wraps
from celery.utils.log import get_task_logger
from tzdatadws.celery import app

logger = get_task_logger(__name__)


def celery_task_conflict_detection(task_func=None):
    """celery任务冲突的检查，避免对数据重复操作

    :param task_func: 默认为None检查自己，否则检查传入的task_func
    :return: 通过返回要执行的task，否则返回None
    """
    def decorator(f):
        @wraps(f)
        def wrapper(*args, **kwargs):
            tasks_name_list = []
            r = app.control.inspect().active()  # 获得激活中的任务
            for k, v in r.items():
                tasks_name_list = [x['name'].split('.')[-1] for x in v]

            # 如果没有传入指定，检查自己
            if task_func is None:
                detected_func_name = f.__name__
            else:  # 否则检查传入的任务函数名
                detected_func_name = task_func.__name__

            # 检查的是自己是否冲突，因为task一旦delay就能在激活状态中查到
            # 所以条件为自身激活数量大于1时，判定为冲突
            if tasks_name_list.count(detected_func_name) > 1 and (not task_func):
                # logger.debug(tasks_name_list.count(decorated_func_name))
                logger.info("没有通过装饰器检查，任务重复：{}".format(detected_func_name))
                return

            # 检查其他的则直接看有无激活即可
            if (detected_func_name in tasks_name_list) and task_func:
                # logger.debug(tasks_name_list.count(decorated_func_name))
                logger.info("没有通过装饰器检查，任务重复：{}".format(detected_func_name))
                return

            # logger.info("通过装饰器")
            return f(*args, **kwargs)

        return wrapper

    return decorator
```
