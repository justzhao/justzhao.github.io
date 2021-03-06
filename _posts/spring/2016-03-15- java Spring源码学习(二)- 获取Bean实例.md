---
layout: post
title: Spring 源码学习(二) 获取Bean实例化
category: Spring
tags: JAVA Spring  获取Bean实例化

---

### 介绍

继续接上次文章的例子。

      ApplicationContext context =  new AnnotationConfigApplicationContext(Application.class);//获取山下文环境
      MessagePrinter printer = context.getBean(MessagePrinter.class);//对应的Bean实例化
      printer.printMessage();

在我们介绍好ApplicationContext上下文的环境生成之后，继续看context.getBean()是如何工作的。首先看一张图，当我们查看getBean()方法的具体实现的调用栈。
![](http://7x00ae.com1.z0.glb.clouddn.com/getBean%E7%9A%84%E8%B0%83%E7%94%A8%E6%A0%88.png)

我们可以看到DefaultListableBeanFactory 这个类。在看下面这张DefaultListableBeanFactory类关系图。
![](http://7x00ae.com1.z0.glb.clouddn.com/DefaultListableBeanFactory%E7%B1%BB%E5%85%B3%E7%B3%BB%E5%9B%BE.png)


DefaultListableBeanFactory继承很多父类和接口。这个类是默认实现了ListableBeanFactory和BeanDefinitionRegistry接口，可以看成一个标准的BeanFactory。如果需要定义自己的的BeanFactory则只要实现此类就可以了。

contex.getBean() 首先调AbstractApplicationContext中的getBean。获取此类的BeanFactory。而DefaultListableBeanFactory作为AbstractApplicationContext中的一个对象引用。随后会直接调用DefaultListableBeanFactory中的getBean方法。getBean方法中，可以获取传进来类路的的类名字。最终会调用到AbstractBeanFactory的doGetBean方法。在这里因为AbstractBeanFactory是DefaultListableBeanFactory的抽象父类。

下面是DefaultListableBeanFactory对应的getBean的方法


		public <T> T getBean(Class<T> requiredType, Object... args) throws BeansException {
		Assert.notNull(requiredType, "Required type must not be null");
		//根据类路径获取到类的名字
		String[] beanNames = getBeanNamesForType(requiredType);
         //根据参数个数不同的情况。
		if (beanNames.length > 1) {
			ArrayList<String> autowireCandidates = new ArrayList<String>();
			for (String beanName : beanNames) {
				if (!containsBeanDefinition(beanName) || getBeanDefinition(beanName).isAutowireCandidate()) {
					autowireCandidates.add(beanName);
				}
			}
			if (autowireCandidates.size() > 0) {
				beanNames = autowireCandidates.toArray(new String[autowireCandidates.size()]);
			}
		}
		if (beanNames.length == 1) {
 			//此个例子中，我们调用这里的方法，将会调用AbstractBeanFactory的getBean()方法，最终会调用到doGetBean()方法
			return getBean(beanNames[0], requiredType, args);
		}
		else if (beanNames.length > 1) {
			Map<String, Object> candidates = new HashMap<String, Object>();
			for (String beanName : beanNames) {
				candidates.put(beanName, getBean(beanName, requiredType, args));
			}
			String primaryCandidate = determinePrimaryCandidate(candidates, requiredType);
			if (primaryCandidate != null) {
				return getBean(primaryCandidate, requiredType, args);
			}
			String priorityCandidate = determineHighestPriorityCandidate(candidates, requiredType);
			if (priorityCandidate != null) {
				return getBean(priorityCandidate, requiredType, args);
			}
			throw new NoUniqueBeanDefinitionException(requiredType, candidates.keySet());
		}
		else if (getParentBeanFactory() != null) {
			return getParentBeanFactory().getBean(requiredType, args);
		}
		else {
			throw new NoSuchBeanDefinitionException(requiredType);
		}
	}



在将会调用AbstractBeanFactory的doGetBean()方法。在doGetBean()方法中，会有

		Object sharedInstance = getSingleton(beanName);
		bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
		bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
这些方法在不同的条件下生成实例。最后返回实例。

### 理解

生成Bean的过程涉及到的类比较多，主要是抓住DefaultListableBeanFactory这个类。