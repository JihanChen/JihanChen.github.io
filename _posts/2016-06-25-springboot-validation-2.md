---
layout: postlayout
title: springboot使用@Validated进行请求参数校验
categories: [java]
tags: [validation]
---

@Validated 是springmvc 对@Valid的二次封装，是使用spring Validator的校验机制，而@Valid是JSR303-Bean Validation的一个数据验证规范。使用@Validated可以进行分组校验，大大减少了代码量，非常的好用。

## 说明
之前一篇文章主要是对@Valid做了简单的使用，@Valid是一种校验规范，而@Validated是在规范上的一次封装，所以使用@Validated 注解来进行校验在平时工作中用的会比较多，该篇文章主要是对@Validated校验进行简单的使用。查看@Valid使用说明可以查看 [springboot使用@Valid进行请求参数校验]({% post_url 2016-06-17-springboot-validation %})  

## 代码操作

### 分组校验
定义实体对象

```
public class SpringMvcEntity {

    @NotNull(message = "id不能为空", groups = { update.class })
    private String id;

    @NotEmpty(message = "用户名不能为空", groups = { Insert.class })
    private String uname;

    @NotEmpty(message = "密码不能为空", groups = { Insert.class,update.class })
    private String pwd;


    public interface Insert {
    }
    public interface update {
    }

    public String getUname() {
        return uname;
    }

    public void setUname(String uname) {
        this.uname = uname;
    }

    public String getPwd() {
        return pwd;
    }

    public void setPwd(String pwd) {
        this.pwd = pwd;
    }

    public String getId() {
        return id;
    }

    public void setId(String id) {
        this.id = id;
    }
}
```
我们在实体属性上面的的注解多了一个groups，名字上就可以看出是一个分组的概念，Insert和update两个接口作为分组接口标识，给id 标注了一个update标记,uname标注了Insert标记，pwd同时标注了Insert和update组，所以不管在更新还是新增pwd 都是必传的参数

controller层代码如下，定义了一个insert 和 update 方法。  
insert方法里参数包含了```@Validated(SpringMvcEntity.Insert.class)```表示只对标注了groups=Insert的校验生效  
update方法参数包含了```@Validated(SpringMvcEntity.update.class)```表示只对标注了groups=update的校验生效    

```
@RestController
public class SpringMvcValidationController {

    @RequestMapping("/springmvc/Validated/insert")
    public ValidationResponse insert(@Validated(SpringMvcEntity.Insert.class)  	@RequestBody SpringMvcEntity SpringMvcEntity){
       return new ValidationResponse("0","ok");
    }

    @RequestMapping("/springmvc/Validated/update")
    public ValidationResponse update(@Validated(SpringMvcEntity.update.class) 	@RequestBody SpringMvcEntity SpringMvcEntity){
        return new ValidationResponse("0","ok");
    }

    @ExceptionHandler(value = MethodArgumentNotValidException.class)
    @ResponseBody
    public ValidationResponse resolveMethodArgumentNotValidException(MethodArgumentNotValidException e){

        ObjectError objectError = e.getBindingResult().getAllErrors().get(0);
        String validationMsg = objectError.getDefaultMessage();

        return new ValidationResponse("1002",validationMsg);
    }

}
```


``` 
POST /springmvc/Validated/insert
{
"uname": "皮卡丘",
}
```
请求之后返回的结果：

```
{
"errorCode": "1002",
"msg": "密码不能为空"
}
```
调用insert方法的时候由于对请求的参数pwd进行了非空校验，所以这边校验没有通过，报了“密码不能为空”错误


``` 
POST /springmvc/Validated/update
{
"uname": "皮卡丘",
}
```
请求之后返回的结果：

```
{
"errorCode": "1002",
"msg": "id不能为空"
}
```
调用update方法的时候由于对请求的参数id进行了非空校验，所以这边校验没有通过，由于我这里捕获异常时候只获取了第一个错误显示到前台，其实是有两个异常,具体操作的地方是```resolveMethodArgumentNotValidException``` 方法下```e.getBindingResult().getAllErrors().get(0)```获取错误列表的第一个错误。

### 组顺序校验
上面例子校验都是没有顺序的，标注了校验的注解，都会进行校验，那是不是可以做到一个组的校验不通过就不去校验下面其他组的校验呢？答案是可以的，我们可以通过@GroupSequence来指定分组验证顺序。

定义几个组如下

```
public class UserGroup {

    private String id;

    @NotEmpty(message = "用户名不能为空",groups ={Sequence2.class} )
    private String uname;

    @NotEmpty(message = "密码不能为空",groups = {Sequence1.class})
    private String pwd;

    @NotNull(message = "年龄不能为空",groups = {Default.class})
    private Integer age;


    public interface Sequence1 {
    }
    public interface Sequence2 {
    }
    @GroupSequence( { Default.class, Sequence1.class, Sequence2.class })
    public interface Group {
    }

    public String getId() {
        return id;
    }

    public void setId(String id) {
        this.id = id;
    }

    public String getUname() {
        return uname;
    }

    public void setUname(String uname) {
        this.uname = uname;
    }

    public String getPwd() {
        return pwd;
    }

    public void setPwd(String pwd) {
        this.pwd = pwd;
    }

    public Integer getAge() {
        return age;
    }

    public void setAge(Integer age) {
        this.age = age;
    }
}
```
Default组是默认的，javax.validation.groups.Default，我们可以直接拿来使用
上面定义了三个组，使用@GroupSequence 对组进行了排序，先进行Default组的校验，然后是Sequence1，最后是Sequence2。

controller层定义如下

```
    @RequestMapping("/springmvc/Validated/group")
    public ValidationResponse group(@Validated({UserGroup.Group.class})
    @RequestBody UserGroup userGroup){
        return new ValidationResponse("0","ok");
    }
```
可以看到我使用了 @Validated({UserGroup.Group.class}使用的是Group组校验，前者又进行了组顺序校验定义，所以程序会进行一个个顺序校验。
实际测试情况如上面定义顺序，先校验 age，然后校验pwd，最后去校验uname。


## 结尾
上面例子使用了@Validated注解方式进行了请求参数的校验，引进了组校验的概念，精简了很多代码，在最后还使用了@GroupSequence进行组的顺序校验，组顺序校验在有些场景还是很有用的，比如有两个组，第二个组中的约束校验需要依赖第一个组校验通过后的数据，或者可能第二个组的验证比较耗时，耗内存等系统资源，组顺序就可以把该校验放在最后进行验证了，节省了一些不必要的开销。总的来说使用@Validated比@Valid更加的强大方便，推荐使用@Validated。




