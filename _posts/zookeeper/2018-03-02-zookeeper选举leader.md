---
layout: post
title: zookeeper 选举leader
category: zookeeper
tags: zk,选举
description: zookeeper 选举
--- 

#### 选举算法
zk 的Leader选算法有三种，AuthFastLeaderElection，LeaderElection，FastLeaderElection，
其中前面两种选举算法AuthFastLeaderElection(Udp版本授权模式)，LeaderElection (udp版本选举)，已经在zk3.4.0废弃，所以着重介绍FastLeaderElection


下面是一个选举日志的例子，可以发现server.id=2 成为了leader，其他两台机器称为了Follower。

##### server.id=0
follower 服务器


        2018-02-13 11:26:49,916 [myid:0] - INFO  [QuorumPeer[myid=0]/0:0:0:0:0:0:0:0:2180:QuorumPeer@909] - LOOKING
        2018-02-13 11:26:49,916 [myid:0] - INFO  [QuorumPeer[myid=0]/0:0:0:0:0:0:0:0:2180:FastLeaderElection@820] - New election. My id =  0, proposed zxid=0x200000000
        2018-02-13 11:26:49,916 [myid:0] - INFO  [WorkerReceiver[myid=0]:FastLeaderElection@602] - Notification: 1 (message format version), 0 (n.leader), 0x200000000 (n.zxid), 0xb (n.round), LOOKING (n.state), 0 (n.sid), 0x9 (n.peerEpoch) LOOKING (my state)
        2018-02-13 11:26:49,916 [myid:0] - INFO  [WorkerReceiver[myid=0]:FastLeaderElection@602] - Notification: 1 (message format version), 2 (n.leader), 0x200000000 (n.zxid), 0xa (n.round), LEADING (n.state), 2 (n.sid), 0x9 (n.peerEpoch) LOOKING (my state)
        2018-02-13 11:26:49,916 [myid:0] - INFO  [WorkerReceiver[myid=0]:FastLeaderElection@602] - Notification: 1 (message format version), 2 (n.leader), 0x200000000 (n.zxid), 0xa (n.round), FOLLOWING (n.state), 1 (n.sid), 0x9 (n.peerEpoch) LOOKING (my state)
        2018-02-13 11:26:49,916 [myid:0] - INFO  [QuorumPeer[myid=0]/0:0:0:0:0:0:0:0:2180:QuorumPeer@979] - FOLLOWING
        2018-02-13 11:26:49,916 [myid:0] - INFO  [QuorumPeer[myid=0]/0:0:0:0:0:0:0:0:2180:ZooKeeperServer@173] - Created server with tickTime 2000 minSessionTimeout 4000 maxSessionTimeout 40000 datadir \zk\instance0\data\version-2 snapdir \zk\instance0\data\version-2
        

##### server.id=1

Follower服务器

    2018-02-13 14:35:30,800 [myid:1] - INFO  [/127.0.0.1:3888:QuorumCnxManager$Listener@743] - Received connection request /127.0.0.1:57830
    2018-02-13 14:35:30,800 [myid:1] - WARN  [RecvWorker:0:QuorumCnxManager$RecvWorker@1028] - Interrupting SendWorker
    2018-02-13 14:35:30,800 [myid:1] - WARN  [SendWorker:0:QuorumCnxManager$SendWorker@951] - Send worker leaving thread
    2018-02-13 14:35:30,804 [myid:1] - INFO  [WorkerReceiver[myid=1]:FastLeaderElection@602] - Notification: 1 (message format version), 2 (n.leader), 0x400000000 (n.zxid), 0x2 (n.round), LOOKING (n.state), 0 (n.sid), 0x4 (n.peerEpoch) LOOKING (my state)
    2018-02-13 14:35:31,014 [myid:1] - INFO  [QuorumPeer[myid=1]/0:0:0:0:0:0:0:0:2181:QuorumPeer@979] - FOLLOWING
    2018-02-13 14:35:31,014 [myid:1] - INFO  [QuorumPeer[myid=1]/0:0:0:0:0:0:0:0:2181:ZooKeeperServer@173] - Created server with tickTime 2000 minSessionTimeout 4000 maxSessionTimeout 40000 datadir \zk\instance1\data\version-2 snapdir \zk\instance1\data\version-2
    2018-02-13 14:35:31,014 [myid:1] - INFO  [QuorumPeer[myid=1]/0:0:0:0:0:0:0:0:2181:Follower@65] - FOLLOWING - LEADER ELECTION TOOK - 4282
    2018-02-13 14:35:31,014 [myid:1] - INFO  [QuorumPeer[myid=1]/0:0:0:0:0:0:0:0:2181:QuorumPeer$QuorumServer@184] - Resolved hostname: 127.0.0.1 to address: /127.0.0.1
    2018-02-13 14:35:31,576 [myid:1] - INFO  [QuorumPeer[myid=1]/0:0:0:0:0:0:0:0:2181:Learner@336] - Getting a snapshot from leader 0x500000000
    2018-02-13 14:35:31,581 [myid:1] - INFO  [QuorumPeer[myid=1]/0:0:0:0:0:0:0:0:2181:FileTxnSnapLog@248] - Snapshotting: 0x500000000 to \zk\instance1\data\version-2\snapshot.500000000
     
     
##### server.id=2
     
 Leader   服务器
 
    2018-02-13 14:35:30,783 [myid:2] - INFO  [WorkerReceiver[myid=2]:FastLeaderElection@602] - Notification: 1 (message format version), 0 (n.leader), 0x300000000 (n.zxid), 0x1 (n.round), LOOKING (n.state), 0 (n.sid), 0x4 (n.peerEpoch) LOOKING (my state)
    2018-02-13 14:35:30,783 [myid:2] - INFO  [WorkerReceiver[myid=2]:FastLeaderElection@602] - Notification: 1 (message format version), 2 (n.leader), 0x400000000 (n.zxid), 0x2 (n.round), LOOKING (n.state), 0 (n.sid), 0x4 (n.peerEpoch) LOOKING (my state)
    2018-02-13 14:35:30,800 [myid:2] - INFO  [WorkerReceiver[myid=2]:FastLeaderElection@602] - Notification: 1 (message format version), 2 (n.leader), 0x400000000 (n.zxid), 0x2 (n.round), LOOKING (n.state), 1 (n.sid), 0x4 (n.peerEpoch) LOOKING (my state)
    2018-02-13 14:35:31,061 [myid:2] - INFO  [QuorumPeer[myid=2]/0:0:0:0:0:0:0:0:2182:QuorumPeer@991] - LEADING
    2018-02-13 14:35:31,076 [myid:2] - INFO  [QuorumPeer[myid=2]/0:0:0:0:0:0:0:0:2182:Leader@63] - TCP NoDelay set to: true
    2018-02-13 14:35:31,076 [myid:2] - INFO  [QuorumPeer[myid=2]/0:0:0:0:0:0:0:0:2182:ZooKeeperServer@173] - Created server with tickTime 2000 minSessionTimeout 4000 maxSessionTimeout 40000 datadir \zk\instance2\data\version-2 snapdir \zk\instance2\data\version-2
    2018-02-13 14:35:31,076 [myid:2] - INFO  [QuorumPeer[myid=2]/0:0:0:0:0:0:0:0:2182:Leader@372] - LEADING - LEADER ELECTION TOOK - 4359
    2018-02-13 14:35:31,114 [myid:2] - INFO  [LearnerHandler-/127.0.0.1:57834:LearnerHandler@346] - Follower sid: 0 : info : org.apache.zookeeper.server.quorum.QuorumPeer$QuorumServer@6ad8bf42
    2018-02-13 14:35:31,135 [myid:2] - INFO  [LearnerHandler-/127.0.0.1:57834:LearnerHandler@401] - Synchronizing with Follower sid: 0 maxCommittedLog=0x0 minCommittedLog=0x0 peerLastZxid=0x300000000
    2018-02-13 14:35:31,135 [myid:2] - INFO  [LearnerHandler-/127.0.0.1:57834:LearnerHandler@475] - Sending SNAP
    2018-02-13 14:35:31,135 [myid:2] - INFO  [LearnerHandler-/127.0.0.1:57834:LearnerHandler@499] - Sending snapshot last zxid of peer is 0x300000000  zxid of leader is 0x500000000sent zxid of db as 0x400000000
    2018-02-13 14:35:31,160 [myid:2] - INFO  [LearnerHandler-/127.0.0.1:57834:LearnerHandler@535] - Received NEWLEADER-ACK message from 0
    2018-02-13 14:35:31,160 [myid:2] - INFO  [QuorumPeer[myid=2]/0:0:0:0:0:0:0:0:2182:Leader@962] - Have quorum of supporters, sids: [ 0,2 ]; starting up and setting last processed zxid: 0x500000000
    2018-02-13 14:35:31,558 [myid:2] - INFO  [LearnerHandler-/127.0.0.1:57833:LearnerHandler@346] - Follower sid: 1 : info : org.apache.zookeeper.server.quorum.QuorumPeer$QuorumServer@38bb5aa9
    2018-02-13 14:35:31,574 [myid:2] - INFO  [LearnerHandler-/127.0.0.1:57833:LearnerHandler@401] - Synchronizing with Follower sid: 1 maxCommittedLog=0x0 minCommittedLog=0x0 peerLastZxid=0x300000000
    2018-02-13 14:35:31,574 [myid:2] - INFO  [LearnerHandler-/127.0.0.1:57833:LearnerHandler@475] - Sending SNAP
    2018-02-13 14:35:31,575 [myid:2] - INFO  [LearnerHandler-/127.0.0.1:57833:LearnerHandler@499] - Sending snapshot last zxid of peer is 0x300000000  zxid of leader is 0x500000000sent zxid of db as 0x500000000
    2018-02-13 14:35:31,598 [myid:2] - INFO  [LearnerHandler-/127.0.0.1:57833:LearnerHandler@535] - Received NEWLEADER-ACK message from 1
    





#### 术语

##### SID
服务器的id，myid文件中的数字。


##### ZXID

事务id，每个事务的编号，标识每次事务变更。由Epoch和自增数字组成。

##### VOTE

Leader选举投票信息

##### Quorum
过半机器，过半的仲裁人群，如果总机器数位n
,那么quorun=n/2+1;


##### FastLeaderElection
快速选举算法，这个类英文介绍如下:


   
     Implementation of leader election using TCP. It uses an object of the class
      QuorumCnxManager to manage connections. Otherwise, the algorithm is push-based
      as with the other UDP implementations.
     
      There are a few parameters that can be tuned to change its behavior. First,
      finalizeWait determines the amount of time to wait until deciding upon a leader.
      This is part of the leader election algorithm.


使用tcp协议进行快速选举，QuorumCnxManager会管理tcp链接，此外还会使用udp协议实现推送。




##### 选举细节


当前节点状态变为looking后，会调用lookForLeaer方法，寻找leader节点或者参与选举。


###### 选举步骤


1. 每个机器发出一个投票。初始情况下，每台机器都会选择自己做完leader。比如选票如下(myid,zxid),myid表示选取作为机器的serverID。

2. 接收来自其他机器投票。收到选票后会检测投票的合法性，包括是否本轮选票，是否来自Looking机器的选票。

3. 处理投票。收到每个选票后，需要把选票和自己的投票pk，pk的规则如下:
- 首先判断机器的纪元，纪元大的选票获胜
- 纪元相等的情况下zxid大的选票获胜
- zxid相等的情况下，serverId大的获胜。


4. 统计选票，针对选票选为leader的机器，查看是否有half的机器都支持这台机器作为leader。half是大于集群机器数量的一般，大于等于(n/2+1)


###### 投票数据
    // 提议的leader的serverId
    long proposedLeader;
    // 提议的zxid
    long proposedZxid;
    // 提议的纪元
    long proposedEpoch;
    
    
###### VOTE信息


    final private int version;
    //服务器id
    final private long id;
    
    final private long zxid;
    //选举轮次
    final private long electionEpoch;
    //机器纪元
    final private long peerEpoch;
    
    


###### 快速选举代码

    public Vote lookForLeader() throws InterruptedException {
        //注册 LeaderElectionBean
        try {
            self.jmxLeaderElectionBean = new LeaderElectionBean();
            MBeanRegistry.getInstance().register(
                    self.jmxLeaderElectionBean, self.jmxLocalPeerBean);
        } catch (Exception e) {
            LOG.warn("Failed to register with JMX", e);
            self.jmxLeaderElectionBean = null;
        }
        if (self.start_fle == 0) {
           self.start_fle = Time.currentElapsedTime();
        }
        try {
            HashMap<Long, Vote> recvset = new HashMap<Long, Vote>();

            HashMap<Long, Vote> outofelection = new HashMap<Long, Vote>();
            //投票后等待ack的时间
            int notTimeout = finalizeWait;

           //获取 serverId，最大的zxid，和纪元
            synchronized(this){
                logicalclock.incrementAndGet();
                updateProposal(getInitId(), getInitLastLoggedZxid(), getPeerEpoch());
            }

            LOG.info("New election. My id =  " + self.getId() +
                    ", proposed zxid=0x" + Long.toHexString(proposedZxid));
            //发出投票信息
            sendNotifications();

            /*
             * 在下面循环体中，直到整个集群找到leader
             */
                
            while ((self.getPeerState() == ServerState.LOOKING) &&
                    (!stop)){
                //从队列中获取消息
                Notification n = recvqueue.poll(notTimeout,
                        TimeUnit.MILLISECONDS);

                //收到投票后第一步是验证，验证投票来源的合法性，
                /*
                 * Sends more notifications if haven't received enough.
                 * Otherwise processes new notification.
                 */
                if(n == null){
                    if(manager.haveDelivered()){
                        sendNotifications();
                    } else {
                        manager.connectAll();
                    }

                    /*
                     * Exponential backoff
                     */
                    int tmpTimeOut = notTimeout*2;
                    notTimeout = (tmpTimeOut < maxNotificationInterval?
                            tmpTimeOut : maxNotificationInterval);
                    LOG.info("Notification time out: " + notTimeout);
                }
                else if(self.getVotingView().containsKey(n.sid)) {
                     //如果该消息 serverId的在投票机器中
                    switch (n.state) {
                    // 如果是选举
                    case LOOKING:
            
                        // If notification > current, replace and send messages out
                        //判定逻辑时钟，如果选举的轮次大于当前机器的轮次，就需要统计投票。否则直接丢弃
                        if (n.electionEpoch > logicalclock.get()) {
                            logicalclock.set(n.electionEpoch);
                            recvset.clear();
                            //收到的选票的优先级是否更高，这里有三个依据
                            //机器纪元大的会被选择为leader，否则zxid大的服务器会被选择为leader，zxid相等的情况下server.id大的会被选择为leader
                            if(totalOrderPredicate(n.leader, n.zxid, n.peerEpoch,
                                    getInitId(), getInitLastLoggedZxid(), getPeerEpoch())) {
                                updateProposal(n.leader, n.zxid, n.peerEpoch);
                            } else {
                                updateProposal(getInitId(),
                                        getInitLastLoggedZxid(),
                                        getPeerEpoch());
                            }
                            sendNotifications();
                        } else if (n.electionEpoch < logicalclock.get()) {
                            if(LOG.isDebugEnabled()){
                                LOG.debug("Notification election epoch is smaller than logicalclock. n.electionEpoch = 0x"
                                        + Long.toHexString(n.electionEpoch)
                                        + ", logicalclock=0x" + Long.toHexString(logicalclock.get()));
                            }
                            break;
                            //轮次相同的情况下直接比较选票
                        } else if (totalOrderPredicate(n.leader, n.zxid, n.peerEpoch,
                                proposedLeader, proposedZxid, proposedEpoch)) {
                            updateProposal(n.leader, n.zxid, n.peerEpoch);
                            sendNotifications();
                        }

                        if(LOG.isDebugEnabled()){
                            LOG.debug("Adding vote: from=" + n.sid +
                                    ", proposed leader=" + n.leader +
                                    ", proposed zxid=0x" + Long.toHexString(n.zxid) +
                                    ", proposed election epoch=0x" + Long.toHexString(n.electionEpoch));
                        }
                        //把本轮的选票放入到set中
                        recvset.put(n.sid, new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch));      
                        // 如果当前选取某个leader的选票大于half ，就可以选出leader
                        if (termPredicate(recvset,
                                new Vote(proposedLeader, proposedZxid,
                                        logicalclock.get(), proposedEpoch))) {

                            // Verify if there is any change in the proposed leader
                            while((n = recvqueue.poll(finalizeWait,
                                    TimeUnit.MILLISECONDS)) != null){
                                if(totalOrderPredicate(n.leader, n.zxid, n.peerEpoch,
                                        proposedLeader, proposedZxid, proposedEpoch)){
                                    recvqueue.put(n);
                                    break;
                                }
                            }

                            /*
                             * This predicate is true once we don't read any new
                             * relevant message from the reception queue
                             */
                            if (n == null) {
                                self.setPeerState((proposedLeader == self.getId()) ?
                                        ServerState.LEADING: learningState());

                                Vote endVote = new Vote(proposedLeader,
                                                        proposedZxid,
                                                        logicalclock.get(),
                                                        proposedEpoch);
                                leaveInstance(endVote);
                                return endVote;
                            }
                        }
                        break;
                    case OBSERVING:
                        LOG.debug("Notification from observer: " + n.sid);
                        break;
                    case FOLLOWING:
                    case LEADING:
                    
                                //如果收到的投票中发现某台机器已经被选取为leader，需要验证它的合法性。 如果该leader的选举时钟和本机的逻辑时钟一致，则记录下这个投票结果，然后对这个leader的合法性作最终的判断，包括是否过半的server选择了这个leader(totalPredicate()方法),以及本机是否已经收到了这个leader自己发过来的状态为LEADING的消息（checkLeader（）方法），如果是，则选举可以结束了。
            //如果投票中的选举轮次和当前机器的轮次不一致，记录这个server的投票结果，然后确认是否过半数的服务器都选择了这个leader，如果是，则选举结束。
                        /*
                         * Consider all notifications from the same epoch
                         * together.
                         */
                        if(n.electionEpoch == logicalclock.get()){
                            recvset.put(n.sid, new Vote(n.leader,
                                                          n.zxid,
                                                          n.electionEpoch,
                                                          n.peerEpoch));
                           
                            if(ooePredicate(recvset, outofelection, n)) {
                                self.setPeerState((n.leader == self.getId()) ?
                                        ServerState.LEADING: learningState());

                                Vote endVote = new Vote(n.leader, 
                                        n.zxid, 
                                        n.electionEpoch, 
                                        n.peerEpoch);
                                leaveInstance(endVote);
                                return endVote;
                            }
                        }

                        /*
                         * Before joining an established ensemble, verify
                         * a majority is following the same leader.
                         */
                        outofelection.put(n.sid, new Vote(n.version,
                                                            n.leader,
                                                            n.zxid,
                                                            n.electionEpoch,
                                                            n.peerEpoch,
                                                            n.state));
           
                        if(ooePredicate(outofelection, outofelection, n)) {
                            synchronized(this){
                                logicalclock.set(n.electionEpoch);
                                self.setPeerState((n.leader == self.getId()) ?
                                        ServerState.LEADING: learningState());
                            }
                            Vote endVote = new Vote(n.leader,
                                                    n.zxid,
                                                    n.electionEpoch,
                                                    n.peerEpoch);
                            leaveInstance(endVote);
                            return endVote;
                        }
                        break;
                    default:
                        LOG.warn("Notification state unrecognized: {} (n.state), {} (n.sid)",
                                n.state, n.sid);
                        break;
                    }
                } else {
                    LOG.warn("Ignoring notification from non-cluster member " + n.sid);
                }
            }
            return null;
        } finally {
            try {
                if(self.jmxLeaderElectionBean != null){
                    MBeanRegistry.getInstance().unregister(
                            self.jmxLeaderElectionBean);
                }
            } catch (Exception e) {
                LOG.warn("Failed to unregister with JMX", e);
            }
            self.jmxLeaderElectionBean = null;
            LOG.debug("Number of connection processing threads: {}",
                    manager.getConnectionThreadCount());
        }
    }
      

#### 选举流程图



![](http://7x00ae.com1.z0.glb.clouddn.com/18-2-15/26971791.jpg)
    
    

#### 总结

在zookeeper的选举过程中，为了保证选举过程最后能选出leader，就一定不能出现两台机器得票相同的僵局，所以一般的，要求zk集群的server数量一定要是奇数，也就是2n+1台，并且，如果集群出现问题，其中存活的机器必须大于n+1台，否则leader无法获得多数server的支持，系统就自动挂掉。所以一般是3个或者3个以上节点。

	如果是配置4台服务器，因为要过半的机器可用，这时候4台最多宕机1台，而3台zk实例最多也只宕机1台，从节省资源的角度看，没必要部署4台服务器。