---
title: "一个不带插件的flask应用"
publish_time: "2023-08-14"
hidden: false
---
## 起因

事情的起因是这样的, 曾经维护过一个别人做的flask项目,引用了大量的插件, 在维护过程中需要调查插件的源码来解决问题, 发现有的插件功能很单一,但是却写的很复杂(因为开源插件要考虑各种实际使用场景). 有时间精力还不如自己写一个. 恰好得到一个从零开始的项目. 决定尝试做一个零插件的flask项目. 效果还不错.

> 注意: 这里的零插件是指零flask插件,不是不用任何python插件.

## 项目构成

一个很简单的dashboard, 主要用到如下组件

```txt
python = "~3.10"
Flask = "~2.1.1"
python-dotenv = "~0.20.0"
pydantic = "~1.9.0"
requests = "~2.27.1"
peewee = "~3.14.10"
arrow = "~1.2.2"
psycopg2 = "~2.9.3"
wtf-peewee = "~3.0.4"
APScheduler = "~3.9.1"
redis = "~4.3.3"
SQLAlchemy = "~1.4.37"
gunicorn = "~20.1.0"
Cython = "^0.29.30"
```

可能用到的flask插件是: `flask-redis`, `flask-sqlalchemy`, `flask-peewee`,`flask-pydantic`

### flask-redis

先说flask-redis
src/extentions.py

```python
from redis import Redis
class FlaskRedis:
    def __init__(self) -> None:
        self.client = None

    def init(self, app):
        self.client = Redis.from_url(app.config["REDIS_URL"])
redis_client = FlaskRedis()
```

只需要在需要的地方

```python
from extensions import redis_client

def get_redis_data(key) -> list[Any] | dict[Any, Any] | None:
    ret = redis_client.client.get(f"{key}")
```

是不是超级简单,完全满足项目需要. 也不需要记忆开源插件的配置.

## flask apscheduler

再譬如scheduler
src/extentioons.py

```python
from apscheduler.schedulers.blocking import BlockingScheduler
class FlaskScheduler:
    def init(self, app):
        scheduler = BlockingScheduler()

        scheduler.configure(**app.config["SCHEDULER_OPTIONS"])

        jobs = app.config.get("SCHEDULER_JOBS", [])
        for job in jobs:
            scheduler.add_job(**job)
        self.scheduler = scheduler
scheduler = FlaskScheduler()
```

当然,这里需要在create_app中init一下

```python
def create_app(test_config: dict[str, Any] = None) -> Flask:
    ...
    redis_client.init(app)
    scheduler.init(app)
    ...
```

scheduler的使用需要通过flask中的click配合实现:
/src/cli/scheduler.py

```python
from flask.cli import with_appcontext
from extensions import scheduler
from flask.cli import AppGroup

scheduler_cli = AppGroup("scheduler")


@scheduler_cli.command("start")
@with_appcontext
def start():
    scheduler.scheduler.start()


@scheduler_cli.command("stop")
@with_appcontext
def stop():
    scheduler.scheduler.stop()
```

文章写到这里, 发现这个项目还有一个替换`flask-migrate`的实现. 但是现在已经是早上8点半了. 我要去上班了. 等有时间在下篇文章中介绍.
