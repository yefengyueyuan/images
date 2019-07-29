### 前言
>在新框架中需要集成消息中间件，通过各项数据对比决定用RabbitMq来做为我们的消息中间件怎么将它高度集成来，从而使得开发人员不需要要关心太多的技术细节直接调用接口就可以用下面为大家一一介绍。
### 消息中间件可以解决什么应用场景
- 1、异步处理(缓存处理)
- 2、应用解耦
- 3、处理高并发(一般在秒杀或团抢活动中使用广泛)
 应用场景：秒杀活动，一般会因为流量过大，导致流量暴增，应用挂掉。为解决这个问题，一般需要在应用前端加入消息队列
-   1. 可以控制活动的人数；
-   2. 可以缓解短时间内高流量压垮应用；
- 4、分布式事物一致性

### 中间件产品选型
> rabbitMQ、activeMQ、zeroMQ、Kafka、Redis

> RabbitMQ是使用Erlang编写的一个开源的消息队列，本身支持很多的协议：AMQP，XMPP, SMTP, STOMP，也正因如此，它非常重量级，更适合于企业级的开发。同时实现了Broker构架，这意味着消息在发送给客户端时先在中心队列排队。对路由，负载均衡或者数据持久化都有很好的支持。

### RabbitMQ系统架构
![image](http://upload-images.jianshu.io/upload_images/5018582-8176cbf2db1246a2.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 1.几个概念说明:
- Virtual Host:其实是一个虚拟概念，类似于权限控制组，一个Virtual Host里面可以有若干个Exchange和Queue，但是权限控制的最小粒度是Virtual Host
-  Broker:它提供一种传输服务,它的角色就是维护一条从生产者到消费者的路线，保证数据能按照指定的方式进行传输,
-  Exchange：消息交换机,它指定消息按什么规则,路由到哪个队列。
- Queue:消息的载体,每个消息都会被投到一个或多个队列。
- Message: 由Header和Body组成，Header是由生产者添加的各种属性的集合，包括Message是否被持久化、由哪个Message Queue接受、优先级是多少等。而Body是真正需要传输的APP数据。


-  Binding:绑定，它的作用就是把exchange和queue按照路由规则绑定起来.
-  Routing Key:路由关键字,exchange根据这个关键字进行消息投递。
-  vhost:虚拟主机,一个broker里可以有多个vhost，用作不同用户的权限分离。
-  Producer:消息生产者,就是投递消息的程序.
-  Consumer:消息消费者,就是接受消息的程序.
-  Channel:信道，仅仅创建了客户端到Broker之间的连接后，客户端还是不能发送消息的。需要为每一个Connection创建Channel，AMQP协议规定只有通过Channel才能执行AMQP的命令。一个Connection可以包含多个Channel。之所以需要Channel，是因为TCP连接的建立和释放都是十分昂贵的，如果一个客户端每一个线程都需要与Broker交互，如果每一个线程都建立一个TCP连接，暂且不考虑TCP连接是否浪费，就算操作系统也无法承受每秒建立如此多的TCP连接。RabbitMQ建议客户端线程之间不要共用Channel，至少要保证共用Channel的线程发送消息必须是串行的，但是建议尽量共用Connection。
- Command:AMQP的命令，客户端通过Command完成与AMQP服务器的交互来实现自身的逻辑。例如在RabbitMQ中，客户端可以通过publish命令发送消息，txSelect开启一个事务，txCommit提交一个事务。
- routing key:生产者在将消息发送给Exchange的时候，一般会指定一个routing key，来指定这个消息的路由规则，而这个routing key需要与Exchange Type及binding key联合使用才能最终生效。在Exchange Type与binding key固定的情况下（在正常使用时一般这些内容都是固定配置好的），我们的生产者就可以在发送消息给Exchange时，通过指定routing key来决定消息流向哪里。RabbitMQ为routing key设定的长度限制为255 bytes
- binding key:在绑定（Binding）Exchange与Queue的同时一般会指定一个bindingkey；消费者将消息发送给Exchange时，一般会指定一个routingkey；当bindingkey与routingkey相匹配时，消息将会被路由到对应的Queue中。这个将在Exchange在绑定多个Queue到同一个Exchange的时候，这些Binding允许使用相同的binding key。binding key并不是在所有情况下都生效，它依赖于ExchangeType，比如fanout类型的Exchange就会无视bindingkey，而是将消息路由到所有绑定到该Exchange的Queue。

- headers:类型的Exchange不依赖于routing key与binding key的匹配规则来路由消息，而是根据发送的消息内容中的headers属性进行匹配。
在绑定Queue与Exchange时指定一组键值对；当消息发送到Exchange时，RabbitMQ会取到该消息的headers（也是一个键值对的形式），对比其中的键值对是否完全匹配Queue与Exchange绑定时指定的键值对；如果完全匹配则消息会路由到该Queue，否则不会路由到该Queue。





#### 2.任务分发机制
##### 2.1Round-robin dispathching循环分发
> RabbbitMQ的分发机制非常适合扩展,而且它是专门为并发程序设计的,如果现在load加重,那么只需要创建更多的Consumer来进行任务处理。
默认情况下，RabbitMQ 会顺序的分发每个Message。当每个queue收到ack后，会将该Message删除，然后将下一个Message分发到下一个Consumer。这种分发方式叫做round-robin
>

#####  2.2Message acknowledgment消息确认
> 为了保证数据不被丢失,RabbitMQ支持消息确认机制,为了保证数据能被正确处理而不仅仅是被Consumer收到,那么我们不能采用no-ack，而应该是在处理完数据之后发送ack.
> 在处理完数据之后发送ack,就是告诉RabbitMQ数据已经被接收,处理完成,RabbitMQ可以安全的删除它了.
> 如果Consumer退出了但是没有发送ack,那么RabbitMQ就会把这个Message发送到下一个Consumer，这样就保证在Consumer异常退出情况下数据也不会丢失.
> RabbitMQ它没有用到超时机制.RabbitMQ仅仅通过Consumer的连接中断来确认该Message并没有正确处理，也就是说RabbitMQ给了Consumer足够长的时间做数据处理。
> 如果忘记ack,那么当Consumer退出时,Mesage会重新分发,然后RabbitMQ会占用越来越多的内存.

#### 3.Message durability消息持久化
> 要持久化队列queue的持久化需要在声明时指定durable=True;
> 这里要注意,队列的名字一定要是Broker中不存在的,不然不能改变此队列的任何属性.
> 队列和交换机有一个创建时候指定的标志durable,durable的唯一含义就是具有这个标志的队列和交换机会在重启之后重新建立,它不表示说在队列中的消息会在重启后恢复
> 消息持久化包括3部分
1. exchange持久化,在声明时指定durable => true
1. queue持久化,在声明时指定durable => true
1. 消息持久化,在投递时指定delivery_mode => 2(1是非持久化).
> 如果exchange和queue都是持久化的,那么它们之间的binding也是持久化的,如果exchange和queue两者之间有一个持久化，一个非持久化,则不允许建立绑定.
> 注意：一旦创建了队列和交换机,就不能修改其标志了,例如,创建了一个non-durable的队列,然后想把它改变成durable的,唯一的办法就是删除这个队列然后重现创建。

#### 4.Fair dispath 公平分发
> 你可能也注意到了,分发机制不是那么优雅,默认状态下,RabbitMQ将第n个Message分发给第n个Consumer。n是取余后的,它不管Consumer是否还有unacked Message，只是按照这个默认的机制进行分发.
> 那么如果有个Consumer工作比较重,那么就会导致有的Consumer基本没事可做,有的Consumer却毫无休息的机会,那么,Rabbit是如何处理这种问题呢?
![image](http://upload-images.jianshu.io/upload_images/5018582-8895ed118f5b2e07.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> 通过basic.qos方法设置prefetch_count=1，这样RabbitMQ就会使得每个Consumer在同一个时间点最多处理一个Message，换句话说,在接收到该Consumer的ack前,它不会将新的Message分发给它
>
注意，这种方法可能会导致queue满。当然，这种情况下你可能需要添加更多的Consumer，或者创建更多的virtualHost来细化你的设计。

#### 5.分发到多个Consumer
##### 5.1Exchange
交换机路由的几种类型:
- Direct Exchange:直接匹配,通过Exchange名称+RountingKey来发送与接收消息,是RabbitMQ默认的交换机模式，也是最简单的模式，根据key全文匹配去寻找队列.
- Fanout Exchange:广播订阅,向所有的消费者发布消息,但是只有消费者将队列绑定到该路由器才能收到消息,忽略Routing Key.
- Topic Exchange：主题匹配订阅,这里的主题指的是RoutingKey,RoutingKey可以采用通配符,如:*或#，RoutingKey命名采用.来分隔多个词,只有消息这将队列绑定到该路由器且指定RoutingKey符合匹配规则时才能收到消息;
- Headers Exchange:消息头订阅,消息发布前,为消息定义一个或多个键值对的消息头,然后消费者接收消息同时需要定义类似的键值对请求头:(如:x-mactch=all或者x_match=any)，只有请求头与消息头匹配,才能接收消息,忽略RoutingKey.
- 默认的exchange:如果用空字符串去声明一个exchange，那么系统就会使用”amq.direct”这个exchange，我们创建一个queue时,默认的都会有一个和新建queue同名的routingKey绑定到这个默认的exchange上去

> 如果有两个接收程序都是用了同一个的queue和相同的routingKey去绑定direct exchange的话，分发的行为是负载均衡的，也就是说第一个是程序1收到，第二个是程序2收到，以此类推。
> 如果有两个接收程序用了各自的queue，但使用相同的routingKey去绑定direct exchange的话，分发的行为是复制的，也就是说每个程序都会收到这个消息的副本。行为相当于fanout类型的exchange

##### 5.2 Bindings 绑定
> 绑定其实就是关联了exchange和queue，或者这么说:queue对exchange的内容感兴趣,exchange要把它的Message deliver到queue。

##### 5.3Direct exchange
> Driect exchange的路由算法非常简单:通过bindingkey的完全匹配，可以用下图来说明.
DirectExchange是RabbitMQ Broker的默认Exchange，它有一个特别的属性对一些简单的应用来说是非常有用的，在使用这个类型的Exchange时，可以不必指定routing key的名字，在此类型下创建的Queue有一个默认的routing key，这个routing key一般同Queue同名
![image](http://upload-images.jianshu.io/upload_images/5018582-d44e78469b5f83eb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### 5.4Fanout Exchange
> 任何发送到Fanout Exchange的消息都会被转发到与该Exchange绑定(Binding)的所有Queue上。
>
> 1. 可以理解为路由表的模式
> 1. 这种模式不需要routingKey
> 1. 这种模式需要提前将Exchange与Queue进行绑定，一个Exchange可以绑定多个Queue，一个Queue可以同多个Exchange进行绑定。
> 1. 4.如果接受到消息的Exchange没有与任何Queue绑定，则消息会被抛弃。
![image](http://upload-images.jianshu.io/upload_images/5018582-51650570bbe4dd69.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### 5.5Topic Exchange
> 任何发送到Topic Exchange的消息都会被转发到所有关心routingKeyy中指定话题的Queue上
1. 这种模式较为复杂，简单来说，就是每个队列都有其关心的主题，所有的消息都带有一个“标题”(routingKey)，Exchange会将消息转发到所有关注主题能与routingKey模糊匹配的队列。
1. 这种模式需要routingKey，也许要提前绑定Exchange与Queue。
1. 在进行绑定时，要提供一个该队列关心的主题，如“#.log.#”表示该队列关心所有涉及log的消息(一个RouteKey为”MQ.log.error”的消息会被转发到该队列)。
1. “#”表示0个或若干个关键字，“*”表示一个关键字。如“log.*”能与“log.warn”匹配，无法与“log.warn.timeout”匹配；但是“log.#”能与上述两者匹配。
1. 同样，如果Exchange没有发现能够与RouteKey匹配的Queue，则会抛弃此消息。
[图片上传失败...(image-c534a0-1527837390951)]

#### 6.消息序列化
> RabbitMQ使用ProtoBuf序列化消息,它可作为RabbitMQ的Message的数据格式进行传输,由于是结构化的数据,这样就极大的方便了Consumer的数据高效处理,当然也可以使用XML，与XML相比,ProtoBuf有以下优势:
1. 简单
1. size小了3-10倍
1. 速度快了20-100倍
1. 易于编程
1. 减少了语义的歧义.
ProtoBuf具有速度和空间的优势，使得它现在应用非常广泛

#### RabbitMQ安装

```
docker pull rabbitmq
```
> docker run -d --name rabbit-management \
   -p 5672:5672 \
   -p 15672:15672 \ registry.eyd.com:5000/library/rabbitmq:3.6.6-management

> http://172.20.32.126:15672/#/

> guest/guest

### 7.blf-rabbitmq-core
> rabbitmq-core是消息基础jar，业务工程如有需要发送消息则引入
```
<dependency>
   <groupId>com.belle.blf</groupId>
   <artifactId>blf-rabbitmq-core</artifactId>
</dependency>
```
#### Configuration

```
@Configuration
//@EnableConfigurationProperties(RabbitGeneralProperties.class)
@ConditionalOnBean(annotation = EnableDirect.class)
public class RabbitConfig {

    @Autowired
    ConnectionFactory connectionFactory;
    //@Autowired
    //RabbitGeneralProperties rabbitGeneralProperties;

    @Value("${rabbit.general.direct.queue}")
    private String directQueue;

    /*@Bean
    public Queue queueGeneral(){
        return  new Queue(RabbitConstants.QUEUE_DIRECT_GENERAL);
    }*/


    /**
     * 动态生成队列
     * @return
     * @throws AmqpException
     * @throws IOException
     */
    @Bean
    public String[] mqMsgQueues() throws AmqpException ,IOException{

        if(StringUtils.isEmpty(directQueue)){
            directQueue=RabbitConstants.QUEUE_DIRECT_GENERAL;
        }
        String[] queueNames =directQueue.split(",");
        for(int i=0;i<queueNames.length;i++){
            connectionFactory.createConnection().createChannel(false).queueDeclare(queueNames[i], true, false, false, null);
        }
        return queueNames;
    }


}

/**
 * @description: 广播发送
 * @author: tan.bin
 * @create: 2018-05-30 17:54
 **/
@Configuration
@ConditionalOnBean(annotation = EnableFanout.class)
public class FanoutRabbitConfig {


    @Autowired
    ConnectionFactory connectionFactory;
    /**
     * exchange queue 多对多关系 一个交换下可以绑定多个队列 一个队列可以被多个交换机绑定
     */

    @Value("${rabbit.general.fanout.exchange}")
    private String fanoutExchange;


    @Value("${rabbit.general.fanout.queue}")
    private String fanoutQueue;


    /**
     * 声明交换机
     * @return
     * @throws AmqpException
     * @throws IOException
     */
    @Primary
    @Bean
    public String mqMsgFanoutExchange() throws AmqpException,IOException {

        if(StringUtils.isEmpty(fanoutExchange)){
            fanoutQueue=RabbitConstants.FANOUT_EXCHANGE_GENERAL;
        }
        String exchangeNames =fanoutExchange;
        //创建队列
        connectionFactory.createConnection().createChannel(false).exchangeDeclare(fanoutExchange,BuiltinExchangeType.FANOUT,true);

        return fanoutExchange;
    }

    /**
     * 队列声明、与绑定与交换机关系
     * @return
     * @throws AmqpException
     * @throws IOException
     */
    @Bean
    public String[] mqMsgFanoutQueues() throws AmqpException,IOException {

        if(StringUtils.isEmpty(fanoutQueue)){
            fanoutQueue=RabbitConstants.FANOUT_EXCHANGE_GENERAL;
        }
        String[] queueNames =fanoutQueue.split(",");
        for(int i=0;i<queueNames.length;i++){
            //创建队列
            connectionFactory.createConnection().createChannel(false).queueDeclare(queueNames[i], true, false, false, null);
            //绑定交换机与队列
            connectionFactory.createConnection().createChannel(false).queueBind(queueNames[i],fanoutExchange,"");
        }
        return queueNames;
    }
}

@Configuration
@ConditionalOnBean(annotation = EnableTopic.class)
public class TopicRabbitConfig {


    private final Logger logger = LoggerFactory.getLogger(TopicRabbitConfig.class);

    @Autowired
    ConnectionFactory connectionFactory;
    /**
     * exchange queue 多对多关系 一个交换下可以绑定多个队列 一个队列可以被多个交换机绑定
     */

    @Value("${rabbit.general.topic.exchange}")
    private String topicExchange;


    @Value("${rabbit.general.topic.queue}")
    private String topicQueue;


    //主题关键字
    @Value("${rabbit.general.topic.routingKey}")
    private String routingKey;

    /**
     * 声明交换机
     * @return
     * @throws AmqpException
     * @throws IOException
     */
    @Primary
    @Bean
    public String mqMsgTopicExchange() throws AmqpException,IOException {

        if(StringUtils.isEmpty(topicExchange)){
            topicExchange=RabbitConstants.TOPIC_EXCHANGE_GENERAL;
        }
        String exchangeNames =topicExchange;
        //声明交换机
        connectionFactory.createConnection().createChannel(false).exchangeDeclare(topicExchange,BuiltinExchangeType.TOPIC,true);

        return topicExchange;
    }

    /**
     * 队列声明、与绑定与交换机关系
     * @return
     * @throws AmqpException
     * @throws IOException
     */
    @Bean
    public String[] mqMsgTopicQueues() throws AmqpException,IOException {

        if(StringUtils.isEmpty(topicQueue)){
            topicQueue=RabbitConstants.FANOUT_EXCHANGE_GENERAL;
        }
        String[] queueNames =topicQueue.split(",");
        String[] routingKeyName=routingKey.split(",");//与队列是一一对应
        for(int i=0;i<queueNames.length;i++){
            logger.info("queueNames:{}==routingKey:{}",queueNames[i],routingKeyName[i]);
            //创建队列
            connectionFactory.createConnection().createChannel(false).queueDeclare(queueNames[i], true, false, false, null);
            //绑定交换机与队列
            connectionFactory.createConnection().createChannel(false).queueBind(queueNames[i],topicExchange,routingKeyName[i]);
        }
        return queueNames;
    }
}
```

```
选择发送消息对方式
@SpringBootApplication
@EnableFanout
@EnableDirect
@EnableTopic
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

#### 7.1消息三种方式


```
spring:
  rabbitmq:
    host: 172.20.32.126
    port: 5672
    username: guest
    password: guest



rabbit:
  general:
    direct:
      queue: text_queue #队列名称
    fanout:                               #需要绑定交换机与队列关系
      exchange: fanout_exchange_general   #交换机名称
      queue:  queue_fanout_general,queue_fanout_general1 #队列名称
    topic:                               #需要绑定交换机与队列关系还需要设置ROUTE_KEY
      exchange: topic_exchange_general
      queue:  queue_topic_general,queue_topic_general1  #一对一关系 顺序队列
      routingKey: blf.#,itg.uc
```


##### 7.1.1Direct

```
 > producer

@RunWith(SpringRunner.class) // SpringJUnit支持，由此引入Spring-Test框架支持！
@SpringBootTest(classes = Application.class, webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public class ApplicationTest {

    @Autowired
    RabbitmqSend rabbitmqSend;
    @Test
    public void doTest(){

        //发送10条件消息
        for(int i=0;i<10;i++){
            rabbitmqSend.sendDirect(RabbitConstants.QUEUE_GENERAL,("hello==="+i).toString());
        }

    }
}

> consumer

@Component
@RabbitListener(queues = RabbitConstants.QUEUE_GENERAL)
public class HelloReceiverA {

    @RabbitHandler
    public void process(String context) {
        System.out.println("客户端监听2:"+context);
    }
}

@Component
@RabbitListener(queues = RabbitConstants.QUEUE_GENERAL)
public class HelloReceiverB {

    @RabbitHandler
    public void process(String context) {
        System.out.println("客户端监听1:"+context);
    }
}

> 结果输出
客户端监听2:hello2
客户端监听2:hello3
客户端监听1:hello4
客户端监听2:hello5
客户端监听1:hello6
客户端监听2:hello7
客户端监听1:hello8
客户端监听2:hello9
```


##### 7.1.2Fanout

```
 > producer

public void doFanoutTest(){

    //发送10条件消息
    for(int i=0;i<10;i++){
        System.out.println("消息发送："+("hello==="+i).toString());
        rabbitmqSend.sendFanoutMessage(RabbitConstants.FANOUT_EXCHANGE_GENERAL,("hello==="+i).toString());
    }

}

> consumer

@Component
@RabbitListener(queues = RabbitConstants.QUEUE_FANOUT_GENERAL)
public class HelloFanoutReceiverA {

    @RabbitHandler
    public void process(String context) {
        System.out.println("Fanout客户端监听2:"+context);
    }
}
@RabbitListener(queues = "queue_fanout_general1")
public class HelloFanoutReceiverB {

    @RabbitHandler
    public void process(String context) {
        System.out.println("Fanout客户端监听2:"+context);
    }
}
Fanout客户端监听2:hello===0
Fanout客户端监听1:hello===0
Fanout客户端监听1:hello===1
Fanout客户端监听2:hello===1
Fanout客户端监听1:hello===2
Fanout客户端监听2:hello===2
Fanout客户端监听1:hello===3
Fanout客户端监听2:hello===3
Fanout客户端监听2:hello===4
Fanout客户端监听1:hello===4
Fanout客户端监听2:hello===5
Fanout客户端监听1:hello===5
Fanout客户端监听2:hello===6
Fanout客户端监听1:hello===6
Fanout客户端监听2:hello===7
Fanout客户端监听1:hello===7
Fanout客户端监听2:hello===8
Fanout客户端监听1:hello===8
Fanout客户端监听2:hello===9
Fanout客户端监听1:hello===9
```
##### 7.1.2Topic

```
 > producer
//发送10条件消息
for(int i=0;i<10;i++){
    System.out.println("消息发送："+("hello==="+i).toString());
    if(i<5) {
        rabbitmqSend.senTopicMessage(RabbitConstants.TOPIC_EXCHANGE_GENERAL, "itg.uc", ("hello===" + i).toString());
    }else {
        rabbitmqSend.senTopicMessage(RabbitConstants.TOPIC_EXCHANGE_GENERAL, "blf.uc", ("hello===" + i).toString());

    }
}

> consumer

@Component
@RabbitListener(queues = RabbitConstants.QUEUE_TOPIC_GENERAL)
public class HelloTopReceiverA {

    @RabbitHandler
    public void process(String context) {
        System.out.println("top客户端监听2:"+context);
    }
}

@Component
@RabbitListener(queues = "queue_topic_general1")
public class HelloTopReceiverB {

    @RabbitHandler
    public void process(String context) {
        System.out.println("top客户端监听1:"+context);
    }
}

top客户端监听1:hello===1
top客户端监听2:hello===6
top客户端监听1:hello===2
top客户端监听2:hello===7
top客户端监听1:hello===3
top客户端监听2:hello===8
top客户端监听1:hello===4
top客户端监听2:hello===9
```
