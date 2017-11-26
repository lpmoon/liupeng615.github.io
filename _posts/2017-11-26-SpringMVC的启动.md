---
categories: Spring 
tags: Spring 
---

传统的使用SpringMVC的方法是在web.xml中配置DispatchServlet，在SpringMVC 3.1版本后提供了一种新的启动方式。在介绍这种新的方式之前，有必要先介绍一下Servlet 3.0的一个新特性 **ServletContainerInitializer** 。先看一下javadoc对该接口的介绍

```
实现该接口的类会成为web应用的入口，并且在启动期间可以注入servlets，filters，和listeners。

该接口的实现类可能会包含 **javax.servlet.annotation.HandlesTypes** 注解，该注解的value对应的Class<?> []， 用于表示该接口的实现类的onStartup方法会接收Class<?> []对应的实现或者子类作为参数。

如果该接口的实现类不包含 **javax.servlet.annotation.HandlesTypes** 注解，或者应用中没有该注解中对应的Class，则需要传递null给onStartup方法。

如果实现了该接口则需要在jar的META-INF/services目录下放置一个文件，改文件的名称必须是该接口的全名称。
```

从上面javadoc的注释可以看出，当应用启动的时候，如果有 **ServletContainerInitializer** 的实现类并且有相应的配置，则会使用该类作为web的入口。

SpringMVC新的启动方式正是基于上面提到的 **ServletContainerInitializer** 接口，其实现类是 **SpringServletContainerInitializer** ，下面看看该类的实现，
```
@HandlesTypes(WebApplicationInitializer.class)
public class SpringServletContainerInitializer implements ServletContainerInitializer {

	@Override
	public void onStartup(Set<Class<?>> webAppInitializerClasses, ServletContext servletContext)
			throws ServletException {

		List<WebApplicationInitializer> initializers = new LinkedList<WebApplicationInitializer>();

		if (webAppInitializerClasses != null) {
			for (Class<?> waiClass : webAppInitializerClasses) {
				// Be defensive: Some servlet containers provide us with invalid classes,
				// no matter what @HandlesTypes says...
				if (!waiClass.isInterface() && !Modifier.isAbstract(waiClass.getModifiers()) &&
						WebApplicationInitializer.class.isAssignableFrom(waiClass)) {
					try {
						initializers.add((WebApplicationInitializer) waiClass.newInstance());
					}
					catch (Throwable ex) {
						throw new ServletException("Failed to instantiate WebApplicationInitializer class", ex);
					}
				}
			}
		}

		if (initializers.isEmpty()) {
			servletContext.log("No Spring WebApplicationInitializer types detected on classpath");
			return;
		}

		servletContext.log(initializers.size() + " Spring WebApplicationInitializers detected on classpath");
		AnnotationAwareOrderComparator.sort(initializers);
		for (WebApplicationInitializer initializer : initializers) {
			initializer.onStartup(servleContext);
		}
	}

}
```

上面的代码功能比较明确，针对每一个传入的 **WebApplicationInitializer** 的实现类进行实例化，然后将这些实例化的对象进行排序操作。排序的规则是

> 如果类带有 **org.springframework.core.annotation.Order** 标签，则获取对应的Order值，如果类实现 **org.springframework.core.Ordered** 接口，则调用getOrder方法获取Order值，如果都没有则为最低Order值。获取所有实例的Order值后，根据Order值进行排序。

排序好后，依次调用实例的onStartup方法进行初始化。

**SpringServletContainerInitialize** 作为SpringMVC初始化的入口，最终的实现依赖于 **WebApplicationInitializer** 的实现类 **AbstractAnnotationConfigDispatcherServletInitializer** ，

```
public abstract class AbstractAnnotationConfigDispatcherServletInitializer
		extends AbstractDispatcherServletInitializer {

	/**
	 * {@inheritDoc}
	 * <p>This implementation creates an {@link AnnotationConfigWebApplicationContext},
	 * providing it the annotated classes returned by {@link #getRootConfigClasses()}.
	 * Returns {@code null} if {@link #getRootConfigClasses()} returns {@code null}.
	 */
	@Override
	protected WebApplicationContext createRootApplicationContext() {
		Class<?>[] configClasses = getRootConfigClasses();
		if (!ObjectUtils.isEmpty(configClasses)) {
			AnnotationConfigWebApplicationContext rootAppContext = new AnnotationConfigWebApplicationContext();
			rootAppContext.register(configClasses);
			return rootAppContext;
		}
		else {
			return null;
		}
	}

	/**
	 * {@inheritDoc}
	 * <p>This implementation creates an {@link AnnotationConfigWebApplicationContext},
	 * providing it the annotated classes returned by {@link #getServletConfigClasses()}.
	 */
	@Override
	protected WebApplicationContext createServletApplicationContext() {
		AnnotationConfigWebApplicationContext servletAppContext = new AnnotationConfigWebApplicationContext();
		Class<?>[] configClasses = getServletConfigClasses();
		if (!ObjectUtils.isEmpty(configClasses)) {
			servletAppContext.register(configClasses);
		}
		return servletAppContext;
	}

	/**
	 * Specify {@link org.springframework.context.annotation.Configuration @Configuration}
	 * and/or {@link org.springframework.stereotype.Component @Component} classes to be
	 * provided to the {@linkplain #createRootApplicationContext() root application context}.
	 * @return the configuration classes for the root application context, or {@code null}
	 * if creation and registration of a root context is not desired
	 */
	protected abstract Class<?>[] getRootConfigClasses();

	/**
	 * Specify {@link org.springframework.context.annotation.Configuration @Configuration}
	 * and/or {@link org.springframework.stereotype.Component @Component} classes to be
	 * provided to the {@linkplain #createServletApplicationContext() dispatcher servlet
	 * application context}.
	 * @return the configuration classes for the dispatcher servlet application context or
	 * {@code null} if all configuration is specified through root config classes.
	 */
	protected abstract Class<?>[] getServletConfigClasses();

}
```
该类提供了两个抽象方法 **getRootConfigClasses** 和 **getServletConfigClasses** ，前者用于获取创建RootApplicationContext的配置，后者用于获取创建ServletApplicationContext的配置。另外两个方法 **createRootApplicationContext** ，**createServletApplicationContext** 用于创建RootApplicationContext和ServletApplicationContext。这个类的入口onStartup在其父类 **AbstractDispatcherServletInitializer** 中。
```
	@Override
	public void onStartup(ServletContext servletContext) throws ServletException {
		super.onStartup(servletContext);
		registerDispatcherServlet(servletContext);
	}
```

onStartup会首先调用其父类 **AbstractContextLoaderInitializer** 的onStartup方法进行初始化，
```
	@Override
	public void onStartup(ServletContext servletContext) throws ServletException {
		registerContextLoaderListener(servletContext);
	}

	protected void registerContextLoaderListener(ServletContext servletContext) {
		WebApplicationContext rootAppContext = createRootApplicationContext();
		if (rootAppContext != null) {
			ContextLoaderListener listener = new ContextLoaderListener(rootAppContext);
			listener.setContextInitializers(getRootApplicationContextInitializers());
			servletContext.addListener(listener);
		}
		else {
			logger.debug("No ContextLoaderListener registered, as " +
					"createRootApplicationContext() did not return an application context");
		}
	}
```
**AbstractContextLoaderInitializer** 的onStartup主要用于初始化RootApplicationContext。在RootApplicationContext初始化好后，**AbstractDispatcherServletInitializer** 会继续初始化DispatcherServlet。

```
	protected void registerDispatcherServlet(ServletContext servletContext) {
		String servletName = getServletName();
		Assert.hasLength(servletName, "getServletName() must not return empty or null");

		WebApplicationContext servletAppContext = createServletApplicationContext();
		Assert.notNull(servletAppContext,
				"createServletApplicationContext() did not return an application " +
				"context for servlet [" + servletName + "]");

		FrameworkServlet dispatcherServlet = createDispatcherServlet(servletAppContext);
		dispatcherServlet.setContextInitializers(getServletApplicationContextInitializers());

		ServletRegistration.Dynamic registration = servletContext.addServlet(servletName, dispatcherServlet);
		Assert.notNull(registration,
				"Failed to register servlet with name '" + servletName + "'." +
				"Check if there is another servlet registered under the same name.");

		registration.setLoadOnStartup(1);
		registration.addMapping(getServletMappings());
		registration.setAsyncSupported(isAsyncSupported());

		Filter[] filters = getServletFilters();
		if (!ObjectUtils.isEmpty(filters)) {
			for (Filter filter : filters) {
				registerServletFilter(servletContext, filter);
			}
		}

		customizeRegistration(registration);
	}
```

**registerDispatcherServlet** 调用 **createServletApplicationContext** 创建ServletApplicationContext，然后创建DispatcherServlet，设置ServletMapping，然后注册Filter。

经过上面的流程，SpringMVC的加载过程就完成了后续的请求处理就和通过web.xml启动的SpringMVC一样了。
