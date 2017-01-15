---
layout: post
title: Dubbo SPI机制
category: dubbo
tags: Dubbo SPI机制

---

#### SPI机制

SPI机制：全名为Service Provider Interface

JAVA接口定义许多抽象方法的集合(扩展点)，可以理解为服务高度的抽象，此时某个高度抽象的服务可能有许多不同的实现，不同的公司提供同一种服务，但是实现不同。这时候，在类路径的META-INF/services/目录里同时创建一个以服务接口命名的文件，文件里面填充的内容就是该服务接口实现类的路径。
JDK能够运行java.util.ServiceLoader 来加载所有服务的实现，其实是一种服务发现机制。

JAVA中的SPI机制存在一些问题

- JDK标准SPI会一次性实例化扩展点所有的实现，如果有些扩展点的实现很耗时，系统没用上某个实现也会实例化。
- 如果扩展点加载失败，可能无法找到原因
- 无法支持一个扩展点中的属性是另外一个扩展点。


Dubbo SPI机制是在JDK的SPI机制发展而来，也解决了上述的问题。


#### Dubbo SPI机制


在Dubbo中主要使用com.alibaba.dubbo.common.extension 包下的类实现SPI机制，
常用的有注解SPI，Adaptive，Activate以及类ExtensionLoader

在Dubbo 中，使用SPI和Adaptive注解，Dubbo的SPI机制会自动生成和编译一个动态的Adpative类。


##### ExtensionLoader类

在Dubbo中有许多被SPI注解标注的接口(扩展点)，这时候可以使用ExtensionLoader的方法拿到对应接口实现类的动态代理类(动态的Adpative类，适配器类)。
    
    //返回一个动态扩展代理类
    public T getAdaptiveExtension() {}
 
    //返回一组动态扩展代理类
    public List<T> getActivateExtension(URL url, String key) {}


在Dubbo中，获得适配器类的方式有两种:

- 静态代码形式的默认适配器。这些类会被Adaptive注解修饰，且一个接口只能有一个这样的静态适配器。这种形式仅应用于一些特殊的接口，如：AdaptiveCompiler、AdaptiveExtensionFactory这两个适配器，ExtensionLoader需要依赖它们来工作，所以使用了这种特殊的构建方式。

- 动态代码适配器。实际上其余的接口都是使用动态适配器，ExtensionLoader根据接口定义动态生成一段适配器代码，并构建这个动态类的实例。这个时候接口中的一些方法具有Adaptive标记，它提供了一些用于查找具体Extension的key，如果这些方法中有URL类型的参数，则会依次在url中查找这些key对应的value，再以此为name确定要使用的Extension。如果没有从url中找到该参数，则会使用SPI注解中的默认值name进行构建。

##### @SPI

应用在接口上，用户加载指定的扩展点的实现    

    @Documented
    @Retention(RetentionPolicy.RUNTIME)
    @Target({ElementType.TYPE})
    public @interface SPI {
    
        /**
         * 缺省扩展点名。
         */
    	String value() default "";
    
    }
    

    
##### @Adaptive


应用在接口的方法上，可以指定多个值，{"key1", "key2"}


    @Documented
    @Retention(RetentionPolicy.RUNTIME)
    @Target({ElementType.TYPE, ElementType.METHOD})
    public @interface Adaptive {
    
    /**
     * 从{@link URL}的Key名，对应的Value作为要Adapt成的Extension名。
     * <p>
     * 如果{@link URL}这些Key都没有Value，使用 用 缺省的扩展（在接口的{@link SPI}中设定的值）。<br>
     * 比如，<code>String[] {"key1", "key2"}</code>，表示
     * <ol>
     * <li>先在URL上找key1的Value作为要Adapt成的Extension名；
     * <li>key1没有Value，则使用key2的Value作为要Adapt成的Extension名。
     * <li>key2没有Value，使用缺省的扩展。
     * <li>如果没有设定缺省扩展，则方法调用会抛出{@link IllegalStateException}。
     * </ol>
     * <p>
     * 如果不设置则缺省使用Extension接口类名的点分隔小写字串。<br>
     * 即对于Extension接口{@code                 com.alibaba.dubbo.xxx.YyyInvokerWrapper}的缺省值为<code>String[] {"yyy.invoker.wrapper"}</code>
     * 
     * @see SPI#value()
     */
    String[] value() default {};
    
    }


##### @Activate

Activate 注解用于在某个条件下扩展点的实现类才被激活。

一个接口有多个实现类，Activate标记在每个实现类的type上，并注明“何时被激活”的条件，如group和key的信息，以及在所有被激活实现类中的排序信息。通过ExtensionLoader的getActivateExtension可以获取到被激活实现类构成的List。


    @Documented
    @Retention(RetentionPolicy.RUNTIME)
    @Target({ElementType.TYPE, ElementType.METHOD})
    public @interface Activate {
    /**
     * Group过滤条件。
     * <br />
     * 包含{@link ExtensionLoader#getActivateExtension}的group参数给的值，则返回扩展。
     * <br />
     * 如没有Group设置，则不过滤。
     */
    String[] group() default {};

    /**
     * Key过滤条件。包含{@link ExtensionLoader#getActivateExtension}的URL的参数Key中有，则返回扩展。
     * <p />
     * 示例：<br/>
     * 注解的值 <code>@Activate("cache,validatioin")</code>，
     * 则{@link ExtensionLoader#getActivateExtension}的URL的参数有<code>cache</code>Key，或是<code>validatioin</code>则返回扩展。
     * <br/>
     * 如没有设置，则不过滤。
     */
    String[] value() default {};

    /**
     * 排序信息，可以不提供。
     */
    String[] before() default {};

    /**
     * 排序信息，可以不提供。
     */
    String[] after() default {};

    /**
     * 排序信息，可以不提供。
     */
    int order() default 0;
    }


#### 例子
以Dubbo启动时候，启动netty服务为例子。


Spring解析ServiceBean 时候，调用onApplicationEvent方法，对中调用到Transporters中的

    public static Transporter getTransporter() {
        return ExtensionLoader.getExtensionLoader(Transporter.class).getAdaptiveExtension();
    }

来启动底层通信服务(Netty).

Transporter 接口的源码如下：

    @SPI("netty")
    public interface Transporter {

    /**
     * Bind a server.
     * 
     * @see com.alibaba.dubbo.remoting.Transporters#bind(URL, Receiver, ChannelHandler)
     * @param url server url
     * @param handler
     * @return server
     * @throws RemotingException 
     */
    @Adaptive({Constants.SERVER_KEY, Constants.TRANSPORTER_KEY})
    Server bind(URL url, ChannelHandler handler) throws RemotingException;

    /**
     * Connect to a server.
     * 
     * @see com.alibaba.dubbo.remoting.Transporters#connect(URL, Receiver, ChannelListener)
     * @param url server url
     * @param handler
     * @return client
     * @throws RemotingException 
     */
    @Adaptive({Constants.CLIENT_KEY, Constants.TRANSPORTER_KEY})
    Client connect(URL url, ChannelHandler handler) throws RemotingException;

    }
    
    
@SPI("netty") 注解表示默认使用NettyServer。dubbo会去加载类路径meta-inf/dubbo/internal下的配置文件。可以看到com.alibaba.dubbo.remoting.Transporter配置文件，里面的内容如下：

    netty=com.alibaba.dubbo.remoting.transport.netty.NettyTransporter
    mina=com.alibaba.dubbo.remoting.transport.mina.MinaTransporter
    grizzly=com.alibaba.dubbo.remoting.transport.grizzly.GrizzlyTransporter

可以看到dubbo底层通信默认有三种实现。


Server bind()方法会具体的去启动一个服务。他上面有如下注解

    @Adaptive({Constants.SERVER_KEY, Constants.TRANSPORTER_KEY})

bind方法具有Adaptive标记，所以在生成动态Adapter类的时候，该方法会根据运行时的参数决定使用那个具体的实现类，也就是会去尝试启动两个参数对应的扩展点实现类。

Dubbo在加载此处代码的时候会，生成类似如下代码。

    public class Transporter$Adaptive implements Transporter {
 
    public Server bind(URL arg0, ChannelHandler arg1) throws RemotingException {
        URL url = arg0;
        String extName = url.getParameter("server",
                url.getParameter("transporter", "netty"));
        Transporter extension = (Transporter) ExtensionLoader
                .getExtensionLoader(Transporter.class).getExtension(extName);
        return extension.bind(arg0, arg1);
    }
 
    }


在适配器的方法中，首先要确定具体实现类的类型extName。当方法中有URL类型的参数时，这个配置会从url参数中获取。Adaptive注解中的两个key即是在url查找的key，分别为server和transporter。当两个key都不存在时，使用SPI注解中得默认值netty。在确定了extName后，就可以从事先加载好的实例中获取extension对象，最终将方法的调用传递给extension的对应方法。这样最后能启动到NettyServer。

如图
![image](http://7x00ae.com1.z0.glb.clouddn.com/dubbo%20spi.png)

    



#### 总结

在Dubbo 中有学多使用SPI加载的扩展点，比如 Filterr，Protocol 等，使用SPI机制的优点是方便热拔插，自定义自己的实现的时候不会入侵代码。





