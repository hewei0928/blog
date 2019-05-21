---
title: Spring之DispatcherServlet详解
date: 2018-06-13 09:39:52
tags:
    - Java
    - Spring
---

> DispatcherServlet是Spring mvc中的最重要一环，可以说Spring mvc是基于这个类实现的。之前只是知道大致的流程，但对中间的具体实现不甚了解。特此记录一下。

<!--more-->

## DispatcherServlet

`DispatcherServlet`的类结构如下图所示：
![image_1cfrii75e1m5fg22aj7svgv6a9.png-71.4kB][1]

### DispatcherServlet初始化
`DispatcherServlet`在web.xml中的声明如下所示：
```xml
<servlet>
	<servlet-name>spring</servlet-name>
	<servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
	<init-param>
		<param-name>contextConfigLocation</param-name>
		<!--spring-mvc 中主要是对Controller及HandlerMapping等web层类的配置-->
		<!--Service及DAO层中的类由ContextLoaderListener进行初始化-->
		<param-value>classpath*:spring-mvc.xml</param-value>
	</init-param>
	<load-on-startup>1</load-on-startup>
	<async-supported>true</async-supported>
</servlet>
<servlet-mapping>
	<servlet-name>spring</servlet-name>
	<url-pattern>*.do</url-pattern>
</servlet-mapping>
```
`DispatcherServlet`本质上还是一个`Servlet`因此在Web项目启动时会创建该对象实例并调用`init`方法进行初始化。阅读源码可知初始化`DispatcherServlet`时调用的是其父类`HttpServletBean`的`init`方法。

```java
public final void init() throws ServletException {
	if (logger.isDebugEnabled()) {
		logger.debug("Initializing servlet '" + getServletName() + "'");
	}

	// 解析servlet中的<init-param>标签，并设置入pvs中
	PropertyValues pvs = new ServletConfigPropertyValues(getServletConfig(), this.requiredProperties);
	if (!pvs.isEmpty()) {
		try {
		    //将当前Servlet转换为`BeanWrapper`，使Spring能将对bean进行加载
			BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(this);
			// ResourceLoader：接口仅有一个getResource(String location)的方法，可以根据一个资源地址加载文件资源。简单来说就是就是用这个类来加载<init-param>中指定的xml文件,再交由BeanWrapper进行加载
			ResourceLoader resourceLoader = new ServletContextResourceLoader(getServletContext());
			bw.registerCustomEditor(Resource.class, new ResourceEditor(resourceLoader, getEnvironment()));
			initBeanWrapper(bw);
			bw.setPropertyValues(pvs, true);
		}
		catch (BeansException ex) {
			if (logger.isErrorEnabled()) {
				logger.error("Failed to set bean properties on servlet '" + getServletName() + "'", ex);
			}
			throw ex;
		}
	}

	// FrameworkServlet中实现
	initServletBean();

	if (logger.isDebugEnabled()) {
		logger.debug("Servlet '" + getServletName() + "' configured successfully");
	}
}
```
里面各方法的细节日后再去深究，总之概括来讲就是：先通过`PropertyValues`获取`web.xml`文件中`DispatcherServlet`标签中`init-param`的参数值，然后通过`ResourceLoader`读取类似`classpath*:spring-mvc.xml`的配置信息，`BeanWrapper`对配置的标签进行解析和将系统默认的bean的各种属性设置到对应的bean属性。

继续往下阅读，可以看到`initServlet`方法(在`FrameworkServlet`中实现)，而`initServlet`方法中又调用了`initWebApplicationContext()`方法，`Spring mvc`用到的类(`handlermapping`、`handlerAdapter`、`viewResolver`等)都是在这个方法中进行初始化的：
```java
protected WebApplicationContext initWebApplicationContext() {
		WebApplicationContext rootContext =
				WebApplicationContextUtils.getWebApplicationContext(getServletContext());
		WebApplicationContext wac = null;

		if (this.webApplicationContext != null) {
			// A context instance was injected at construction time -> use it
			wac = this.webApplicationContext;
			if (wac instanceof ConfigurableWebApplicationContext) {
				ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) wac;
				if (!cwac.isActive()) {
					// The context has not yet been refreshed -> provide services such as
					// setting the parent context, setting the application context id, etc
					if (cwac.getParent() == null) {
						// The context instance was injected without an explicit parent -> set
						// the root application context (if any; may be null) as the parent
						cwac.setParent(rootContext);
					}
					configureAndRefreshWebApplicationContext(cwac);
				}
			}
		}
		if (wac == null) {
		    //2、查找已经绑定的上下文
			wac = findWebApplicationContext();
		}
		if (wac == null) {
		    //3、如果没有找到相应的上下文，并指定父亲为ContextLoaderListener
		    //配置文件中注入的bean（Controller）都在这个方法下加载（更具体在AbstractApplicationContext中refresh（）方法的finishBeanFactoryInitialization(beanFactory)这一行）
			wac = createWebApplicationContext(rootContext);
		}

		if (!this.refreshEventReceived) {
			onRefresh(wac);
		}

		if (this.publishContext) {
			String attrName = getServletContextAttributeName();
			getServletContext().setAttribute(attrName, wac);
			if (this.logger.isDebugEnabled()) {
				this.logger.debug("Published WebApplicationContext of servlet '" + getServletName() +
						"' as ServletContext attribute with name [" + attrName + "]");
			}
		}

		return wac;
	}
```

`onRefresh(wac)`之前的`configureAndRefreshWebApplicationContext`、 `wac = findWebApplicationContext()`、`wac = createWebApplicationContext(rootContext)`方法主要是对`WebApplicationContext`的寻找创建，打断点可知`DispatcherServlet`对应配置文件中的`Bean`在`createWebApplicationContext(rootContext)`这一步中实例化（`Controller`，自定义的`handlerMapping`、`Spring`自动加载的`Handlermapping`等）。不过这里主要看`onRefresh()`方法：

```java
protected void onRefresh(ApplicationContext context) {
    this.initStrategies(context);
}

protected void initStrategies(ApplicationContext context) {
    this.initMultipartResolver(context);
    this.initLocaleResolver(context);
    this.initThemeResolver(context);
    this.initHandlerMappings(context);
    this.initHandlerAdapters(context);
    this.initHandlerExceptionResolvers(context);
    this.initRequestToViewNameTranslator(context);
    this.initViewResolvers(context);
    this.initFlashMapManager(context);
}
```
看到这个方法中那些熟悉的名词有点感动... 这个方法可以看到是对`Spring mvc`中用到的重要类的加载。

```java
// 用于文件上传
private void initMultipartResolver(ApplicationContext context) {
	try {
	    //Spring不会默认创建MultipartResolver对象，需要在`xml`文件中自定义
		this.multipartResolver = context.getBean(MULTIPART_RESOLVER_BEAN_NAME, MultipartResolver.class);
		if (logger.isDebugEnabled()) {
			logger.debug("Using MultipartResolver [" + this.multipartResolver + "]");
		}
	}
	catch (NoSuchBeanDefinitionException ex) {
		// Default is no multipart resolver.
		this.multipartResolver = null;
		if (logger.isDebugEnabled()) {
			logger.debug("Unable to locate MultipartResolver with name '" + MULTIPART_RESOLVER_BEAN_NAME +
					"': no multipart request handling provided");
		}
	}
}

// 用于文件上传
private void initLocaleResolver(ApplicationContext context) {
	try {
	    //Spring不会默认创建LocalResolver对象，需要在`xml`文件中自定义
		this.localeResolver = context.getBean(LOCALE_RESOLVER_BEAN_NAME, LocaleResolver.class);
		if (logger.isDebugEnabled()) {
			logger.debug("Using LocaleResolver [" + this.localeResolver + "]");
		}
	}
	catch (NoSuchBeanDefinitionException ex) {
		// We need to use the default.
		this.localeResolver = getDefaultStrategy(context, LocaleResolver.class);
		if (logger.isDebugEnabled()) {
			logger.debug("Unable to locate LocaleResolver with name '" + LOCALE_RESOLVER_BEAN_NAME +
					"': using default [" + this.localeResolver + "]");
		}
	}
}

//主题解析器
private void initThemeResolver(ApplicationContext context) {
	try {
	    // Spring不会默认创建ThemeResolver对象，需要在`xml`文件中自定义
		this.themeResolver = context.getBean(THEME_RESOLVER_BEAN_NAME, ThemeResolver.class);
		if (logger.isDebugEnabled()) {
			logger.debug("Using ThemeResolver [" + this.themeResolver + "]");
		}
	}
	catch (NoSuchBeanDefinitionException ex) {
		// We need to use the default.
		this.themeResolver = getDefaultStrategy(context, ThemeResolver.class);
		if (logger.isDebugEnabled()) {
			logger.debug("Unable to locate ThemeResolver with name '" + THEME_RESOLVER_BEAN_NAME +
					"': using default [" + this.themeResolver + "]");
		}
	}
}

//HandlerMapping对象
private void initHandlerMappings(ApplicationContext context) {
	this.handlerMappings = null;

    //Spring 默认会加载RequestMappingHandlerMapping， SimpleUrlHandlerMapping， BeanNameUrlHandlerMapping
	if (this.detectAllHandlerMappings) {
		// Find all HandlerMappings in the ApplicationContext, including ancestor contexts.
		Map<String, HandlerMapping> matchingBeans =
				BeanFactoryUtils.beansOfTypeIncludingAncestors(context, HandlerMapping.class, true, false);
		if (!matchingBeans.isEmpty()) {
			this.handlerMappings = new ArrayList<HandlerMapping>(matchingBeans.values());
			// We keep HandlerMappings in sorted order.
			AnnotationAwareOrderComparator.sort(this.handlerMappings);
		}
	}
	else {
	    // 在web.xml中设置了detectAllHandlerMappings为false,则会去加载用户自定义的handlerMapping
		try {
			HandlerMapping hm = context.getBean(HANDLER_MAPPING_BEAN_NAME, HandlerMapping.class);
			this.handlerMappings = Collections.singletonList(hm);
		}
		catch (NoSuchBeanDefinitionException ex) {
			// Ignore, we'll add a default HandlerMapping later.
		}
	}

	//在web.xml中设置了detectAllHandlerMappings为false，且没有在xml文件中自定义handlermapping则会去根据DispatcherServlet目录下的DispatcherServlet.properties文件去创建对应DispatcherServlet
	if (this.handlerMappings == null) {
		this.handlerMappings = getDefaultStrategies(context, HandlerMapping.class);
		if (logger.isDebugEnabled()) {
			logger.debug("No HandlerMappings found in servlet '" + getServletName() + "': using default");
		}
	}
}

// HandlerAdapter对象，加载原理同handlerMapping
private void initHandlerAdapters(ApplicationContext context) {
	this.handlerAdapters = null;
    
    //Spring 会默认加载RequestMappingHandlerAdapter、HttpRequestHandlerAdapter、SimpleControllerHandlerAdapter
	if (this.detectAllHandlerAdapters) {
		// Find all HandlerAdapters in the ApplicationContext, including ancestor contexts.
		Map<String, HandlerAdapter> matchingBeans =
				BeanFactoryUtils.beansOfTypeIncludingAncestors(context, HandlerAdapter.class, true, false);
		if (!matchingBeans.isEmpty()) {
			this.handlerAdapters = new ArrayList<HandlerAdapter>(matchingBeans.values());
			// We keep HandlerAdapters in sorted order.
			AnnotationAwareOrderComparator.sort(this.handlerAdapters);
		}
	}
	else {
		try {
			HandlerAdapter ha = context.getBean(HANDLER_ADAPTER_BEAN_NAME, HandlerAdapter.class);
			this.handlerAdapters = Collections.singletonList(ha);
		}
		catch (NoSuchBeanDefinitionException ex) {
			// Ignore, we'll add a default HandlerAdapter later.
		}
	}

	// Ensure we have at least some HandlerAdapters, by registering
	// default HandlerAdapters if no other adapters are found.
	if (this.handlerAdapters == null) {
		this.handlerAdapters = getDefaultStrategies(context, HandlerAdapter.class);
		if (logger.isDebugEnabled()) {
			logger.debug("No HandlerAdapters found in servlet '" + getServletName() + "': using default");
		}
	}
}


private void initHandlerExceptionResolvers(ApplicationContext context) {
	this.handlerExceptionResolvers = null;

    //默认加载ExceptionHandlerExceptionResolver、 ResponseStatusExceptionResolver、DefaultHandlerExceptionResolver
	if (this.detectAllHandlerExceptionResolvers) {
		// Find all HandlerExceptionResolvers in the ApplicationContext, including ancestor contexts.
		Map<String, HandlerExceptionResolver> matchingBeans = BeanFactoryUtils
				.beansOfTypeIncludingAncestors(context, HandlerExceptionResolver.class, true, false);
		if (!matchingBeans.isEmpty()) {
			this.handlerExceptionResolvers = new ArrayList<HandlerExceptionResolver>(matchingBeans.values());
			// We keep HandlerExceptionResolvers in sorted order.
			AnnotationAwareOrderComparator.sort(this.handlerExceptionResolvers);
		}
	}
	else {
		try {
			HandlerExceptionResolver her =
					context.getBean(HANDLER_EXCEPTION_RESOLVER_BEAN_NAME, HandlerExceptionResolver.class);
			this.handlerExceptionResolvers = Collections.singletonList(her);
		}
		catch (NoSuchBeanDefinitionException ex) {
			// Ignore, no HandlerExceptionResolver is fine too.
		}
	}

	// Ensure we have at least some HandlerExceptionResolvers, by registering
	// default HandlerExceptionResolvers if no other resolvers are found.
	if (this.handlerExceptionResolvers == null) {
		this.handlerExceptionResolvers = getDefaultStrategies(context, HandlerExceptionResolver.class);
		if (logger.isDebugEnabled()) {
			logger.debug("No HandlerExceptionResolvers found in servlet '" + getServletName() + "': using default");
		}
	}
}


private void initRequestToViewNameTranslator(ApplicationContext context) {
	try {
	    // Spring不会默认创建RequestToViewNameTranslator对象，需要在`xml`文件中自定义
		this.viewNameTranslator =
				context.getBean(REQUEST_TO_VIEW_NAME_TRANSLATOR_BEAN_NAME, RequestToViewNameTranslator.class);
		if (logger.isDebugEnabled()) {
			logger.debug("Using RequestToViewNameTranslator [" + this.viewNameTranslator + "]");
		}
	}
	catch (NoSuchBeanDefinitionException ex) {
		// We need to use the default.
		this.viewNameTranslator = getDefaultStrategy(context, RequestToViewNameTranslator.class);
		if (logger.isDebugEnabled()) {
			logger.debug("Unable to locate RequestToViewNameTranslator with name '" +
					REQUEST_TO_VIEW_NAME_TRANSLATOR_BEAN_NAME + "': using default [" + this.viewNameTranslator +
					"]");
		}
	}
}


//ViewResolver对象
private void initViewResolvers(ApplicationContext context) {
	this.viewResolvers = null;

    // Spring 默认不会创建viewResolver对象， 需要自己在web.xml中创建
	if (this.detectAllViewResolvers) {
		// Find all ViewResolvers in the ApplicationContext, including ancestor contexts.
		Map<String, ViewResolver> matchingBeans =
				BeanFactoryUtils.beansOfTypeIncludingAncestors(context, ViewResolver.class, true, false);
		if (!matchingBeans.isEmpty()) {
			this.viewResolvers = new ArrayList<ViewResolver>(matchingBeans.values());
			// We keep ViewResolvers in sorted order.
			AnnotationAwareOrderComparator.sort(this.viewResolvers);
		}
	}
	else {
		try {
			ViewResolver vr = context.getBean(VIEW_RESOLVER_BEAN_NAME, ViewResolver.class);
			this.viewResolvers = Collections.singletonList(vr);
		}
		catch (NoSuchBeanDefinitionException ex) {
			// Ignore, we'll add a default ViewResolver later.
		}
	}

    //没有在xml文件中自定义ViewResolver,则会去根据DispatcherServlet目录下的DispatcherServlet.properties文件去创建对应ViewResolver
	// Ensure we have at least one ViewResolver, by registering
	// a default ViewResolver if no other resolvers are found.
	if (this.viewResolvers == null) {
		this.viewResolvers = getDefaultStrategies(context, ViewResolver.class);
		if (logger.isDebugEnabled()) {
			logger.debug("No ViewResolvers found in servlet '" + getServletName() + "': using default");
		}
	}
}


private void initFlashMapManager(ApplicationContext context) {
	try {
		this.flashMapManager = context.getBean(FLASH_MAP_MANAGER_BEAN_NAME, FlashMapManager.class);
		if (logger.isDebugEnabled()) {
			logger.debug("Using FlashMapManager [" + this.flashMapManager + "]");
		}
	}
	catch (NoSuchBeanDefinitionException ex) {
		// We need to use the default.
		this.flashMapManager = getDefaultStrategy(context, FlashMapManager.class);
		if (logger.isDebugEnabled()) {
			logger.debug("Unable to locate FlashMapManager with name '" +
					FLASH_MAP_MANAGER_BEAN_NAME + "': using default [" + this.flashMapManager + "]");
		}
	}
}
```

### DispatcherServlet处理请求

`DispatcherServlet`初始化之后，项目中的请求都会通过这个`Servlet`来处理，如`HttpServlet`中的`doGet`、`doPost`、`doDelete`、`doOptions`、`doTrace`。`DispatcherServlet`中的这些方法在父类`FramworkServlet`中实现：

```java
protected final void doGet(HttpServletRequest request, HttpServletResponse response)
		throws ServletException, IOException {
	processRequest(request, response);
}

protected final void doPost(HttpServletRequest request, HttpServletResponse response)
		throws ServletException, IOException {
	processRequest(request, response);
}

protected final void doPut(HttpServletRequest request, HttpServletResponse response)
		throws ServletException, IOException {
	processRequest(request, response);
}

protected final void doDelete(HttpServletRequest request, HttpServletResponse response)
		throws ServletException, IOException {
	processRequest(request, response);
}
```

都是交由`processRequest`方法处理：
```java
protected final void processRequest(HttpServletRequest request, HttpServletResponse response)
		throws ServletException, IOException {

	long startTime = System.currentTimeMillis();
	Throwable failureCause = null;

	LocaleContext previousLocaleContext = LocaleContextHolder.getLocaleContext();
	LocaleContext localeContext = buildLocaleContext(request);

	RequestAttributes previousAttributes = RequestContextHolder.getRequestAttributes();
	ServletRequestAttributes requestAttributes = buildRequestAttributes(request, response, previousAttributes);

	WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
	asyncManager.registerCallableInterceptor(FrameworkServlet.class.getName(), new RequestBindingInterceptor());

	initContextHolders(request, localeContext, requestAttributes);

	try {
		doService(request, response);
	}
	catch (ServletException ex) {
		failureCause = ex;
		throw ex;
	}
	catch (IOException ex) {
		failureCause = ex;
		throw ex;
	}
	catch (Throwable ex) {
		failureCause = ex;
		throw new NestedServletException("Request processing failed", ex);
	}

	finally {
		resetContextHolders(request, previousLocaleContext, previousAttributes);
		if (requestAttributes != null) {
			requestAttributes.requestCompleted();
		}

		if (logger.isDebugEnabled()) {
			if (failureCause != null) {
				this.logger.debug("Could not complete request", failureCause);
			}
			else {
				if (asyncManager.isConcurrentHandlingStarted()) {
					logger.debug("Leaving response open for concurrent processing");
				}
				else {
					this.logger.debug("Successfully completed request");
				}
			}
		}

		publishRequestHandledEvent(request, response, startTime, failureCause);
	}
}
```
可以看到最主要的方法是`doService`, 之前的那些方法主要是将当前请求的`LocalContenxt`和`RequestAttributes`与当前线程进行绑定，使请求结束后仍能继续访问。继续看`doService()`方法：
```java
protected void doService(HttpServletRequest request, HttpServletResponse response) throws Exception {
	if (logger.isDebugEnabled()) {
		String resumed = WebAsyncUtils.getAsyncManager(request).hasConcurrentResult() ? " resumed" : "";
		logger.debug("DispatcherServlet with name '" + getServletName() + "'" + resumed +
				" processing " + request.getMethod() + " request for [" + getRequestUri(request) + "]");
	}

	// Keep a snapshot of the request attributes in case of an include,
	// to be able to restore the original attributes after the include.
	Map<String, Object> attributesSnapshot = null;
	if (WebUtils.isIncludeRequest(request)) {
		attributesSnapshot = new HashMap<String, Object>();
		Enumeration<?> attrNames = request.getAttributeNames();
		while (attrNames.hasMoreElements()) {
			String attrName = (String) attrNames.nextElement();
			if (this.cleanupAfterInclude || attrName.startsWith(DEFAULT_STRATEGIES_PREFIX)) {
				attributesSnapshot.put(attrName, request.getAttribute(attrName));
			}
		}
	}

	// Make framework objects available to handlers and view objects.
	request.setAttribute(WEB_APPLICATION_CONTEXT_ATTRIBUTE, getWebApplicationContext());
	request.setAttribute(LOCALE_RESOLVER_ATTRIBUTE, this.localeResolver);
	request.setAttribute(THEME_RESOLVER_ATTRIBUTE, this.themeResolver);
	request.setAttribute(THEME_SOURCE_ATTRIBUTE, getThemeSource());

	FlashMap inputFlashMap = this.flashMapManager.retrieveAndUpdate(request, response);
	if (inputFlashMap != null) {
		request.setAttribute(INPUT_FLASH_MAP_ATTRIBUTE, Collections.unmodifiableMap(inputFlashMap));
	}
	request.setAttribute(OUTPUT_FLASH_MAP_ATTRIBUTE, new FlashMap());
	request.setAttribute(FLASH_MAP_MANAGER_ATTRIBUTE, this.flashMapManager);

	try {
		doDispatch(request, response);
	}
	finally {
		if (!WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted()) {
			// Restore the original attribute snapshot, in case of an include.
			if (attributesSnapshot != null) {
				restoreAttributesAfterInclude(request, attributesSnapshot);
			}
		}
	}
}
```
这个方法任然是做一些准备工作，将`WebApplicationContext`、`LocalResolver`、`ThemeResolver`对象存储至`request`中，供接下来的方法调用。主要的处理逻辑还是在`doDispatch`中，这个方法中编写了完整的请求处理过程：
```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
	HttpServletRequest processedRequest = request;
	HandlerExecutionChain mappedHandler = null;
	boolean multipartRequestParsed = false;

	WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

	try {
		ModelAndView mv = null;
		Exception dispatchException = null;

		try {
			processedRequest = checkMultipart(request);
			multipartRequestParsed = (processedRequest != request);

			// 利用handlerMapping去查找对应请求的地址由哪个方法处理
			mappedHandler = getHandler(processedRequest);
			// mappedHandler.getHandler()返回HandlerM
			if (mappedHandler == null || mappedHandler.getHandler() == null) {
				noHandlerFound(processedRequest, response);
				return;
			}

			// web中前后端的请求基本返回RequestMappingHandlerAdapter
			HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

			// Process last-modified header, if supported by the handler.
			String method = request.getMethod();
			boolean isGet = "GET".equals(method);
			if (isGet || "HEAD".equals(method)) {
				long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
				if (logger.isDebugEnabled()) {
					logger.debug("Last-Modified value for [" + getRequestUri(request) + "] is: " + lastModified);
				}
				if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
					return;
				}
			}

			if (!mappedHandler.applyPreHandle(processedRequest, response)) {
				return;
			}

			// Actually invoke the handler.
			mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

			if (asyncManager.isConcurrentHandlingStarted()) {
				return;
			}

			applyDefaultViewName(processedRequest, mv);
			mappedHandler.applyPostHandle(processedRequest, response, mv);
		}
		catch (Exception ex) {
			dispatchException = ex;
		}
		catch (Throwable err) {
			// As of 4.3, we're processing Errors thrown from handler methods as well,
			// making them available for @ExceptionHandler methods and other scenarios.
			dispatchException = new NestedServletException("Handler dispatch failed", err);
		}
		processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
	}
	catch (Exception ex) {
		triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
	}
	catch (Throwable err) {
		triggerAfterCompletion(processedRequest, response, mappedHandler,
				new NestedServletException("Handler processing failed", err));
	}
	finally {
		if (asyncManager.isConcurrentHandlingStarted()) {
			// Instead of postHandle and afterCompletion
			if (mappedHandler != null) {
				mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
			}
		}
		else {
			// Clean up any resources used by a multipart request.
			if (multipartRequestParsed) {
				cleanupMultipart(processedRequest);
			}
		}
	}
}
```

根据以上代码： `HandlerMapping`作用是根据请求url查找对应的处理方法，并根据`Controller`类以及方法生成`HandlerMethod`对象。而`HandlerAdapter`则是根据`HandlerMethod`对象去调用实际处理的方法。





  [1]: http://static.zybuluo.com/hewei0928/6lnuj5o6h1tu9dm2xiw8k4je/image_1cfrii75e1m5fg22aj7svgv6a9.png