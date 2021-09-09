# Bean生命周期源码_1

1. 关键流程图

   <iframe id="embed_dom" name="embed_dom" frameborder="0" style="display:block;width:525px; height:245px;" src="https://www.processon.com/embed/61397f4b0e3e7412ece12032"> </iframe>
   
2. 源码

   ```java
   // AbstractAutowireCapableBeanFactory#doCreateBean
       
       
   protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
   			throws BeanCreationException {
   
   		// Instantiate the bean.
   		BeanWrapper instanceWrapper = null;
   		if (mbd.isSingleton()) {
   			instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
   		}
   		if (instanceWrapper == null) {
               // 1. 实例化
   			instanceWrapper = createBeanInstance(beanName, mbd, args);
   		}
   		final Object bean = instanceWrapper.getWrappedInstance();
   		Class<?> beanType = instanceWrapper.getWrappedClass();
   		if (beanType != NullBean.class) {
   			mbd.resolvedTargetType = beanType;
   		}
   
   		// Allow post-processors to modify the merged bean definition.
   		synchronized (mbd.postProcessingLock) {
   			if (!mbd.postProcessed) {
   				try {
   					applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
   				}
   				catch (Throwable ex) {
   					throw new BeanCreationException(mbd.getResourceDescription(), beanName,
   							"Post-processing of merged bean definition failed", ex);
   				}
   				mbd.postProcessed = true;
   			}
   		}
   
   		// Eagerly cache singletons to be able to resolve circular references
   		// even when triggered by lifecycle interfaces like BeanFactoryAware.
   		boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
   				isSingletonCurrentlyInCreation(beanName));
   		if (earlySingletonExposure) {
   			if (logger.isTraceEnabled()) {
   				logger.trace("Eagerly caching bean '" + beanName +
   						"' to allow for resolving potential circular references");
   			}
   			addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
   		}
   
   		// Initialize the bean instance.
   		Object exposedObject = bean;
   		try {
               // 2. 属性填充
   			populateBean(beanName, mbd, instanceWrapper);
               // 3. 初始化
   			exposedObject = initializeBean(beanName, exposedObject, mbd);
   		}
   		catch (Throwable ex) {
   			if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
   				throw (BeanCreationException) ex;
   			}
   			else {
   				throw new BeanCreationException(
   						mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
   			}
   		}
   
   		if (earlySingletonExposure) {
   			Object earlySingletonReference = getSingleton(beanName, false);
   			if (earlySingletonReference != null) {
   				if (exposedObject == bean) {
   					exposedObject = earlySingletonReference;
   				}
   				else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
   					String[] dependentBeans = getDependentBeans(beanName);
   					Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
   					for (String dependentBean : dependentBeans) {
   						if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
   							actualDependentBeans.add(dependentBean);
   						}
   					}
   					if (!actualDependentBeans.isEmpty()) {
   						throw new BeanCurrentlyInCreationException(beanName,
   								"Bean with name '" + beanName + "' has been injected into other beans [" +
   								StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
   								"] in its raw version as part of a circular reference, but has eventually been " +
   								"wrapped. This means that said other beans do not use the final version of the " +
   								"bean. This is often the result of over-eager type matching - consider using " +
   								"'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
   					}
   				}
   			}
   		}
   
   		// Register bean as disposable.
   		try {
               // 4. 销毁--注册回调
   			registerDisposableBeanIfNecessary(beanName, bean, mbd);
   		}
   		catch (BeanDefinitionValidationException ex) {
   			throw new BeanCreationException(
   					mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
   		}
   
   		return exposedObject;
   	}
   ```



第三步、初始化

3.1 Aware接口

* BeanNameAware：注入当前 bean 对应 beanName；
* BeanClassLoaderAware：注入加载当前 bean 的 ClassLoader；
* BeanFactoryAware：注入 当前BeanFactory容器 的引用。

```java
// AbstractAutowireCapableBeanFactory#initializeBean
    
protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
		if (System.getSecurityManager() != null) {
			AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
				invokeAwareMethods(beanName, bean);
				return null;
			}, getAccessControlContext());
		}
		else {
            // 3.1 Aware接口
			invokeAwareMethods(beanName, bean);
		}

		Object wrappedBean = bean;
		if (mbd == null || !mbd.isSynthetic()) {
            // 3.2 BeanPostProcessor前置处理器
			wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
		}

		try {
            // 3.3 InitializingBean接口  和  3.4 init-method
			invokeInitMethods(beanName, wrappedBean, mbd);
		}
		catch (Throwable ex) {
			throw new BeanCreationException(
					(mbd != null ? mbd.getResourceDescription() : null),
					beanName, "Invocation of init method failed", ex);
		}
		if (mbd == null || !mbd.isSynthetic()) {
            // 3.25 BeanPostProcessor后置处理器
			wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
		}

		return wrappedBean;
	}
```



参考： 

[如何记忆 Spring Bean 的生命周期]: https://www.toutiao.com/i6838849657989235213/?tt_from=weixin&amp;utm_campaign=client_share&amp;wxshare_count=1&amp;timestamp=1631074182&amp;app=news_article&amp;utm_source=weixin&amp;utm_medium=toutiao_android&amp;use_new_style=1&amp;req_id=2021090812094201013515503057006B85&amp;share_token=c6b43a7f-7658-4961-b2c6-ecc6ab5ca0df&amp;group_id=6838849657989235213

