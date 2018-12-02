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
</pre>

<pre>
Producer
</pre>

<pre>
NameSrv
</pre>

<pre>
Consumer
</pre>

<pre>
RocketMq与Kafka
</pre>