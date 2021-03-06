---
date: 2019-03-16
status: public
title: ASR训练
---
# 1 总体概览
ASR（Automatic Speech Recognition）是将语音转换为文字的一项技术，其中语言模型是其中的关键，主要涉及到特定领域的说法和词库，通过语言模型能够增加识别的准确率。`ASR-Train` 服务提供了语言模型训练的基本功能，主要的服务架构如下：
![](./_image/ASR-flow.png)
- `asr-trainer-sever`是一个 `HTTP` 服务，接受来自前端的训练请求；
- `asr-product/solution-trainer` 是训练端，通过订阅 `RabbitMQ` 消息来完成训练；
- `resserver` 是一个外部服务，主要负责将训练生成的模型进行拷贝和同步；
- `lmcheck` 是对生成的模型进行检查；
- `callback server` 为通知训练成功的接口。

除此之外，通过 `redis` 保存训练状态；`protobuf` 进行消息序列化合反序列化；`Etcd` 保存热配置。训练请求和训练服务是通过 `RabbitMQ` 解耦完成异步设计，训练请求和`resserver` 是异步设计。
整个服务使用 `Python` 语言开发，使用 `tornado` 框架开发。
# 2 基础内容
## 2.1 RabbitMQ 
`RabbitMQ`是一款优秀的消息中间件， 提供了 `pika` 的 `Python` 语言客户端，而且配合 `tornado` 提供了基于`tornado.ioloop`异步连接。使用的一般步骤为：创建`conneciton`，创建`channel`,  发送消息或者开始接受消息。为了保证服务的可靠性，`RabbitMQ` 还提供了队列序列化，消息序列化和 `ack` 等机制。
但是官方提供的异步的消息发送和接受例子太过于简单，因此基于 `tornado` 和 `pika` 开发一套消息队列 `Adapter`.
```python
class TornadoAdapter(object):
    @corotuine
    def publish(self, exchange, routing_key, body,
                properties=None,  mandatory=True, return_callback=None, close_callback=None):
                    pass
                    
    @coroutine
    def receive(self, exchange, routing_key, queue_name, handler, no_ack=False, prefetch_count=0,
                return_callback=None, close_callback=None, cancel_callback=None):
                    pass
```
为了方便使用，还增加了 `rpc` 调用接口
```python
@coroutine
def rpc(self, exchange, routing_key, body, timeout=None,
            close_callback=None, return_callback=None, cancel_callback=None):
                pass
```
## 2.2 异步进程
语言模型训练是调用的脚本，也就是在另外一个进程中执行，`tornado` 提供了一个 `Subprocess` 可以将一个进程封装成一个 `IO` 事件。为了保证非零退出抛出异常和日志的输出，封装成 `AsyncProcess` 类。
```python
class AsyncProcess(object):
    @coroutine
    def invoke():
        pass
```
## 2.3 协程池
在训练服务中，会出现多个异步进程同时运行，从机器资源角度来看，需要控制进程运行数量，考虑设计协程池，所有异步进程都运行在协程池中。
```python
class CoroutinePool(object):
    def __init__(self, pool_size):
        pass

    @coroutine
    def push(self, f, *args, **kwargs):
           pass
```
`pool_size` 指定协程池大小，所有进程都 `push` 进协程池中。

## 2.4 单元测试
使用 `coverage` 单元测试工具，抱枕代码单元测试覆盖率达到 `90%` 以上。
# 3 设计缺陷
## 3.1 Redis 保存状态
整个训练状态保存在 `redis`，一旦 `redis` 出现宕机，导致整个服务处于无状态的状况。`redis` 中主要包含了如下信息：
```json
{
    "step": "training",
    "callback": "www.dui.ai/asr-train",
    "error": ""
}
```
从函数式编程的角度来看，正式因为所有服务都依赖 `redis` 保存的状态，导致整个架构是 `statusful`，因此需要在调用其他的服务的时候，考虑将这个保存在 `redis` 的信息一并传递给其他服务。

## 3.2 状态丢失
由于在设计架构，和其他部分服务是异步的，如果异步的服务并没有发起回调，那么导致状态出现问题。考虑设计一套定时扫描状态机制，对外部服务超时进行清理。

## 3.3 资源使用监控
在语言模型训练中，如果语料设计的不合理，将会生成的语言模型占用的磁盘空间非常大， 需要对脚本调用进行一次封装，启动一个后台进程对目录的文件大小监控，一旦超出阈值，则将脚本运行的进程 `kill` 掉。
