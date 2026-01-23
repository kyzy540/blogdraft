---
title: "Kombu + Redis的Worker发现"
date: 2026-01-23T21:35:55+08:00
tags:
  - it
keywords:
  - python
---

分布式系统中，发现远端worker是一种常见需求。例如：一个计算任务需要GPU，而集群中只有少数机器配备。常见思路是让 管控/调度服务 发现GPU机部（worker），并把计算任务代理给worker执行，接收返回结果

一种经典解决方案是心跳和注册中心模式：
* **心路广播**: Worker 启动后，每隔 N 秒向一个 fanout 交换机发送包含自己IP 的消息
* **存活判定（TTL）**: 注册中心监听该交换机，将收到的IP 和当前时间存入内存字典
* **自动剔除**: 注册中心定期检查字典，如果某个 IP 超过 N*3秒没有更新，则判定该 Worker 已下线

Python中可以基于Kombu和Redis实现：*worker.py*上报心跳，*proxy.py*是注册中心（将计算任务代理给worker执行）

```python
import time
from kombu import Connection, Exchange

def run_worker():
    with Connection("redis://127.0.0.1:6379") as conn:
        heartbeat_exchange = Exchange("worker_discovery", type="fanout")
        producer = conn.Producer()
        worker_ip = "127.0.0.1"
        print(f"[*] Worker {worker_ip} started...")
        while True:
            payload = {"ip": worker_ip, "ts": time.time()}
            producer.publish(
                payload,
                exchange=heartbeat_exchange,
                declare=[heartbeat_exchange], # 不加declare proxy收不到消息
                serializer="json",
            )
            print(f"[#] Heartbeat sent: {time.ctime()}")
            time.sleep (5)

if __name__ == "__main__":
    run_worker()

```

```python
import time
import socket
from kombu import Connection, Exchange, Queue

def start_proxy():
    active_workers = {}
    timeout_threshold = 15
    conn = Connection("redis://127.0.0.1:6379")
    heartbeat_exchange = Exchange("worker_discovery", type="fanout")

    # 注意：如果不指定名字 Kombu 会随机生成一个
    discovery_queue = Queue("example", exchange=heartbeat_exchange, exclusive-True)
    def process_message(body, message):
        ip = body.get("ip")
        active_workers[ip] = body.get("ts")
        print(f"[+] Discovered/Updated worker: {ip}")
        message.ack()

    print("[*] Proxy is listening for heartbeats...")
    with conn.Consumer(queues=[discovery_queue], callbacks=[process_message]):
        last_check = time.time()
        while True:
            try:
                conn.drain_events(timeout=1)
            except socket.timeout:
                # 即使没有收到消息也定期检查 worker 存活状态
                pass

            # 每隔 5 秒打印一次当前在线列表
            if time.time() - last_check > 5:
                now = time.time()
                # 清理超时 Worker
                dead_workers = [
                    ip
                    for ip, ts in active workers.items()
                    if now - ts > timeout_threshold
                ]
                for ip in dead_workers:
                    del active_workers[ip]
                    print(f"[-] Worker {ip} timed out.")

                print(f"Current Online: {list(active_workers.keys())}")
                last_check = time.time()

if __name__ == "__main__":
    try:
        start_proxy()
    except KeyboardInterrupt:
        print("Exit.")
```

## fanout交换机
`Exchange(type="fanout")`是广播交换机，对应AMOP中topic广播概念，对应[《Redis Pub/sub》](https://redis.io/docs/latest/develop/pubsub/)。广播消息指生产者向topic发一次消息，此时订阅topic的所有消费者都会收到消息。但对于发消息时没订阅topic的消费者，没法收到历史消息。Redis的channel机制和topic一样

Exchange交换机是Kombu特有的概念，核心是为了解耦生产者与消费者，并提供灵活的路由策略。Exchange可以绑定多个消息队列，配置复杂路由规则。如此，生成者可以只关心“消息怎么发”，把“消息发给谁”的问题留给Exchange

在例子中，proxy.py启动时会在Redis中创建名为"worker_discovery"的set，写入一个key: "example"，并订阅名为"example"的channel。worker.py会先读"worker_discovery"set，把心跳消息发给set里所有key对应的channel

这意味着worker和proxy可以是多对多的。通过"worker_discovery"确定消息发给谁就是Exchange的作用
如果只启动worker不启动proxy呢？程序也不会报错，在worker看来消息已经发出去了。很健壮

## publish(declare=[heartbeat_exchange], ...)
worker发心跳消息的代码有另一种写法，比上面简洁，publish少传declare参数
```python
producer = conn.Producer(exchange=heartbeat_exchange)
producer.publish(payload, serializer="json")
```

如果publish不传declare会如何？
```python
producer = conn.Producer()
producer.publish(payload, exchange=heartbeat_exchange, serializer="json")
```

虽然代码看似语义相同，但功能完全不对。worker会创建名为"example"的list，写入心跳消息，而非将消息发到channel。proxy永远收不到心跳

问题关键在[Producer.publish](https://github.com/celery/kombu/blob/5208431c95bda47c7f422638dd273e086ab34be9/kombu/messaging.py#L182)的代码中
```python
if self.auto_declare and self.exchange.name:
    if self.exchange not in declare:
        # XXX declare should be a Set.
        declare.append(self.exchange)
```

把exchange作为初始化参数或declare参数传给Producer时，会触发`maybe_declare`逻辑，自动绑定exchange到channel，从而正确广播消息。否则将执行默认行为: 发到消息队列也就是Redis的list中

因此，正确代码还有第三种写法，给exchange先绑定channel，然后发消息
```python
heartbeat_exchange.declare(channel=conn.channel())
producer = conn.Producer()
producer.publish(payload, exchange=heartbeat_exchange, serializer="json")
```

对了，Redis的channel广播不区分数据库，因此链接字符串不用写数据库序号
