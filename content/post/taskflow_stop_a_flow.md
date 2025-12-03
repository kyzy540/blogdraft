---
title: "如何中断Taskflow工作流"
date: 2025-12-03T16:55:28+08:00
tags:
  - it
keywords:
  - taskflow
---

taskflow的poster+conductor架构下，conductor以多线程并发运行job。线程池被封装在内部。因此一旦conductor开始执行任务，想中断它就很麻烦。通常要重启整个conductor线程池

是否有更优雅的办法中断conductor正在执行的任务？

## 不可行——JobBoard.abandon()/trash()

`JobBoard`的`abandon`和`trash`方法不能终止job。前者只删除job.lock，后者会把job数据拷贝到`/taskflow/.trash/`下再删除job node。二者都没解决核心问题: 阻止工作流继续执行下一步骤。

## 不推荐——修改数据库atomdetails状态

设想: 修改taskflow persistence数据库(如postresql) `flowdetails`和`atomdetails`表。把`RUNNING`和`PENDING`的步骤都UPDATE成`REVERTED`。期望工作流在进入下一个task时会异常自动中断

测试结果: taskflow会检查任务状态 **合法性**。一个步骤`SUCCESS`，但后续步骤状态是`FAILURE`或`REVERTED`会被识别为状态非法。这会导致conductor异常退出，影响其他线程。此外，taskflow会将失败任务的`failure`字段转换成`dict`。写入的数据结构必须也合法，否则任务无法被重试

![](https://docs.openstack.org/taskflow/latest/_images/task_states.svg)

同理state设为`REVERT_FAILURE`或`SUCCESS`可以中断工作流，但也会报错状态非法

总之方案可行，但不高效

## 可行——kill conductor进程

每个conductor都只维护一个线程，在更上层用 **进程** 维护多个conductor。这样就可通过kill conductor进程。但代价是任务的`revert`方法不会被执行。如果所有任务完全幂等，没有资源泄漏风险，是可行的

但不推荐这个方案。它实际上提高了任务设计难度，也不符合taskflow的设计理念

## 推荐——execute()装饰器

示例代码如下

```python
from threading import Event
from taskflow import task

cancel_event = Event()


class CancelableTask(task.Task):

    def __init_subclass__(cls, **kwargs):
        super().__init_subclass__(**kwargs)

        if hasattr(cls, "execute") and callable(cls.execute):
            original_execute = cls.execute

            def wrapped_execute(self, *args, **kwargs):
                if cancel_event and cancel_event.is_set():
                    raise Exception("Flow cancelled at start")
                result = original_execute(self, *args, **kwargs)
                if cancel_event and cancel_event.is_set():
                    raise Exception("Flow cancelled after execution")

                return result

            cls.execute = wrapped_execute


class DownloadTask(CancelableTask):
    def execute(self, url, **kwargs):
        print(f"Downloading {url}...")
        for i in range(10):
            print(f"Progress: {i*10}%")
        return f"Downloaded {url}"

    def revert(self, result, *args, **kwargs):
        pass # TODO 清理工作

```

`__init_subclass__`是Python 3.6引入的特性，在子类初始化时执行。赋予父类修改子类方法的能力。示例代码中，`DownloadTask`继承了`CancelableTask`，它的`execute`方法被 **透明** 地封装了一层装饰器。如果检测到`cancel_event`会抛出异常。这种抛异常方式是taskflow可以处理的，`DownloadTask`的`revert`方法随后会被调用，完成清理工作

这个方案相对优雅。它符合taskflow的设计理念: 任务失败应该先 **回滚(revert)**，后退出。虽然不能在execute过程中断任务，但换来的好处是确保会执行`revert`，并且后续可以重试并继续任务
