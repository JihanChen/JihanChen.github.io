---
layout: postlayout
title: 自定义参数解析
categories: [springboot]
tags: [springmvc,springboot]
---

Springmvc为我们提供了一系列的参数解析器，不管你是要获取Cookie中的值，Header中的值，JSON格式的数据，URI中的值。

## 起源
在开发过程中我们会遇到这样的需求

1.app终端请求服务端，把app版本信息和终端类型存放在http请求的Header里，我们需要在controller层获取到多个Header，有没有可能直接把Header的这些数据信息直接整合到客户端的请求参数里（post的body或者request的param请求连接上）

 2.在使用shiro 的后端管理系统中，我们很多时候需要获取当前用户的信息，比如user_id，username等信息，  比如这样获取SecurityUtils.getSubject().getSession().getAttribute("currentUserId") ，这样总觉得很麻烦，有没有可能直接敷于一个User 对象，然后controller层拿到之后后续都可以操作



## 发现

	在SpringMVC中其实就有相应的解决方法，我们在controller中获取请求参数用的最多的 @RequestParam、@RequestHeader、@RequestBody、@PathVariable、@ModelAttribute，使用了这些注解定义参数就可以获取对应的请求数据，为什么使用这些注解就能拿到值了呢？



其实springmvc针对每一个注解都有一个对应的 MethodArgumentResolver：

@RequestHeader  		    RequestHeaderMapMethodArgumentResolver处理器

@RequestBody @Response   RequestResponseBodyMethodProcessor处理器

@ModelAttribute  		   ModelAttributeMethodProcessor处理器

@PathVariable   			   PathVariableMapMethodArgumentResolver处理器

@RequestParam 			   RequestParamMapMethodArgumentResolver处理器

下面我们通过查看RequestHeaderMapMethodArgumentResolver类来查看springmvc 是如何对参数进行注入数据的。



```java
public class RequestHeaderMapMethodArgumentResolver implements HandlerMethodArgumentResolver {

@Override
public boolean supportsParameter(MethodParameter parameter) {
	return (parameter.hasParameterAnnotation(RequestHeader.class) &&
			Map.class.isAssignableFrom(parameter.getParameterType()));
}

@Override
public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer,
		NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {

	Class<?> paramType = parameter.getParameterType();
	if (MultiValueMap.class.isAssignableFrom(paramType)) {
		MultiValueMap<String, String> result;
		if (HttpHeaders.class.isAssignableFrom(paramType)) {
			result = new HttpHeaders();
		}
		else {
			result = new LinkedMultiValueMap<String, String>();
		}
		for (Iterator<String> iterator = webRequest.getHeaderNames(); iterator.hasNext();) {
			String headerName = iterator.next();
			String[] headerValues = webRequest.getHeaderValues(headerName);
			if (headerValues != null) {
				for (String headerValue : headerValues) {
					result.add(headerName, headerValue);
				}
			}
		}
		return result;
	}
	else {
		Map<String, String> result = new LinkedHashMap<String, String>();
		for (Iterator<String> iterator = webRequest.getHeaderNames(); iterator.hasNext();) {
			String headerName = iterator.next();
			String headerValue = webRequest.getHeader(headerName);
			if (headerValue != null) {
				result.put(headerName, headerValue);
			}
		}
		return result;
	}
}
}
```

我们可以看到RequestHeaderMapMethodArgumentResolver实现了HandlerMethodArgumentResolver接口

```java
public interface HandlerMethodArgumentResolver {

 boolean supportsParameter(MethodParameter parameter);

 Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer,
		NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws   Exception;

}
```

`HandlerMethodArgumentResolver` 其实就是将 `HttpServletRequest`中的header 和body解析为HandlerMethod方法的参数的解析器，而两个方法`supportsParameter`和`resolveArgument`扮演着重要作用，`supportsParameter` 判断 `HandlerMethodArgumentResolver` 是否支持 `MethodParameter`，可以理解为条件入口，满足就可以进入下面`resolveArgument`方法中，`resolveArgument`才是真正的处理参数分解的方法，返回的Object就是controller方法上的形参对象。



我们重新回到`RequestHeaderMapMethodArgumentResolver` 中，其`supportsParameter`方法判断是否有`@RequestHeader` 注解，并且判断方法参数的类型是否Map或Map子类，同时满足返回true。

```java
public boolean supportsParameter(MethodParameter parameter) {
	return (parameter.hasParameterAnnotation(RequestHeader.class) &&
	Map.class.isAssignableFrom(parameter.getParameterType()));
}
```

我们有些时候获取 request请求的Header 快速的获取其实就是在controller的方法上使用注解`@RequestHeader`  ，然后定义参数名称获取数据

```java
    @PostMapping("/getHeaders")
    @ResponseBody
    public String getHeader(@RequestHeader("headers") Map<String,String> headers){
        // 获取版本号
        String version = headers.get("version");
        // 返回结果
        return version;
    }
```



我们具体来看看注解了 @RequestHeader 后`RequestHeaderMapMethodArgumentResolver` 到底做了什么处理，让我们直接通过 version 就可以拿到数据。

针对注解了`@RequestHeader` 并且是Map或其子类的方法，`supportsParameter`返回true，然后将调用`resolveArgument`方法。

我们通过一个定义了方法参数类型是Map，根据上面的代码将进入else里面

```java
// 定义返回结果
Map<String, String> result = new LinkedHashMap<String, String>();
// 遍历获取request请求中的 header，遍历名称获取对应的数值
for (Iterator<String> iterator = webRequest.getHeaderNames(); iterator.hasNext();) {
	String headerName = iterator.next();
	String headerValue = webRequest.getHeader(headerName);
	if (headerValue != null) {
	result.put(headerName, headerValue);
	}
}
return result;
```



我们进入`webRequest`类中可以看到 `getHeaderNames` 方法，注释上备注说返回一个迭代器，里面存放了request请求的所有header名称。

```java
/**
 * Return a Iterator over request header names.
 * @since 3.0
 * @see javax.servlet.http.HttpServletRequest#getHeaderNames()
 */
Iterator<String> getHeaderNames();
```

通过转化对象到`LinkedHashMap`中，key等于header的名称，value等于header的具体数值，返最终回了一个`LinkedHashMap` 。

这样我们就可以在contrller 方法中通过@RequestHeader("headers") Map<String,String> headers 

获取所有的 headers。



## 解决

通过上面`RequestHeaderMapMethodArgumentResolver` 解析器，我们也可以自定义一个参数解析器，来解决上面两个场景的业务。

编写`DeviceDataHandlerMethodArgumentResolver` 用来处理app 的版本信息

```java
public class DeviceDataHandlerMethodArgumentResolver implements HandlerMethodArgumentResolver {
    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return parameter.getParameterType().equals(DeviceData.class);
    }
    @Override
    public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer, NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {
        String appType = webRequest.getHeader("type");
        String appVersion = webRequest.getHeader("version");
        return new DeviceData(appType, appVersion);
    }
}
```



编写`CurrentUserMethodArgumentResolver` 用来处理当前登录用户的信息

```java
public class CurrentUserMethodArgumentResolver implements HandlerMethodArgumentResolver {
    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        if (parameter.hasParameterAnnotation(CurrentUser.class)) {
            return true;
        }
        return false;
    }
    @Override
    public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer, NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {
        Session session = SecurityUtils.getSubject().getSession();
        // 获取session中的用户信息
        Object currentUser = session.getAttribute("current_user");
        return currentUser;
    }
}
```



springboot 中我们将写好的具体实现类`DeviceDataHandlerMethodArgumentResolver` 和`CurrentUserMethodArgumentResolver` 配置到springmvc的 WebMvcConfigurerAdapter

```java
@Configuration
public class WebParameterConfig extends WebMvcConfigurerAdapter {
	@Override
	public void addArgumentResolvers(List<HandlerMethodArgumentResolver> 	argumentResolvers) {
	argumentResolvers.add(new DeviceDataHandlerMethodArgumentResolver());
    argumentResolvers.add(new CurrentUserMethodArgumentResolver());
  }
}
```



非boot项目springmvc 配置文件方式可以在spring-mvc.xml文件里配置如下

```java
<mvc:annotation-driven>
   <mvc:argument-resolvers>
        <bean class="com.test.bind.method.DeviceDataHandlerMethodArgumentResolver"/>
        <bean class="com.test.bind.method.CurrentUserMethodArgumentResolver"/>
    </mvc:argument-resolvers>
</mvc:annotation-driven>
```

最后在controller中获取对应的数据

```java
@PostMapping("/getHeaders")
@ResponseBody
public String getHeader(DeviceData deviceData){
    // 获取版本号
    String version = deviceData.getAppVersion()
    // 返回结果
    return version;
}
```

```java
@PostMapping("/getCurrnentUser")
@ResponseBody
public User getCurrnentUser(@CurrentUser User user){
    return user;
}
```



## 结尾

通过上面的实现是不是获取一些数据变得非常方便，代码也变得简洁，springmvc 不仅提供了入参的解析注入，还提供了出参的解析注入，我们可以很灵活的进行一系列的灵活操作。


