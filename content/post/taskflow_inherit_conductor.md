---
title: "Taskflow自定义Conductor"
date: 2025-12-03T13:44:42+08:00
tags:
  - it
keywords:
  - taskflow
---

Taskflow原生的 `blocking/nonblocking` Conductor不支持Job过滤，也就是无差别消费所有Job。如果要实现特殊Job特殊Conductor处理，需要自定义Conductor。什么Job算特殊？例如有某个工作流需要GPU资源，且一个Conductor集群里只有个别机器有GPU。此时最好的调度策略是有GPU的机器运行GPU Job。如果GPU机器空闲且其他机器忙，也可以运行不需要GPU的Job

## 过滤Job

Conductor过滤Job重载`_can_claim_more_jobs`即可。下面代码的`has_gpu`没实现，仅用于说明逻辑: 无GPU的机器不会执行需要GPU的Job；有GPU的机器会执行所有Job
```python
from taskflow.conductors.backends.impl_nonblocking import NonBlockingConductor


class GpuConductor(NonBlockingConductor):

    def has_gpu(self):
        pass # TODO 获取GPU状态

    def _can_claim_more_jobs(self, job):
        if not super()._can_claim_more_jobs(job):
            return False

        gpu = job.details.get("gpu")
        if gpu and not self.has_gpu():
            return False
        return True

```

## stevedore
Taskflow实例化Conductor依赖 [stevedore.driver.DriverManager](https://github.com/openstack/taskflow/blob/master/taskflow/conductors/backends/__init__.py#L36)，其基于命名空间和entry point在运行时加载Conductor。因此`GpuConductor`必须添加到`taskflow.conductors`命名空间下。办法是写`pyproject.toml`
```toml
[project.entry-points."taskflow.conductors"]
gpu = "custom_conductor.gpu_conductor:GpuConductor"
```

编辑安装包。注意，必须安装后stevedore才能在环境的分发元数据里发现新的entry point。直接跑源代码而不安装，stevedore看不到
```bash
pip install -e .
```

安装验证
```python
from stevedore.extension import ExtensionManager
em = ExtensionManager("taskflow.conductors")
print([e.name for e in em.extensions]) # 应包含 "gpu"
```

注册entry point不影响修改源代码，即修改`GpuConductor`代码不需要重新安装

## job post

发布Job时，需要指定`details`

```python
job_backend = job_backends.fetch(name, conf, persistence=persist_backend)

details = {"gpu": True}
job_backend.post(flow_name, logbook, details)
```