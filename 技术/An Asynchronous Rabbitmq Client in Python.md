---
title: An Asynchronous Rabbitmq client in Python
date: 2018-07-22
status: public
---
# 1 Background
Rabbitmq is high performance of `message-queueing` software writing in Erlang. It is used to dispatch messages it receives. The client  connecting  and sending messsages to it is called `producer` and that waiting for message is called `consumer`. The basic skeleton of rabbitmq is as follow:
![](./_image/rabbitmq.png)
Rabbitmq is widely used in a large system which helps to load balances. Many time-wasting processes will bring out wrose user experences. For example, if you purchase online on credit card, what does happen when in backend?  Your order information will be split into various messages and those will be sent to rabbitmq or any other message queue system. The repostiory client will consume this message and prepares the goods for you. The payment client will consume the message and email the bill to you. As you can see, all the works are done by different worker client. As you can see, you don't have to wait the  above jobs and all you to do is "one-click".
If thousands of client send messages to rabbitmq simultaneously, what would like to  occur? 
Traditional rabbitmq's  architecture is the publishers , consumers and rabbitmq server are ditributed among different hosts. publishers and cosumers connect to server with tcp conneciton. As we all kown, IO will hurt system performance because it will invoke system call. Multithreads solution seems to be nice but it also brings out  disasters if you take `C10K` problem into consideration. 
Is any strategies that we can take? The answer is yes, using  asychronous no-blocking framework. 
# 2 Pika RabbitMq Driver
`pika` is rabbitmq driver for python programming language. 
## 2.1 Synchronous Producer
Let's take the first glancing at synchronous producer. 
```python:n
class SyncRabbitMQProducer(object):
    """
    synchronize amqp producer
    usage:
        with SyncAMQPProducer("127.0.0.1") as p:
            p.publish("exchange_name", "dog.black", "message1", "message2")
    """
    def __init__(self, rabbitmq_url):
        """

        :param rabbitmq_url:
        """
        self._parameter = ConnectionParameters("127.0.0.1") if rabbitmq_url in ["localhost", "127.0.0.1"] else \
            URLParameters(rabbitmq_url)
        self._logger = logging.getLogger(__name__)
        self._connection = None
        self._channel = None

    @property
    def logger(self):
        """
        logger
        :return:
        """
        return self._logger

    def _connect(self):
        self.logger.info("initialize connection and channel")
        self._connection = BlockingConnection(self._parameter)
        self._channel = self._connection.channel()

    def _disconnect(self):
        self.logger.info("tear down channel and connection")
        if self._channel is not None:
            self._channel.close()
        if self._connection is not None:
            self._connection.close()

    def __enter__(self):
        self._connect()
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        self._disconnect()
        return not isinstance(exc_val, Exception)

    def publish(self, exchange, routing_key, *messages, **kwargs):
        self.logger.info("[publishing] exchange name: %s; routing key: %s" % (exchange, routing_key,))
        if self._channel is None:
            raise RabbitMQError("channel has not been initialized")
        properties = kwargs.pop("properties") if kwargs.has_key("properties") else None
        for message in messages:
            self.logger.info("publish message: %s" % message)
            self._channel.basic_publish(exchange=exchange,
                                        routing_key=routing_key,
                                        body=str(message),
                                        properties=properties)

    def publish_messages(self, exchange, messages, **kwargs):
        self.logger.info("[publish_message] exchange name: %s" % exchange)
        if self._channel is None:
            raise RabbitMQError("channel has not been initialized")
        if not isinstance(messages, dict):
            raise RabbitMQError("messages is not dict")
        properties = kwargs.pop("properties") if kwargs.has_key("properties") else None
        for routing_key, message in messages.items():
            self.logger.info("routing key:%s, message: %s" % (routing_key, message, ))
            self._channel.basic_publish(exchange=exchange,
                                        routing_key=routing_key,
                                        body=str(message),
                                        properties=properties)

    def connect(self):
        """
        This is method doesn't recommend. using `with` context instead
        :return: None
        """
        warnings.warn("Call connect() method, using `with` context instead.", category=DeprecationWarning, stacklevel=2)
        self._connect()

    def disconnect(self):
        """
        This is method doesn't recommend. using `with` context instead
        :return: None
        """
        warnings.warn("Call disconnect() method, using `with` context instead.", category=DeprecationWarning, stacklevel=2)
        self._disconnect()
```
How can we use this publisher client?
```python
with SyncAMQPProducer("127.0.0.1", "exchange_name") as p:
        p.publish("topic.one", "message to send")
```
When we create a instance of `SyncRabbitMQProducer`, it establishes a tcp connection with rabbitmq server with `BlockingConnection` which means it will block the process.  When invokes `publish` method, it will check whether channel has been built and then publish messages.  
## 2.2 Drawbacks
The handshake process for a RabbimtMQ connection is actually quite involved and requires at least 7 TCP packets (more if TLS is used). Each publisher establishes a connection, creates a new channel, sends messages and closes channel and conenction. Most of time is wasted in system IO. How can we solve this problem? The answer is `TornadoConnection` which is provided by `pika` library. You can using `tornado` io loop to achieve aynchronouse connection. 
# 3 Asynchronous client
## 3.1 Guidelines
- Each rabbitmq client has two connections, one for publishing message and the other for consuming message.
- Creating a new channel as long as it publish message.
- Aside from publisher and consumer, it have to  plays a role of rpc.
- The client must to be roubusted which means you have to handler errors as much as possible.
## 3.2 Synchronous connection
```python:n
@gen.coroutine
def connect(self):
        """
        establishing two connections for publishing and receiving respectively.
        :return: True if establish successfully.
        """
        self._publish_connection = yield self._create_connection(self._parameter)
        self._receive_connection = yield self._create_connection(self._parameter)
        raise gen.Return(True)

def _create_connection(self, parameter):
        self.logger.info("creating connection")
        future = Future()

        def open_callback(unused_connection):
            self.logger.info("created connection")
            future.set_result(unused_connection)

        def open_error_callback(connection, exception):
            self.logger.error("open connection with error: %s" % exception)
            future.set_exception(exception)

        def close_callback(connection, reply_code, reply_text):
            if reply_code not in [self._NORMAL_CLOSE_CODE,]:
                self.logger.error("closing connection: reply code:%s, reply_text: %s. system will exist" % (reply_code, reply_text,))
                sys.exit(self._EXIST_CODE)

        TornadoConnection(parameter,
                          on_open_callback=open_callback,
                          on_open_error_callback=open_error_callback,
                          on_close_callback=close_callback,
                          custom_ioloop=self._io_loop)
        return future
```
Here in line `28`, we create new instance of `TornadoConnection` but its initialization is asynchronous, it will yield out by io loop. Once connection is open, it will callback `open_callback` in line `15`. We set connection into `Future` as result. In line `2`, the `connect` method wait those two connection to establish before it return. But we should be aware of `connect` method is also asynchronous. 
## 3.3 Create new channel
We have to admit the fact that RabbiMQ connections are a lot more resouce heavy than channel. Connection should be long live, and channels can be open and closed frequently. So our strategy is that we create a new chanel for each publish and consume. After publishing message, the channel will be useless so we choose to  close it as soon as possible. 
```python:n
    def _create_channel(self, connection):
        self.logger.info("creating channel")
        future = Future()

        def on_channel_closed(channel, reply_code, reply_txt):
            if reply_code not in [self._NORMAL_CLOSE_CODE, self._USER_CLOSE_CODE]:
                self.logger.error("channel closed. reply code: %s; reply text: %s. system will exist"
                                  % (reply_code, reply_txt,))
                sys.exit(self._EXIST_CODE)

        def open_callback(channel):
            self.logger.info("created channel")
            channel.add_on_close_callback(on_channel_closed)
            future.set_result(channel)

        connection.channel(on_open_callback=open_callback)
        return future
```
I guess you have known how the code works. It does not create a new channel as `channel()` method called in line `16`. It will invoke `open_callback` in line `11` once the channel is open. 
## 3.4 Exchange, Queue and Routing Key
`exchange`, `queue` and `routing key` are basic terminologies in RabbiMQ. All of them can be declared or bound in a synchronous way. 
```python:n
def _exchange_declare(self, channel, exchange=None, exchange_type='topic', **kwargs):
        self.logger.info("declaring exchange: %s " % exchange)
        future = Future()

        def callback(unframe):
            self.logger.info("declared exchange: %s" % exchange)
            future.set_result(unframe)

        channel.exchange_declare(callback=callback,
                                 exchange=exchange, exchange_type=exchange_type, **kwargs)
        return future

def _queue_declare(self, channel, queue='', **kwargs):
        self.logger.info("declaring queue: %s" % queue)
        future = Future()

        def callback(method_frame):
            self.logger.info("declared queue: %s" % method_frame.method.queue)
            future.set_result(method_frame.method.queue)

        channel.queue_declare(callback=callback, queue=queue, **kwargs)
        return future

def _queue_bind(self, channel, queue, exchange, routing_key=None, **kwargs):
        self.logger.info("binding queue: %s to exchange: %s" % (queue, exchange,))
        future = Future()

        def callback(unframe):
            self.logger.info("bound queue: %s to exchange: %s" % (queue, exchange,))
            future.set_result(unframe)
        channel.queue_bind(callback, queue=queue, exchange=exchange, routing_key=routing_key, **kwargs)
        return future
```
Those three methods share the same idea with previous asynchronouse code. 

## 3.4 publish
```python:n
    @gen.coroutine
    def publish(self, exchange, routing_key, body, properties=None):
        """
        publish message. creating a brand new channel once invoke this method. After publishing, it closes the
        channel.
        :param exchange: exchange name
        :type exchange; str or unicode
        :param routing_key: routing key (e.g. dog.yellow, cat.big)
        :param body: message
        :param properties: properties
        :return: None
        """
        self.logger.info("[publishing] exchange: %s; routing key: %s; body: %s." % (exchange, routing_key, body,))
        channel = yield self._create_channel(self._publish_connection)
        channel.basic_publish(exchange=exchange, routing_key=routing_key, body=body, properties=properties)
        channel.close()
```
All the codes are self-explanatory.
## 3.5 receive
As we metioned above, we have to take rpc into consideration. So after consuming the message by passing handler, we check the message properties whether it was not `None`. If it was not none, we create a channel to publish result into RabbitMQ. Here is the code:
```python：n
@gen.coroutine
def receive(self, exchange, routing_key, queue_name, handler, no_ack=False, prefetch_count=0):
        """
        receive message. creating a brand new channel to consume message. Before consuming, it have to declaring
        exchange and queue. And bind queue to particular exchange with routing key. if received properties is not
        none, it publishes result back to `reply_to` queue.
        :param exchange: exchange name
        :param routing_key: routing key (e.g. dog.*, *.big)
        :param queue_name: queue name
        :param handler: message handler,
        :type handler def fn(body)
        :param no_ack: ack
        :param prefetch_count: prefetch count
        :return: None
        """
        self.logger.info("[receive] exchange: %s; routing key: %s; queue name: %s" % (exchange, routing_key, queue_name,))
        channel = yield self._create_channel(self._publish_connection)
        yield self._exchange_declare(channel, exchange=exchange)
        yield self._queue_declare(channel, queue=queue_name, auto_delete=True)
        yield self._queue_bind(channel, exchange=exchange, queue=queue_name, routing_key=routing_key)
        self.logger.info("[start consuming] exchange: %s; routing key: %s; queue name: %s" % (exchange,
                                                                                              routing_key, queue_name,))
        channel.basic_qos(prefetch_count=prefetch_count)
        channel.basic_consume(functools.partial(self._on_message, exchange=exchange, handler=handler)
                              , queue=queue_name, no_ack=no_ack)

def _on_message(self, unused_channel, basic_deliver, properties, body, exchange, handler=None):
        self.logger.info("consuming message: %s" % body)
        self._io_loop.spawn_callback(self._process_message, unused_channel, basic_deliver, properties, body,
                                     exchange, handler)

@gen.coroutine
def _process_message(self, unused_channel, basic_deliver, properties, body, exchange, handler=None):
        try:
            result = yield handler(body)
            self.logger.info("%s has been processed successfully and result is  %s" % (body, result,))
            if properties is not None \
                    and properties.reply_to is not None \
                    and properties.correlation_id is not None:
                self.logger.info("sending result back to %s" % properties.reply_to)
                self.publish(exchange=exchange,
                             routing_key=properties.reply_to,
                             properties=BasicProperties(correlation_id=properties.correlation_id),
                             body=str(result))
            unused_channel.basic_ack(basic_deliver.delivery_tag)
        except Exception:
            unused_channel.basic_ack(basic_deliver.delivery_tag)
            import traceback
            self.logger.error(traceback.format_exc())
```
Different from `publish`, `receive` declares exchange and queue and binds routing key ahead. 

## 3.6 rpc
RabbitMQ helps us to implement a micro `Remote Process Call`(rpc) framework. We send message to the queue and consumer receive and handle it. After then  sends result back into reabbitmq. Finally, we complete a rpc procedure. 
```python:n
@gen.coroutine
def rpc(self, exchange, routing_key, body, timeout=None):
        """
        rpc call. It create a queue randomly when encounters first call with the same exchange name. Then, it starts
        consuming the created queue(waiting result). It publishes message to rabbitmq with properties that has correlation_id
        and reply_to. if timeout is set, it starts a coroutine to wait timeout and raises an `Exception("timeout")`.
        If server has been sent result, it return it asynchronously.
        :param exchange: exchange name
        :param routing_key: routing key(e.g. dog.Yellow, cat.big)
        :param body: message
        :param timeout: timeout
        :return: result or Exception("timeout")
        """
        self.logger.info("rpc call. exchange: %s; routing_key: %s; body: %s" % (exchange, routing_key, body,))
        if exchange not in self._rpc_exchange_dict:
            self._rpc_exchange_dict[exchange] = Queue(maxsize=1)
            callback_queue = yield self._initialize_rpc_callback(exchange)
            yield self._rpc_exchange_dict[exchange].put(callback_queue)
        callback_queue = yield self._rpc_exchange_dict[exchange].get()
        yield self._rpc_exchange_dict[exchange].put(callback_queue)
        self.logger.info("starting calling. %s" % body)
        result = yield self._call(exchange, callback_queue, routing_key, body, timeout)
        raise gen.Return(result)

@gen.coroutine
def _initialize_rpc_callback(self, exchange):
        self.logger.info("initialize rpc callback queue")
        rpc_channel = yield self._create_channel(self._receive_connection)
        yield self._exchange_declare(rpc_channel, exchange)
        callback_queue = yield self._queue_declare(rpc_channel, auto_delete=True)
        self.logger.info("callback queue: %s" % callback_queue)
        yield self._queue_bind(rpc_channel, exchange=exchange, queue=callback_queue, routing_key=callback_queue)
        rpc_channel.basic_consume(self._rpc_callback_process, queue=callback_queue)
        raise gen.Return(callback_queue)

def _rpc_callback_process(self, unused_channel, basic_deliver, properties, body):
        if properties.correlation_id in self._rpc_corr_id_dict:
            self._rpc_corr_id_dict[properties.correlation_id].set_result(body)

def _call(self, exchange, callback_queue, routing_key, body, timeout=None):
        future = Future()
        corr_id = str(uuid.uuid1())
        self._rpc_corr_id_dict[corr_id] = future
        self.publish(exchange, routing_key, body,
                     properties=BasicProperties(correlation_id=corr_id,
                                                reply_to=callback_queue))

        def on_timeout():
            self.logger.error("timeout")
            del self._rpc_corr_id_dict[corr_id]
            future.set_exception(RabbitMQError('rpc timeout'))

        if timeout is not None:
            self._io_loop.add_timeout(float(timeout), on_timeout)
        return future
```
To avoid declaring exchange and queue repeatly, we take `exchange` as key to prevent it. And Creating callback queue randomly by RabbitMQ. Before we send mesage to RabbitMQ, we start a consume wait result from `server` in callback queue.  In `_call` method,  we using `uuid` to generate a unique key for this call and a `Future` to wait result asynchronous. Apart from basic feature, the `rpc` provider timeout to prevent unexpected condition from `server`.  

# 4 Conclusion
Using  `TornadoConnection`, our codes hit a high performance. All the code can be viewed in [gist](https://gist.github.com/gaufung/ba64df0fb210f5e8f41e15b279589ea7).