---
title: "分析一个Python MongoEngine的多层装饰器"
date: 2023-08-01T16:48:08+08:00
tags:
  - it
keywords:
  - python
  - 装饰器
---

在读MongoEngine文档时候看到了一段很牛逼的嵌套装饰器 [@update_modified.apply](https://github.com/MongoEngine/mongoengine/blob/master/docs/guide/signals.rst) ，核心代码如下
```python
def handler(event):
    """Signal decorator to allow use of callback functions as class decorators."""

    def decorator(fn):
        def apply(cls):
            event.connect(fn, sender=cls)
            return cls

        fn.apply = apply
        return fn

    return decorator

@handler(signals.pre_save)
def update_modified(sender, document):
    document.modified = datetime.utcnow()

@update_modified.apply
class Record(Document):
    modified = DateTimeField()
```

上述代码实现等同如下代码的功能，核心目的是让 `event/fn/cls` 都可以组装。但……这个嵌套装饰器具体是如何生效的?
```python
signals.pre_save.connect(update_modified, sender=Record)
```

参考 [《如何理解python装饰器》](https://www.zhihu.com/question/26930016) 可知道如下关系:
* cls = Record
* fn = update_modified
* event = signals.pre_save

用 `handler` 装饰 `update_modified` 返回了装饰器`decorator`，即一个组装的函数定义。将嵌套装饰器转换成单层装饰器
```python
def decorator(fn):
    def apply(cls):
        signals.pre_save.connect(fn, sender=cls)
        return cls

    fn.apply = apply
    return fn
```

装饰器`decorator`有点特别，它没直接返回内部函数`apply`，而是将`apply`赋值给传入函数`fn.apply`后返回`fn`。Python中函数是一等公民，即函数也是对象，`fn.apply = apply` 就使用了这一特性

`@update_modified.apply` 关联了 `event/fn/cls` 3个参数，在`Record`定义时触发`apply`，完成signal注册并`return cls`