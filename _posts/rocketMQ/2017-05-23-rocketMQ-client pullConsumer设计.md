---
layout: post
title: rocketMQ的pullConsumer
category: rocketMQ
tags: 消息队列 消费者
---

#### 消费消息
rocketMQ 客户端在fetch消息一共有两种方式，第一种为pu
ll，主动拉取，第二种为push，客户端被动推送。


##### pull

客户端主动根据topic fetch到消息队列MessageQueue，然后从MessageQueue中消费消息，客户端需要自己维护消息消费的位置(offset)

##### push
push 模式是被动消费，客户端通过订阅topic，然后注册MessageListener 进行被动消费。push 底层还是通过pull实现的，底层实现是pullService，通过pullService不动轮询拉取消息，然后回调
MessageListener的consumeMessage。此时客户端也不需要关系消息消费的offset。



#### 例子

下面是一个Pull 消费的例子，启动了一个叫做order-system 的consumer。然后根据topic="TopicTest"，获取到MessageQueue。最后在MessageQueue中拉取消息进行消费。

      public static void main(String[] args) throws MQClientException, InterruptedException {

        DefaultMQPullConsumer consumer = new DefaultMQPullConsumer("order-system");
        consumer.start();
        // fetch 到主题为TopicTest的队列。
        Set<MessageQueue> mqs = consumer.fetchSubscribeMessageQueues("TopicTest");
        for (MessageQueue mq : mqs) {
            while (true) {
                try {
                    // 拉取消息，一次性拉取32条消息，
                    PullResult pullResult =
                            consumer.pullBlockIfNotFound(mq, null, getMessageQueueOffset(mq), 32);
                    // 记录每个messageQueue的消费偏移量        
                    putMessageQueueOffset(mq, pullResult.getNextBeginOffset());
                    if (pullResult.getPullStatus() == PullStatus.FOUND) {
                        List<MessageExt> msgs = pullResult.getMsgFoundList();

                        for (MessageExt m : msgs) {
                            // 生产者发送的是字符串，座椅这里直接生成string对象。    
                            System.out.println(new String(m.getBody()));
                        }
                    }

                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }
        consumer.shutdown();
    }

##### 源码分析

  DefaultMQPullConsumer consumer = new DefaultMQPullConsumer("order-system"); 最终调用如下代码
  
    //所有的consumer操作都会委托到DefaultMQPullConsumerImpl
      public DefaultMQPullConsumer(final String consumerGroup, RPCHook rpcHook) {
        this.consumerGroup = consumerGroup;
        defaultMQPullConsumerImpl = new DefaultMQPullConsumerImpl(this, rpcHook);
    }
    
    
    
  consumer.start() 分析
  
start 方法最终调用到DefaultMQPullConsumerImpl的start方法。  
 
 
       public void start() throws MQClientException {
       // 初始状态为 ServiceState.CREATE_JUST;
        switch (this.serviceState) {
            case CREATE_JUST:
                this.serviceState = ServiceState.START_FAILED;
                // 检查配置    
                this.checkConfig();
                //获取订阅的topic    
                this.copySubscription();

                if (this.defaultMQPullConsumer.getMessageModel() == MessageModel.CLUSTERING) {
                    this.defaultMQPullConsumer.changeInstanceNameToPID();
                }

                // 实例化MQClientInstance，用来发出网络请求的实例
                this.mQClientFactory = MQClientManager.getInstance().getAndCreateMQClientInstance(this.defaultMQPullConsumer, this.rpcHook);

                //RebalanceImpl 负载均衡类，负责选择messageQueue拉取消息，和维护Topic的路由信息.
                this.rebalanceImpl.setConsumerGroup(this.defaultMQPullConsumer.getConsumerGroup());
                this.rebalanceImpl.setMessageModel(this.defaultMQPullConsumer.getMessageModel());
                this.rebalanceImpl.setAllocateMessageQueueStrategy(this.defaultMQPullConsumer.getAllocateMessageQueueStrategy());
                this.rebalanceImpl.setmQClientFactory(this.mQClientFactory);
                
                // 拉取消息的API 
                this.pullAPIWrapper = new PullAPIWrapper(//
                        mQClientFactory, //
                        this.defaultMQPullConsumer.getConsumerGroup(), isUnitMode());
                this.pullAPIWrapper.registerFilterMessageHook(filterMessageHookList);

                if (this.defaultMQPullConsumer.getOffsetStore() != null) {
                    this.offsetStore = this.defaultMQPullConsumer.getOffsetStore();
                } else {
                    switch (this.defaultMQPullConsumer.getMessageModel()) {
                        case BROADCASTING:
                            this.offsetStore = new LocalFileOffsetStore(this.mQClientFactory, this.defaultMQPullConsumer.getConsumerGroup());
                            break;
                        case CLUSTERING:
                            this.offsetStore = new RemoteBrokerOffsetStore(this.mQClientFactory, this.defaultMQPullConsumer.getConsumerGroup());
                            break;
                        default:
                            break;
                    }
                }

                this.offsetStore.load();

                boolean registerOK = mQClientFactory.registerConsumer(this.defaultMQPullConsumer.getConsumerGroup(), this);
                if (!registerOK) {
                    this.serviceState = ServiceState.CREATE_JUST;

                    throw new MQClientException("The consumer group[" + this.defaultMQPullConsumer.getConsumerGroup()
                            + "] has been created before, specify another name please." + FAQUrl.suggestTodo(FAQUrl.GROUP_NAME_DUPLICATE_URL),
                            null);
                }
                
                // MQClientInstance 的start方法，启动consumer的一些服务方法
                // 比如 mQClientAPIImpl =========>netty 
                // startScheduledTask ======>定时调度任务
                // pullMessageService===========> 拉取消息的任务
                // RebalanceService=========>启动 doRebalance
                mQClientFactory.start();
                log.info("the consumer [{}] start OK", this.defaultMQPullConsumer.getConsumerGroup());
                this.serviceState = ServiceState.RUNNING;
                break;
            case RUNNING:
            case START_FAILED:
            case SHUTDOWN_ALREADY:
                throw new MQClientException("The PullConsumer service state not OK, maybe started once, "//
                        + this.serviceState//
                        + FAQUrl.suggestTodo(FAQUrl.CLIENT_SERVICE_NOT_OK),
                        null);
            default:
                break;
        }
    } 
    
    
Set<MessageQueue> mqs = consumer.fetchSubscribeMessageQueues("TopicTest"); 分析,根据topic从Namsrv中拉取到messageQueue。

调用到MQClientAPIImpl 中的getTopicRouteInfoFromNameServer 方法。使用netty向namesrv获取top的路由信息

       public TopicRouteData getTopicRouteInfoFromNameServer(final String topic, final long timeoutMillis)
            throws RemotingException, MQClientException, InterruptedException {
        GetRouteInfoRequestHeader requestHeader = new GetRouteInfoRequestHeader();
        requestHeader.setTopic(topic);
        // GET_ROUTEINTO_BY_TOPIC 是根据topic 获取路由信息
        RemotingCommand request = RemotingCommand.createRequestCommand(RequestCode.GET_ROUTEINTO_BY_TOPIC, requestHeader);

        RemotingCommand response = this.remotingClient.invokeSync(null, request, timeoutMillis);
        assert response != null;
        switch (response.getCode()) {
            case ResponseCode.TOPIC_NOT_EXIST: {
                if (!topic.equals(MixAll.DEFAULT_TOPIC))
                    log.warn("get Topic [{}] RouteInfoFromNameServer is not exist value", topic);
                break;
            }
            case ResponseCode.SUCCESS: {
                byte[] body = response.getBody();
                if (body != null) {
                    return TopicRouteData.decode(body, TopicRouteData.class);
                }
            }
            default:
                break;
        }

        throw new MQClientException(response.getCode(), response.getRemark());
    }
    

// MQClientInstance 把 TopicRouteData 转化成MessageQueue。   
    
    
        public static Set<MessageQueue> topicRouteData2TopicSubscribeInfo(final String topic, final TopicRouteData route) {
        Set<MessageQueue> mqList = new HashSet<MessageQueue>();
        // 此Topic可能会有多台broker。一个Queue对应一个broker
        List<QueueData> qds = route.getQueueDatas();
        for (QueueData qd : qds) {
            if (PermName.isReadable(qd.getPerm())) {
                //每个broker可以有多个读队列。
                for (int i = 0; i < qd.getReadQueueNums(); i++) {
                    MessageQueue mq = new MessageQueue(topic, qd.getBrokerName(), i);
                    mqList.add(mq);
                }
            }
        }

        return mqList;
    }
    
    
    
PullResult pullResult =consumer.pullBlockIfNotFound(mq, null, getMessageQueueOffset(mq), 32); 分析


最终调用到MQClientAPIImpl 的pullMessage

     public PullResult pullMessage(//
                                  final String addr, //
                                  final PullMessageRequestHeader requestHeader, //
                                  final long timeoutMillis, //
                                  final CommunicationMode communicationMode, //
                                  final PullCallback pullCallback//
    ) throws RemotingException, MQBrokerException, InterruptedException {
        // 业务代码为PullMessage。    
        RemotingCommand request =
        RemotingCommand.createRequestCommand(RequestCode.PULL_MESSAGE, requestHeader);

        switch (communicationMode) {
            case ONEWAY:
                assert false;
                return null;
            case ASYNC:
                this.pullMessageAsync(addr, request, timeoutMillis, pullCallback);
                return null;
            case SYNC:
                return this.pullMessageSync(addr, request, timeoutMillis);
            default:
                assert false;
                break;
        }

        return null;
    }
    
    


#### 时序图。

下面是pull消息的时序图

![image](http://7x00ae.com1.z0.glb.clouddn.com/rocketMq%20pullMessage%E6%97%B6%E5%BA%8F.png)

可以到consumer先fetch到MessageQueue，然后再MessageQueue上pull消息





#### 总结

rocketMQ 中的pullConsumer,需要客户端维护每个队列的消费进度(offset)。pull 消费的好处是客户端能消费多少就拉取多少。