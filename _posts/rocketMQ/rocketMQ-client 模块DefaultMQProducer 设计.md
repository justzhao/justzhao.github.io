---
layout: post
title: rocketMQ  defaultProducer
category: rocketMQ
tags: 消息队列 生产者
---

#### rokectMQ-client 

client模块主要包含了消息生产者和消息消费者的逻辑代码，
消费者通过namesrv发现broker，从broker获取消息然后消费。
生产者通过namesrv，向broker发送消息。


client模块包含了 客户端针对消息的所有操作，投放消息，拉取消息，负载均和，消息投放，消息管理等。

![image](http://7x00ae.com1.z0.glb.clouddn.com/rocketMQclient%E7%BB%93%E6%9E%84%E5%9B%BE.jpg)



#### 消息生产

rocketMQ默认消费者的类主要由DefaultMQProducer  实现。

![image](http://7x00ae.com1.z0.glb.clouddn.com/deafultproducer%E9%BB%98%E8%AE%A4%E7%BB%93%E6%9E%84.jpg)

DefaultMQProducer  根据topic向broker发送消息


 
 
#### 例子

下面代码是一个生成消息的过程

        //新建一个topic=order_system 的producer
         DefaultMQProducer producer = new DefaultMQProducer("order_system");
        // 启动客户端
        producer.start();
        //发送1000条消息
        for (int i = 0; i < 1000; i++) {
            try {
                Message msg = new Message("Ttestzp",// topic
                        "TagA",// tag
                        ("Hello RocketMQ " + i).getBytes(RemotingHelper.DEFAULT_CHARSET)// body
                );
                SendResult sendResult = producer.send(msg);
                LocalTransactionExecuter tranExecuter = new LocalTransactionExecuter() {

                    @Override
                    public LocalTransactionState executeLocalTransactionBranch(Message msg, Object arg) {
                        // TODO Auto-generated method stub
                        return null;
                    }
                };

                //producer.sendMessageInTransaction(msg, tranExecuter, arg)
                System.out.println(sendResult);
            } catch (Exception e) {
                e.printStackTrace();
                Thread.sleep(1000);
            }
        }
        
        
 
 
        

##### 初始化分析

    // 步骤1
    DefaultMQProducer producer = new DefaultMQProducer("order_system");
     
初始化了producer客户端，指定topic，做一系列的实例化对象操作。

    //步骤2
    producer.start();

    启动客户端服务，consumer的操作基本都委托到DefaultMQProducerImpl这个类中，
    
    
DefaultMQProducerImpl start()方法


        public void start(final boolean startFactory) throws MQClientException {
        switch (this.serviceState) {
            //  初始状态为CREATE_JUST
            case CREATE_JUST:
                this.serviceState = ServiceState.START_FAILED;
          // 启动服务包含，Channel服务，定时任务，pullMessageService，等
                this.checkConfig();

                if (!this.defaultMQProducer.getProducerGroup().equals(MixAll.CLIENT_INNER_PRODUCER_GROUP)) {
                    this.defaultMQProducer.changeInstanceNameToPID();
                }
                // 实例化MQClientInstance 所有的客户端操作都会委托到这个实例            
                this.mQClientFactory = MQClientManager.getInstance().getAndCreateMQClientInstance(this.defaultMQProducer, rpcHook);

                boolean registerOK = mQClientFactory.registerProducer(this.defaultMQProducer.getProducerGroup(), this);
                if (!registerOK) {
                    this.serviceState = ServiceState.CREATE_JUST;
                    throw new MQClientException("The producer group[" + this.defaultMQProducer.getProducerGroup()
                            + "] has been created before, specify another name please." + FAQUrl.suggestTodo(FAQUrl.GROUP_NAME_DUPLICATE_URL),
                            null);
                }
                
                // 首先写入默认的topic 后续代码回修正    
                this.topicPublishInfoTable.put(this.defaultMQProducer.getCreateTopicKey(), new TopicPublishInfo());

                if (startFactory) {
                    // 执行MQClientInstance  的start 方法
                    mQClientFactory.start();
                }

                log.info("the producer [{}] start OK. sendMessageWithVIPChannel={}", this.defaultMQProducer.getProducerGroup(),
                        this.defaultMQProducer.isSendMessageWithVIPChannel());
                this.serviceState = ServiceState.RUNNING;
                break;
            case RUNNING:
            case START_FAILED:
            case SHUTDOWN_ALREADY:
                throw new MQClientException("The producer service state not OK, maybe started once, "//
                        + this.serviceState//
                        + FAQUrl.suggestTodo(FAQUrl.CLIENT_SERVICE_NOT_OK),
                        null);
            default:
                break;
        }

        this.mQClientFactory.sendHeartbeatToAllBrokerWithLock();
    }
    
 
MQClientInstance 的start方法    
    
       public void start() throws MQClientException {

        synchronized (this) {
            switch (this.serviceState) {
                case CREATE_JUST:
                    this.serviceState = ServiceState.START_FAILED;
                    // If not specified,looking address from name server
                    if (null == this.clientConfig.getNamesrvAddr()) {
                        this.clientConfig.setNamesrvAddr(this.mQClientAPIImpl.fetchNameServerAddr());
                    }
                    // Start request-response channel
                    // 启动netty
                    this.mQClientAPIImpl.start();
                    // Start various schedule tasks
                    //启动一下定时任务，比如 fetchNameServerAddr
                    this.startScheduledTask();
                    // Start pull service
                    // 启动pullMessageService 服务，拉取消息服务。
                    this.pullMessageService.start();
                    // Start rebalance service
                    this.rebalanceService.start();
                    // Start push service
                    this.defaultMQProducer.getDefaultMQProducerImpl().start(false);
                    log.info("the client factory [{}] start OK", this.clientId);
                    this.serviceState = ServiceState.RUNNING;
                    break;
                case RUNNING:
                    break;
                case SHUTDOWN_ALREADY:
                    break;
                case START_FAILED:
                    throw new MQClientException("The Factory object[" + this.getClientId() + "] has been created before, and failed.", null);
                default:
                    break;
            }
        }
    }
    
    
###### 启动流程图如下
![image](http://7x00ae.com1.z0.glb.clouddn.com/rocket%20DefaultMQPullConsumer%20%E5%90%AF%E5%8A%A8.png)


##### 发送消息分析

以下面代码为例

        Message msg = new Message("Ttestzp",topic "TagA",  ("Hello RocketMQ").getBytes(RemotingHelper.DEFAULT_CHARSET)
                );
                
        SendResult sendResult = producer.send(msg);



producer.send(msg) 核心方法委托到 DefaultMQProducerImpl中的sendDefaultImpl


     private SendResult sendDefaultImpl(//
                                       Message msg, //
                                       final CommunicationMode communicationMode, //
                                       final SendCallback sendCallback, //
                                       final long timeout//
    ) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
        // 检测consumer状态
        this.makeSureStateOK();
        Validators.checkMessage(msg, this.defaultMQProducer);

        final long invokeID = random.nextLong();
        long beginTimestampFirst = System.currentTimeMillis();
        long beginTimestampPrev = beginTimestampFirst;
        long endTimestamp = beginTimestampFirst;
        // 根据topic获取路由消息，见下面分析
        TopicPublishInfo topicPublishInfo = this.tryToFindTopicPublishInfo(msg.getTopic());
        // 拿到路由信心
        if (topicPublishInfo != null && topicPublishInfo.ok()) {
            MessageQueue mq = null;
            Exception exception = null;
            SendResult sendResult = null;
            int timesTotal = communicationMode == CommunicationMode.SYNC ? 1 + this.defaultMQProducer.getRetryTimesWhenSendFailed() : 1;
            int times = 0;
            String[] brokersSent = new String[timesTotal];
            
            // for 循环主要是针对发送错误重试发送消息
            for (; times < timesTotal; times++) {
                String lastBrokerName = null == mq ? null : mq.getBrokerName();
                // 从路由中选择一个messageQueue
                MessageQueue tmpmq = this.selectOneMessageQueue(topicPublishInfo, lastBrokerName);
                if (tmpmq != null) {
                    mq = tmpmq;
                    brokersSent[times] = mq.getBrokerName();
                    try {
                        beginTimestampPrev = System.currentTimeMillis();
                        // 发送消息
                        sendResult = this.sendKernelImpl(msg, mq, communicationMode, sendCallback, topicPublishInfo, timeout);
                        endTimestamp = System.currentTimeMillis();
                        this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, false);
                        switch (communicationMode) {
                            case ASYNC:
                                return null;
                            case ONEWAY:
                                return null;
                            case SYNC:
                                if (sendResult.getSendStatus() != SendStatus.SEND_OK) {
                                    if (this.defaultMQProducer.isRetryAnotherBrokerWhenNotStoreOK()) {
                                        continue;
                                    }
                                }

                                return sendResult;
                            default:
                                break;
                        }
                        // 捕捉异常，如果有异常需要重试发送消息
                    } catch (RemotingException e) {
                
                        endTimestamp = System.currentTimeMillis();
                        this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, true);
                        log.warn(String.format("sendKernelImpl exception, resend at once, InvokeID: %s, RT: %sms, Broker: %s", invokeID, endTimestamp - beginTimestampPrev, mq), e);
                        log.warn(msg.toString());
                        exception = e;
                        continue;
                    } catch (MQClientException e) {
                        endTimestamp = System.currentTimeMillis();
                        this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, true);
                        log.warn(String.format("sendKernelImpl exception, resend at once, InvokeID: %s, RT: %sms, Broker: %s", invokeID, endTimestamp - beginTimestampPrev, mq), e);
                        log.warn(msg.toString());
                        exception = e;
                        continue;
                    } catch (MQBrokerException e) {
                        // 如果是broker 一次，则不重试了，直接返回结果。
                        endTimestamp = System.currentTimeMillis();
                        this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, true);
                        log.warn(String.format("sendKernelImpl exception, resend at once, InvokeID: %s, RT: %sms, Broker: %s", invokeID, endTimestamp - beginTimestampPrev, mq), e);
                        log.warn(msg.toString());
                        exception = e;
                        switch (e.getResponseCode()) {
                            case ResponseCode.TOPIC_NOT_EXIST:
                            case ResponseCode.SERVICE_NOT_AVAILABLE:
                            case ResponseCode.SYSTEM_ERROR:
                            case ResponseCode.NO_PERMISSION:
                            case ResponseCode.NO_BUYER_ID:
                            case ResponseCode.NOT_IN_CURRENT_UNIT:
                                continue;
                            default:
                                if (sendResult != null) {
                                    return sendResult;
                                }

                                throw e;
                        }
                    } catch (InterruptedException e) {
                        endTimestamp = System.currentTimeMillis();
                        this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, false);
                        log.warn(String.format("sendKernelImpl exception, throw exception, InvokeID: %s, RT: %sms, Broker: %s", invokeID, endTimestamp - beginTimestampPrev, mq), e);
                        log.warn(msg.toString());

                        log.warn("sendKernelImpl exception", e);
                        log.warn(msg.toString());
                        throw e;
                    }
                } else {
                    break;
                }
            } // end of for

            if (sendResult != null) {
                return sendResult;
            }

            String info = String.format("Send [%d] times, still failed, cost [%d]ms, Topic: %s, BrokersSent: %s", //
                    times, //
                    (System.currentTimeMillis() - beginTimestampFirst), //
                    msg.getTopic(), //
                    Arrays.toString(brokersSent));

            info += FAQUrl.suggestTodo(FAQUrl.SEND_MSG_FAILED);

            MQClientException mqClientException = new MQClientException(info, exception);
            if (exception instanceof MQBrokerException) {
                mqClientException.setResponseCode(((MQBrokerException) exception).getResponseCode());
            } else if (exception instanceof RemotingConnectException) {
                mqClientException.setResponseCode(ClientErrorCode.ConnectBrokerException);
            } else if (exception instanceof RemotingTimeoutException) {
                mqClientException.setResponseCode(ClientErrorCode.AccessBrokerTimeout);
            } else if (exception instanceof MQClientException) {
                mqClientException.setResponseCode(ClientErrorCode.BrokerNotExistException);
            }

            throw mqClientException;
        }

        List<String> nsList = this.getmQClientFactory().getMQClientAPIImpl().getNameServerAddressList();
        if (null == nsList || nsList.isEmpty()) {
            throw new MQClientException(
                    "No name server address, please set it." + FAQUrl.suggestTodo(FAQUrl.NAME_SERVER_ADDR_NOT_EXIST_URL), null).setResponseCode(ClientErrorCode.NoNameServerException);
        }

        throw new MQClientException("No route info of this topic, " + msg.getTopic() + FAQUrl.suggestTodo(FAQUrl.NO_TOPIC_ROUTE_INFO),
                null).setResponseCode(ClientErrorCode.NotFoundTopicException);
    }


    // 发送消息的时候，根据topic获取路由信息。
    // 如果是新的topic，会根据topic =TBW102(默认存在的topic) 去获取路由信息，然后新的topic 会使用默认topic的路由信息 
    private TopicPublishInfo tryToFindTopicPublishInfo(final String topic) {
        TopicPublishInfo topicPublishInfo = this.topicPublishInfoTable.get(topic);
        if (null == topicPublishInfo || !topicPublishInfo.ok()) {
            this.topicPublishInfoTable.putIfAbsent(topic, new TopicPublishInfo());
            this.mQClientFactory.updateTopicRouteInfoFromNameServer(topic);
            topicPublishInfo = this.topicPublishInfoTable.get(topic);
        }

        if (topicPublishInfo.isHaveTopicRouterInfo() || topicPublishInfo.ok()) {
            return topicPublishInfo;
        } else {
            // 获取topic=TBW102 的路由信息
            this.mQClientFactory.updateTopicRouteInfoFromNameServer(topic, true, this.defaultMQProducer);
            topicPublishInfo = this.topicPublishInfoTable.get(topic);
            return topicPublishInfo;
        }
    }



###### 发送消息流程图
![image](http://7x00ae.com1.z0.glb.clouddn.com/rocket%20DefaultMQPullConsumer%20%E5%8F%91%E9%80%81%E6%B6%88%E6%81%AF.png)


#### 总结

rocketMQ生产者的主要关键类为 DefaultMQProducer,DefaultMQProducerImpl,MQClientInstance.

在发送消息时候，会通过namesrv根据topic信息找到路由，这里注意的时，如果topic不存在，则默认获取topic=tbw102的路由信息进行发送消息。
