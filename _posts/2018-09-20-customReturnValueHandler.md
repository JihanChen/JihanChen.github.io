---
layout: postlayout
title: 统一处理返回结果
categories: [springboot]
tags: [springmvc,springboot]
---

Springmvc 不仅提供了请求参数解析入口，还提供了返回值的解析入口。我们可以通过扩展来实现个性化的功能需求。

## 起源
	现在越来越多的公司实施前后端分离方案，后端通过提供给前端接口的方式，数据传输采用json方式，一般都会有一个统一的格式，比如每个接口外层返回都带有errorCode，errorMsg，真正的数据在一个result里，这样就需要我们在controller 每次返回结果时候把service层处理完的结果放入一个Result中，说到这里可以发现我们需要在每个方法去包装这个一样的Result，那有没有让这个包装动作只写一次，框架自动包装呢？



## 思考

我们在controller层中返回json 数据给前端一般都是在方法上注解`@ResponseBody`，或者在controller类上使用注解`@RestController`，这样客户端就可以得到json 格式的数据。

比如下面的

```java
@Controller
@RequestMapping("/")
public class TestController {
    @PostMapping("/getHeaders")
    @ResponseBody
    public String getHeader(DeviceData deviceData){
        // 获取版本号
        String version = deviceData.getAppVersion();
        // 返回结果
        return version;
    }

}
```

或者

```java
@RestController
@RequestMapping("/rest")
public class TestRestController {
    @PostMapping("/getHeaders")
    public String getHeader(DeviceData deviceData){
        // 获取版本号
        String version = deviceData.getAppVersion();
        // 返回结果
        return version;
    }

}
```

既然springmvc 能通过`@ResponseBody` 对方法返回的结果进行处理，让其返回json 的格式，那我们应该也能通过其中类似方式去改下。上一篇  [自定义参数解析]({% post_url 2018-09-20-customMethodArgumentResolver %})   我们知道`@RequestBody` 是通过`RequestResponseBodyMethodProcessor` 处理器处理入参解析的，通过类名我们也能猜出这个类不仅处理request，还处理response的解析操作。带着猜想我们找到对应的`RequestResponseBodyMethodProcessor`类追究下原因。

```java
public class RequestResponseBodyMethodProcessor extends AbstractMessageConverterMethodProcessor {
   public RequestResponseBodyMethodProcessor(List<HttpMessageConverter<?>> converters) 		{
      super(converters);
   }

   public RequestResponseBodyMethodProcessor(List<HttpMessageConverter<?>> converters,
         ContentNegotiationManager manager) {
      super(converters, manager);
   }

   public RequestResponseBodyMethodProcessor(List<HttpMessageConverter<?>> converters,
         List<Object> requestResponseBodyAdvice) {

      super(converters, null, requestResponseBodyAdvice);
   }

   public RequestResponseBodyMethodProcessor(List<HttpMessageConverter<?>> converters,
         ContentNegotiationManager manager, List<Object> requestResponseBodyAdvice) {

      super(converters, manager, requestResponseBodyAdvice);
   }

   @Override
   public boolean supportsParameter(MethodParameter parameter) {
      return parameter.hasParameterAnnotation(RequestBody.class);
   }

   @Override
   public boolean supportsReturnType(MethodParameter returnType) {
      return (AnnotatedElementUtils.hasAnnotation(returnType.getContainingClass(), ResponseBody.class) ||
            returnType.hasMethodAnnotation(ResponseBody.class));
   }

   @Override
   public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer,
         NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {

      parameter = parameter.nestedIfOptional();
      Object arg = readWithMessageConverters(webRequest, parameter, parameter.getNestedGenericParameterType());
      String name = Conventions.getVariableNameForParameter(parameter);

      WebDataBinder binder = binderFactory.createBinder(webRequest, arg, name);
      if (arg != null) {
         validateIfApplicable(binder, parameter);
         if (binder.getBindingResult().hasErrors() && isBindExceptionRequired(binder, parameter)) {
            throw new MethodArgumentNotValidException(parameter, binder.getBindingResult());
         }
      }
      mavContainer.addAttribute(BindingResult.MODEL_KEY_PREFIX + name, binder.getBindingResult());

      return adaptArgumentIfNecessary(arg, parameter);
   }

   @Override
   protected <T> Object readWithMessageConverters(NativeWebRequest webRequest, MethodParameter parameter,
         Type paramType) throws IOException, HttpMediaTypeNotSupportedException, HttpMessageNotReadableException {

      HttpServletRequest servletRequest = webRequest.getNativeRequest(HttpServletRequest.class);
      ServletServerHttpRequest inputMessage = new ServletServerHttpRequest(servletRequest);

      Object arg = readWithMessageConverters(inputMessage, parameter, paramType);
      if (arg == null) {
         if (checkRequired(parameter)) {
            throw new HttpMessageNotReadableException("Required request body is missing: " +
                  parameter.getMethod().toGenericString());
         }
      }
      return arg;
   }

   protected boolean checkRequired(MethodParameter parameter) {
      return (parameter.getParameterAnnotation(RequestBody.class).required() && !parameter.isOptional());
   }

   @Override
   public void handleReturnValue(Object returnValue, MethodParameter returnType,
         ModelAndViewContainer mavContainer, NativeWebRequest webRequest)
         throws IOException, HttpMediaTypeNotAcceptableException, HttpMessageNotWritableException {

      mavContainer.setRequestHandled(true);
      ServletServerHttpRequest inputMessage = createInputMessage(webRequest);
      ServletServerHttpResponse outputMessage = createOutputMessage(webRequest);

      // Try even with null return value. ResponseBodyAdvice could get involved.
      writeWithMessageConverters(returnValue, returnType, inputMessage, outputMessage);
   }

}
```



发现`RequestResponseBodyMethodProcessor` 类继承了 `AbstractMessageConverterMethodProcessor`，然后查看`AbstractMessageConverterMethodProcessor`发现它继承了`AbstractMessageConverterMethodArgumentResolver`类并且实现了`HandlerMethodReturnValueHandler`接口，这个`HandlerMethodReturnValueHandler` 就是关键，对返回值的解析。

```java
public abstract class AbstractMessageConverterMethodProcessor extends AbstractMessageConverterMethodArgumentResolver
      implements HandlerMethodReturnValueHandler {
      .... 省咯其中代码
      }
```

 我们再进入`HandlerMethodReturnValueHandler` 可以看到其有两个方法,和`HandlerMethodArgumentResolver`非常的像，`supportsReturnType`作为入口，`handleReturnValue`真正的处理解析的方法

```java
public interface HandlerMethodReturnValueHandler {
	// 判断条件
   boolean supportsReturnType(MethodParameter returnType);
	// 处理返回值操作
   void handleReturnValue(Object returnValue, MethodParameter returnType,
         ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception;

}
```

到这里我们重新回到`RequestResponseBodyMethodProcessor`查看其中实现`HandlerMethodReturnValueHandler`中的两个方法

```java
	
@Override
	public boolean supportsReturnType(MethodParameter returnType) {
        // 判断类和方法是否有注解@ResponseBody
   	 return (AnnotatedElementUtils.hasAnnotation(returnType.getContainingClass(), 			    ResponseBody.class) ||returnType.hasMethodAnnotation(ResponseBody.class));
	}

	@Override
	public void handleReturnValue(Object returnValue, MethodParameter returnType,
			ModelAndViewContainer mavContainer, NativeWebRequest webRequest)
			throws IOException, HttpMediaTypeNotAcceptableException, HttpMessageNotWritableException {

		mavContainer.setRequestHandled(true);
		ServletServerHttpRequest inputMessage = createInputMessage(webRequest);
		ServletServerHttpResponse outputMessage = createOutputMessage(webRequest);

		// 开始进行解析结果 然后返回
		writeWithMessageConverters(returnValue, returnType, inputMessage, outputMessage);
	}

```

条件入口`supportsReturnType` 方法判断controller类和方法上是否有`@ResponseBody`注解，有就进入handleReturnValue 方法处理后续的解析操作。

我们来看`handleReturnValue` 方法获得到`ServletServerHttpRequest`和`ServletServerHttpResponse`，然后直接调用`writeWithMessageConverters` 方法进行主要的解析。

```java
protected <T> void writeWithMessageConverters(T value, MethodParameter returnType,
      ServletServerHttpRequest inputMessage, ServletServerHttpResponse outputMessage)
      throws IOException, HttpMediaTypeNotAcceptableException, HttpMessageNotWritableException {

   Object outputValue;
   Class<?> valueType;
   Type declaredType;
	// 判断是否是String 类型的
   if (value instanceof CharSequence) {
      outputValue = value.toString();
      valueType = String.class;
      declaredType = String.class;
   }
   else {
       // 非String获取具体的类型
      outputValue = value;
      valueType = getReturnValueType(outputValue, returnType);
      declaredType = getGenericType(returnType);
   }
	// 通过request请求获取其中的MediaType类型
   HttpServletRequest request = inputMessage.getServletRequest();
   List<MediaType> requestedMediaTypes = getAcceptableMediaTypes(request);
   List<MediaType> producibleMediaTypes = getProducibleMediaTypes(request, valueType, declaredType);

   if (outputValue != null && producibleMediaTypes.isEmpty()) {
      throw new IllegalArgumentException("No converter found for return value of type: " + valueType);
   }

   Set<MediaType> compatibleMediaTypes = new LinkedHashSet<MediaType>();
   for (MediaType requestedType : requestedMediaTypes) {
      for (MediaType producibleType : producibleMediaTypes) {
         if (requestedType.isCompatibleWith(producibleType)) {
            compatibleMediaTypes.add(getMostSpecificMediaType(requestedType, producibleType));
         }
      }
   }
   if (compatibleMediaTypes.isEmpty()) {
      if (outputValue != null) {
         throw new HttpMediaTypeNotAcceptableException(producibleMediaTypes);
      }
      return;
   }
	
   List<MediaType> mediaTypes = new ArrayList<MediaType>(compatibleMediaTypes);
   MediaType.sortBySpecificityAndQuality(mediaTypes);

   MediaType selectedMediaType = null;
   for (MediaType mediaType : mediaTypes) {
      if (mediaType.isConcrete()) {
         selectedMediaType = mediaType;
         break;
      }
      else if (mediaType.equals(MediaType.ALL) || mediaType.equals(MEDIA_TYPE_APPLICATION)) {
         selectedMediaType = MediaType.APPLICATION_OCTET_STREAM;
         break;
      }
   }
	// 重点开始调用HttpMessageConverter 进行输出结果
   if (selectedMediaType != null) {
      selectedMediaType = selectedMediaType.removeQualityValue();
      for (HttpMessageConverter<?> messageConverter : this.messageConverters) {
         if (messageConverter instanceof GenericHttpMessageConverter) {
            if (((GenericHttpMessageConverter) messageConverter).canWrite(
                  declaredType, valueType, selectedMediaType)) {
               outputValue = (T) getAdvice().beforeBodyWrite(outputValue, returnType, selectedMediaType,
                     (Class<? extends HttpMessageConverter<?>>) messageConverter.getClass(),
                     inputMessage, outputMessage);
               if (outputValue != null) {
                  addContentDispositionHeader(inputMessage, outputMessage);
                  ((GenericHttpMessageConverter) messageConverter).write(
                        outputValue, declaredType, selectedMediaType, outputMessage);
                  if (logger.isDebugEnabled()) {
                     logger.debug("Written [" + outputValue + "] as \"" + selectedMediaType +
                           "\" using [" + messageConverter + "]");
                  }
               }
               return;
            }
         }
         else if (messageConverter.canWrite(valueType, selectedMediaType)) {
            outputValue = (T) getAdvice().beforeBodyWrite(outputValue, returnType, selectedMediaType,
                  (Class<? extends HttpMessageConverter<?>>) messageConverter.getClass(),
                  inputMessage, outputMessage);
            if (outputValue != null) {
               addContentDispositionHeader(inputMessage, outputMessage);
               ((HttpMessageConverter) messageConverter).write(outputValue, selectedMediaType, outputMessage);
               if (logger.isDebugEnabled()) {
                  logger.debug("Written [" + outputValue + "] as \"" + selectedMediaType +
                        "\" using [" + messageConverter + "]");
               }
            }
            return;
         }
      }
   }

   if (outputValue != null) {
      throw new HttpMediaTypeNotAcceptableException(this.allSupportedMediaTypes);
   }
}
```

这么多代码可以总结就是解析结果将结果用流的方式输出。而`HttpMessageConverter`就是实现了Spring的消息转换机制。
![httpmessageconvert]({{ site.BASE_PATH }}/assets/img/2018-09-20/httpmessageconvert.jpg)

`HttpMessageConverter`工作就是将java 对象转化流和把流转化java对象。

通过上面的分析，我们想到就是去定义一个类实现`HandlerMethodReturnValueHandler`,然后对结果去进行解析处理，统一包装controller方法返回的结果。



## 实现

定义一个类似@ResponseBody 注解自定义注解

```java
@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface ResponseBodyWrapper {
}
```



```java
public class ResponseMethodReturnValueHandler implements HandlerMethodReturnValueHandler {
    @Override
    public boolean supportsReturnType(MethodParameter returnType) {
        // 判断类和方法是否有注解@ResponseBodyWrapper
   	 return (AnnotatedElementUtils.hasAnnotation(returnType.getContainingClass(), 			    ResponseBodyWrapper.class) ||returnType.hasMethodAnnotation(ResponseBodyWrapper.class));
	}
    
    @Override
    public void handleReturnValue(Object returnValue, MethodParameter returnType, ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {

        mavContainer.setRequestHandled(true);
        // 获取response
        HttpServletResponse response = webRequest.getNativeResponse(HttpServletResponse.class);
        // 设置MediaType
        response.setContentType(MediaType.APPLICATION_JSON_UTF8_VALUE);
        // 将结果包装输出
        write(response,getResponseStr(returnValue));

    }
	
    // 对结果进行包装 
    private String getResponseStr(Object returnValue){
        ResultModel resultModle = new ResultModel(returnValue);
        return JSON.toJSONString(resultModle);
    }
	// 输出流
    private final void write(HttpServletResponse response,String responseValue){
        PrintWriter writer = null;
        try {
            writer = response.getWriter();
            writer.write(responseValue);
            writer.flush();
        } catch (IOException ex) {
            ex.printStackTrace();
        } finally {
            if (writer != null){
                writer.close();
            }
        }
    }


}
```

返回结果的包装类，统一errorCode，errorMsg

```java
public class ResultModel {
    private String errorCode;
    private String errorMsg;
    private Object result;

    public ResultModel(Object result){
        this.result = result;
        this.errorCode = "0";
        this.errorMsg = "ok";
    }
    ...省咯set，get方法

}
```



在springboot项目中我们需要把这个自定义的`ResponseMethodReturnValueHandler` 注册到springmvc中。

```java
@Configuration
public class WebConfig extends WebMvcConfigurationSupport {
    @Override

    public void addReturnValueHandlers(List<HandlerMethodReturnValueHandler> returnValueHandlers) {
        super.addReturnValueHandlers(returnValueHandlers);
        returnValueHandlers.add(new ResponseMethodReturnValueResolver());
    }

}
```

最后我们只需要在controller 中注解`@ResponseBodyWrapper` 代替之前的`@ResponseBody` 。

但是会发现返回根本就没有结果，是不是很让人疑惑，原因其实就是springmvc有默认的处理器，自定义的处理器会排在最后面，只有默认的处理器全部失败才能轮到自己的。



网上找了下解决方案https://github.com/AndreasKl/spring-boot-mvc-completablefuture/blob/master/src/main/java/net/andreaskluth/WebMvcConfiguration.java 

```java
@Configuration
public class WebConfig extends WebMvcConfigurationSupport {
    @Bean
    public HandlerMethodReturnValueHandler responseMethodReturnValueHandler() {
        return new ResponseMethodReturnValueHandler();
    }
    @PostConstruct
    public void init() {
        final List<HandlerMethodReturnValueHandler> originalHandlers = new ArrayList<>(
                requestMappingHandlerAdapter.getReturnValueHandlers());
        final int deferredPos = obtainValueHandlerPosition(originalHandlers, DeferredResultMethodReturnValueHandler.class);
        // Add our handler directly after the deferred handler.
        originalHandlers.add(deferredPos + 1, responseMethodReturnValueHandler());
        requestMappingHandlerAdapter.setReturnValueHandlers(originalHandlers);
    }

    private int obtainValueHandlerPosition(final List<HandlerMethodReturnValueHandler> originalHandlers, Class<?> handlerClass) {
        for (int i = 0; i < originalHandlers.size(); i++) {
            final HandlerMethodReturnValueHandler valueHandler = originalHandlers.get(i);
            if (handlerClass.isAssignableFrom(valueHandler.getClass())) {
                return i;
            }
        }
        return -1;
    }
}
```





## 结尾

通过自定义`HandlerMethodReturnValueHandler`  ，我们实现了对返回结果统一包装，不需要在每个controller方法中去设置ResultModel 对象，极大的简化了代码编写，都交给框架统一处理。
