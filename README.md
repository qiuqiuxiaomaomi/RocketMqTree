# RocketMqTree
RocketMq消息中间件技术研究


![](https://i.imgur.com/qOqISuz.png)

<pre>
集群工作流程：
      1）启动NameSrv，NameSrv起来后监听端口，等待Broker， Producer， Consumer连接，
         NameSrv相当于一个路由注册中心，Kafka中的Zookeeper
      2）Broker启动，跟所有的NameSrv保持长连接，定时发送心跳包。心跳包中包含当前Broker
         的信息（IP+端口）以及存储所有topic信息，注册成功后，NameSrv集群中就有Topic跟
         Broker的映射关系。
      3）收发消息前，先创建Topic，创建Topic时需要指定该Topic要存储在哪些Broker上，也可以
         在发送消息时自动创建Topic
      5) Producer发送消息，启动时限跟NameSrv集群中的其中一台建立长连接，并从NameSrv中
         获取当前发送的Topic存在哪些Broker上，然后跟对应的Broker建立长连接，直接向
         Broker发消息
      6）Consumer跟Producer类似，跟其中一台NameSrv建立长连接，获取当前定于的Topic在哪些
         Broker上，然后直接跟Broker建立连接通道，开始消费消息。
</pre>

<pre>
Broker
      1：高并发读写服务
         Broker的高并发读写主要依靠以下两点：
            1）消息顺序写
               所有的Topic数据同时只会写一个文件，一个文件满1G，再写新文件，真正的顺序写
               盘，是的发消息TPS大幅提高。
            2）消息随机读
               RocketMq尽可能让读命中系统pageCache，因为操作系统访问pageCache时，即使
               只访问1k的消息，系统也会提前预读出更多的数据，在下次读时就可能命中pageCache,减少I/O

      2：负载均衡与动态伸缩
         1)负载均衡：Broker上存Topic信息，Topic由多个队列组成，队列会平均分散在多个
           Broker上，而Producer的发送机制保证消息尽量平均分配到所有队列中，最终效果
           就是所有消息都平均落在每个Broker上。
         2）动态伸缩能力：Broker的伸缩性体现在两个维度。Topic, Broker
            Topic维度：假如一个Topic的消息量特别大，但集群水位压力还是很低，就可以扩大
                      该Topic的队列数，Topic的队列数跟发送，消费速度成正比。
            Broker维度：如果集群水位很高了，需要扩容，直接加机器部署Broker就可以，Broker
                      启动以后向NameSrv注册，Producer,Consumer通过NameSrv发现新的
                      Broker，立即跟该Broker直连，发送消息。

     单个Broker跟所有Namesrv保持心跳请求，心跳间隔为30秒，心跳请求中包括当前Broker所有
     的Topic信息。Namesrv会反查Broer的心跳信息，如果某个Broker在2分钟之内都没有心跳，则
     认为该Broker下线，调整Topic跟Broker的对应关系。但此时Namesrv不会主动通知Producer、Consumer有Broker宕机。
</pre>

<pre>
Producer
      Producer启动时，也需要指定Namesrv的地址，从Namesrv集群中选一台建立长连接。如果
      该Namesrv宕机，会自动连其他Namesrv。直到有可用的Namesrv为止。

      生产者每30秒从Namesrv获取Topic跟Broker的映射关系，更新到本地内存中。再跟Topic涉
      及的所有Broker建立长连接，每隔30秒发一次心跳。在Broker端也会每10秒扫描一次当前注
      册的Producer，如果发现某个Producer超过2分钟都没有发心跳，则断开连接。

      这里需要注意一点：假如某个Broker宕机，意味生产者最长需要30秒才能感知到。在这期间会
      向宕机的Broker发送消息。当一条消息发送到某个Broker失败后，会往该broker自动再重
      发2次，假如还是发送失败，则抛出发送失败异常。业务捕获异常，重新发送即可。客户端里会自
      动轮询另外一个Broker重新发送，这个对于用户是透明的。
</pre>

<pre>
NameSrv
      1）NameSrv用于存储Topic，Broker关系信息，功能简单，稳定性高，多个NameSrv之间没有
         通信，单台NameSrv宕机不影响其他NameSrv与集群；即使整个NameSrv集群宕机，已经正常
         工作的Producer,Consumer，Broker仍然能正常工作，但新起的Producer, Broker,
         Consumer就无法工作。
      2）NameSrv压力不会太大，平时主要开销是维持心跳和提供Topic-Broker的关系数据。但有一
         点需要注意，Broker向NameSrv发心跳时，会带上当前自己所负责的所有Topic信息，如果
         Topic个数太多（万级别），会导致一次心跳中，就Topic的数据就几十M，网络情况差的
         话，网络传输失败，心跳失败，导致NameSrv误认为Broker心跳失败。
</pre>

<pre>
Consumer

      消费者启动时需要指定Namesrv地址，与其中一个Namesrv建立长连接。消费者每隔30秒
      从nameserver获取所有topic的最新队列情况，这意味着某个broker如果宕机，客户端最多
      要30秒才能感知。连接建立后，从namesrv中获取当前消费Topic所涉及的Broker，直连Broker。

      Consumer跟Broker是长连接，会每隔30秒发心跳信息到Broker。Broker端每10秒检查一次当
      前存活的Consumer，若发现某个Consumer 2分钟内没有心跳，就断开与该Consumer的连接，
      并且向该消费组的其他实例发送通知，触发该消费者集群的负载均衡。
</pre>

<pre>
RocketMq与Kafka
   数据可靠性对比： 
      1）RocketMQ支持异步实时刷盘，同步刷盘，同步复制，异步复制
      2）Kafka使用异步刷盘方式，异步复制/同步复制

   性能对比
      3) 卡夫卡单机写入TPS约在百万条/秒，消息大小10个字节
      5) RocketMQ单机写入TPS单实例约7万条/秒，单机部署3个Broker，可以跑到最高12万条/秒，消息大小10个字节

          客户端通常使用的Java语言，缓存过多消息，GC是个很严重的问题
          Producer调用发送消息接口，消息未发送到Broker，向业务返回成功，此时Producer宕机，会导致消息丢失，业务出错
          Producer通常为分布式系统，且每台机器都是多线程发送，我们认为线上的系统单个Producer每秒产生的数据量有限，不可能上万。
          缓存的功能完全可以由上层业务完成。

   单机支持的队列数
      6) Kafka单机超过64个队列/分区，Load会发生明显的飙高现象，队列越多，load越高，发送消息响应时间变长。Kafka分区数无法过多的问题
      7) RocketMQ单机支持最高5万个队列，负载不会发生明显变化
      
      队列多有什么好处？
         单机可以创建更多话题，因为每个主题都是由一批队列组成
         消费者的集群规模和队列数成正比，队列越多，消费类集群可以越大

   消息投递实时性
      8) Kafka使用短轮询方式，实时性取决于轮询间隔时间，0.8以后版本支持长轮询。
      9) RocketMQ使用长轮询，同Push方式实时性一致，消息的投递延时通常在几个毫秒。
 
   消费失败重试
      1) 卡夫卡消费失败不支持重试
      2) RocketMQ消费失败支持定时重试，每次重试间隔时间顺延

   严格的消息顺序
      卡夫卡支持消息顺序，但是一台代理宕机后，就会产生消息乱序
      RocketMQ支持严格的消息顺序，在顺序消息场景下，一台Broker宕机后，发送消息会失败，但是不会乱序

   分布式事务
      卡夫卡不支持分布式事务消息
      阿里云MQ支持分布式事务消息，未来开源版本的RocketMQ也有计划支持分布式事务消息

   消息查询
      卡夫卡不支持消息查询
      RocketMQ支持根据消息标识查询消息，也支持根据消息内容查询消息（发送消息时指定一个消息密钥，任意字符串，例如指定为订单编号）

   消息回溯
      卡夫卡理论上可以按照偏移来回溯消息
      RocketMQ支持按照时间来回溯消息，精度毫秒，例如从一天之前的某时某分某秒开始重新消费消息
   
   消息消费并行度
      Kafka的消费并行度依赖Topic配置的分区数，如分区数为10，那么最多10台机器来并行消
      费（每台机器只能开启一个线程），或者一台机器消费（10个线程并行消费）。即消费并行度和分区数一致。
      RocketMQ消费并行度分两种情况：
          顺序消费方式并行度同卡夫卡完全一致
          乱序方式并行度取决于Consumer的线程数，如Topic配置10个队列，10台机器消费，每
              台机器100个线程，那么并行度为1000。

   消息轨迹
      卡夫卡不支持消息轨迹
      阿里云MQ支持消息轨迹
   
   开发语言友好性
      卡夫卡采用斯卡拉编写
       RocketMQ采用的Java语言编写
</pre>