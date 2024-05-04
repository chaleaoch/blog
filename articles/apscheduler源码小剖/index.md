---
title: "apscheduler小剖"
publish_time: "2024-05-04"
hidden: false
---

<p style="color: rgba(127, 127, 127, 0.9);">前段时间同事问我, 如果直接修改apscheduler的数据库信息, 是否可以让job快速运行. 答案当然是否定的, 具体原因需要结合代码解释, 这就是本文的由来.<p>

无论是官网还是源码, 都很清晰的表达apscheduler由四个部分组成.

1. executor 管干活的, 譬如线程池
2. jobstores 存储job的地方, 主要用途是将job持久化, 重启apscheduler之后可以继续执行.
3. triggers 触发器, 用于计算job下次运行时间
4. scheduler 主体进程, 负责调度job

源码只取各个组件的最简单的例子进行讲解, 基本原理搞懂就可以了, 对自己的要求不高😀

## scheduler

取最简单的BlockingScheduler, 他的核心方法是`_process_jobs`, 用于循环调度job[<https://github.com/agronholm/apscheduler/blob/3.x/apscheduler/schedulers/base.py#L941>](https://github.com/agronholm/apscheduler/blob/3.x/apscheduler/schedulers/base.py#L941)

1. 第一步, 从job store中获取所有到期的job列表 `due_jobs = jobstore.get_due_jobs(now)`
2. 第二步, 上job中找executor `executor = self._lookup_executor(job.executor)`
3. 第三步, 在executor上执行job `executor.submit_job(job, run_times)`
4. 第四步, 计算这个job下一次什么时候运行, `job.trigger.get_next_fire_time(run_times[-1], now)`
5. 第五步, 更新job store信息, 方便下一次循环中的第一步执行获取到期job列表 `jobstore.update_job(job)`
6. 第六步, 通过jobstore计算下一次唤醒时间, 然后sleep, 释放cpu资源

```python
while self.state != STATE_STOPPED:
    self._event.wait(wait_seconds)
    self._event.clear()
    wait_seconds = self._process_jobs()
```

余下的功能无非就是add_job, remove_job等方法在正常处理业务逻辑之外,需要实时唤醒scheduler,原理是event中的set()方法, 在源码中,可以看到多次调用`self.wakeup()` 实际上执行的代码是 `self._event.set()`

## jobstore

以redis举例吧.

```python
def get_due_jobs(self, now):
    timestamp = datetime_to_utc_timestamp(now)
    job_ids = self.redis.zrangebyscore(self.run_times_key, 0, timestamp)
    if job_ids:
        job_states = self.redis.hmget(self.jobs_key, *job_ids)
        return self._reconstitute_jobs(six.moves.zip(job_ids, job_states))
    return []
```

看到这里, 文章开头的问题实际上就可以解答了, 如果直接修改数据库信息, get_next_fire_time并不会更新,需要在一个循环之后才会更新, 所以job并不会立即执行.

## executor

简单解释一下上文中的`submit_job`
`f = self._pool.submit(run_job, job, job._jobstore_alias, run_times, self._logger.name)`

## 其他

trigger我没看, 因为crontab太复杂, interval太简单, 不需要看哈哈...
apscheduler看到这种程度,针对应用层的开发我觉得足够了.调用api的时候大概知道其中的原理, 做到心中有数还是很有必要的, 关键是这个库的源码很简单, 也很好理解, 看了也就看了, 不会花太多的时间.
还有就是apscheduler中的monitor感兴趣的同行可以看看.
