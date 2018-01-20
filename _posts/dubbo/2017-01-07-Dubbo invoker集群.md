---
layout: post
title: Dubbo Invoker集群
category: dubbo
tags:  dubbo Invoker，集群
description:  dubbo ，Invoker
--- 

Dubbo 的集群方式有下面几种，在dubbo config 配置Cluster="failfast" 即可以配置集群方式为failFast，dubbo默认集群方式是failOver.



-  failFast

快速失败，只发起一次调用，失败立即报错，通常用于非幂等性的写操作

- failOver

失败转移，当出现失败，重试其它服务器，通常用于读操作，但重试会带来更长延迟。

-  failBack

失败自动恢复，后台记录失败请求，定时重发，通常用于消息通知操作。
- failsafe

失败安全，出现异常时，直接忽略，通常用于写入审计日志等操作。
- forking

并行调用，只要一个成功即返回，通常用于实时性要求较高的操作，但需要浪费更多服务资源。 
- available
 
选择一个可用的Invoker调用

- broadcast

广播调用，所有提供逐个调用，任意一台报错则报错。通常用于更新提供方本地状态 速度慢，任意一台报错则报错 。 




####  生成Invoker

ReferenceConfig#createProxy



下面是创建代理的关键代码，主要是生成一个Invoker


            // 单个注册中心，直接refer 出invoker    
          if (urls.size() == 1) {
                invoker = refprotocol.refer(interfaceClass, urls.get(0));
            } else {
                // 
                List<Invoker<?>> invokers = new ArrayList<Invoker<?>>();
                URL registryURL = null;
                for (URL url : urls) {
                
                    // 这里生成Invoker 比较复杂，经过ProtocolFilterWrapper，ProtocolListenerWrapper，RegistryProtocol，RegitryDerictor
                    invokers.add(refprotocol.refer(interfaceClass, url));
                    if (Constants.REGISTRY_PROTOCOL.equals(url.getProtocol())) {
                        registryURL = url; // 用了最后一个registry url
                    }
                }
                if (registryURL != null) { // 有 注册中心协议的URL
                    // 对有注册中心的Cluster 只用 AvailableCluster
                    URL u = registryURL.addParameter(Constants.CLUSTER_KEY, AvailableCluster.NAME);
                    invoker = cluster.join(new StaticDirectory(u, invokers));
                } else { // 不是 注册中心的URL
                    invoker = cluster.join(new StaticDirectory(invokers));
                }
            }
        }




上述代码可以看到，根据注册中心的个数，dubbo会生成单个invoker和invoker集群。



##### 单个注册中心生成invoker


      invoker = refprotocol.refer(interfaceClass, urls.get(0));
      
单个注册中心使用Protobuf.refer 方法，这里refprotocol实例是有SPI机制生的。Protocol接口如下所示


    @SPI("dubbo")
public interface Protocol {

    /**
     * 获取缺省端口，当用户没有配置端口时使用。
     *
     * @return 缺省端口
     */
    int getDefaultPort();

    /**
     * 暴露远程服务：<br>
     * 1. 协议在接收请求时，应记录请求来源方地址信息：RpcContext.getContext().setRemoteAddress();<br>
     * 2. export()必须是幂等的，也就是暴露同一个URL的Invoker两次，和暴露一次没有区别。<br>
     * 3. export()传入的Invoker由框架实现并传入，协议不需要关心。<br>
     *
     * @param <T>     服务的类型
     * @param invoker 服务的执行体
     * @return exporter 暴露服务的引用，用于取消暴露
     * @throws RpcException 当暴露服务出错时抛出，比如端口已占用
     */
    @Adaptive
    <T> Exporter<T> export(Invoker<T> invoker) throws RpcException;

    /**
     * 引用远程服务：<br>
     * 1. 当用户调用refer()所返回的Invoker对象的invoke()方法时，协议需相应执行同URL远端export()传入的Invoker对象的invoke()方法。<br>
     * 2. refer()返回的Invoker由协议实现，协议通常需要在此Invoker中发送远程请求。<br>
     * 3. 当url中有设置check=false时，连接失败不能抛出异常，并内部自动恢复。<br>
     *
     * @param <T>  服务的类型
     * @param type 服务的类型
     * @param url  远程服务的URL地址
     * @return invoker 服务的本地代理
     * @throws RpcException 当连接服务提供方失败时抛出
     */
    @Adaptive
    <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException;

    /**
     * 释放协议：<br>
     * 1. 取消该协议所有已经暴露和引用的服务。<br>
     * 2. 释放协议所占用的所有资源，比如连接和端口。<br>
     * 3. 协议在释放后，依然能暴露和引用新的服务。<br>
     */
    void destroy();

}

 这里Url中的Protocol属性是Register，因为是通过注册中心获取服务引用，所以会生成RegisterProtocol实例，但是RegisterProtocol 实例会被ProtocolListenerWrapper 和ProtocolFilterWrapper 包装起来，所以这里实际上是 ProtocolListenerWrapper 实例。refer 方法的调用链如下
 
    ProtocolListenerWrapper=》ProtocolFilterWrapper=》RegisterProtocol

RegisterProtocol的关键refer方法如下:    

    
        
        //实现Protocol方法
        
      public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
        url = url.setProtocol(url.getParameter(Constants.REGISTRY_KEY, Constants.DEFAULT_REGISTRY)).removeParameter(Constants.REGISTRY_KEY);
        Registry registry = registryFactory.getRegistry(url);
        if (RegistryService.class.equals(type)) {
            return proxyFactory.getInvoker((T) registry, type, url);
        }

        // group="a,b" or group="*"
        Map<String, String> qs = StringUtils.parseQueryString(url.getParameterAndDecoded(Constants.REFER_KEY));
        String group = qs.get(Constants.GROUP_KEY);
        if (group != null && group.length() > 0) {
            if ((Constants.COMMA_SPLIT_PATTERN.split(group)).length > 1
                    || "*".equals(group)) {
                return doRefer(getMergeableCluster(), registry, type, url);
            }
        }
        return doRefer(cluster, registry, type, url);
    }

    private Cluster getMergeableCluster() {
        return ExtensionLoader.getExtensionLoader(Cluster.class).getExtension("mergeable");
    }

    private <T> Invoker<T> doRefer(Cluster cluster, Registry registry, Class<T> type, URL url) {
        // 生成directory对象，类似于zk节点的目录，该目标包括接口，方法，类，注册中心等信息，consumer和provider信息
        RegistryDirectory<T> directory = new RegistryDirectory<T>(type, url);
        directory.setRegistry(registry);
        directory.setProtocol(protocol);
        // REFER_KEY的所有属性
        Map<String, String> parameters = new HashMap<String, String>(directory.getUrl().getParameters());
        URL subscribeUrl = new URL(Constants.CONSUMER_PROTOCOL, parameters.remove(Constants.REGISTER_IP_KEY), 0, type.getName(), parameters);
        if (!Constants.ANY_VALUE.equals(url.getServiceInterface())
                && url.getParameter(Constants.REGISTER_KEY, true)) {
            registry.register(subscribeUrl.addParameters(Constants.CATEGORY_KEY, Constants.CONSUMERS_CATEGORY,
                    Constants.CHECK_KEY, String.valueOf(false)));
        }
        directory.subscribe(subscribeUrl.addParameter(Constants.CATEGORY_KEY,
                Constants.PROVIDERS_CATEGORY
                        + "," + Constants.CONFIGURATORS_CATEGORY
                        + "," + Constants.ROUTERS_CATEGORY));

        // 调用Cluster 获取invoker。或者用Directory生成invoker加入到集群中。 后面详细介绍
        Invoker invoker = cluster.join(directory);
    
        ProviderConsumerRegTable.registerConsuemr(invoker, url, subscribeUrl, directory);
        return invoker;
    }




##### 多个注册中心生成invoker

当有多个注册中心的时候，先为每个注册中心生成invoker，生成invoker列表。然后统一包装成一个invoker 返回。

在上面可以看到多个注册中心时候，会有如下代码 


      URL u = registryURL.addParameter(Constants.CLUSTER_KEY, AvailableCluster.NAME);

我们会获取到AvaliableCluster,但是AvaliableCluster会被包装成 MockClusterWrapper。 





#### Cluster



下面是Cluster接口和通过SPI生成的代码


    @SPI(FailoverCluster.NAME)
    public interface Cluster {

    /**
     * Merge the directory invokers to a virtual invoker.
     *
     * @param <T>
     * @param directory
     * @return cluster invoker
     * @throws RpcException
     */
    @Adaptive
    <T> Invoker<T> join(Directory<T> directory) throws RpcException;

    }




##### FailOverCluster

默认情况下是failOver

    public <T> Invoker<T> join(Directory<T> directory) throws RpcException {
        return new FailoverClusterInvoker<T>(directory);
    }


返回FailoverClusterInvoker 实例。

##### AvailableCluster


    public class AvailableCluster implements Cluster {

    public static final String NAME = "available";

    public <T> Invoker<T> join(Directory<T> directory) throws RpcException {

        return new AbstractClusterInvoker<T>(directory) {
            public Result doInvoke(Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
                for (Invoker<T> invoker : invokers) {
                    if (invoker.isAvailable()) {
                        return invoker.invoke(invocation);
                    }
                }
                throw new RpcException("No provider available in " + invokers);
            }
        };

    }

    }
    

    返回一个匿名内部类



##### MockClusterWrapper

Cluster会被包装成MockCluster，如果配置了mock方法，就直接返回mock方法的内容。 这里返回MockClusterInvoker。   

        
MockClusterInvoker的关键方法，invoker,发现存在mock，就直接返回了。

        public Result invoke(Invocation invocation) throws RpcException {
        Result result = null;

        String value = directory.getUrl().getMethodParameter(invocation.getMethodName(), Constants.MOCK_KEY, Boolean.FALSE.toString()).trim();
        if (value.length() == 0 || value.equalsIgnoreCase("false")) {
            //没有mock方法，继续调用。调用的FaillOverInvoker或者 AvailableCluster
            result = this.invoker.invoke(invocation);
        } else if (value.startsWith("force")) {
            if (logger.isWarnEnabled()) {
                logger.info("force-mock: " + invocation.getMethodName() + " force-mock enabled , url : " + directory.getUrl());
            }
            //force:direct mock
            result = doMockInvoke(invocation, null);
        } else {
            //fail-mock
            try {
                result = this.invoker.invoke(invocation);
            } catch (RpcException e) {
                if (e.isBiz()) {
                    throw e;
                } else {
                    if (logger.isWarnEnabled()) {
                        logger.info("fail-mock: " + invocation.getMethodName() + " fail-mock enabled , url : " + directory.getUrl(), e);
                    }
                    result = doMockInvoke(invocation, e);
                }
            }
        }
        return result;
    }

   
##### FailoverClusterInvoker

关键doInvoke 方法

    public Result doInvoke(Invocation invocation, final List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
        List<Invoker<T>> copyinvokers = invokers;
        checkInvokers(copyinvokers, invocation);
        int len = getUrl().getMethodParameter(invocation.getMethodName(), Constants.RETRIES_KEY, Constants.DEFAULT_RETRIES) + 1;
        if (len <= 0) {
            len = 1;
        }
        // retry loop.
        RpcException le = null; // last exception.
        // 获取invoker列表。
        List<Invoker<T>> invoked = new ArrayList<Invoker<T>>(copyinvokers.size()); // invoked invokers.
        Set<String> providers = new HashSet<String>(len);
        for (int i = 0; i < len; i++) {
            //重试时，进行重新选择，避免重试时invoker列表已发生变化.
            //注意：如果列表发生了变化，那么invoked判断会失效，因为invoker示例已经改变
            if (i > 0) {
                checkWhetherDestroyed();
                copyinvokers = list(invocation);
                //重新检查一下
                checkInvokers(copyinvokers, invocation);
            }
            // 负载均衡的实现，随机选择一个invoker
            Invoker<T> invoker = select(loadbalance, invocation, copyinvokers, invoked);
            invoked.add(invoker);
            RpcContext.getContext().setInvokers((List) invoked);
            try {
                Result result = invoker.invoke(invocation);
                if (le != null && logger.isWarnEnabled()) {
                    logger.warn("Although retry the method " + invocation.getMethodName()
                            + " in the service " + getInterface().getName()
                            + " was successful by the provider " + invoker.getUrl().getAddress()
                            + ", but there have been failed providers " + providers
                            + " (" + providers.size() + "/" + copyinvokers.size()
                            + ") from the registry " + directory.getUrl().getAddress()
                            + " on the consumer " + NetUtils.getLocalHost()
                            + " using the dubbo version " + Version.getVersion() + ". Last error is: "
                            + le.getMessage(), le);
                }
                return result;
            } catch (RpcException e) {
                if (e.isBiz()) { // biz exception.
                    throw e;
                }
                le = e;
            } catch (Throwable e) {
                le = new RpcException(e.getMessage(), e);
            } finally {
                providers.add(invoker.getUrl().getAddress());
            }
        }
        throw new RpcException(le != null ? le.getCode() : 0, "Failed to invoke the method "
                + invocation.getMethodName() + " in the service " + getInterface().getName()
                + ". Tried " + len + " times of the providers " + providers
                + " (" + providers.size() + "/" + copyinvokers.size()
                + ") from the registry " + directory.getUrl().getAddress()
                + " on the consumer " + NetUtils.getLocalHost() + " using the dubbo version "
                + Version.getVersion() + ". Last error is: "
                + (le != null ? le.getMessage() : ""), le != null && le.getCause() != null ? le.getCause() : le);
    }




##### AbstractClusterInvoker

   
   AbstractClusterInvoker 主要是做负载均衡，从多个invoker中选择一个出来。
    
    
     使用loadbalance选择invoker.</br>
     * a)先lb选择，如果在selected列表中 或者 不可用且做检验时，进入下一步(重选),否则直接返回</br>
     * b)重选验证规则：selected > available .保证重选出的结果尽量不在select中，并且是可用的
     *
     * @param availablecheck 如果设置true，在选择的时候先选invoker.available == true
     * @param selected       已选过的invoker.注意：输入保证不重复
     */
    protected Invoker<T> select(LoadBalance loadbalance, Invocation invocation, List<Invoker<T>> invokers, List<Invoker<T>> selected) throws RpcException {
        if (invokers == null || invokers.size() == 0)
            return null;
        String methodName = invocation == null ? "" : invocation.getMethodName();

        boolean sticky = invokers.get(0).getUrl().getMethodParameter(methodName, Constants.CLUSTER_STICKY_KEY, Constants.DEFAULT_CLUSTER_STICKY);
        {
            //ignore overloaded method
            if (stickyInvoker != null && !invokers.contains(stickyInvoker)) {
                stickyInvoker = null;
            }
            //ignore cucurrent problem
            if (sticky && stickyInvoker != null && (selected == null || !selected.contains(stickyInvoker))) {
                if (availablecheck && stickyInvoker.isAvailable()) {
                    return stickyInvoker;
                }
            }
        }
        Invoker<T> invoker = doselect(loadbalance, invocation, invokers, selected);

        if (sticky) {
            stickyInvoker = invoker;
        }
        return invoker;
    }

    private Invoker<T> doselect(LoadBalance loadbalance, Invocation invocation, List<Invoker<T>> invokers, List<Invoker<T>> selected) throws RpcException {
        if (invokers == null || invokers.size() == 0)
            return null;
        if (invokers.size() == 1)
            return invokers.get(0);
        // 如果只有两个invoker，退化成轮循
        if (invokers.size() == 2 && selected != null && selected.size() > 0) {
            return selected.get(0) == invokers.get(0) ? invokers.get(1) : invokers.get(0);
        }
        Invoker<T> invoker = loadbalance.select(invokers, getUrl(), invocation);

        //如果 selected中包含（优先判断） 或者 不可用&&availablecheck=true 则重试.
        if ((selected != null && selected.contains(invoker))
                || (!invoker.isAvailable() && getUrl() != null && availablecheck)) {
            try {
                Invoker<T> rinvoker = reselect(loadbalance, invocation, invokers, selected, availablecheck);
                if (rinvoker != null) {
                    invoker = rinvoker;
                } else {
                    //看下第一次选的位置，如果不是最后，选+1位置.
                    int index = invokers.indexOf(invoker);
                    try {
                        //最后在避免碰撞
                        invoker = index < invokers.size() - 1 ? invokers.get(index + 1) : invoker;
                    } catch (Exception e) {
                        logger.warn(e.getMessage() + " may because invokers list dynamic change, ignore.", e);
                    }
                }
            } catch (Throwable t) {
                logger.error("clustor relselect fail reason is :" + t.getMessage() + " if can not slove ,you can set cluster.availablecheck=false in url", t);
            }
        }
        return invoker;
    }




#### Directory

用来管理某个方法所有的invoker的类，类似于某个文件夹下所有的文件。比如一个接口下面多个方法。

    public interface Directory<T> extends Node {

    /**
     * get service type.
     *
     * @return service type.
     */
    Class<T> getInterface();

    /**
     * 获取当前ServiceType面所有的invoker，一个注册中心，invocation是调用参数，包括方法名称，参数等。
     *
     * @return invokers
     */
    List<Invoker<T>> list(Invocation invocation) throws RpcException;

}

##### RegisterDiretory

动态目录服务， 它的Invoker集合是从zk获取的， 它实现了NotifyListener接口实现了回调接口notify(List<Url>),刷新invoker。


    public class RegistryDirectory<T> extends AbstractDirectory<T> implements NotifyListener {
        public synchronized void notify(List<URL> urls) {}
        
        private void refreshInvoker(List<URL> invokerUrls)
    }



##### StatisticDirectory

 和RegisterDirecttory的区别是不会刷新，静态的目录服务。

#### 总结


dubbo 在解析<dubbo:reference> 标签类时候，生成代理类，代理实例化过程中根据urls(消费者信息) 参数invoker实例，并且根据dubbo cluster的配置值选用不同的集群方式，默认的集群方式是FailOver，然后通过通过Cluster.join方法生成Invoker集群，这样同一个service对外暴露一个Invoker，但是内部存在多个Invoker。通过调用的时候，选用不同的负载均和策略，选用invoker。   
   
      