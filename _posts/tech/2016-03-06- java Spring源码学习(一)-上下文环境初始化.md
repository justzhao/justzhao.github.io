---
layout: post
title: JAVA Spring 上下文环境初始化
category: 技术
tags: JAVA Spring  上下文环境初始化

---

### 介绍

Spring 是本人接触的最多的开源项目，里面多东西值得学习。

我们可以进入官网https://spring.io/ 可以有一些指导页面，然后进入 http://projects.spring.io/spring-framework/ 新建一个MAVEN工程直接DOWN 下。此过程可以下载到源码

		<dependencies>
	    <dependency>
	        <groupId>org.springframework</groupId>
	        <artifactId>spring-context</artifactId>
	        <version>4.2.5.RELEASE</version>
	    </dependency>
		</dependencies>
上面是pom.xml 依赖。

直接使用官网上的例子，下面是主要代码，其他的可以在官网查看
	
	@Configuration
	@ComponentScan  //自动检测扫描组件
	public class Application {
    @Bean
    MessageService mockMessageService() {
        return new MessageService() {
            public String getMessage() {
              return "Hello World!";
            }
        };
    }
	 public static void main(String[] args) {
      ApplicationContext context =  new AnnotationConfigApplicationContext(Application.class);
      MessagePrinter printer = context.getBean(MessagePrinter.class);
      
      printer.printMessage();
	}

	}

此官网的例子并不是从一个配置文件XML中加载上下文环境，而是使用注解的方式，加载Application类来返回一个上下文环境。下面详细介绍如何返回这个上下文环境ApplicationContext。
通过查看源码发现ApplicationContext 是实现了BeanFactory的接口，BeanFactory接口，故名思议，就是一个制造Bean的工厂，此接口有重要的getBean方法。AnnotationConfigApplicationContext是实现ApplicationContext的一个子类，有两个重要的私有变量。

	public class AnnotationConfigApplicationContext extends GenericApplicationContext implements AnnotationConfigRegistry {

	private final AnnotatedBeanDefinitionReader reader;//读取器

	private final ClassPathBeanDefinitionScanner scanner;//扫描器
	}
个人理解为读取器和扫描器。reader 会把当前传递过来的Application.class注册到AnnotatedBeanDefinitionReader类中。 调用AnnotatedBeanDefinitionReader的 registerBean最后完成所有的工作。

	public void registerBean(Class<?> annotatedClass, String name,
			@SuppressWarnings("unchecked") Class<? extends Annotation>... qualifiers) {
		AnnotatedGenericBeanDefinition abd = new AnnotatedGenericBeanDefinition(annotatedClass);//AnnotatedGenericBeanDefinition是Bean实例化，会使用classLoad来加载。说白了就是反射。Bean的定义和包装
		//判断Application上是否存在Conditional注解，如果不满足Conditional对应条件则该bean不被创建；
		if (this.conditionEvaluator.shouldSkip(abd.getMetadata())) {
			return;
		}

		ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(abd);
		abd.setScope(scopeMetadata.getScopeName());
		String beanName = (name != null ? name : this.beanNameGenerator.generateBeanName(abd, this.registry));
	   //处理一些其他注解
		AnnotationConfigUtils.processCommonDefinitionAnnotations(abd);
		if (qualifiers != null) {
			for (Class<? extends Annotation> qualifier : qualifiers) {
				if (Primary.class == qualifier) {
					abd.setPrimary(true);
				}
				else if (Lazy.class == qualifier) {
					abd.setLazyInit(true);
				}
				else {
					abd.addQualifier(new AutowireCandidateQualifier(qualifier));
				}
			}
		}

		BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(abd, beanName);
		definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
		BeanDefinitionReaderUtils.registerBeanDefinition(definitionHolder, this.registry);
	}

整个类结构图如下：

![](http://7x00ae.com1.z0.glb.clouddn.com/20160306blog.jpg)

此图忽略了许多继承关系，但是能清楚表现出在初始化上下文环境设计到的接口和类对象。
完成这个步骤后就可以去获取我们的Bean了。

### 理解

