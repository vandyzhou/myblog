---
title: spring factorybean提前初始化
date: 2021-06-18 11:46:14
tags: case
categories: spring
---

## 版本

Spring 5.0.11.RELEASE
Spring Boot 2.0.7.RELEASE

## 背景

使用方初始化一个实现 FactoryBean 的Bean（ThriftClient），而这个FactoryBean的启动需要依赖另一个经过BeanPostProcessor（MdpConfigBeanPostProcessor）处理之后的值作为配置项用来启动；
这种依赖的方式会导致所有的 FactoryBean 被提前初始化；


## 问题分析

* Spring启动时，首先会初始化 BeanPostProcessor 这类Bean；将所有的 BeanPostProcessor 根据顺序（PriorityOrdered > Ordered > regular BeanPostProcessor）依次进行beanFactory.getBean(name, BeanPostProcessor.class)

> PostProcessorRegistrationDelegate.registerBeanPostProcessors

![registerBeanPostProcessors](/imgs/case/spring-bean-early-init/spring-early-init-1.png)

* 而springboot的自动配置里面存在ValidationAutoConfiguration的启动类，会自动加载MethodValidationPostProcessor的BeanPostProcessor

> ValidationAutoConfiguration.methodValidationPostProcessor

``` java
@Bean
@ConditionalOnMissingBean
public static MethodValidationPostProcessor methodValidationPostProcessor(
		Environment environment, @Lazy Validator validator) {
	MethodValidationPostProcessor processor = new MethodValidationPostProcessor();
	boolean proxyTargetClass = environment
			.getProperty("spring.aop.proxy-target-class", Boolean.class, true);
	processor.setProxyTargetClass(proxyTargetClass);
	processor.setValidator(validator);
	return processor;
}
```

* 而spring对于初始化带有构造方法的Bean时的逻辑如下：

> AbstractAutowireCapableBeanFactory.createBeanInstance

``` java
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
	// Make sure bean class is actually resolved at this point.
	Class<?> beanClass = resolveBeanClass(mbd, beanName);

	if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
		throw new BeanCreationException(mbd.getResourceDescription(), beanName,
				"Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
	}

	Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
	if (instanceSupplier != null) {
		return obtainFromSupplier(instanceSupplier, beanName);
	}

	if (mbd.getFactoryMethodName() != null)  {
		// 重点在这里
		return instantiateUsingFactoryMethod(beanName, mbd, args);
	}

	// 省略一些逻辑...

	// No special handling: simply use no-arg constructor.
	return instantiateBean(beanName, mbd);
}
//调用ConstructorResolver的构造
protected BeanWrapper instantiateUsingFactoryMethod(
		String beanName, RootBeanDefinition mbd, @Nullable Object[] explicitArgs) {

	return new ConstructorResolver(this).instantiateUsingFactoryMethod(beanName, mbd, explicitArgs);
}
```

> ConstructorResolver.instantiateUsingFactoryMethod --> createArgumentArray --> resolveAutowiredArgument

![createArgumentArray](/imgs/case/spring-bean-early-init/spring-early-init-3.png)

``` java
protected Object resolveAutowiredArgument(MethodParameter param, String beanName,
		@Nullable Set<String> autowiredBeanNames, TypeConverter typeConverter) {

	if (InjectionPoint.class.isAssignableFrom(param.getParameterType())) {
		InjectionPoint injectionPoint = currentInjectionPoint.get();
		if (injectionPoint == null) {
			throw new IllegalStateException("No current InjectionPoint available for " + param);
		}
		return injectionPoint;
	}
	return this.beanFactory.resolveDependency(
			new DependencyDescriptor(param, true), beanName, autowiredBeanNames, typeConverter);
}
**
 * Create a new descriptor for a method or constructor parameter.
 * Considers the dependency as 'eager'.
 * @param methodParameter the MethodParameter to wrap
 * @param required whether the dependency is required
 */
public DependencyDescriptor(MethodParameter methodParameter, boolean required) {
	//注意：第三个参数 earge = true
	this(methodParameter, required, true);
}
```

* 最后，对于初始化 带有构造方法的@Bean的BeanPostProcessor 时，最终会执行到 BeanFoctoryUtils.beanNamesForTypeIncludingAncestors 并且 eager = true

> DefaultListableBeanFactory.findAutowireCandidates

![findAutowireCandidates](/imgs/case/spring-bean-early-init/spring-early-init-2.png)

> BeanFoctoryUtils.beanNamesForTypeIncludingAncestors --> DefaultListableBeanFactory.doGetBeanNamesForType

``` java
private String[] doGetBeanNamesForType(ResolvableType type, boolean includeNonSingletons, boolean allowEagerInit) {
	List<String> result = new ArrayList<>();

	// 重点：Check all bean definitions.
	for (String beanName : this.beanDefinitionNames) {
		// Only consider bean as eligible if the bean name
		// is not defined as alias for some other bean.
		if (!isAlias(beanName)) {
			try {
				RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
				// Only check bean definition if it is complete.
				if (!mbd.isAbstract() && (allowEagerInit ||
						(mbd.hasBeanClass() || !mbd.isLazyInit() || isAllowEagerClassLoading()) &&
								!requiresEagerInitForType(mbd.getFactoryBeanName()))) {
					// In case of FactoryBean, match object created by FactoryBean.
					boolean isFactoryBean = isFactoryBean(beanName, mbd);
					BeanDefinitionHolder dbd = mbd.getDecoratedDefinition();
					boolean matchFound =
							(allowEagerInit || !isFactoryBean ||
									(dbd != null && !mbd.isLazyInit()) || containsSingleton(beanName)) &&
							(includeNonSingletons ||
									(dbd != null ? mbd.isSingleton() : isSingleton(beanName))) &&
							isTypeMatch(beanName, type);
					if (!matchFound && isFactoryBean) {
						// In case of FactoryBean, try to match FactoryBean instance itself next.
						beanName = FACTORY_BEAN_PREFIX + beanName;
						matchFound = (includeNonSingletons || mbd.isSingleton()) && isTypeMatch(beanName, type);
					}
					if (matchFound) {
						result.add(beanName);
					}
				}
			}
			catch (CannotLoadBeanClassException ex) {
				if (allowEagerInit) {
					throw ex;
				}
				// Probably a class name with a placeholder: let's ignore it for type matching purposes.
				if (logger.isDebugEnabled()) {
					logger.debug("Ignoring bean class loading failure for bean '" + beanName + "'", ex);
				}
				onSuppressedException(ex);
			}
			catch (BeanDefinitionStoreException ex) {
				if (allowEagerInit) {
					throw ex;
				}
				// Probably some metadata with a placeholder: let's ignore it for type matching purposes.
				if (logger.isDebugEnabled()) {
					logger.debug("Ignoring unresolvable metadata in bean definition '" + beanName + "'", ex);
				}
				onSuppressedException(ex);
			}
		}
	}

	// 省略...Check manually registered singletons too.
	
	return StringUtils.toStringArray(result);
}
```