---
layout: post
title: Spring AOP原理
category: Spring
tags: JAVA，Sping AOP

---

#### AOP细节

经常可以看到在spring的配置文件中配置  <aop:aspectj-autoproxy/> 开启aop，其实是开启动态代理，为所需要的一些bean植入一段业务逻辑(切面)。
    
        此外，EnableAspectJAutoProxy 和 <aop:aspectj-autoproxy/> 效果一样，
    开启自动植入 aop增强
    
  
     

#### 配置文件     
     
     <aop:aspectj-autoproxy/>
    <bean id="orderService" class="com.zhaopeng.service.consumer.OrderService">
    </bean>
    <bean class="com.zhaopeng.service.consumer.log.LogService"/>
    <bean class="com.zhaopeng.service.consumer.biz.Pay" />
     
     
     
       public static void main(String args[]) throws IOException {

        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext("ApplicationContext.xml");
        context.start();
          // 返回的是Enhancer
        OrderService orderService = (OrderService) context.getBean("orderService");
        orderService.processOrder(1);
    }
     
  
 如果一个bean被植入了一段逻辑，则称为被增强。    
 
- 切面自己不会被增强。
- 某个类不能被增加的正则匹配到，则不会被增强
 
 
### 实现细节 
 
当配置文件中写有<aop:aspectj-autoproxy/>的时候，会自动开启aop。是spring会解析<aop>标签，加载相关的类，对指定的bean创建动态代理，植入逻辑。

如果aop 的属性 proxy-target-class="true" 则表示使用cglib的方式实现动态代理，否则使用jdk的动态代理。



#### AopNamespaceHandler


AopNamespaceHandler 用来解析标签<aop:>.实现自动创建AOP代理，初始化AOP工具类。


    public class AopNamespaceHandler extends NamespaceHandlerSupport {
    public AopNamespaceHandler() {
    }

    public void init() {
        this.registerBeanDefinitionParser("config", new ConfigBeanDefinitionParser());
        this.registerBeanDefinitionParser("aspectj-autoproxy", new AspectJAutoProxyBeanDefinitionParser());
        this.registerBeanDefinitionDecorator("scoped-proxy", new ScopedProxyBeanDefinitionDecorator());
        this.registerBeanDefinitionParser("spring-configured", new SpringConfiguredBeanDefinitionParser());
    }
 } 
 
 
#####  AspectJAutoProxyBeanDefinitionParser

用来解析 <aop:aspectj-autoproxy/>标签，注册一个name=org.springframework.aop.config.internalAutoProxyCreator 类为AnnotationAwareAspectJAutoProxyCreator的
beanDefinition

##### AnnotationAwareAspectJAutoProxyCreator
类结构如下:

![](http://7x00ae.com1.z0.glb.clouddn.com/17-12-10/17896669.jpg)


注意这个类实现了BeanPostProcessor 接口，所以在相关的bean在初始前后会调用对应的postProcessBeforeInitialization和postProcessAfterInitialization方法




postProcessAfterInitialization 方法。如果需要进行动态代理增强，就会创建代理

    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		if (bean != null) {
			Object cacheKey = getCacheKey(bean.getClass(), beanName);
			if (!this.earlyProxyReferences.contains(cacheKey)) {
				return wrapIfNecessary(bean, beanName, cacheKey);
			}
		}
		return bean;
	}







spring启动自动植入AOP之后，会在实例化每个bean的时候，发现需要植入代理的时候就会执行相关操作。

#### bean的实例化


DefaultListableBeanFactory#preInstantiateSingletons 方法，获取到所有的beanNames ，根据beanName调用getBean(beanName)来实例化对象，按照如下顺序调用。


调用到 AbstractBeanFactory#doGetBean。

调用到 DefaultSingletonBeanRegistry#getSingleton

调用到 AbstractAutowireCapableBeanFactory#createBean

调用到 AbstractAutowireCapableBeanFactory#doCreateBean

调用到 AbstractAutowireCapableBeanFactory#initializeBean

调用到 AbstractAutowireCapableBeanFactory#applyBeanPostProcessorsBeforeInitialization和 applyBeanPostProcessorsAfterInitialization

调用到 AbstractAutoProxyCreator#postProcessAfterInitialization 如果符合条件，会创建代理。

调用到   AbstractAutoProxyCreator#wrapIfNecessary

调用到  DefaultAopProxyFactory#isInfrastructureClass


调用到   AspectJAwareAdvisorAutoProxyCreator#shouldSkip

调用到   AbstractAdvisorAutoProxyCreator#findEligibleAdvisors

调用到  AbstractAutoProxyCreator#createProxy


调用到 AbstractAutoProxyCreator#buildAdvisors


调用到 BeanFactoryAspectJAdvisorsBuilder#buildAspectJAdvisors

调用到 ReflectiveAspectJAdvisorFactory#getAdvice

调用到 DefaultAopProxyFactory#createAopProxy


可以看到在AbstractAutoProxyCreator 调用createProxy，需要buildAdvisors。在这里先说说advisor的概念

######  Advisor：充当Advice和Pointcut的适配器，类似使用Aspect的@Aspect注解的类。一般有advice和pointcut属性




###### Advice：拦截器，拦截行为。

buildAdvisors 上是获取该方法需要增强的拦截器(intercept)， 获取的Advisor列表的顺序是由要求的，入下面顺序所示。
 
     Advisor 排序：AfterThroing，AfterReturn，after，around，before。


![](http://7x00ae.com1.z0.glb.clouddn.com/17-12-16/26014201.jpg)


当逻辑执行到指定的切点，会依次调用如上的advisor。


获取advice的关键代码如下：



     	@Override
	public Advice getAdvice(Method candidateAdviceMethod, AspectJExpressionPointcut expressionPointcut,
			MetadataAwareAspectInstanceFactory aspectInstanceFactory, int declarationOrder, String aspectName) {

		Class<?> candidateAspectClass = aspectInstanceFactory.getAspectMetadata().getAspectClass();
		validate(candidateAspectClass);

		AspectJAnnotation<?> aspectJAnnotation =
				AbstractAspectJAdvisorFactory.findAspectJAnnotationOnMethod(candidateAdviceMethod);
		if (aspectJAnnotation == null) {
			return null;
		}

		// If we get here, we know we have an AspectJ method.
		// Check that it's an AspectJ-annotated class
		if (!isAspect(candidateAspectClass)) {
			throw new AopConfigException("Advice must be declared inside an aspect type: " +
					"Offending method '" + candidateAdviceMethod + "' in class [" +
					candidateAspectClass.getName() + "]");
		}

		if (logger.isDebugEnabled()) {
			logger.debug("Found AspectJ method: " + candidateAdviceMethod);
		}

		AbstractAspectJAdvice springAdvice;

		switch (aspectJAnnotation.getAnnotationType()) {
			case AtBefore:
				springAdvice = new AspectJMethodBeforeAdvice(
						candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
				break;
			case AtAfter:
				springAdvice = new AspectJAfterAdvice(
						candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
				break;
			case AtAfterReturning:
				springAdvice = new AspectJAfterReturningAdvice(
						candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
				AfterReturning afterReturningAnnotation = (AfterReturning) aspectJAnnotation.getAnnotation();
				if (StringUtils.hasText(afterReturningAnnotation.returning())) {
					springAdvice.setReturningName(afterReturningAnnotation.returning());
				}
				break;
			case AtAfterThrowing:
				springAdvice = new AspectJAfterThrowingAdvice(
						candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
				AfterThrowing afterThrowingAnnotation = (AfterThrowing) aspectJAnnotation.getAnnotation();
				if (StringUtils.hasText(afterThrowingAnnotation.throwing())) {
					springAdvice.setThrowingName(afterThrowingAnnotation.throwing());
				}
				break;
			case AtAround:
				springAdvice = new AspectJAroundAdvice(
						candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
				break;
			case AtPointcut:
				if (logger.isDebugEnabled()) {
					logger.debug("Processing pointcut '" + candidateAdviceMethod.getName() + "'");
				}
				return null;
			default:
				throw new UnsupportedOperationException(
						"Unsupported advice type on method: " + candidateAdviceMethod);
		}

		// Now to configure the advice...
		springAdvice.setAspectName(aspectName);
		springAdvice.setDeclarationOrder(declarationOrder);
		String[] argNames = this.parameterNameDiscoverer.getParameterNames(candidateAdviceMethod);
		if (argNames != null) {
			springAdvice.setArgumentNamesFromStringArray(argNames);
		}
		springAdvice.calculateArgumentBindings();
		return springAdvice;
	}




注意如下代码是创建aop代理的关键代码，可以看到可以根据proxy-target-class 来判断是否用clib的方式实现动态代理，


    public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
		if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
			Class<?> targetClass = config.getTargetClass();
			if (targetClass == null) {
				throw new AopConfigException("TargetSource cannot determine target class: " +
						"Either an interface or a target is required for proxy creation.");
			}
			if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
				return new JdkDynamicAopProxy(config);
			}
			return new ObjenesisCglibAopProxy(config);
		}
		else {
			return new JdkDynamicAopProxy(config);
		}
	}


调用到  CglibAopProxy#getProxy，使用 Enhancer来创建动态代理。


    public Object getProxy(ClassLoader classLoader) {
		if (logger.isDebugEnabled()) {
			logger.debug("Creating CGLIB proxy: target source is " + this.advised.getTargetSource());
		}

		try {
			Class<?> rootClass = this.advised.getTargetClass();
			Assert.state(rootClass != null, "Target class must be available for creating a CGLIB proxy");

			Class<?> proxySuperClass = rootClass;
			if (ClassUtils.isCglibProxyClass(rootClass)) {
				proxySuperClass = rootClass.getSuperclass();
				Class<?>[] additionalInterfaces = rootClass.getInterfaces();
				for (Class<?> additionalInterface : additionalInterfaces) {
					this.advised.addInterface(additionalInterface);
				}
			}

			// Validate the class, writing log messages as necessary.
			validateClassIfNecessary(proxySuperClass, classLoader);

			// Configure CGLIB Enhancer...
			Enhancer enhancer = createEnhancer();
			if (classLoader != null) {
				enhancer.setClassLoader(classLoader);
				if (classLoader instanceof SmartClassLoader &&
						((SmartClassLoader) classLoader).isClassReloadable(proxySuperClass)) {
					enhancer.setUseCache(false);
				}
			}
			enhancer.setSuperclass(proxySuperClass);
			enhancer.setInterfaces(AopProxyUtils.completeProxiedInterfaces(this.advised));
			enhancer.setNamingPolicy(SpringNamingPolicy.INSTANCE);
			enhancer.setStrategy(new ClassLoaderAwareUndeclaredThrowableStrategy(classLoader));

			Callback[] callbacks = getCallbacks(rootClass);
			Class<?>[] types = new Class<?>[callbacks.length];
			for (int x = 0; x < types.length; x++) {
				types[x] = callbacks[x].getClass();
			}
			// fixedInterceptorMap only populated at this point, after getCallbacks call above
			enhancer.setCallbackFilter(new ProxyCallbackFilter(
					this.advised.getConfigurationOnlyCopy(), this.fixedInterceptorMap, this.fixedInterceptorOffset));
			enhancer.setCallbackTypes(types);

			// Generate the proxy class and create a proxy instance.
			return createProxyClassAndInstance(enhancer, callbacks);
		}
		catch (CodeGenerationException ex) {
			throw new AopConfigException("Could not generate CGLIB subclass of class [" +
					this.advised.getTargetClass() + "]: " +
					"Common causes of this problem include using a final class or a non-visible class",
					ex);
		}
		catch (IllegalArgumentException ex) {
			throw new AopConfigException("Could not generate CGLIB subclass of class [" +
					this.advised.getTargetClass() + "]: " +
					"Common causes of this problem include using a final class or a non-visible class",
					ex);
		}
		catch (Exception ex) {
			// TargetSource.getTarget() failed
			throw new AopConfigException("Unexpected AOP exception", ex);
		}
	}


#### 增强类

aop中Advice 增强类(拦截器类)有如下五种。

    AspectJMethodBeforeAdvice
    
    AspectJAfterAdvice
    
    
    AspectJAfterReturningAdvice
    
    AspectJAfterThrowingAdvice
    
    AspectJAroundAdvice
    
    
    
    
    
#### 执行过程

当执行到切点逻辑时候，执行到代理对象的逻辑，如下图所示。得到的orderService实际上是一个代理类。


![](http://7x00ae.com1.z0.glb.clouddn.com/17-12-16/87004432.jpg)


下一步执行会进入DynamicAdvisedInterceptor的逻辑。



    	private static class DynamicAdvisedInterceptor implements MethodInterceptor, Serializable {

		private final AdvisedSupport advised;

		public DynamicAdvisedInterceptor(AdvisedSupport advised) {
			this.advised = advised;
		}

		@Override
		public Object intercept(Object proxy, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
			Object oldProxy = null;
			boolean setProxyContext = false;
			Class<?> targetClass = null;
			Object target = null;
			try {
				if (this.advised.exposeProxy) {
					// Make invocation available if necessary.
					oldProxy = AopContext.setCurrentProxy(proxy);
					setProxyContext = true;
				}
				// May be null. Get as late as possible to minimize the time we
				// "own" the target, in case it comes from a pool...
				target = getTarget();
				if (target != null) {
					targetClass = target.getClass();
				}
				List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
				Object retVal;
				// Check whether we only have one InvokerInterceptor: that is,
				// no real advice, but just reflective invocation of the target.
				if (chain.isEmpty() && Modifier.isPublic(method.getModifiers())) {
					// We can skip creating a MethodInvocation: just invoke the target directly.
					// Note that the final invoker must be an InvokerInterceptor, so we know
					// it does nothing but a reflective operation on the target, and no hot
					// swapping or fancy proxying.
					Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
					retVal = methodProxy.invoke(target, argsToUse);
				}
				else {
					// 执行。
					retVal = new CglibMethodInvocation(proxy, target, method, args, targetClass, chain, methodProxy).proceed();
				}
				retVal = processReturnType(proxy, target, method, retVal);
				return retVal;
			}
			finally {
				if (target != null) {
					releaseTarget(target);
				}
				if (setProxyContext) {
					// Restore old proxy.
					AopContext.setCurrentProxy(oldProxy);
				}
			}
		}

执行到ReflectiveMethodInvocation的proceed方法



查看变量interceptorsAndDynamicMethodMatchers如下图所示，可以看到该方法有如下几个拦截器。

![](http://7x00ae.com1.z0.glb.clouddn.com/17-12-16/10792470.jpg)






    public Object proceed() throws Throwable {
		//	We start with an index of -1 and increment early.
		if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
			return invokeJoinpoint();
		}

		Object interceptorOrInterceptionAdvice =
				this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
		if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
			// Evaluate dynamic method matcher here: static part will already have
			// been evaluated and found to match.
			InterceptorAndDynamicMethodMatcher dm =
					(InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;
			if (dm.methodMatcher.matches(this.method, this.targetClass, this.arguments)) {
				return dm.interceptor.invoke(this);
			}
			else {
				// Dynamic matching failed.
				// Skip this interceptor and invoke the next in the chain.
				return proceed();
			}
		}
		else {
			// It's an interceptor, so we just invoke it: The pointcut will have
			// been evaluated statically before this object was constructed.
			return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
		}
	}


advice 会按照一定的顺序(前面介绍过顺序)AfterThroing，AfterReturn，after，around，before 执行，但是这里的调用方式是递归调用m.proceed(), 每种advice会递归调用proceed()方法，
当所有的拦截器都被执行过后，会去执行真正的业务切点方法。如下代码所示。



    if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
    			return invokeJoinpoint();
    		}
    
    
所以advice的执行顺序如下;


![](http://7x00ae.com1.z0.glb.clouddn.com/17-12-16/70376112.jpg)




整体实例化aop过程。核心思想在实例化bean之后，去spring容器获取AutoProxyCreator,获取所有的advice，创建代理对象返回。


![](http://7x00ae.com1.z0.glb.clouddn.com/17-12-16/36371793.jpg)





















