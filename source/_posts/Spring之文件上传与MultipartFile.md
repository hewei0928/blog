---
title: Spring之文件上传与MultipartFile
date: 2018-06-20 09:44:46
tags:
    - Java
    - Spring
---

> Spring 文件上传使用的总结及MultipartFile的解析

<!--more-->

# 文件上传

## 前后台交互

### 使用MultipartFile参数接收：

#### 单文件上传
1. 前端`<form>`标签
```html
<form action="/UploadOneServlet.do" method="post" name="f_upload" enctype="multipart/form-data">
	<input type="file" name="multipartFile" />
	<input type="submit" value="上传" />
</form>
```
在`<form>`标签中添加`enctype="multipart/form-data"`,即可实现`multipart`类型数据上传。

2. 后台接口
```java
@RequestMapping(value = "/UploadOneServlet.do", method = RequestMethod.POST)
@ResponseBody
public String upload(@RequestParam("multipartFile") MultipartFile multipartFile) {
    //...文件处理代码
}
```
后端中的对应方法中使用`MultipartFile`类型参数进行接收。必须添加`@RequestParam("multipartFile")`参数，否则报错。`@RequestParam`的`value`与前端`imput`标签中的`name`对应。

#### 一个input标签上传多个文件

1. 前端`<form>`标签
```html
<form action="/UploadOneServlet.do" method="post" name="f_upload" enctype="multipart/form-data">
	<input type="file" name="multipartFiles" multiple/>
	<input type="submit" value="上传" />
</form>
```
在`<input>`标签中添加`multiple`参数即可在一个框中选择多个文件上传
2. 后台接口
```java
@RequestMapping(value = "/UploadOneServlet.do", method = RequestMethod.POST)
@ResponseBody
public String upload(@RequestParam("multipartFile") List<MultipartFile> multipartFile) {
    //...文件处理代码
}
```
后台使用`List`,即可前台上传的多个文件。如果使用单个`MultipartFile`类型进行接收则只能接收一个文件。

#### 多个name相同的单文件input标签

1. 前端`<form>`标签
```html
<form action="/UploadOneServlet.do" method="post" name="f_upload" enctype="multipart/form-data">
	<input type="file" name="multipartFiles" />
	<input type="file" name="multipartFiles" />
	<input type="file" name="multipartFiles" />
	<input type="submit" value="上传" />
</form>
```
前台的多个`<input>`标签的`name`相同
2. 后台接口
```java
@RequestMapping(value = "/UploadOneServlet.do", method = RequestMethod.POST)
@ResponseBody
public String upload(@RequestParam("multipartFiles") List<MultipartFile> multipartFiles) {
}
```
后台接收方法与**一个input标签上传多个文件**时相同，使用`List<MultipartFile>`进行接收。`@RequestParam`的`value`与前端`imput`标签中的`name`对应。

#### 混合使用
1. 前端`<form>`标签
```html
<form action="/UploadOneServlet.do" method="post" name="f_upload" enctype="multipart/form-data">
	<input type="file" name="multipartFiles" />
	<input type="file" name="multipartFile" multiple/>
	<input type="submit" value="上传" />
</form>
```
前台使用`name`不同的`<input>`标签上传文件

2. 后台接口
```java
@RequestMapping(value = "/UploadOneServlet.do", method = RequestMethod.POST)
@ResponseBody
public String upload(@RequestParam("multipartFile") List<MultipartFile> multipartFiles, @RequestParam("multipartFiles") MultipartFile multipartFile) {
}
```
后台使用一个`List<MultiPartFile>`类型和`MultiPartFile`类型进行接收。`@RequestParam`的`value`与前端`imput`标签中的`name`一一对应。


### 使用HttpServletRequest接收
1. 前端`<input>`标签
```html
<form action="/UploadOneServlet.do" method="post" name="f_upload" enctype="multipart/form-data">
	<input type="file" name="multipartFiles" />
	<input type="file" name="multipartFile" multiple/>
	<input type="submit" value="上传" />
</form>
```
2. 后台接口
```java
@RequestMapping(value = "/UploadOneServlet.do", method = RequestMethod.POST)
@ResponseBody
public String upload(HttpServletRequest request){
	List<MultipartFile> files = new ArrayList<>();
	MultipartHttpServletRequest multipartHttpServletRequest = (MultipartHttpServletRequest) request;
	Iterator<String> a = multipartHttpServletRequest.getFileNames();//返回的数量与前端input数量相同, 返回的字符串即为前端input标签的name
	while (a.hasNext()) {
		String name = a.next();
		List<MultipartFile> multipartFiles = multipartHttpServletRequest.getFiles(name);//获取单个input标签上传的文件，可能为多个
		files.addAll(multipartFiles);
	}
	upload(files);
	return "success";
}
```

## 文件上传

之前的几种情况已经接收到了前台上传的`List<MultipartFile>`文件集合，遍历集合实现文件上传
```java
private void upload(List<MultipartFile> multipartFiles) throws Exception{
	for (MultipartFile multipartFile : multipartFiles) {
		String fileName = multipartFile.getOriginalFilename();
		String filePath = ContextLoader.getCurrentWebApplicationContext().getServletContext().getRealPath("/") + "fileUpload";
		String fileTotalName = filePath + File.separator + fileName;
		File f = new File(fileTotalName);
		multipartFile.transferTo(f);
	}
}
```

# MultiPartFile类解析

`Spring MVC`中要使用`MultipartFile`进行文件上传，需要先注入`MultipartResolver`类型的Bean:
```xml
<!-- 支持上传文件 -->
<bean id="multipartResolver"
	class="org.springframework.web.multipart.commons.CommonsMultipartResolver">
	<!-- 设置上传文件的最大尺寸为50MB -->
	<property name="maxUploadSize">
		<value>52428800</value>
	</property>
</bean>
```

之前解析`DispatcherServlet`类时得知所有最终会调用这个类中的`doDispatch`方法对request进行处理：
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

				// Determine handler for the current request.
				mappedHandler = getHandler(processedRequest);
				if (mappedHandler == null || mappedHandler.getHandler() == null) {
					noHandlerFound(processedRequest, response);
					return;
				}
				//...剩余代码
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

可以看到`processedRequest = checkMultipart(request)`对reuqest进行判断：
```java
protected HttpServletRequest checkMultipart(HttpServletRequest request) throws MultipartException {
	if (this.multipartResolver != null && this.multipartResolver.isMultipart(request)) {
		if (WebUtils.getNativeRequest(request, MultipartHttpServletRequest.class) != null) {
			logger.debug("Request is already a MultipartHttpServletRequest - if not in a forward, " +
					"this typically results from an additional MultipartFilter in web.xml");
		}
		else if (hasMultipartException(request) ) {
			logger.debug("Multipart resolution failed for current request before - " +
					"skipping re-resolution for undisturbed error rendering");
		}
		else {
			try {
				return this.multipartResolver.resolveMultipart(request);
			}
			catch (MultipartException ex) {
				if (request.getAttribute(WebUtils.ERROR_EXCEPTION_ATTRIBUTE) != null) {
					logger.debug("Multipart resolution failed for error dispatch", ex);
				}
				else {
					throw ex;
				}
			}
		}
	}
	return request;
}
```
通过以上方法可以看到以下逻辑：
1. 判断用户是否注入了`MultipartResolver`类型的Bean,如果注入了则使用它去检测当前请求是否为文件上传请求
2. 如果为文件上传请求，就通过`MultipartResolver`去处理当前请求。
3. 返回处理后的请求或为处理的请求。

### 文件上传请求判断

接下来看看`Spring`是如何判断当前请求是文件上传请求的, 这里主要看`CommonsMultipartResolver`：
```java
public boolean isMultipart(HttpServletRequest request) {
	return (request != null && ServletFileUpload.isMultipartContent(request));
}
```

这里直接使用了`Apache`中的`ServletFileUpload`:
```java
public static final boolean isMultipartContent(
        HttpServletRequest request) {
    if (!POST_METHOD.equalsIgnoreCase(request.getMethod())) {
        return false;
    }
    return FileUploadBase.isMultipartContent(new ServletRequestContext(request));
}
```
这里可以看到request请求必须是POST类型，如果是，则用`FileUploadBase`进行进一步判断：
```java
public static final boolean isMultipartContent(RequestContext ctx) {
    String contentType = ctx.getContentType();
    if (contentType == null) {
        return false;
    }
    if (contentType.toLowerCase(Locale.ENGLISH).startsWith(MULTIPART)) {// MULTIPART = "multipart/"
        return true;
    }
    return false;
}
```
以上方法说明：
只有当当前请求的`contentType`是以"multipart/"开头的时候，才会将此请求当做文件上传的请求。

综上，`Spring`判断当前请求是否是文件上传请求主要有两个条件：
1. 当前请求为POST请求
2. 当前请求的`contextType`必须以"multipart/"开头
这正符合了前端`<form>`标签中的设置。

### 文件上传请求解析

如果是文件上传请求，则会调用`multipartResolver.resolveMultipart(request)`对请求进行解析：
```java
public MultipartHttpServletRequest resolveMultipart(final HttpServletRequest request) throws MultipartException {
    MultipartParsingResult parsingResult = parseRequest(request);
	return new DefaultMultipartHttpServletRequest(request, parsingResult.getMultipartFiles(),
    					parsingResult.getMultipartParameters(), parsingResult.getMultipartParameterContentTypes());
}
```

可以看到任然是调用parseRequest方法对request进行解析，并将结果存入`MultipartParsingResult`：

```java
protected MultipartParsingResult parseRequest(HttpServletRequest request) throws MultipartException {
	String encoding = determineEncoding(request);
	FileUpload fileUpload = prepareFileUpload(encoding);
	try {
	    // 获取到上传文件的信息 一个FileItem对应一个文件，FileItem的fieldName与input框的name对应
		List<FileItem> fileItems = ((ServletFileUpload) fileUpload).parseRequest(request);
		//将其存入MultipartParsingResult类中
		return parseFileItems(fileItems, encoding);
	}
	catch (FileUploadBase.SizeLimitExceededException ex) {
		throw new MaxUploadSizeExceededException(fileUpload.getSizeMax(), ex);
	}
	catch (FileUploadBase.FileSizeLimitExceededException ex) {
		throw new MaxUploadSizeExceededException(fileUpload.getFileSizeMax(), ex);
	}
	catch (FileUploadException ex) {
		throw new MultipartException("Failed to parse multipart servlet request", ex);
	}
}
```

```java
protected MultipartParsingResult parseFileItems(List<FileItem> fileItems, String encoding) {
		MultiValueMap<String, MultipartFile> multipartFiles = new LinkedMultiValueMap<String, MultipartFile>();
		Map<String, String[]> multipartParameters = new HashMap<String, String[]>();
		Map<String, String> multipartParameterContentTypes = new HashMap<String, String>();

		// Extract multipart files and multipart parameters.
		for (FileItem fileItem : fileItems) {
			if (fileItem.isFormField()) {
				//...省略代码
			}
			else {
				// multipart file field
				CommonsMultipartFile file = createMultipartFile(fileItem);
				// file.getName()得到fileItem.getfieldName  map键位input框的name, 值为具体文件类
				multipartFiles.add(file.getName(), file);
				if (logger.isDebugEnabled()) {
					logger.debug("Found multipart file [" + file.getName() + "] of size " + file.getSize() +
							" bytes with original filename [" + file.getOriginalFilename() + "], stored " +
							file.getStorageDescription());
				}
			}
		}
		return new MultipartParsingResult(multipartFiles, multipartParameters, multipartParameterContentTypes);
}
```

初始化了`MultipartParsingResult`对象之后就能根据该对象去初始化`MultipartHttpServletRequest`对象，供接口中调用。

