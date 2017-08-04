---
layout: post
title: Spring 加载property
category: Spring
tags: JAVA Spring 加载property

---



经常在spring工程中 使用如下代码加载 jdbc的配置文件。
    
     <bean class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">
        <property name="locations">
            <array>
                <value>classpath:jdbc.properties</value>
            </array>
        </property>
    </bean>
  
    
    
    
      <bean id="dataSource" class="com.zhaopeng.action.DataSource">
        <property name="url" value="${url}"></property>
    </bean>
    
jdbc.properties 文件如下
    
    url=127.0.0.1
    
    

spring 启动后。在实例化dataSource的时候把jdbc.properties 里面的的url赋予给dataSource.
首先查看 PropertyPlaceholderConfigurer 类，

PropertyPlaceholderConfigurer 的继承关系如下：


![image](http://7x00ae.com1.z0.glb.clouddn.com/spring%E5%8A%A0%E8%BD%BD%E8%B5%84%E6%BA%90%E6%96%87%E4%BB%B6.jpg)

    

如继承图中可以看到 PropertyPlaceholderConfigurer 是 BeanFactoryPostProcessor的子类。

public interface BeanFactoryPostProcessor {

	void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;

    }


BeanFactoryPostProcessor 是一个接口，它的作用在于 在beanFactory实例化后，可以让子类获取到
beanFactory，从而对beanDefintion做一些修改操作。


在此图中可以看到 PropertyResourceConfigurer 实现了BeanFactoryPostProcessor接口



从AbstractApplicationContext 的refresh方法调用 	invokeBeanFactoryPostProcessors(beanFactory)开始。 调用到PostProcessorRegistrationDelegate 的 invokeBeanFactoryPostProcessors;



    public abstract class PropertyResourceConfigurer extends PropertiesLoaderSupport
    		implements BeanFactoryPostProcessor, PriorityOrdered {

    	@Override
	public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
		try {
		    // 从properties 文件获取所有的属性
			Properties mergedProps = mergeProperties();

			// Convert the merged properties, if necessary.
			convertProperties(mergedProps);

			// Let the subclass process the properties.
			// 调用子类实现，把${...} 替换成真正的属性值
			// 调用PropertyPlaceholderConfigurer 的方法
			processProperties(beanFactory, mergedProps);
		}
		catch (IOException ex) {
			throw new BeanInitializationException("Could not load properties", ex);
		}
	}
	
	
	
	public class PropertyPlaceholderConfigurer extends PlaceholderConfigurerSupport {
	
	//替换bean definition 的${...} 
	        @Override
    	protected void processProperties(ConfigurableListableBeanFactory beanFactoryToProcess, Properties props)
    			throws BeansException {
    
    		StringValueResolver valueResolver = new PlaceholderResolvingStringValueResolver(props);
    		// 调用PlaceholderConfigurerSupport 中的方法
    		doProcessProperties(beanFactoryToProcess, valueResolver);
    	}
	
	}



public abstract class PlaceholderConfigurerSupport extends PropertyResourceConfigurer
		implements BeanNameAware, BeanFactoryAware {


    protected void doProcessProperties(ConfigurableListableBeanFactory beanFactoryToProcess,
			StringValueResolver valueResolver) {

		BeanDefinitionVisitor visitor = new BeanDefinitionVisitor(valueResolver);

		String[] beanNames = beanFactoryToProcess.getBeanDefinitionNames();
		// 根据beanName 迭代每一个beanDefinition
		for (String curName : beanNames) {
			// Check that we're not parsing our own bean definition,
			// to avoid failing on unresolvable placeholders in properties file locations.
			
			if (!(curName.equals(this.beanName) && beanFactoryToProcess.equals(this.beanFactory))) {
				BeanDefinition bd = beanFactoryToProcess.getBeanDefinition(curName);
				try {
				 //执行 替换，
					visitor.visitBeanDefinition(bd);
				}
				catch (Exception ex) {
					throw new BeanDefinitionStoreException(bd.getResourceDescription(), curName, ex.getMessage(), ex);
				}
			}
		}

		// New in Spring 2.5: resolve placeholders in alias target names and aliases as well.
		beanFactoryToProcess.resolveAliases(valueResolver);

		// New in Spring 3.0: resolve placeholders in embedded values such as annotation attributes.
		beanFactoryToProcess.addEmbeddedValueResolver(valueResolver);
	}



    }




以上代码加载时序图如下。


![image](http://7x00ae.com1.z0.glb.clouddn.com/spring%E5%8A%A0%E8%BD%BD%E8%B5%84%E6%BA%90%E6%97%B6%E5%BA%8F%E5%9B%BE.png)


#### 总结



spring 启动时候，主要通过实现 BeanFactoryPostProcessor的子类 PropertyResourceConfigurer 先加载好所有的property文件到内存，然后修改beanDefinition ，把变量${}替换成真正的属性值，这样就完成了配置文件的加载。