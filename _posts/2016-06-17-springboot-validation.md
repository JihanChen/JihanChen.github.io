---
layout: postlayout
title: springboot使用@Valid进行请求参数校验
categories: [java]
tags: [springboot,validation]
---

最近在研究springboot,发现实在是太方便了，各种注解，各种爽。这篇文章主要是在controller层使用@Valid注解进行参数校验，校验不通过返回自定义的结果。

## 起源
使用框架最大的好处就是加快了开发速度，很多场景我们需要对客户端请求的参数进行一些常规校验，比如非空校验，位数校验，格式是否正确等等。以前我们可能在控制层获取到请求参数，然后一个个对参数进行校验，不通过校验就无法进行下面的业务。使用springboot的时候用@Valid对参数进行校验,大大加快的开发速度，精简了代码。

## 开始使用

### 代码操作

controller层代码
```
@RequestMapping("/validation/msg")
@ResponseBody
public String validationMsg(@Valid @RequestBody User user){
        return "ok";
}
```


User实体对象
```
public class User{
    @NotEmpty(message = "用户名不能为空")
    private String uname;  

    @NotEmpty(message = "密码不能为空")
    private String pwd; 

    private int age; 
    // 此处省略掉set get方法
}```



下面客户端开始请求
POST /validation/msg
``` 
{
    "pwd": "123456",
    "age": "28"
}
```



请求之后返回的结果：

```
{
    "timestamp":1468634431233,
    "status":400,
    "error":"Bad Request",
    "exception":"org.springframework.web.bind.MethodArgumentNotValidException",
    "errors":[
        {
            "codes":[
                "NotEmpty.user.uname",
                "NotEmpty.uname",
                "NotEmpty.java.lang.String",
                "NotEmpty"
            ],
            "arguments":[
                {
                    "codes":[
                        "user.uname",
                        "uname"
                    ],
                    "arguments":null,
                    "defaultMessage":"uname",
                    "code":"uname"
                }
            ],
            "defaultMessage":"用户名不能为空",
            "objectName":"user",
            "field":"uname",
            "rejectedValue":null,
            "bindingFailure":false,
            "code":"NotEmpty"
        }
    ],
    "message":"Validation failed for object='user'. Error count: 1",
    "path":"/validation/msg"
}
```




springboot 对异常进行了封装，针对json请求和普通的页面请求做了处理，具体可以查看<http://www.cnblogs.com/xinzhao/p/4934247.html>

显然返回的参数太多了很乱，看的不直观，一般框架对于请求也都会有一个固定的返回格式，比如 

```{"errorCode": "0","msg": "ok"}```   请求正常 

```{"errorCode": "1001","errorMsg": "用户名不能为空"}``` 请求异常

当你看了上面链接的异常处理其实就很简单的可以显示自定义的返回结果。在controller层添加如下代码




```     
@ExceptionHandler(value=MethodArgumentNotValidException.class) 
@ResponseBody
public ValidationResponse resolveMValidException(MethodArgumentNotValidException e){

         // 只获取第一个异常的错误返回
        ObjectError objectError = e.getBindingResult().getAllErrors().get(0);
        // 获取错误描述－上面注解的信息
        String validationMsg = objectError.getDefaultMessage();

        return new ValidationResponse("1001",validationMsg);
 }
```




定义了异常处理 然后输出了自定义的返回结果，最终输出了

```{"errorCode": "1001","errorMsg": "用户名不能为空"} ```

是不是很赞啊，非常简单友好的输出了错误异常信息。

下面我们使用正确的请求发给服务端
```
POST /validation/msg

{
    "uname":"皮卡丘",
    "pwd": "123456",
    "age": "1"
}

```




结果没有出错了哦，返回结果
```
{
"errorCode": "0",
"msg": "ok"
}
```




### JSR303定义的校验类型

由于 @Valid 校验是基于JSR303,使用方式可以查看文章 <http://www.ibm.com/developerworks/cn/java/j-lo-jsr303/>

> 校验类型

空检查
@Null       验证对象是否为null
@NotNull    验证对象是否不为null, 无法查检长度为0的字符串
@NotBlank 检查约束字符串是不是Null还有被Trim的长度是否大于0,只对字符串,且会去掉前后空格.
@NotEmpty 检查约束元素是否为NULL或者是EMPTY.


Booelan检查
@AssertTrue     验证 Boolean 对象是否为 true  
@AssertFalse    验证 Boolean 对象是否为 false  


长度检查
@Size(min=, max=) 验证对象（Array,Collection,Map,String）长度是否在给定的范围之内  
@Length(min=, max=) Validates that the annotated string is between min and max included.

 

日期检查
@Past        验证 Date 和 Calendar 对象是否在当前时间之前  
@Future     验证 Date 和 Calendar 对象是否在当前时间之后  
@Pattern    验证 String 对象是否符合正则表达式的规则

 
数值检查

@Min            验证 Number 和 String 对象是否大等于指定的值  
@Max            验证 Number 和 String 对象是否小等于指定的值  
@DecimalMax 被标注的值必须不大于约束中指定的最大值. 这个约束的参数是一个通过BigDecimal定义的最大值的字符串表示.小数存在精度
@DecimalMin 被标注的值必须不小于约束中指定的最小值. 这个约束的参数是一个通过BigDecimal定义的最小值的字符串表示.小数存在精度
@Digits     验证 Number 和 String 的构成是否合法  
@Digits(integer=,fraction=) 验证字符串是否是符合指定格式的数字，interger指定整数精度，fraction指定小数精度。
@Range(min=, max=) Checks whether the annotated value lies between (inclusive) the specified minimum and maximum.
@Range(min=10000,max=50000,message="range.bean.wage")
private BigDecimal wage;

@Valid 递归的对关联对象进行校验, 如果关联对象是个集合或者数组,那么对其中的元素进行递归校验,如果是一个map,则对其中的值部分进行校验.(是否进行递归验证)
@CreditCardNumber信用卡验证
@Email  验证是否是邮件地址，如果为null,不进行验证，算通过验证。
@URL(protocol=,host=, port=,regexp=, flags=)


## 结尾
在springmvc使用@Valid会一点，需要引入validation-api和hibernate-validator依赖，这个下次再讲。springboot简单几行代码就完成了整个校验过程，全程都不需要配置文件，全是基于注解，“约定优于配置” 的理念确实强大，快速开发的不二选择。