---
title: "TaskFlow学习笔记"
date: 2023-09-05T14:51:44+08:00
draft: true
tags:
  - it
keywords:
  - python
  - taskflow
---

Taskflow是一个Python库，用于构建工作流。它可以处理异步操作、分布式部署和错误跟踪。相比 [Airflow](https://airflow.apache.org/) 它更轻量，不需要部署client&server架构。代码引入taskflow库，配合外部存储，就可以实现分布式工作流应用

[![TaskFlow官网](/img/taskflow_note/taskflow_icon.png#center)](https://wiki.openstack.org/wiki/TaskFlow)

## 设计理念
从官方[设计理念](https://wiki.openstack.org/wiki/TaskFlow/Paradigm_shifts) 可以看出，taskflow将 **可靠性** (Resilience，即故障处理和恢复) 放在首位。设计文档和使用文档都是从最小可恢复单元 (atom) 讲起，自底向上地介绍整个架构

这个介绍方式很好，但对于初衷就是想用taskflow实现分布式工作流的开发者来说，读起来可能有点冗长。我想换个方式，把它分成了两部分，第一部分是介绍如何使用taskflow，第二部分是如何使用taskflow来实现分布式工作流应用。

## 分布式工作流样例

[Conductor running 99 bottles of beer song requests](https://docs.openstack.org/taskflow/latest/user/examples.html#conductor-running-99-bottles-of-beer-song-requests) 是一个典型例子，只要把代码中的sqlite数据库改成mysql就能实现分布式。但为了方便对照代码，本文还是以sqlite为例做介绍。整个流程分6步：
1. [构建工作流meta data数据库](https://opendev.org/openstack/taskflow/src/branch/master//taskflow/examples/99_bottles.py#L185)
2. [发布job: 将工作流meta data宣告到zookeeper](https://opendev.org/openstack/taskflow/src/branch/master//taskflow/examples/99_bottles.py#L189)
3. [conductor从zookeeper承接job](https://opendev.org/openstack/taskflow/src/branch/master//taskflow/examples/99_bottles.py#L149)
4. [conductor从数据库获悉工作流meta data](https://opendev.org/openstack/taskflow/src/branch/master//taskflow/examples/99_bottles.py#L151)
5. [conductor调用make_bottles实例化工作流并执行完成](https://opendev.org/openstack/taskflow/src/branch/master//taskflow/examples/99_bottles.py#L157)
6. conductor删除zookeeper中的job，完成job

![](/img/taskflow_note/99_bottles.png#center)

**`Conductor`** 是工作流的核心，实例化和执行工作流都由他完成。剩下部分都是辅助它完成工作。例如，`poster`的作用是宣告job，供`Conductor`认领。`Persistence`是用来存储job的meta data，`Conductor` 要维护meta data

至此，读者认识了由若干陌生名词堆砌而成的一张流程图，同时收获了一头雾水。不要紧，接下来将逐步分解

## 自顶向下

 一个job的生命周期包括: *发布(post) -> 承接(require) -> 执行(execute) -> 完成(done)* 。`Jobboard`就是发布和承接的中心

### Jobboard
打个比方，`Jobboard`是《巫师》世界里的公告板，`job`是委托，`Conductor`是承接委托的猎魔人 (可能有多位)，`poster`是发布委托的村民
![](/img/taskflow_note/notice_board.jpg#center)

`poster`发布到`Jobboard`的数据很少。读者可以先运行`run_poster`，然后暂停，看看zookeeper里的job数据。只包含了poster id、和 uuid。从代码也可以看出确实没别的信息了

```python
lb = models.LogBook("post-from-%s" % my_name)
fd = models.FlowDetail("song-from-%s" % my_name, uuidutils.generate_uuid())
lb.add(fd)
with contextlib.closing(persist_backend.get_connection()) as conn:
    conn.save_logbook(lb)
    engines.save_factory_details(fd, make_bottles, 
                                 [HOW_MANY_BOTTLES], {}, 
                                 backend=persist_backend)
# Post, and be done with it!
jb = job_backend.post("song-from-%s" % my_name, book=lb)
```

那么`Conductor`如何通过这么少的数据来承接job呢？答案是查`Persistence` (数据库)

`engines.save_factory_details(fd, make_bottles...` 这行代码的作用就是把job的meta data保存到数据库，包括poster id、uuid和 关键的"make_bottles"方法名。"make_bottles" 在这里被称为`factory function` 是因为它真就只往数据库存了一个方法名。承接job的`Conductor`会具体方法名实例化工作流，但这放到下文展开

因此，`poster`和`Jobboard`要做的事情很简单，就是发布一个`make_bottles`的job，然后等`Conductor`来承接并完成。至于有没有`Conductor`承接，job具体如何完成二者都不关心。哪怕`Conductor`接了`make_bottles`的job却造出一辆汽车也可以。

### Conductors
先吐槽一句，`Conductor`是乐团指挥的意思，但这很难反应出它在整个架构中的作用。为了方便记忆，建议读者可以将`Conductor`直接类比为一般分布式框架中的 *worker* 角色，至少在`make_bottles`例子中这么理解没问题

```python
print("Starting conductor with pid: %s" % ME)
my_name = "conductor-%s" % ME
persist_backend = persistence_backends.fetch(PERSISTENCE_URI)
with contextlib.closing(persist_backend):
    with contextlib.closing(persist_backend.get_connection()) as conn:
        conn.upgrade()
    job_backend = job_backends.fetch(my_name, JB_CONF,
                                     persistence=persist_backend)
    job_backend.connect()
    with contextlib.closing(job_backend):
        cond = conductor_backends.fetch('blocking', my_name, job_backend,
                                        persistence=persist_backend)
        # Run forever, and kill -9 or ctrl-c me...
        try:
            cond.run()
        finally:
            cond.stop()
            cond.wait()
```

`Conductor`的核心代码不长 (`on_conductor_event`方法只是记日志，不重要)，连接`Jobboard`和`Persistence`，承接并完成job。

问题来了，代码只写了`PERSISTENCE_URI`和`JB_CONF`，`Conductor`如何获悉`poster`, `job`等其他信息？答案都隐藏在`cond.run()`里，它包含了*承接*，*构建工作流*，*执行* 3个部分

#### 承接
`Conductor`不挑活儿。它会逐一承接`Jobboard`上所有job，直到无job可接。承接job的方式很简单，就是在zookeeper上生成一个`jobX.lock`文件，标记任务已经被某个`Conductor`承接，并在完成job后删除`jobX.lock`。但如果job异常了呢？

"承接"步骤个反直觉的设计: 不管能不能完成，先承接再说。也就是job一旦被某个`Conductor`承接，即便无法完成，也会从`Jobboard`上下架。异常会被记录到数据库，但已经被承接的job不会再分发给其他`Conductor`

还以《巫师》的公告板作比，一旦杰洛特摘下委托书，这份委托就只属于他。无论能否完成，都不会有其他猎魔人抢活。除非委托人 (`poster`)再次往告示板发布新委托书

#### 构建工作流
`Conductor`承接job后会查`Persistence`的meta data，找到`factory function` ("make_bottles")创建工作流。但如果`Conductor`的代码版本有误，"make_bottles"方法有异常，或压根没定义"make_bottles"方法呢？那会在数据库里记录job异常，对其他组件没有更多影响了

#### 执行
如果工作流构建成功，`Conductor`就会开始执行并完成工作流(`Flow`)。整个执行流程是由`Engine`负责

在进一步下沉之前，必须先了解2个问题: 上文提到多次的"工作流"定义是什么？为什么要由Engine执行工作流？

### 工作流定义
* Task (任务): 代表了最基础的工作单元，理想情况下幂等。失败可以回滚，也允许重试。在 Taskflow 中，一个 Task 通常是由一个 Python 函数或类实例化而成的对象，它包含了执行特定工作的逻辑。每个 Task 可以具有输入和输出，这些输入和输出可以用来与其他 Task 之间传递数据
* Flow (工作流): Flow 是由一系列 Task 组成的集合，它定义了 Task 之间的执行顺序或依赖关系。一个 Flow 可以是线性的，即任务按照特定的顺序执行；也可以是更复杂的结构，如图或树状结构。Flow 负责描述 Task 之间的关系，而不是执行它们

因此，**构建工作流** 是指调用`factory function`构建 **有向无环图（DAG）**，并没有真正执行。`Flow`是job的核心抽象，只有当它构建成功后job才进入核心阶段

代码动态构建DAG提供了灵活性，同时也提高了学习成本。对于复杂工作流，有了比"基于Task的输入/输出分支"更高一层的自定义工具。但对于简单工作流，例如拷贝脚本到远端然后执行，它反而有点蹩脚。因为一旦修改`factory function`，集群内的所有`Conductor`都要更新代码并重启。这只能说是设计上的取舍

### Engine (引擎)
一旦有了工作流DAG，那么就得有模块负责维护任务 (Task)之前的关系和输入/输出

Engine (引擎)模块执行 Flow 中的 Task。它负责遍历 Flow 中定义的 Task，并根据 Flow 中指定的依赖和顺序执行它们。Engine 会考虑任务之间的依赖关系，并在必要时进行适当的调度，以确保任务按照预定的方式执行。Engine 还要处理并发执行、故障恢复和重试机制等复杂场景。

如下是官网的[工作流的状态机](https://docs.openstack.org/taskflow/latest/user/states.html)
![](/img/taskflow_note/flow_states.svg#center)

"make_bottles"包含3个任务: "take_bottle", "pass_it", "next_bottles"。其中"take_bottle" return的输出会被作为入参数传给"next_bottles"。这些都是由`Engine`完成的，但仅仅涉及了`start` -> `RUNNING` -> `SUCCESS` 最简单的3个状态变化。`Engine`还要处理`FAILURE`、`REVERTED`等异常情况

### Persistence

### Worker
笔者猜测`Conductor`可能是后加的，是原本*worker*高层抽象。Taskflow里有 [Worker](https://docs.openstack.org/taskflow/latest/user/workers.html#workers)，但它仅用在更复杂分布式架构下，即`Conductor`承接job，实例化工作流`make_bottles`，然后工作流内的每个`Action`发送到远端`Worker`执行。下图是`Conductor`本地执行Action (conductor-1) 和 远程搭配 `Worker`(conductor-2) 的区别
![](/img/taskflow_note/worker_action_engine.png#center)

但什么场景要用到`Conductor`搭配`Worker` 使用呢？我想不到