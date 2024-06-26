# RabbitMQ 原理

> 整体阅读时间，在 40 分钟左右。

常见的消息队列很多，主要包括 RabbitMQ、Kafka、RocketMQ 和 ActiveMQ，相关的选型可以看我之前的系列，**这篇文章只讲 RabbitMQ，先讲原理，后搞实战。**

思维导图：

<img src="./images/sXFqMxQoVLEBqF1pxoH0xnM8THYI4LSL5cRfpJNPML2UIEeDHWf5tdj6Rc7Ep3QPjiaSwzMhU91hR72dYD73Vfw.png" alt="图片" style="zoom:50%;"/>



# 1. 消息队列

##  1.1 消息队列模式

消息队列目前主要 2 种模式，分别为“点对点模式”和“发布/订阅模式”。

#### 1.1.1 点对点模式

一个具体的消息只能由一个消费者消费，多个生产者可以向同一个消息队列发送消息，但是一个消息在被一个消息者处理的时候，这个消息在队列上会被锁住或者被移除并且其他消费者无法处理该消息。

需要额外注意的是，如果消费者处理一个消息失败了，消息系统一般会把这个消息放回队列，这样其他消费者可以继续处理。

<img src="./images/sXFqMxQoVLEBqF1pxoH0xnM8THYI4LSLaKTHNFbmRshmZtCMmVIDWuHO6303CPfhDe0azAUAFk1eIQG7LhUhxQ.png" alt="图片" style="zoom:80%;" />

#### 1.1.2 发布/订阅模式

单个消息可以被多个订阅者并发的获取和处理。一般来说，订阅有两种类型：

- **临时（ephemeral）订阅**：这种订阅只有在消费者启动并且运行的时候才存在。一旦消费者退出，相应的订阅以及尚未处理的消息就会丢失。
- **持久（durable）订阅**：这种订阅会一直存在，除非主动去删除。消费者退出后，消息系统会继续维护该订阅，并且后续消息可以被继续处理。

<img src="./images/sXFqMxQoVLEBqF1pxoH0xnM8THYI4LSLJCEZDaMHKMCicqOEWgS5PTpe4tpkeb0msIamDxJ7g2pJOQWzg8JXKwA.png" alt="图片" style="zoom:80%;" />



## 1.2 衡量标准

对消息队列进行技术选型时，需要通过以下指标衡量你所选择的消息队列，是否可以满足你的需求：

- **消息顺序**：发送到队列的消息，消费时是否可以保证消费的顺序，比如A先下单，B后下单，应该是A先去扣库存，B再去扣，顺序不能反。
- **消息路由**：根据路由规则，只订阅匹配路由规则的消息，比如有A/B两者规则的消息，消费者可以只订阅A消息，B消息不会消费。
- 消息可靠性：是否会存在丢消息的情况，比如有A/B两个消息，最后只有B消息能消费，A消息丢失。
- **消息时序**：主要包括“消息存活时间”和“延迟/预定的消息”，“消息存活时间”表示生产者可以对消息设置TTL，如果超过该TTL，消息会自动消失；“延迟/预定的消息”指的是可以延迟或者预订消费消息，比如延时5分钟，那么消息会5分钟后才能让消费者消费，时间未到的话，是不能消费的。
- **消息留存**：消息消费成功后，是否还会继续保留在消息队列。
- **容错性**：当一条消息消费失败后，是否有一些机制，保证这条消息是一种能成功，比如异步第三方退款消息，需要保证这条消息消费掉，才能确定给用户退款成功，所以必须保证这条消息消费成功的准确性。
- **伸缩**：当消息队列性能有问题，比如消费太慢，是否可以快速支持库容；当消费队列过多，浪费系统资源，是否可以支持缩容。
- **吞吐量**：支持的最高并发数。

# 2. RabbitMQ 原理初探

RabbitMQ 2007 年发布，是使用 Erlang 语言开发的开源消息队列系统，基于 AMQP 协议来实现。

## 2.1 基本概念

提到RabbitMQ，就不得不提AMQP协议。AMQP协议是具有现代特征的二进制协议。是一个提供统一消息服务的应用层标准高级消息队列协议，是应用层协议的一个开放标准，为面向消息的中间件设计。

先了解一下AMQP协议中间的几个重要概念：

- Server：接收客户端的连接，实现AMQP实体服务。
- Connection：连接，应用程序与Server的网络连接，TCP连接。
- Channel：信道，消息读写等操作在信道中进行。客户端可以建立多个信道，每个信道代表一个会话任务。
- Message：消息，应用程序和服务器之间传送的数据，消息可以非常简单，也可以很复杂。由Properties和Body组成。Properties为外包装，可以对消息进行修饰，比如消息的优先级、延迟等高级特性；Body就是消息体内容。
- Virtual Host：虚拟主机，用于逻辑隔离。一个虚拟主机里面可以有若干个Exchange和Queue，同一个虚拟主机里面不能有相同名称的Exchange或Queue。
- Exchange：交换器，接收消息，按照路由规则将消息路由到一个或者多个队列。如果路由不到，或者返回给生产者，或者直接丢弃。RabbitMQ常用的交换器常用类型有direct、topic、fanout、headers四种，后面详细介绍。
- Binding：绑定，交换器和消息队列之间的虚拟连接，绑定中可以包含一个或者多个RoutingKey。
- RoutingKey：路由键，生产者将消息发送给交换器的时候，会发送一个RoutingKey，用来指定路由规则，这样交换器就知道把消息发送到哪个队列。路由键通常为一个“.”分割的字符串，例如“com.rabbitmq”。
- Queue：消息队列，用来保存消息，供消费者消费。

## 2.2 工作原理

AMQP 协议模型由三部分组成：生产者、消费者和服务端，执行流程如下：

1. 生产者是连接到 Server，建立一个连接，开启一个信道。
2. 生产者声明交换器和队列，设置相关属性，并通过路由键将交换器和队列进行绑定。
3. 消费者也需要进行建立连接，开启信道等操作，便于接收消息。
4. 生产者发送消息，发送到服务端中的虚拟主机。
5. 虚拟主机中的交换器根据路由键选择路由规则，发送到不同的消息队列中。
6. 订阅了消息队列的消费者就可以获取到消息，进行消费。

<img src="./images/sXFqMxQoVLEBqF1pxoH0xnM8THYI4LSLfGKVOuJwUEecElnO27UfgEORmamd6qnr7AZbNpXn8kLvjWkEwtWkjQ.png" alt="图片" style="zoom:50%;" />



## 2.3 常用交换器

RabbitMQ常用的交换器类型有direct、topic、fanout、headers四种：

- Direct Exchange：见文知意，直连交换机意思是此交换机需要绑定一个队列，要求该消息与一个特定的路由键完全匹配。简单点说就是一对一的，点对点的发送。

<img src="./images/sXFqMxQoVLEBqF1pxoH0xnM8THYI4LSLOEDMv7DlLR5XK5AaNmQjJicNural7bQlYL3cGibB0TUqA6mVhlqqKfMg.png" alt="图片" style="zoom:50%;" />

- Fanout Exchange：这种类型的交换机需要将队列绑定到交换机上。一个发送到交换机的消息都会被转发到与该交换机绑定的所有队列上。很像子网广播，每台子网内的主机都获得了一份复制的消息。简单点说就是发布订阅。

<img src="./images/sXFqMxQoVLEBqF1pxoH0xnM8THYI4LSLmOPzRtptiagyIxB2gANA8zhExhXK9xKcHCtgzVJL2DngRK4lWj0nfGA.png" alt="图片" style="zoom:50%;" />

- Topic Exchange：直接翻译的话叫做主题交换机，如果从用法上面翻译可能叫通配符交换机会更加贴切。这种交换机是使用通配符去匹配，路由到对应的队列。通配符有两种："*" 、 "#"。需要注意的是通配符前面必须要加上"."符号。
- - - *符号：有且只匹配一个词。比如 a.*可以匹配到"a.b"、"a.c"，但是匹配不了"a.b.c"。
    - \#符号：匹配一个或多个词。比如"rabbit.#"既可以匹配到"rabbit.a.b"、"rabbit.a"，也可以匹配到"rabbit.a.b.c"。

<img src="./images/sXFqMxQoVLEBqF1pxoH0xnM8THYI4LSL6tdcfxDibT4x8jiaUfFzjAjbkBISLkF302AjbZ3COpnibdnz5BAicoicH1g.png" alt="图片" style="zoom:50%;" />

- Headers Exchange：这种交换机用的相对没这么多。它跟上面三种有点区别，它的路由不是用routingKey进行路由匹配，而是在匹配请求头中所带的键值进行路由。创建队列需要设置绑定的头部信息，有两种模式：全部匹配和部分匹配。如上图所示，交换机会根据生产者发送过来的头部信息携带的键值去匹配队列绑定的键值，路由到对应的队列。

<img src="./images/sXFqMxQoVLEBqF1pxoH0xnM8THYI4LSLL0WkSsAlU7jwic8y84gKMEclWXTUYicHa4h9TS02bt1hicPJZbib09BX3A.png" alt="图片" style="zoom:50%;" />

## 2.4 消费原理

我们先看几个基本概念：

- broker：每个节点运行的服务程序，功能为维护该节点的队列的增删以及转发队列操作请求。
- master queue：每个队列都分为一个主队列和若干个镜像队列。
- mirror queue：镜像队列，作为master queue的备份。在master queue所在节点挂掉之后，系统把mirror queue提升为master queue，负责处理客户端队列操作请求。注意，mirror queue只做镜像，设计目的不是为了承担客户端读写压力。

集群中有两个节点，每个节点上有一个broker，每个broker负责本机上队列的维护，并且borker之间可以互相通信。集群中有两个队列A和B，每个队列都分为master queue和mirror queue（备份）。那么队列上的生产消费怎么实现的呢？

<img src="./images/sXFqMxQoVLEBqF1pxoH0xnM8THYI4LSLQe3iaJleYL1EmuN00KdaJ8qGgB9zP4PRIgRzZicCuzFDIG7Fn3p5RvdQ.png" alt="图片" style="zoom:50%;" />

对于消费队列，如下图有两个consumer消费队列A，这两个consumer连在了集群的不同机器上。RabbitMQ集群中的任何一个节点都拥有集群上所有队列的元信息，所以连接到集群中的任何一个节点都可以，主要区别在于有的consumer连在master queue所在节点，有的连在非master queue节点上。

因为mirror queue要和master queue保持一致，故需要同步机制，正因为一致性的限制，导致所有的读写操作都必须都操作在master queue上（想想，为啥读也要从master queue中读？和数据库读写分离是不一样的），然后由master节点同步操作到mirror queue所在的节点。即使consumer连接到了非master queue节点，该consumer的操作也会被路由到master queue所在的节点上，这样才能进行消费。

<img src="./images/sXFqMxQoVLEBqF1pxoH0xnM8THYI4LSLaRoOYUSQiboEMJSR7SHzPYF9djSJia9d1lM9LMGhJCQmCSKHf7Aib8plw.png" alt="图片" style="zoom:50%;" />

对于生成队列，原理和消费一样，如果连接到非 master queue 节点，则路由过去。

<img src="./images/sXFqMxQoVLEBqF1pxoH0xnM8THYI4LSLQBcpdYF0GjOncSNGaaKxHN7ObOOqBE4MqibAQntd5FviaIY1XDBLBtTg.png" alt="图片" style="zoom:50%;" />

> 所以，到这里小伙伴们就可以看到 RabbitMQ的不足：由于master queue单节点，导致性能瓶颈，吞吐量受限。虽然为了提高性能，内部使用了Erlang这个语言实现，但是终究摆脱不了架构设计上的致命缺陷。

## 2.5 高级特性

#### 2.5.1 过期时间

Time To Live，也就是生存时间，是一条消息在队列中的最大存活时间，单位是毫秒，下面看看RabbitMQ过期时间特性：

- RabbitMQ可以对消息和队列设置TTL。
- RabbitMQ支持设置消息的过期时间，在消息发送的时候可以进行指定，每条消息的过期时间可以不同。
- RabbitMQ支持设置队列的过期时间，从消息入队列开始计算，直到超过了队列的超时时间配置，那么消息会变成死信，自动清除。
- 如果两种方式一起使用，则过期时间以两者中较小的那个数值为准。
- 当然也可以不设置TTL，不设置表示消息不会过期；如果设置为0，则表示除非此时可以直接将消息投递到消费者，否则该消息将被立即丢弃。

#### 2.5.2 消息确认

为了保证消息从队列可靠地到达消费者，RabbitMQ提供了消息确认机制。

消费者订阅队列的时候，可以指定autoAck参数，当autoAck为true的时候，RabbitMQ采用自动确认模式，RabbitMQ自动把发送出去的消息设置为确认，然后从内存或者硬盘中删除，而不管消费者是否真正消费到了这些消息。

当autoAck为false的时候，RabbitMQ会等待消费者回复的确认信号，收到确认信号之后才从内存或者磁盘中删除消息。

消息确认机制是RabbitMQ消息可靠性投递的基础，只要设置autoAck参数为false，消费者就有足够的时间处理消息，不用担心处理消息的过程中消费者进程挂掉后消息丢失的问题。

#### 2.5.3 持久化

消息的可靠性是RabbitMQ的一大特色，那么RabbitMQ是如何保证消息可靠性的呢？答案就是消息持久化。持久化可以防止在异常情况下丢失数据。RabbitMQ的持久化分为三个部分：交换器持久化、队列持久化和消息的持久化。

交换器持久化可以通过在声明队列时将durable参数设置为true。如果交换器不设置持久化，那么在RabbitMQ服务重启之后，相关的交换器元数据会丢失，不过消息不会丢失，只是不能将消息发送到这个交换器了。

队列的持久化能保证其本身的元数据不会因异常情况而丢失，但是不能保证内部所存储的消息不会丢失。要确保消息不会丢失，需要将其设置为持久化。队列的持久化可以通过在声明队列时将durable参数设置为true。

设置了队列和消息的持久化，当RabbitMQ服务重启之后，消息依然存在。如果只设置队列持久化或者消息持久化，重启之后消息都会消失。

当然，也可以将所有的消息都设置为持久化，但是这样做会影响RabbitMQ的性能，因为磁盘的写入速度比内存的写入要慢得多。

对于可靠性不是那么高的消息可以不采用持久化处理以提高整体的吞吐量。鱼和熊掌不可兼得，关键在于选择和取舍。在实际中，需要根据实际情况在可靠性和吞吐量之间做一个权衡。

#### 2.5.4 死信队列

当消息在一个队列中变成死信之后，他能被重新发送到另一个交换器中，这个交换器成为死信交换器，与该交换器绑定的队列称为死信队列。

消息变成死信有下面几种情况：

- 消息被拒绝。
- 消息过期
- 队列达到最大长度

DLX也是一个正常的交换器，和一般的交换器没有区别，他能在任何的队列上面被指定，实际上就是设置某个队列的属性。当这个队列中有死信的时候，RabbitMQ会自动将这个消息重新发送到设置的交换器上，进而被路由到另一个队列，我们可以监听这个队列中消息做相应的处理。

死信队列有什么用？当发生异常的时候，消息不能够被消费者正常消费，被加入到了死信队列中。后续的程序可以根据死信队列中的内容分析当时发生的异常，进而改善和优化系统。

#### 2.5.5 延迟队列

一般的队列，消息一旦进入队列就会被消费者立即消费。延迟队列就是进入该队列的消息会被消费者延迟消费，延迟队列中存储的对象是的延迟消息，“延迟消息”是指当消息被发送以后，等待特定的时间后，消费者才能拿到这个消息进行消费。

延迟队列用于需要延迟工作的场景。最常见的使用场景：淘宝或者天猫我们都使用过，用户在下单之后通常有30分钟的时间进行支付，如果这30分钟之内没有支付成功，那么订单就会自动取消。

除了延迟消费，延迟队列的典型应用场景还有延迟重试。比如消费者从队列里面消费消息失败了，可以延迟一段时间以后进行重试。

## 2.6 特性分析

这里才是内容的重点，不仅需要知道Rabbit的特性，还需要知道支持这些特性的原因：

- **消息路由（支持）**：RabbitMQ可以通过不同的交换器支持不同种类的消息路由；
- **消息有序（不支持）**：当消费消息时，如果消费失败，消息会被放回队列，然后重新消费，这样会导致消息无序；
- **消息时序（非常好）**：通过延时队列，可以指定消息的延时时间，过期时间TTL等；
- **容错处理（非常好）**：通过交付重试和死信交换器（DLX）来处理消息处理故障；
- **伸缩（一般）**：伸缩其实没有非常智能，因为即使伸缩了，master queue还是只有一个，负载还是只有这一个master queue去抗，所以我理解RabbitMQ的伸缩很弱（个人理解）。
- **持久化（不太好）**：没有消费的消息，可以支持持久化，这个是为了保证机器宕机时消息可以恢复，但是消费过的消息，就会被马上删除，因为RabbitMQ设计时，就不是为了去存储历史数据的。
- **消息回溯（不支持）**：因为消息不支持永久保存，所以自然就不支持回溯。
- **高吞吐（中等）**：因为所有的请求的执行，最后都是在master queue，它的这个设计，导致单机性能达不到十万级的标准。

# 3. RabbitMQ环境搭建

因为我用的是Mac，所以直接可以参考官网：

> https://www.rabbitmq.com/install-homebrew.html

需要注意的是，一定需要先执行：

```powershell
brew update
```

然后再执行：

```powershell
brew install rabbitmq
```

> 之前没有执行brew update，直接执行brew install rabbitmq时，会报各种各样奇怪的错误，其中“403 Forbidde”居多。

但是在执行“brew install rabbitmq”，会自动安装其它的程序，如果你使用源码安装Rabbitmq，因为启动该服务依赖erlang环境，所以你还需手动安装erlang，但是目前官方已经一键给你搞定，会自动安装Rabbitmq依赖的所有程序，是不是很棒！

<img src="./images/sXFqMxQoVLEBqF1pxoH0xnM8THYI4LSLRgDl6W2qQxnoyhLcYDVJ8LOgrnUnEqzPE8Bwbonic9TtPOiaOJRIJQqw.png" alt="图片" style="zoom:50%;" />

最后执行成功的输出如下：

<img src="./images/sXFqMxQoVLEBqF1pxoH0xnM8THYI4LSLLM7LXibUxu6twuxWr12GPCA7J06xHsx1t26bYIMQX4ZFN2R811ZPALA.png" alt="图片" style="zoom:50%;" />

启动服务：

```powershell
# 启动方式1：后台启动
brew services start rabbitmq
# 启动方式2：当前窗口启动
cd /usr/local/Cellar/rabbitmq/3.8.19
rabbitmq-server
```

在浏览器输入：

```powershell
http://localhost:15672/
```

会出现RabbitMQ后台管理界面（用户名和密码都为guest）：

<img src="./images/sXFqMxQoVLEBqF1pxoH0xnM8THYI4LSLFCHbNANdOCE4iapJRCibLd4pJiaYMx9U5vU7M03XGWhsq2RJCn5oRHcsQ.png" alt="图片" style="zoom:50%;" />

通过brew安装，一行命令搞定，真香！

# 4. RabbitMQ测试

## 4.1 添加账号

首先得启动mq

```powershell
## 添加账号
./rabbitmqctl add_user admin admin
## 添加访问权限
./rabbitmqctl set_permissions -p "/" admin ".*" ".*" ".*"
## 设置超级权限
./rabbitmqctl set_user_tags admin administrator
```

## 4.2 编码实测

因为代码中引入了java 8的特性，pom引入依赖：

```java
<dependency>
    <groupId>com.rabbitmq</groupId>
    <artifactId>amqp-client</artifactId>
    <version>5.5.1</version>
</dependency>

<plugins>
    <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <configuration>
            <source>8</source>
            <target>8</target>
        </configuration>
    </plugin>
</plugins>
```

开始写代码：

```java
public class RabbitMqTest {
    //消息队列名称
    private final static String QUEUE_NAME = "hello";

    @Test
    public void send() throws java.io.IOException, TimeoutException {
        //创建连接工程
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("127.0.0.1");
        factory.setPort(5672);
        factory.setUsername("admin");
        factory.setPassword("admin");
        //创建连接
        Connection connection = factory.newConnection();
        //创建消息通道
        Channel channel = connection.createChannel();
        //生成一个消息队列
        channel.queueDeclare(QUEUE_NAME, true, false, false, null);

        for (int i = 0; i < 10; i++) {
            String message = "Hello World RabbitMQ count: " + i;
            //发布消息，第一个参数表示路由（Exchange名称），为""则表示使用默认消息路由
            channel.basicPublish("", QUEUE_NAME, null, message.getBytes());
            System.out.println(" [x] Sent '" + message + "'");
        }
        //关闭消息通道和连接
        channel.close();
        connection.close();
    }

    @Test
    public void consumer() throws java.io.IOException, TimeoutException {
        //创建连接工厂
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("127.0.0.1");
        factory.setPort(5672);
        factory.setUsername("admin");
        factory.setPassword("admin");
        //创建连接
        Connection connection = factory.newConnection();
        //创建消息信道
        final Channel channel = connection.createChannel();
        //消息队列
        channel.queueDeclare(QUEUE_NAME, true, false, false, null);
        System.out.println("[*] Waiting for message. To exist press CTRL+C");

        DeliverCallback deliverCallback = (consumerTag, delivery) -> {
            String message = new String(delivery.getBody(), "UTF-8");
            System.out.println(" [x] Received '" + message + "'");
        };
        channel.basicConsume(QUEUE_NAME, true, deliverCallback, consumerTag -> {});
    }
}
```

执行send()后控制台输出：

```powershell
[x] Sent 'Hello World RabbitMQ count: 0'
[x] Sent 'Hello World RabbitMQ count: 1'
[x] Sent 'Hello World RabbitMQ count: 2'
[x] Sent 'Hello World RabbitMQ count: 3'
[x] Sent 'Hello World RabbitMQ count: 4'
[x] Sent 'Hello World RabbitMQ count: 5'
[x] Sent 'Hello World RabbitMQ count: 6'
[x] Sent 'Hello World RabbitMQ count: 7'
[x] Sent 'Hello World RabbitMQ count: 8'
[x] Sent 'Hello World RabbitMQ count: 9'
```

<img src="./images/sXFqMxQoVLEBqF1pxoH0xnM8THYI4LSLdeAagu2Z8CWrmO4TsX37JIEm85zNxv6UJapq9HgjVuwzxOP9lapqZQ.png" alt="图片" style="zoom:50%;" />

执行consumer()后：

<img src="./images/sXFqMxQoVLEBqF1pxoH0xnM8THYI4LSLIdBibNY90PafBxEvbYE5ktoVjvVROIA8HPFqaYSLJOiaadGPV5UYKRxQ.png" alt="图片" style="zoom:50%;" />

> 示例中的代码讲解，可以直接参考官网：https://www.rabbitmq.com/tutorials/tutorial-one-java.html

# 5. 基本使用姿势

## 5.1 公共代码封装

封装工厂类：

```java
public class RabbitUtil {
    public static ConnectionFactory getConnectionFactory() {
        //创建连接工程，下面给出的是默认的case
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("127.0.0.1");
        factory.setPort(5672);
        factory.setUsername("admin");
        factory.setPassword("admin");
        factory.setVirtualHost("/");
        return factory;
    }
}
```

封装生成者：

```java
public class MsgProducer {
    public static void publishMsg(String exchange, BuiltinExchangeType exchangeType, String toutingKey, String message) throws IOException, TimeoutException {
        ConnectionFactory factory = RabbitUtil.getConnectionFactory();
        //创建连接
        Connection connection = factory.newConnection();
        //创建消息通道
        Channel channel = connection.createChannel();
        // 声明exchange中的消息为可持久化，不自动删除
        channel.exchangeDeclare(exchange, exchangeType, true, false, null);
        // 发布消息
        channel.basicPublish(exchange, toutingKey, null, message.getBytes());
        System.out.println("Sent '" + message + "'");
        channel.close();
        connection.close();
    }
}
```

封装消费者：

```java
public class MsgConsumer {
    public static void consumerMsg(String exchange, String queue, String routingKey)
            throws IOException, TimeoutException {
        ConnectionFactory factory = RabbitUtil.getConnectionFactory();
        //创建连接
        Connection connection = factory.newConnection();
        //创建消息信道
        final Channel channel = connection.createChannel();
        //消息队列
        channel.queueDeclare(queue, true, false, false, null);
        //绑定队列到交换机
        channel.queueBind(queue, exchange, routingKey);
        System.out.println("[*] Waiting for message. To exist press CTRL+C");

        Consumer consumer = new DefaultConsumer(channel) {
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties,
                                       byte[] body) throws IOException {
                String message = new String(body, "UTF-8");
                try {
                    System.out.println(" [x] Received '" + message);
                } finally {
                    System.out.println(" [x] Done");
                    channel.basicAck(envelope.getDeliveryTag(), false);
                }
            }
        };
        // 取消自动ack
        channel.basicConsume(queue, false, consumer);
    }
}
```

## 5.2 Direct方式

<img src="./images/sXFqMxQoVLEBqF1pxoH0xnM8THYI4LSLAqDxFoyHu8amIHOzh95K7Jp0Xic4NBG4WCBFFYFKe8iaFtz6ib34cqkicw.png" alt="图片" style="zoom:67%;" />

#### 5.2.1 Direct示例

生产者：

```java
public class DirectProducer {
    private static final String EXCHANGE_NAME = "direct.exchange";
    public void publishMsg(String routingKey, String msg) {
        try {
            MsgProducer.publishMsg(EXCHANGE_NAME, BuiltinExchangeType.DIRECT, routingKey, msg);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
    public static void main(String[] args) throws InterruptedException {
        DirectProducer directProducer = new DirectProducer();
        String[] routingKey = new String[]{"aaa", "bbb", "ccc"};
        String msg = "hello >>> ";
        for (int i = 0; i < 10; i++) {
            directProducer.publishMsg(routingKey[i % 3], msg + i);
        }
        System.out.println("----over-------");
        Thread.sleep(1000 * 60 * 100);
    }
}
```

执行生产者，往消息队列中放入10条消息，其中key分别为“aaa”、“bbb”和“ccc”，分别放入qa、qb、qc三个队列：

<img src="./images/sXFqMxQoVLEBqF1pxoH0xnM8THYI4LSLbKDAGibsKNRYl7XQ26c6bSiaQjYmjZvUL0YxrSKMKjy4POlXvk77If7Q.png" alt="图片" style="zoom:50%;" />

下面是qa队列的信息：

<img src="./images/sXFqMxQoVLEBqF1pxoH0xnM8THYI4LSLFWUUGLeXsCdQia0A3cQpgR57syacBLy3dKEwFZPfC5V5NgTh22bWRUQ.png" alt="图片" style="zoom:50%;" />

消费者：

```java
public class DirectConsumer {
    private static final String exchangeName = "direct.exchange";
    public void msgConsumer(String queueName, String routingKey) {
        try {
            MsgConsumer.consumerMsg(exchangeName, queueName, routingKey);
        } catch (IOException e) {
            e.printStackTrace();
        } catch (TimeoutException e) {
            e.printStackTrace();
        }
    }
    public static void main(String[] args) throws InterruptedException {
        DirectConsumer consumer = new DirectConsumer();
        String[] routingKey = new String[]{"aaa", "bbb", "ccc"};
        String[] queueNames = new String[]{"qa", "qb", "qc"};

        for (int i = 0; i < 3; i++) {
            consumer.msgConsumer(queueNames[i], routingKey[i]);
        }
        Thread.sleep(1000 * 60 * 100);
    }
}
```

执行后的输出：

```powershell
[*] Waiting for message. To exist press CTRL+C
 [x] Received 'hello >>> 0
 [x] Done
 [x] Received 'hello >>> 3
 [x] Done
 [x] Received 'hello >>> 6
 [x] Done
 [x] Received 'hello >>> 9
 [x] Done
[*] Waiting for message. To exist press CTRL+C
 [x] Received 'hello >>> 1
 [x] Done
 [x] Received 'hello >>> 4
 [x] Done
 [x] Received 'hello >>> 7
 [x] Done
[*] Waiting for message. To exist press CTRL+C
 [x] Received 'hello >>> 2
 [x] Done
 [x] Received 'hello >>> 5
 [x] Done
 [x] Received 'hello >>> 8
 [x] Done
```

可以看到，分别从qa、qb、qc中将不同的key的数据消费掉。

#### 5.2.2 问题探讨

> 有个疑问：这个队列的名称qa、qb和qc是RabbitMQ自动生成的么，我们可以指定队列名称么？

我做了个简单的实验，我把消费者代码修改了一下：

```java
public static void main(String[] args) throws InterruptedException {
    DirectConsumer consumer = new DirectConsumer();
    String[] routingKey = new String[]{"aaa", "bbb", "ccc"};
    String[] queueNames = new String[]{"qa", "qb", "qc1"}; // 将qc修改为qc1

    for (int i = 0; i < 3; i++) {
        consumer.msgConsumer(queueNames[i], routingKey[i]);
    }
    Thread.sleep(1000 * 60 * 100);
}
```

执行后如下图所示：

<img src="./images/sXFqMxQoVLEBqF1pxoH0xnM8THYI4LSLXibY42YtqywT0VLbmBaEO4aNG7pw5ibjkbon5vgHzfyLBmKGBh8feicoA.png" alt="图片" style="zoom:50%;" />

我们可以发现，多了一个qc1，所以可以判断这个界面中的queues，是消费者执行时，会将消费者指定的队列名称和direct.exchange绑定，绑定的依据就是key。

当我们把队列中的数据全部消费掉，然后重新执行生成者后，会发现qc和qc1中都有3条待消费的数据，因为绑定的key都是“ccc”，所以两者的数据是一样的：

<img src="./images/sXFqMxQoVLEBqF1pxoH0xnM8THYI4LSL3iaDzQWIb01UQcEOv94Dn4VBHjyAPwfDhA9QGBzzZxaIFZ5uVWePN4g.png" alt="图片" style="zoom:50%;" />

绑定关系如下：

<img src="./images/sXFqMxQoVLEBqF1pxoH0xnM8THYI4LSLpD65jicq08bldM1xceCCicicbUrdIOLzTO8soq7hdXsKkDgOljTFlibCqA.png" alt="图片" style="zoom:50%;" />

> 注意：当没有Queue绑定到Exchange时，往Exchange中写入的消息也不会重新分发到之后绑定的queue上。

> 思考：不执行消费者，看不到这个Queres中信息，我其实可以把这个界面理解为消费者信息界面。不过感觉还是怪怪的，这个queues如果是消费者信息，就不应该叫queues，我理解queues应该是RabbitMQ中实际存放数据的queues，难道是我理解错了？

## 5.3 Fanout方式（指定队列）

<img src="./images/sXFqMxQoVLEBqF1pxoH0xnM8THYI4LSLbCDAcU5KfbfRnoc45QZgTbInKXyLTRZ7ibsupcGRROPW11mtPU9JOIA.png" alt="图片" style="zoom:50%;" />

生产者封装：

```java
public class FanoutProducer {
    private static final String EXCHANGE_NAME = "fanout.exchange";
    public void publishMsg(String routingKey, String msg) {
        try {
            MsgProducer.publishMsg(EXCHANGE_NAME, BuiltinExchangeType.FANOUT, routingKey, msg);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
    public static void main(String[] args) {
        FanoutProducer directProducer = new FanoutProducer();
        String msg = "hello >>> ";
        for (int i = 0; i < 10; i++) {
            directProducer.publishMsg("", msg + i);
        }
    }
}
```

消费者：

```java
public class FanoutConsumer {
    private static final String EXCHANGE_NAME = "fanout.exchange";
    public void msgConsumer(String queueName, String routingKey) {
        try {
            MsgConsumer.consumerMsg(EXCHANGE_NAME, queueName, routingKey);
        } catch (IOException e) {
            e.printStackTrace();
        } catch (TimeoutException e) {
            e.printStackTrace();
        }
    }
    public static void main(String[] args) {
        FanoutConsumer consumer = new FanoutConsumer();
        String[] queueNames = new String[]{"qa-2", "qb-2", "qc-2"};
        for (int i = 0; i < 3; i++) {
            consumer.msgConsumer(queueNames[i], "");
        }
    }
}
```

执行生成者，结果如下：

<img src="./images/sXFqMxQoVLEBqF1pxoH0xnM8THYI4LSLt6PxYOBS9SyqyMasAOe9hkzwCiaxgxCyEl1y8J8dLBqz55plOQU2AGQ.png" alt="图片" style="zoom:50%;" />

我们发现，生产者生产的10条数据，在每个消费者中都可以消费，这个是和Direct不同的地方，但是使用Fanout方式时，有几个点需要注意一下：

- 生产者的routkey可以为空，因为生产者的所有数据，会下放到每一个队列，所以不会通过routkey去路由；
- 消费者需要指定queues，因为消费者需要绑定到指定的queues才能消费。

<img src="./images/sXFqMxQoVLEBqF1pxoH0xnM8THYI4LSLRPTSibibdDicMtniafLia9QyicVjWgt2ADibv0yzYCibmpADiacEg2tWEaCbpIg.png" alt="图片" style="zoom:67%;" />

这幅图就画出了Fanout的精髓之处，exchange会和所有的queue进行绑定，不区分路由，消费者需要绑定指定的queue才能发起消费。

> 注意：往队列塞数据时，可能通过界面看不到消息个数的增加，可能是你之前已经开启了消费进程，导致增加的消息马上被消费了。

## 5.4 Fanout方式（随机获取队列）

上面我们是指定了队列，这个方式其实很不友好，比如对于Fanout，我其实根本无需关心队列的名字，如果还指定对应队列进行消费，感觉这个很冗余，所以我们这里就采用随机获取队列名字的方式，下面代码直接Copy官网。

生成者封装：

```java
public static void publishMsgV2(String exchange, BuiltinExchangeType exchangeType, String message) throws IOException, TimeoutException {
    ConnectionFactory factory = RabbitUtil.getConnectionFactory();
    //创建连接
    Connection connection = factory.newConnection();
    //创建消息通道
    Channel channel = connection.createChannel();

    // 声明exchange中的消息
    channel.exchangeDeclare(exchange, exchangeType);

    // 发布消息
    channel.basicPublish(exchange, "", null, message.getBytes("UTF-8"));

    System.out.println("Sent '" + message + "'");
    channel.close();
    connection.close();
}
```

消费者封装：

```java
public static void consumerMsgV2(String exchange) throws IOException, TimeoutException {
    ConnectionFactory factory = RabbitUtil.getConnectionFactory();
    Connection connection = factory.newConnection();
    final Channel channel = connection.createChannel();

    channel.exchangeDeclare(exchange, "fanout");
    String queueName = channel.queueDeclare().getQueue();
    channel.queueBind(queueName, exchange, "");

    System.out.println(" [*] Waiting for messages. To exit press CTRL+C");

    DeliverCallback deliverCallback = (consumerTag, delivery) -> {
        String message = new String(delivery.getBody(), "UTF-8");
        System.out.println(" [x] Received '" + message + "'");
    };
    channel.basicConsume(queueName, true, deliverCallback, consumerTag -> { });
}
```

生产者：

```java
public class FanoutProducer {
    private static final String EXCHANGE_NAME = "fanout.exchange.v2";
    public void publishMsg(String msg) {
        try {
            MsgProducer.publishMsgV2(EXCHANGE_NAME, BuiltinExchangeType.FANOUT, msg);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
    public static void main(String[] args) {
        FanoutProducer directProducer = new FanoutProducer();
        String msg = "hello >>> ";
        for (int i = 0; i < 10000; i++) {
            directProducer.publishMsg(msg + i);
        }
    }
}
```

消费者：

```java
public class FanoutConsumer {
    private static final String EXCHANGE_NAME = "fanout.exchange.v2";
    public void msgConsumer() {
        try {
            MsgConsumer.consumerMsgV2(EXCHANGE_NAME);
        } catch (IOException e) {
            e.printStackTrace();
        } catch (TimeoutException e) {
            e.printStackTrace();
        }
    }
    public static void main(String[] args) {
        FanoutConsumer consumer = new FanoutConsumer();
        for (int i = 0; i < 3; i++) {
            consumer.msgConsumer();
        }
    }
}
```

执行后，管理界面如下：

<img src="./images/sXFqMxQoVLEBqF1pxoH0xnM8THYI4LSLa941gRzBicAMPRrFcTIeczDLCgavUHfMzqcoeZFUwvgiaxQicsRK5FdDg.png" alt="图片" style="zoom:50%;" />

<img src="./images/sXFqMxQoVLEBqF1pxoH0xnM8THYI4LSLtqIUtXbX2GyGO6EXOtthicTjjzvTcgt6LJSs8YQxrX0TCloHecVtwIQ.png" alt="图片" style="zoom:50%;" />

<img src="./images/sXFqMxQoVLEBqF1pxoH0xnM8THYI4LSLeNkcn2pwzWRY43AjJlIB22Mug5AIsIaMia4dXZRTNhA9Up9VrNWyL9A.png" alt="图片" style="zoom:50%;" />

## 5.5 Topic方式

<img src="./images/sXFqMxQoVLEBqF1pxoH0xnM8THYI4LSLibsbLDCrtxSR6XDtbq4XkRaWxLqtP9F5Mm9QNd317M2LKhsbOJLpusQ.png" alt="图片" style="zoom:67%;" />

代码详见官网：https://www.rabbitmq.com/tutorials/tutorial-five-java.html

> 更多方式，请直接查看官网：https://www.rabbitmq.com/getstarted.html

<img src="./images/sXFqMxQoVLEBqF1pxoH0xnM8THYI4LSLLrjvTXSer8gmia4rI3eRfHH1PeeA20ZCBIDdOexiboUJjTav8Ismxicpw.png" alt="图片" style="zoom:50%;" />

# 6. RabbitMQ 进阶

## 6.1 durable 和 autoDeleted

在定义Queue时，可以指定这两个参数：

```java
/**
 * Declare an exchange.
 * @see com.rabbitmq.client.AMQP.Exchange.Declare
 * @see com.rabbitmq.client.AMQP.Exchange.DeclareOk
 * @param exchange the name of the exchange
 * @param type the exchange type
 * @param durable true if we are declaring a durable exchange (the exchange will survive a server restart)
 * @param autoDelete true if the server should delete the exchange when it is no longer in use
 * @param arguments other properties (construction arguments) for the exchange
 * @return a declaration-confirm method to indicate the exchange was successfully declared
 * @throws java.io.IOException if an error is encountered
 */
Exchange.DeclareOk exchangeDeclare(String exchange, BuiltinExchangeType type, boolean durable, boolean autoDelete,
    Map<String, Object> arguments) throws IOException;
    
/**
* Declare a queue
* @see com.rabbitmq.client.AMQP.Queue.Declare
* @see com.rabbitmq.client.AMQP.Queue.DeclareOk
* @param queue the name of the queue
* @param durable true if we are declaring a durable queue (the queue will survive a server restart)
* @param exclusive true if we are declaring an exclusive queue (restricted to this connection)
* @param autoDelete true if we are declaring an autodelete queue (server will delete it when no longer in use)
* @param arguments other properties (construction arguments) for the queue
* @return a declaration-confirm method to indicate the queue was successfully declared
* @throws java.io.IOException if an error is encountered
*/
Queue.DeclareOk queueDeclare(String queue, boolean durable, boolean exclusive, boolean autoDelete,
    Map<String, Object> arguments) throws IOException;
```

#### 6.1.1 durable

持久化，保证RabbitMQ在退出或者crash等异常情况下数据没有丢失，需要将queue，exchange和Message都持久化。

若是将queue的持久化标识durable设置为true，则代表是一个持久的队列，那么在服务重启之后，会重新读取之前被持久化的queue。

虽然队列可以被持久化，但是里面的消息是否为持久化，还要看消息的持久化设置。即重启queue，但是queue里面还没有发出去的消息，那队列里面还存在该消息么？这个取决于该消息的设置。

#### 6.1.2 autoDeleted

自动删除，如果该队列没有任何订阅的消费者的话，该队列会被自动删除。这种队列适用于临时队列。

当一个Queue被设置为自动删除时，当消费者断掉之后，queue会被删除，这个主要针对的是一些不是特别重要的数据，不希望出现消息积累的情况。

#### 6.1.3 小节

- 当一个Queue已经声明好了之后，不能更新durable或者autoDelted值；当需要修改时，需要先删除再重新声明
- 消费的Queue声明应该和投递的Queue声明的 durable,autoDelted属性一致，否则会报错
- 对于重要的数据，一般设置 durable=true, autoDeleted=false
- 对于设置 autoDeleted=true 的队列，当没有消费者之后，队列会自动被删除

## 6.4 ACK

执行一个任务可能需要花费几秒钟，你可能会担心如果一个消费者在执行任务过程中挂掉了。一旦RabbitMQ将消息分发给了消费者，就会从内存中删除。在这种情况下，如果正在执行任务的消费者宕机，会丢失正在处理的消息和分发给这个消费者但尚未处理的消息。

但是，我们不想丢失任何任务，如果有一个消费者挂掉了，那么我们应该将分发给它的任务交付给另一个消费者去处理。

为了确保消息不会丢失，RabbitMQ支持消息应答。消费者发送一个消息应答，告诉RabbitMQ这个消息已经接收并且处理完毕了。RabbitMQ就可以删除它了。

因此手动ACK的常见手段：

```java
// 接收消息之后，主动ack/nak
Consumer consumer = new DefaultConsumer(channel) {
    @Override
    public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties,
            byte[] body) throws IOException {
        String message = new String(body, "UTF-8");
        try {
            System.out.println(" [ " + queue + " ] Received '" + message);
            channel.basicAck(envelope.getDeliveryTag(), false);
        } catch (Exception e) {
            channel.basicNack(envelope.getDeliveryTag(), false, true);
        }
    }
};
// 取消自动ack
channel.basicConsume(queue, false, consumer);
```

# 7. 结语

<img src="./images/sXFqMxQoVLEBqF1pxoH0xnM8THYI4LSL1xtDlZsdwicpgCYAsIJ4p8pBjvmWfdQ3jW4wATqm565icIwoVR2rUvCA.png" alt="图片" style="zoom:50%;" />

前段时间有粉丝问我问题，是否可以只学习理论知识，如何将理论知识和应用结合起来？

**我的回答：不要做 PPT 专家，理论需要和实际相结合。**

那如何结合呢，比如本文的 RabbitMQ，**我之前其实没有用过，但是你可以自己把环境搭起来，然后到机器上跑跑**，虽然和实际应用还有些距离，但至少你实操过，不会浮于表面，也印象深刻，等后续项目需要使用时，就更容易上手。

可能有同学杠上了，楼哥，你写的高并发系列文章，都是纯理论，没有实操，**那是因为高并发系列的东西，楼哥都搞了好几年了，现在只是简单的输出。**

其实还有一个非常重要的原因，那就是**现在的读者，都喜欢看理论，不喜欢大段代码的内容，都认为看完即掌握，或者懒得动**，所以楼哥就投其所好，后面的文章就摘掉实操的内容。

对于勤动手实操的同学，可以翻看楼哥之前的文章，基本每个系列，里面都有大量的实操示例哈。

理论要掌握，实操不能落！





