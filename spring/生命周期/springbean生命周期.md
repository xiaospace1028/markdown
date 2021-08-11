## spring的生命周期

### 生命周期

1. 实例化
2. 装配属性 
3. 执行相关aware 
4. 执行BeanPostProcessor.Before方法 
5. 执行init方法
6. 执行BeanPostProcessor.After方法 
7. 执行destroy方法

---

测试类

```java
package com.xiaospace.jytest.process;

import org.springframework.beans.BeansException;
import org.springframework.beans.factory.*;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.*;
import org.springframework.context.annotation.Bean;
import org.springframework.core.io.ResourceLoader;
import org.springframework.stereotype.Component;
import org.springframework.web.context.ServletContextAware;

import javax.annotation.PostConstruct;
import javax.annotation.PreDestroy;
import javax.servlet.ServletContext;

/**
 * @PostConstruct->InitializingBean.afterPropertiesSet->initMethod
 * 初始化：注解 优先 InitializingBean 优先 initMethod
 * @PreDestroy->DisposableBean.destroy->destroyMethod 
 * 结束：注解 优先 DisposableBean 优先 destroyMethod
 */
@Component
public class BeanConfig {
    @Bean(initMethod = "initMethod", destroyMethod = "destroyMethod")
    public BeanInit getBeanInit() {
        return new BeanInit();
    }

    /**
     * ==>BeanNameAware.setBeanName
     * ==>BeanClassLoaderAware.setBeanClassLoader
     * ==>BeanFactoryAware.setBeanFactory
     * ==>ApplicationEventPublisherAware.setApplicationEventPublisher
     * ==>MessageSourceAware.setMessageSource
     * ==>ApplicationContextAware.setApplicationContext
     * ==>ServletContextAware.setServletContext
     * ==>BeanPostProcessor.postProcessBeforeInitialization
     * ==>@PostConstruct
     * ==>InitializingBean.afterPropertiesSet
     * ==>initMethod
     * ==>BeanPostProcessor.postProcessAfterInitialization
     * ==>@PreDestroy
     * ==>DisposableBean.destroy
     * ==>destroyMethod
     */
    static class BeanInit implements
            BeanNameAware,//在创建此bean的bean工厂中设置bean的名称
            BeanClassLoaderAware,//将 bean ClassLoaderr 提供给 bean 实例的回调
            BeanFactoryAware,//回调提供了自己的bean实例工厂
            ResourceLoaderAware,//设置资源路径
            ApplicationEventPublisherAware,//设置事件通知
            MessageSourceAware,//设置国际化消息
            ApplicationContextAware,//设置上下文
            ServletContextAware,//ServletContext上下文
            InitializingBean,//初始化bean的方法顺序
            DisposableBean//注销bean的的方法
    {
        @Value("${beanInit.val}")
        private String val; //可以打印这个参数说明确实set参数了

        public String getVal() {
            return val;
        }

        @Override
        public void setBeanName(String name) {
            System.out.println("==>BeanNameAware.setBeanName");
        }

        @Override
        public void setBeanClassLoader(ClassLoader classLoader) {
            System.out.println("==>BeanClassLoaderAware.setBeanClassLoader");
        }

        @Override
        public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
            System.out.println("==>BeanFactoryAware.setBeanFactory");
        }

        @Override
        public void setResourceLoader(ResourceLoader resourceLoader) {
            System.out.println("==>ResourceLoaderAware.setResourceLoader");
        }

        @Override
        public void setApplicationEventPublisher(ApplicationEventPublisher applicationEventPublisher) {
            System.out.println("==>ApplicationEventPublisherAware.setApplicationEventPublisher");
        }

        @Override
        public void setMessageSource(MessageSource messageSource) {
            System.out.println("==>MessageSourceAware.setMessageSource");
        }

        @Override
        public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
            System.out.println("==>ApplicationContextAware.setApplicationContext");
        }

        @Override
        public void setServletContext(ServletContext servletContext) {
            System.out.println("==>ServletContextAware.setServletContext");
        }

        @PostConstruct
        public void init() {
            System.out.println("==>@PostConstruct");
        }

        @Override
        public void afterPropertiesSet() throws Exception {
            System.out.println("==>InitializingBean.afterPropertiesSet");
        }

        public void initMethod() {
            System.out.println("==>initMethod");
        }

        @PreDestroy
        public void close() {
            System.out.println("==>@PreDestroy");
        }

        @Override
        public void destroy() throws Exception {
            System.out.println("==>DisposableBean.destroy");
        }

        public void destroyMethod() {
            System.out.println("==>destroyMethod");
        }
    }
}
```

```java
package com.xiaospace.jytest.process;

import org.springframework.beans.BeansException;
import org.springframework.beans.factory.config.BeanPostProcessor;
import org.springframework.stereotype.Component;

@Component
public class BeanProcessor implements BeanPostProcessor {

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        if (bean instanceof BeanConfig.BeanInit) {
            System.out.println("==>BeanPostProcessor.postProcessBeforeInitialization");
        }
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        if (bean instanceof BeanConfig.BeanInit) {
            System.out.println("==>BeanPostProcessor.postProcessAfterInitialization");
        }
        return null;
    }
}
```



其实BeanFactory里面也写了相关的配置文件后的生命周期顺序

### org.springframework.beans.factory.BeanFactory

```java
* @see BeanNameAware#setBeanName
 * @see BeanClassLoaderAware#setBeanClassLoader
 * @see BeanFactoryAware#setBeanFactory
 * @see org.springframework.context.ResourceLoaderAware#setResourceLoader
 * @see org.springframework.context.ApplicationEventPublisherAware#setApplicationEventPublisher
 * @see org.springframework.context.MessageSourceAware#setMessageSource
 * @see org.springframework.context.ApplicationContextAware#setApplicationContext
 * @see org.springframework.web.context.ServletContextAware#setServletContext
 * @see org.springframework.beans.factory.config.BeanPostProcessor#postProcessBeforeInitialization
 * @see InitializingBean#afterPropertiesSet
 * @see org.springframework.beans.factory.support.RootBeanDefinition#getInitMethodName
 * @see org.springframework.beans.factory.config.BeanPostProcessor#postProcessAfterInitialization
 * @see DisposableBean#destroy
 * @see org.springframework.beans.factory.support.RootBeanDefinition#getDestroyMethodName
 */
public interface BeanFactory {
```



下面也有具体生命周期内部源码顺序

#### org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory

##### #initializeBean(java.lang.String, java.lang.Object, org.springframework.beans.factory.support.RootBeanDefinition)

```java
protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
			//...................................................
			invokeAwareMethods(beanName, bean);
      //查看#invokeAwareMethods
  	
			//...................................................
			wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
      //查看#applyBeanPostProcessorsBeforeInitialization

			//...................................................
			invokeInitMethods(beanName, wrappedBean, mbd);
      //查看#invokeInitMethods
  
			//...................................................
			wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
      //查看applyBeanPostProcessorsAfterInitialization
		return wrappedBean;
	}
```

##### #invokeAwareMethods

```java
	private void invokeAwareMethods(final String beanName, final Object bean) {
		if (bean instanceof Aware) {
			if (bean instanceof BeanNameAware) {
				((BeanNameAware) bean).setBeanName(beanName);
			}
			if (bean instanceof BeanClassLoaderAware) {
				ClassLoader bcl = getBeanClassLoader();
				if (bcl != null) {
					((BeanClassLoaderAware) bean).setBeanClassLoader(bcl);
				}
			}
			if (bean instanceof BeanFactoryAware) {
				((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);
			}
		}
	}
```
```java
//==>BeanNameAware.setBeanName
//==>BeanClassLoaderAware.setBeanClassLoader
//==>BeanFactoryAware.setBeanFactory
```



##### #applyBeanPostProcessorsBeforeInitialization

```java
public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName)
			throws BeansException {
		Object result = existingBean;
		for (BeanPostProcessor processor : getBeanPostProcessors()) {
			Object current = processor.postProcessBeforeInitialization(result, beanName);
			if (current == null) {
				return result;
			}
			result = current;
		}
		return result;
	}
```

```java
==>getBeanPostProcessors()
result = {CopyOnWriteArrayList@8660}  size = 17
 0 = {ApplicationContextAwareProcessor@8699} 
  //==>ResourceLoaderAware.setResourceLoader
	//==>ApplicationEventPublisherAware.setApplicationEventPublisher
	//==>MessageSourceAware.setMessageSource
	//==>ApplicationContextAware.setApplicationContext
 1 = {WebApplicationContextServletContextAwareProcessor@8727} 
  //==>ServletContextAware.setServletContext
 2 = {ConfigurationClassPostProcessor$ImportAwareBeanPostProcessor@8748} 
 3 = {PostProcessorRegistrationDelegate$BeanPostProcessorChecker@8768} 
 4 = {ConfigurationPropertiesBindingPostProcessor@8769} 
 5 = {AnnotationAwareAspectJAutoProxyCreator@8770} 
 6 = {DataSourceInitializerPostProcessor@8771} 
 7 = {MethodValidationPostProcessor@8772}
 8 = {PersistenceExceptionTranslationPostProcessor@8773} 
 9 = {BeanProcessor@8774} 
 //==>BeanPostProcessor.postProcessBeforeInitialization
 10 = {WebServerFactoryCustomizerBeanPostProcessor@8775} 
 11 = {ErrorPageRegistrarBeanPostProcessor@8776} 
 12 = {ProjectingArgumentResolverRegistrar$ProjectingArgumentResolverBeanPostProcessor@8777} 
 13 = {PersistenceAnnotationBeanPostProcessor@8778} 
 14 = {CommonAnnotationBeanPostProcessor@8779} 
 //==>@PostConstruct
 15 = {AutowiredAnnotationBeanPostProcessor@8780} 
 16 = {ApplicationListenerDetector@8781} 
```

##### #invokeInitMethods

```java
	protected void invokeInitMethods(String beanName, final Object bean, @Nullable RootBeanDefinition mbd)
			throws Throwable {

		boolean isInitializingBean = (bean instanceof InitializingBean);
		if (isInitializingBean && (mbd == null || !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {
		//......省略权限代码.......
			}
			else {
				((InitializingBean) bean).afterPropertiesSet();//执行实现InitializingBean#afterPropertiesSet()方法
			}
		}

		if (mbd != null && bean.getClass() != NullBean.class) {//执行指定InitMethod方法
			String initMethodName = mbd.getInitMethodName();
			if (StringUtils.hasLength(initMethodName) && //判断该方法是否为空
					!(isInitializingBean && "afterPropertiesSet".equals(initMethodName)) &&//判断是否继承isInitializingBean并且方法名称afterPropertiesSet 为false
					!mbd.isExternallyManagedInitMethod(initMethodName)//判断是不是init方法
         ) {
				invokeCustomInitMethod(beanName, bean, mbd);
			}
		}
	}
```

```java
//==>InitializingBean.afterPropertiesSet
//==>initMethod
```

##### #applyBeanPostProcessorsAfterInitialization

```java
	public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
			throws BeansException {
		Object result = existingBean;
		for (BeanPostProcessor processor : getBeanPostProcessors()) {
			Object current = processor.postProcessAfterInitialization(result, beanName);
			if (current == null) {
				return result;
			}
			result = current;
		}
		return result;
	}
```

```java
==>getBeanPostProcessors()
  result = {CopyOnWriteArrayList@8660}  size = 17
 0 = {ApplicationContextAwareProcessor@8699} 
 1 = {WebApplicationContextServletContextAwareProcessor@9147} 
 2 = {ConfigurationClassPostProcessor$ImportAwareBeanPostProcessor@8748} 
 3 = {PostProcessorRegistrationDelegate$BeanPostProcessorChecker@9148} 
 4 = {ConfigurationPropertiesBindingPostProcessor@9149} 
 5 = {AnnotationAwareAspectJAutoProxyCreator@9150}
 6 = {DataSourceInitializerPostProcessor@9151} 
 7 = {MethodValidationPostProcessor@9152} 
 8 = {PersistenceExceptionTranslationPostProcessor@9153} 
 9 = {BeanProcessor@9154} 
//==>BeanPostProcessor.postProcessAfterInitialization
 10 = {WebServerFactoryCustomizerBeanPostProcessor@9155} 
 11 = {ErrorPageRegistrarBeanPostProcessor@9156} 
 12 = {ProjectingArgumentResolverRegistrar$ProjectingArgumentResolverBeanPostProcessor@9157} 
 13 = {PersistenceAnnotationBeanPostProcessor@9158} 
 14 = {CommonAnnotationBeanPostProcessor@9159} 
 15 = {AutowiredAnnotationBeanPostProcessor@8780} 
 16 = {ApplicationListenerDetector@8781} 
```

