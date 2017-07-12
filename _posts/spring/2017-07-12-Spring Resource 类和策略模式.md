---
layout: post
title: Spring  Resource接口
category: Spring
tags: JAVA Spring Resource接口  略模式 

---


### Spring 资源文件管理

在Spring中改进了JAVA 对资源文件的管理，不在使用URL类。

#### Resource 类实现资源管理

Spring 的资源文件统一使用Resource 接口来管理。Resource 接口如图所示,并且提供如下方法



    public interface Resource extends InputStreamSource {
    
    
    	boolean exists();
    
    
    	boolean isReadable();
    
    
    	boolean isOpen();
    
    
    	URL getURL() throws IOException;
    
    
    	URI getURI() throws IOException;
    
    
    	File getFile() throws IOException;
    
    
    	long contentLength() throws IOException;
    
    	
    	long lastModified() throws IOException;
    
    
    	Resource createRelative(String relativePath) throws IOException;
    
    
    	String getFilename();
    
    
    	String getDescription();
    
    }

在spring 中Resource本身不实现任何资源访问的逻辑方法，它有不同的实现类针对不同的资源负责不同的资源访问。如下图所示：
![](http://7x00ae.com1.z0.glb.clouddn.com/16-8-2/22451155.jpg)

在图中我们可以看到 Resource的三个实现。


    class AbstractResource
    
    interface ContextResource
    
    interface WritableResource


在这三个类中 我们常见的资源访问子类如下：
- UrlResource：访问网络资源的实现类。
- ClassPathResource：访问类加载路径里资源的实现类。
- FileSystemResource：访问文件系统里资源的实现类。
- ServletContextResource：访问相对于 ServletContext 路径里的资源的实现类：
- InputStreamResource：访问输入流资源的实现类。
- ByteArrayResource：访问字节数组资源的实现类

这些类提供对不同的资源文件的逻辑访问。
以下是一些常见的用法。



 UrlResource url = new UrlResource("file:book.xml"); //我们可以使用file前缀访问本地文件，也可以使用http访问网络资源。
 
    ClassPathResource resource = new ClassPathResource("book.xml");//使用类路径加载资源
    FileSystemResource fr = new FileSystemResource("book.xml"); //使用本地文件系统加载资源。

以上子类和Resource类的关系图如下
![](http://7x00ae.com1.z0.glb.clouddn.com/16-8-2/56784692.jpg)


在Spring 中 ，这些类都实现了一种策略模式。下面将介绍策略模式。

#### 策略模式

策略模式简单的说就是实现某一个功能有多种算法或者策略，我们可以根据环境或者条件的不同选择不同的算法或者策略来完成该功能。   
也就是说我们不能子啊外部选择不同的算法，需要从内部实现。

##### 商品出售的例子
假设某个商品出售一批不同折扣的商品，比如旧货是0.5，新货是0.8。如此我们可以定义如下：

###### 折扣计算的接口

     public interface DiscountStrategy 
     { 
     //定义一个用于计算打折价的方法
     double getDiscount(double originPrice); 
     }

分别有两种实现类

旧货折扣实现

     // 实现 DiscountStrategy 接口，实现对 旧货 打折的算法
     public class OldDiscount 
     implements DiscountStrategy 
     { 
        // 重写 getDiscount() 方法，提供 VIP 打折算法
         public double getDiscount(double originPrice) 
         { 
             System.out.println("使用 旧货 折扣 ..."); 
             return originPrice * 0.5; 
         } 
     }

新货折扣实现


     // 实现 DiscountStrategy 接口，实现对 旧货 打折的算法
     public class newDiscount 
     implements DiscountStrategy 
     { 
        // 重写 getDiscount() 方法，提供 VIP 打折算法
         public double getDiscount(double originPrice) 
         { 
             System.out.println("使用 新货 折扣 ..."); 
             return originPrice * 0.8; 
         } 
     }


下面是我们的商品类DiscountContext，也就是我们的出售环境


     public class DiscountContext 
     { 
         // 组合一个 DiscountStrategy 对象
         private DiscountStrategy strategy; 
         // 构造器，传入一个 DiscountStrategy 对象
         public DiscountContext(DiscountStrategy strategy) 
         { 
             this.strategy  = strategy; 
         } 
         // 根据实际所使用的 DiscountStrategy 对象得到折扣价
         public double getDiscountPrice(double price) 
         { 
             // 如果 strategy 为 null，系统自动选择 OldDiscount 类
             if (strategy == null)
             {
                strategy = new OldDiscount();
             }
              return this.strategy.getDiscount(price);
         } 
         // 提供切换算法的方法
         public void changeDiscount(DiscountStrategy strategy) 
         { 
         this.strategy = strategy; 
         } 
     }


上述代码中，我们可以看到商品默认折扣策略是旧货折扣，当我们改变策略的时候需要执行如下代码：


    //修改策略
     changeDiscount(new newDiscount());
     

这样我们在外部环境修改了策略模式。当我们新增一种折扣模式的时候，需要新建一个类实现DiscountStrategy接口。

使用策略模式可以让客户端代码在不同的打折策略之间切换，但也有一个小小的遗憾：客户端代码需要和不同的策略类耦合。
为了弥补这个不足，我们可以考虑使用配置文件来指定 DiscountContext 使用哪种打折策略——这就彻底分离客户端代码和具体打折策略——这正好是 Spring 框架的强项，Spring 框架采用配置文件来管理 Bean，当然也可以管理资源。        
这时候回到Spring是如何使用策略模式访问资源的。

#### Spring 策略模式 访问资源

在介绍Spring 策略模式访问资源之前，必须介绍接口类如下：
##### ResourceLoader和ResourceLoaderAware接口


    public interface ResourceLoader {
    
    
    	String CLASSPATH_URL_PREFIX = ResourceUtils.CLASSPATH_URL_PREFIX;
    	Resource getResource(String location); //这个方法返回一个Resource 资源对象。
    	ClassLoader getClassLoader();
    
    }


Spring 中的ApplicationContext  类实现了ResourceLoader接口，所以能调用getResource方法获得资源对象，代码如下：


    Resource resource=context.getResource("book.xml");//ApplicationContext 也就是spring容器就能管理我们的资源了

在如上代码中，getResource的访问资源的策略由context环境所决定。也就是说有如下情况：
- 如果 ApplicationContext 是 FileSystemXmlApplicationContext，resource 就是 FileSystemResource 实例。
- 如果 ApplicationContext 是 ClassPathXmlApplicationContext，resource 就是 ClassPathResource 实例。
- 如果 ApplicationContext 是 XmlWebApplicationContext，resource 是 ServletContextResource 实例。

可以查看如下例子：

    
            System.out.println("FileSystemXmlApplicationContext");
            
            ApplicationContext context = new FileSystemXmlApplicationContext("E:\\workspace\\spring\\src\\main\\resources\\ApplicationContext.xml");
            MessagePrinter printer = (MessagePrinter) context.getBean("messagePrinter");
            printer.printMessage();
    
            Resource fileres = context.getResource("book.xml");
            // 获取该资源的简单信息
            System.out.println(fileres.getFilename());
            System.out.println(fileres.getDescription());
    
            
            System.out.println("ClassPathXmlApplicationContext");
            ApplicationContext ctx = new ClassPathXmlApplicationContext("ApplicationContext.xml");
            MessagePrinter mp = (MessagePrinter) ctx.getBean("messagePrinter");
    
            mp.printMessage();
            
            Resource classres = ctx.getResource("book.xml");
            // 获取该资源的简单信息
            System.out.println(classres.getFilename());
            System.out.println(classres.getDescription());

这个例子的输出结果如下：


    FileSystemXmlApplicationContext
    file [E:\workspace\spring\src\main\resources\ApplicationContext.xml]
    Hello World!!
    book.xml
    file [E:\workspace\spring\book.xml]
    ClassPathXmlApplicationContext
    
    Hello World!!
    book.xml
    class path resource [book.xml]

由图中，我们可以看到当ApplicationContext的实现不同的时候，Spring 会使用不同的策略来访问当前的资源环境。

此外，Spring访问资源文件时候，可以强行使用前缀来制定访问策略，比如：


            ApplicationContext context = new FileSystemXmlApplicationContext("E:\\workspace\\spring\\src\\main\\resources\\ApplicationContext.xml");
      Resource fileres = context.getResource("book.xml");
      System.out.println(fileres.getFilename());
      System.out.println(fileres.getDescription());

可以发现输出结果如下


    class path resource [book.xml]
    ClassPathXmlApplicationContext


我们使用前缀指定访问策略只会对当前访问有效。

以下是常见前缀及对应的访问策略：
- classpath:以 ClassPathResource 实例来访问类路径里的资源。
- file:以 UrlResource 实例访问本地文件系统的资源。
- http:以 UrlResource 实例访问基于 HTTP 协议的网络资源。
- 无前缀:由于 ApplicationContext 的实现类来决定访问策略。


ResourceLoaderAware 接口则用于指定该接口的实现类必须持有一个 ResourceLoader 实例。类似于 Spring 提供的 BeanFactoryAware、BeanNameAware 接口，ResourceLoaderAware 接口也提供了一个 setResourceLoader() 方法，该方法将由 Spring 容器负责调用，Spring 容器会将一个ResourceLoader对象作为该方法的参数传入。当我们把将 ResourceLoaderAware 实例部署在 Spring 容器中后，Spring 容器会将自身当成 ResourceLoader 作为 setResourceLoader() 方法的参数传入，由于 ApplicationContext 的实现类都实现了 ResourceLoader 接口，Spring 容器自身完全可作为 ResourceLoader 使用。
关于ResourceLoaderAware 看下面的一个例子，我们将上面的MessagePrinter 实现ResourceLoaderAware接口：


    public interface ResourceLoader {
    
    	/** Pseudo URL prefix for loading from the class path: "classpath:" */
    	String CLASSPATH_URL_PREFIX = ResourceUtils.CLASSPATH_URL_PREFIX;
    
         //获取Resource资源对象
    	Resource getResource(String location);
    
    
    	ClassLoader getClassLoader();
    
    }

    
    public class MessagePrinter implements ResourceLoaderAware {
      
        //实现了ResourceLoaderAware 接口，我们可以在这里获取Resource对象
        public void setResourceLoader(ResourceLoader resourceLoader) {
            // TODO Auto-generated method stub
            //resourceLoader.getResource()
             System.out.println(resourceLoader.toString());
            
        }
      
       
    }

接下来我们运行例子，可以发现多了输出信息

    
    FileSystemXmlApplicationContex
    //得到当前Spring Context环境
    org.springframework.context.support.FileSystemXmlApplicationContext@5a6b258d: startup date [Wed Aug 03 10:20:13 CST 2016]; root of context hierarchy
    Hello World!!
    book.xml
    file [E:\workspace\spring\book.xml]
    
    ClassPathXmlApplicationContext
    //得到当前Spring Context环境
    org.springframework.context.support.ClassPathXmlApplicationContext@63072ac9: startup date [Wed Aug 03 10:20:13 CST 2016]; root of context hierarchy
    Hello World!!
    book.xml
    class path resource [book.xml]

##### Spring 使用Resource 作为属性

前文介绍了 Spring 提供的资源访问策略，但这些依赖访问策略要么需要使用 Resource 实现类，要么需要使用 ApplicationContext 来获取资源。实际上，当应用程序中的 Bean 实例需要访问资源时，Spring 有更好的解决方法：直接利用依赖注入。
前文中解释了Spring使用策略模式来访问资源文件，而且它能使用IOC容器来管理资源文件。也就是如下两种方案。
1. 代码中获取 Resource 实例。
2. 使用依赖注入。

对于第一种方案，很明显，需要在代码中指定资源文件的位置，当文件名字改变或者资源文件位置变化了，都需要去修改代码，具有很强的耦合性。

对于第二种方案，使用依赖注入，如果我们能在程序的外部来指定资源位置，让 Spring 为 Bean 实例依赖注入资源。减少耦合性。我们对上面的代码做如下修改。


    
    public class MessagePrinter implements ResourceLoaderAware {
    
        //新增 Resource 属性
        private Resource resource;
        
        public Resource getResource() {
            return resource;
        }
    
        public void setResource(Resource resource) {
            this.resource = resource;
        }
    
    }
    
    // 修改bean.xml
    
       <bean id="messagePrinter" class="com.zhaopeng.action.MessagePrinter">
         <!--依赖注入资源文件-->
         <property name="resource" value="classpath:book.xml"></property>
        
        </bean>
        
    //main函数
    ApplicationContext ctx = new ClassPathXmlApplicationContext("ApplicationContext.xml");
            MessagePrinter mp = (MessagePrinter) ctx.getBean("messagePrinter");
    
    System.out.println("使用依赖注入管理资源文件");
    Resource iocres = mp.getResource();
    System.out.println(iocres.getFilename());
    System.out.println(iocres.getDescription());
    

最后也能同样的输出资源文件信息。

#### 小结

总结了Spring 中 Resource接口和对应的实现子类。了解了设计模式中的策略模式，Spring是如何利用策略模式访问资源文件，加上后来的依赖注入访问资源文件实现解耦。体现了Spring的核心思想 容器解耦。















