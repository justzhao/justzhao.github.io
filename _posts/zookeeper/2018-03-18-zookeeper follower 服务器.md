---
layout: post
title: zookeeper follower 服务器
category: zookeeper
tags: zk,follower
description: follower, 接收请求
--- 

当一个节点QuorumPeer发现自己的状态为Following后，就会调用makerFoller方法，如下代码所示，新生成一个Follower。



     protected Follower makeFollower(FileTxnSnapLog logFactory) throws IOException {
        return new Follower(this, new FollowerZooKeeperServer(logFactory, 
                this,new ZooKeeperServer.BasicDataTreeBuilder(), this.zkDb));
    }
    


#### Follower 对象

Follower对象是继承Learner对象，前文说过，Follower和Observer都是Learner的一种

    public class Follower extends Learner{

    private long lastQueued;
    // This is the same object as this.zk, but we cache the downcast op
    final FollowerZooKeeperServer fzk;
    
    Follower(QuorumPeer self,FollowerZooKeeperServer zk)    {
        this.self = self;
        this.zk=zk;
        this.fzk = zk;
    }

  
    /**
        执行Follow方法，然后可以进行FollowLeader    
     * the main method called by the follower to follow the leader
     *
     * @throws InterruptedException
     */
    void followLeader() throws InterruptedException {
        self.end_fle = Time.currentElapsedTime();
        long electionTimeTaken = self.end_fle - self.start_fle;
        self.setElectionTimeTaken(electionTimeTaken);
        LOG.info("FOLLOWING - LEADER ELECTION TOOK - {}", electionTimeTaken);
        self.start_fle = 0;
        self.end_fle = 0;
        fzk.registerJMX(new FollowerBean(this, zk), self.jmxLocalPeerBean);
        try {
            QuorumServer leaderServer = findLeader();            
            try {
            //和leader服务器建立联系，这里的端口是配置文件的第二个端口也就是数据同步端口
                connectToLeader(leaderServer.addr, leaderServer.hostname);
                // 这里从准Leader获取最新的Epoch（获取LEADERINFO类型消息），并且响应NewEpoch
                long newEpochZxid = registerWithLeader(Leader.FOLLOWERINFO);

                //check to see if the leader zxid is lower than ours
                //this should never happen but is just a safety check
                
                //这里检测下epoch
                long newEpoch = ZxidUtils.getEpochFromZxid(newEpochZxid);
                if (newEpoch < self.getAcceptedEpoch()) {
                    LOG.error("Proposed leader epoch " + ZxidUtils.zxidToString(newEpochZxid)
                            + " is less than our accepted epoch " + ZxidUtils.zxidToString(self.getAcceptedEpoch()));
                    throw new IOException("Error: Epoch of leader is lower");
                }
                //调用父类的方法开始执行同步，NewLeader等消息
                syncWithLeader(newEpochZxid);                
                //到这里说明同步阶段完成，可以开始接受外部的请求。
                QuorumPacket qp = new QuorumPacket();
                while (this.isRunning()) {
                    readPacket(qp);
                    processPacket(qp);
                }
            } catch (Exception e) {
                LOG.warn("Exception when following the leader", e);
                try {
                    sock.close();
                } catch (IOException e1) {
                    e1.printStackTrace();
                }
    
                // clear pending revalidations
                pendingRevalidations.clear();
            }
        } finally {
            zk.unregisterJMX((Learner)this);
        }
    }

    /**
      按照packet的类型处理请求，
     * Examine the packet received in qp and dispatch based on its contents.
     * @param qp
     * @throws IOException
     */
    protected void processPacket(QuorumPacket qp) throws IOException{
        switch (qp.getType()) {
        case Leader.PING:            
            ping(qp);            
            break;
        case Leader.PROPOSAL:            
            TxnHeader hdr = new TxnHeader();
            Record txn = SerializeUtils.deserializeTxn(qp.getData(), hdr);
            if (hdr.getZxid() != lastQueued + 1) {
                LOG.warn("Got zxid 0x"
                        + Long.toHexString(hdr.getZxid())
                        + " expected 0x"
                        + Long.toHexString(lastQueued + 1));
            }
            lastQueued = hdr.getZxid();
            fzk.logRequest(hdr, txn);
            break;
        case Leader.COMMIT:
            fzk.commit(qp.getZxid());
            break;
        case Leader.UPTODATE:
            LOG.error("Received an UPTODATE message after Follower started");
            break;
        case Leader.REVALIDATE:
            revalidate(qp);
            break;
        case Leader.SYNC:
            fzk.sync();
            break;
        default:
            LOG.error("Invalid packet type: {} received by Observer", qp.getType());
        }
    }

     }
  
#### FollowerZooKeeperServer

 Follower服务器的主业务类是FollowerZooKeeperServer，同样也继承了ZooKeeperServer。
 
 

Follower处理请求的流程如下：

 processors: FollowerRequestProcessor -> CommitProcessor ->FinalRequestProcessor
    
如果发现有事务请求就转发request给leader服务器。

此外Follower接收来自leader的Sync请求，处理流程如下

  
SyncRequestProcessor -> SendAckRequestProcessor


以下是FollowerZooKeeperServer 类的代码




    
    public class FollowerZooKeeperServer extends LearnerZooKeeperServer {



    private static final Logger LOG =
        LoggerFactory.getLogger(FollowerZooKeeperServer.class);

    CommitProcessor commitProcessor;

    SyncRequestProcessor syncProcessor;

    /*
     * Pending sync requests
     */
    ConcurrentLinkedQueue<Request> pendingSyncs;
    
    /**
     * @param port
     * @param dataDir
     * @throws IOException
     */
    FollowerZooKeeperServer(FileTxnSnapLog logFactory,QuorumPeer self,
            DataTreeBuilder treeBuilder, ZKDatabase zkDb) throws IOException {
        super(logFactory, self.tickTime, self.minSessionTimeout,
                self.maxSessionTimeout, treeBuilder, zkDb, self);
        this.pendingSyncs = new ConcurrentLinkedQueue<Request>();
    }

    public Follower getFollower(){
        return self.follower;
    }     
    
    设置处理器的逻辑代码 FollowerRequestProcessor，CommitProcessor，FinalRequestProcessor，
    此外还有SyncRequestProcessor和，SendAckRequestProcessor，SyncRequestProcessor用来记录来自leader 的事务请求。

    @Override
    protected void setupRequestProcessors() {
        RequestProcessor finalProcessor = new FinalRequestProcessor(this);
        commitProcessor = new CommitProcessor(finalProcessor,
                Long.toString(getServerId()), true,
                getZooKeeperServerListener());
        commitProcessor.start();
        firstProcessor = new FollowerRequestProcessor(this, commitProcessor);
        ((FollowerRequestProcessor) firstProcessor).start();
        syncProcessor = new SyncRequestProcessor(this,
                new SendAckRequestProcessor((Learner)getFollower()));
        syncProcessor.start();
    }

    LinkedBlockingQueue<Request> pendingTxns = new LinkedBlockingQueue<Request>();

    //记录一个Proposal到硬盘
    public void logRequest(TxnHeader hdr, Record txn) {
        Request request = new Request(null, hdr.getClientId(), hdr.getCxid(),
                hdr.getType(), null, null);
        request.hdr = hdr;
        request.txn = txn;
        request.zxid = hdr.getZxid();
        if ((request.zxid & 0xffffffffL) != 0) {
            pendingTxns.add(request);
        }
        syncProcessor.processRequest(request);
    }

    /**
      用来提交一个Proposal,这里必须按照pendingTxns的顺序来提交日志
     * When a COMMIT message is received, eventually this method is called, 
     * which matches up the zxid from the COMMIT with (hopefully) the head of
     * the pendingTxns queue and hands it to the commitProcessor to commit.
     * @param zxid - must correspond to the head of pendingTxns if it exists
     */
    public void commit(long zxid) {
        if (pendingTxns.size() == 0) {
            LOG.warn("Committing " + Long.toHexString(zxid)
                    + " without seeing txn");
            return;
        }
        long firstElementZxid = pendingTxns.element().zxid;
        if (firstElementZxid != zxid) {
            LOG.error("Committing zxid 0x" + Long.toHexString(zxid)
                    + " but next pending txn 0x"
                    + Long.toHexString(firstElementZxid));
            System.exit(12);
        }
        Request request = pendingTxns.remove();
        commitProcessor.commit(request);
    }
    
    synchronized public void sync(){
        if(pendingSyncs.size() ==0){
            LOG.warn("Not expecting a sync.");
            return;
        }
                
        Request r = pendingSyncs.remove();
		commitProcessor.commit(r);
    }
             
    @Override
    public int getGlobalOutstandingLimit() {
        return super.getGlobalOutstandingLimit() / (self.getQuorumSize() - 1);
    }
    
    @Override
    public void shutdown() {
        LOG.info("Shutting down");
        try {
            super.shutdown();
        } catch (Exception e) {
            LOG.warn("Ignoring unexpected exception during shutdown", e);
        }
        try {
            if (syncProcessor != null) {
                syncProcessor.shutdown();
            }
        } catch (Exception e) {
            LOG.warn("Ignoring unexpected exception in syncprocessor shutdown",
                    e);
        }
    }
    
   
 

![](http://7x00ae.com1.z0.glb.clouddn.com/18-2-22/4925301.jpg)
    


#### FollowerRequestProcessor   


FollowerRequestProcessor 是第一个业务处理器会把事务请求转发给Leader。这里的请求都是直接来自客户端。



     public void run() {
        try {
            while (!finished) {
                Request request = queuedRequests.take();
                if (LOG.isTraceEnabled()) {
                    ZooTrace.logRequest(LOG, ZooTrace.CLIENT_REQUEST_TRACE_MASK,
                            'F', request, "");
                }
                if (request == Request.requestOfDeath) {
                    break;
                }
                // We want to queue the request to be processed before we submit
                // the request to the leader so that we are ready to receive
                // the response
                nextProcessor.processRequest(request);
                
                //这里如果是写请求会转发给leader
                // We now ship the request to the leader. As with all
                // other quorum operations, sync also follows this code
                // path, but different from others, we need to keep track
                // of the sync operations this follower has pending, so we
                // add it to pendingSyncs.
                switch (request.type) {
                case OpCode.sync:
                    zks.pendingSyncs.add(request);
                    zks.getFollower().request(request);
                    break;
                case OpCode.create:
                case OpCode.delete:
                case OpCode.setData:
                case OpCode.setACL:
                case OpCode.createSession:
                case OpCode.closeSession:
                case OpCode.multi:
 					//转发请求给Leader
                    zks.getFollower().request(request);
                    break;
                }
            }
        } catch (Exception e) {
            handleException(this.getName(), e);
        }
        LOG.info("FollowerRequestProcessor exited loop!");
    }





Follower接收的消息都是来自Leader，如果Follower收到的消息类型是PROPOSAL类型，
然后会调用SyncRequestProcessor把事务请求同步到磁盘，然后调用SendAckRequestProcessor当Proposal被append到磁盘之后给leader发ack。

如果Follower收到的消息类型是COMMIT，也就是提交消息类型，会调用 CommitProcessor。

CommitProcessor处理消息后会交给FinalRequestProcessor。



#### CommitProcessor

以下是CommitProcessor的关键代码。Follower会把 commit消息加入到committedRequests列表中，然后CommitProcessor会从列表中拉取数据处理

    public CommitProcessor(RequestProcessor nextProcessor, String id,
            boolean matchSyncs, ZooKeeperServerListener listener) {
        super("CommitProcessor:" + id, listener);
        this.nextProcessor = nextProcessor;
        this.matchSyncs = matchSyncs;
    }

    volatile boolean finished = false;

    @Override
    public void run() {
        try {
            Request nextPending = null;            
            while (!finished) {
                //
                int len = toProcess.size();
                for (int i = 0; i < len; i++) {
                    //交给
                    nextProcessor.processRequest(toProcess.get(i));
                }
                toProcess.clear();
                //这里使用synchronized，当执行这里的写任务时，其他请求都会被阻塞。
                synchronized (this) {
                    //当需要处理的消息都为空时
                    if ((queuedRequests.size() == 0 || nextPending != null)
                            && committedRequests.size() == 0) {
                        wait();
                        continue;
                    }
                    // First check and see if the commit came in for the pending
                    // request
                    // 这里committedRequests 不为空的时候会处理
                    if ((queuedRequests.size() == 0 || nextPending != null)
                            && committedRequests.size() > 0) {
                        Request r = committedRequests.remove();
                        /*
                         * We match with nextPending so that we can move to the
                         * next request when it is committed. We also want to
                         * use nextPending because it has the cnxn member set
                         * properly.
                         */
                        if (nextPending != null
                                && nextPending.sessionId == r.sessionId
                                && nextPending.cxid == r.cxid) {
                            // we want to send our version of the request.
                            // the pointer to the connection in the request
                            nextPending.hdr = r.hdr;
                            nextPending.txn = r.txn;
                            nextPending.zxid = r.zxid;
                            toProcess.add(nextPending);
                            nextPending = null;
                        } else {
                            //Follower 的逻辑会走到这。
                            // this request came from someone else so just
                            // send the commit packet
                            toProcess.add(r);
                        }
                    }
                }

                // We haven't matched the pending requests, so go back to
                // waiting
                if (nextPending != null) {
                    continue;
                }
                //Follower 情况下nextPending一直为null，queuedRequests一直为空下面的逻辑不会走
                synchronized (this) {
                    // Process the next requests in the queuedRequests
                    while (nextPending == null && queuedRequests.size() > 0) {
                        Request request = queuedRequests.remove();
                        switch (request.type) {
                        case OpCode.create:
                        case OpCode.delete:
                        case OpCode.setData:
                        case OpCode.multi:
                        case OpCode.setACL:
                        case OpCode.createSession:
                        case OpCode.closeSession:
                            nextPending = request;
                            break;
                        case OpCode.sync:
                            if (matchSyncs) {
                                nextPending = request;
                            } else {
                                toProcess.add(request);
                            }
                            break;
                        default:
                            toProcess.add(request);
                        }
                    }
                }
            }
        } catch (InterruptedException e) {
            LOG.warn("Interrupted exception while waiting", e);
        } catch (Throwable e) {
            LOG.error("Unexpected exception causing CommitProcessor to exit", e);
        }
        LOG.info("CommitProcessor exited loop!");
    }

    synchronized public void commit(Request request) {
        if (!finished) {
            if (request == null) {
                LOG.warn("Committed a null!",
                         new Exception("committing a null! "));
                return;
            }
            if (LOG.isDebugEnabled()) {
                LOG.debug("Committing request:: " + request);
            }
            committedRequests.add(request);
            notifyAll();
        }
    }

    synchronized public void processRequest(Request request) {
        // request.addRQRec(">commit");
        if (LOG.isDebugEnabled()) {
            LOG.debug("Processing request:: " + request);
        }
        
        if (!finished) {
            queuedRequests.add(request);
            notifyAll();
        }
    }
    

最后由FinalRequestProcessor 做出相应的反应。





    当群首接收到一个新的写请求操作时， 直接地或通过其他追随者服务器来生成一个提议， 
    之后转发到追随者服务器。 当收到一个提议， 追随者服务器会发送这个提议到SyncRequestProcessor处理器，SendAckRequestProcessor会向群首发送确认消息。
     当群首服务器接收到足够确认消息来提交这个提议时， 群首就会发送提交事务消息给追随者（ 同
    时也会发送INFORM消息给观察者服务器）。 

    当接收到提交事务消息时， 追随者就通过CommitRequestProcessor处理器进行处理。
    为了保证执行的顺序，CommitRequestProcessor处理器会在收到一个写请求处理器时暂停后续的请求处理。 
	这就意味着， 在一个写请求之
    后接收到的任何读取请求都将被阻塞， 直到读取请求转给CommitRequestProcessor处理器。 通过等待的方式， 请求可以被保证按照接收的顺序来被执行。
    
    对于观察者服务器的请求流水线（ ObserverZooKeeperServer类） 与追随者服务器的流水线非常相似。 但是因为观察者服务器不需要确认提议消息， 因此观察者服务器并不需要发送确认消息给群首服务器， 也不用持久化事务到硬盘。


#### 总结

在节点变为Follow状态后，首先和leader节点联系，完成必要的数据同步，然后接收到leader的
uptodate指令后可以对客户端服服务。
