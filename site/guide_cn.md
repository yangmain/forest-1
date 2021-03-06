# 一. 新手介绍

## 什么是 Forest？

Forest 是一个开源的 Java HTTP 客户端框架，它能够将 HTTP 的所有请求信息（包括 URL、Header 以及 Body 等信息）绑定到您自定义的 Interface 方法上，能够通过调用本地接口方法的方式发送 HTTP 请求。

## 为什么使用 Forest?

使用 Forest 就像使用类似 Dubbo 那样的 RPC 框架一样，只需要定义接口，调用接口即可，不必关心具体发送 HTTP 请求的细节。同时将 HTTP 请求信息与业务代码解耦，方便您统一管理大量 HTTP 的 URL、Header 等信息。而请求的调用方完全不必在意 HTTP 的具体内容，即使该 HTTP 请求信息发生变更，大多数情况也不需要修改调用发送请求的代码。

## Forest 如何使用?

Forest 不需要您编写具体的 HTTP 调用过程，只需要您定义一个接口，然后通过 Forest 注解将 HTTP 请求的信息添加到接口的方法上即可。请求发送方通过调用您定义的接口便能自动发送请求和接受请求的响应。

## Forest 的工作原理

Forest 会将您定义好的接口通过动态代理的方式生成一个具体的实现类，然后组织、验证 HTTP 请求信息，绑定动态数据，转换数据形式，SSL 验证签名，调用后端 HTTP API(httpclient 等 API)执行实际请求，等待响应，失败重试，转换响应数据到 Java 类型等脏活累活都由这动态代理的实现类给包了。
请求发送方调用这个接口时，实际上就是在调用这个干脏活累活的实现类。

## Forest 的架构

![avater](media/architect.png)

我们讲 HTTP 发送请求的过程分为前端部分和后端部分，Forest 本身是处理前端过程的框架，是对后端 HTTP API 框架的进一步封装。

<b>前端部分：</b>

1. Forest 配置： 负责管理 HTTP 发送请求所需的配置。
2. Forest 注解： 用于定义 HTTP 发送请求的所有相关信息，一般定义在 interface 上和其方法上。
3. 动态代理： 用户定义好的 HTTP 请求的`interface`将通过动态代理产生实际执行发送请求过程的代理类。
4. 模板表达式： 模板表达式可以嵌入在几乎所有的 HTTP 请求参数定义中，它能够将用户通过参数或全局变量传入的数据动态绑定到 HTTP 请求信息中。
5. 数据转换： 此模块将字符串数据和`JSON`或`XML`形式数据进行互转。目前 JSON 转换器支持`Jackson`、`Fastjson`、`Gson`三种，XML 支持`JAXB`一种。
6. 拦截器： 用户可以自定义拦截器，拦截指定的一个或一批请求的开始、成功返回数据、失败、完成等生命周期中的各个环节，以插入自定义的逻辑进行处理。
7. 过滤器： 用于动态过滤和处理传入 HTTP 请求的相关数据。
8. SSL： Forest 支持单向和双向验证的 HTTPS 请求，此模块用于处理 SSL 相关协议的内容。

<b>后端部分：</b>

后端为实际执行 HTTP 请求发送过程的第三方 HTTP API，目前支持`okHttp3`和`httpclient`两种后端 API。

<b>Spring Boot Starter Forest:</b>

提供对`Spring Boot`的支持

## 对应的 Java 版本

Forest 1.0.x 和 Forest 1.1.x 基于 JDK 1.7, Forest 1.2.x及以上版本基于 JDK 1.8

# 二. 安装

## 2.1 在 Spring Boot 项目中安装

若您的项目基于`Spring Boot`，那只要添加下面一个 maven 依赖便可。

```xml
<dependency>
    <groupId>com.dtflys.forest</groupId>
    <artifactId>spring-boot-starter-forest</artifactId>
    <version>1.4.12</version>
</dependency>
```

最新版本为<font color=red>1.4.12</font>，为稳定版本

如果您用的是Gradle:

```gradle
compile group: 'com.dtflys.forest', name: 'spring-boot-starter-forest', version: '1.4.12'
```

## 2.2 在非 Spring Boot 项目中安装

添加 Forest 核心包依赖

```xml
<dependency>
    <groupId>com.dtflys.forest</groupId>
    <artifactId>forest-core</artifactId>
    <version>1.4.12</version>
</dependency>
```

最新版本为<font color=red>1.4.12</font>，为稳定版本

如果您用的是Gradle:

```gradle
compile group: 'com.dtflys.forest', name: 'forest-core', version: '1.4.12'
```


# 三. 构建请求接口

在 Forest 依赖加入好之后，就可以构建 HTTP 请求的接口了。

在 Forest 中，所有的 HTTP 请求信息都要绑定到某一个接口的方法上，不需要编写具体的代码去发送请求。请求发送方通过调用事先定义好 HTTP 请求信息的接口方法，自动去执行 HTTP 发送请求的过程，其具体发送请求信息就是该方法对应绑定的 HTTP 请求信息。

## 3.1 简单请求

创建一个`interface`，并用`@Request`注解修饰接口方法。

```java
public interface MyClient {

    @Request(url = "http://localhost:5000/hello")
    String simpleRequest();

}
```

通过`@Request`注解，将上面的`MyClient`接口中的`simpleRequest()`方法绑定了一个 HTTP 请求，
其 URL 为`http://localhost:5000/hello`
，并默认使用`GET`方式，且将请求响应的数据以`String`的方式返回给调用者。

## 3.2 稍复杂点的请求

```java
public interface MyClient {

    @Request(
            url = "http://localhost:5000/hello/user",
            headers = "Accept: text/plain"
    )
    String sendRequest(@DataParam("uname") String username);
}
```

上面的`sendRequest`方法绑定的 HTTP 请求，定义了 URL 信息，以及把`Accept:text/plain`加到了请求头中，
方法的参数`String username`绑定了注解`@DataParam("uname")`，它的作用是将调用者传入入参 username 时，自动将`username`的值加入到 HTTP 的请求参数`uname`中。

如果调用方代码如下所示：

```java
MyClient myClient;
...
myClient.sendRequest("foo");
```

这段调用所实际产生的 HTTP 请求如下：

    GET http://localhost:5000/hello/user?uname=foo
    HEADER:
        Accept: text/plain

## 3.3 HTTP Method

使用`POST`方式

```java
public interface MyClient {

    /**
     * 通过 @Request 注解的 type 参数指定 HTTP 请求的方式。
     */
    @Request(
            url = "http://localhost:5000/hello",
            type = "POST"
    )
    String simplePost();

    /**
     * 使用 @Post 注解，可以去掉 type = "POST" 这行属性
     */
    @Post(url = "http://localhost:5000/hello")
    String simplePost();

    /**
     * 使用 @PostRequest 注解，和上面效果等价
     */
    @PostRequest(url = "http://localhost:5000/hello")
    String simplePost();

}
```


除了`GET`和`POST`，也可以指定成其他几种 HTTP 请求方式(`PUT`, `HEAD`, `OPTIONS`, `DELETE`)。

其中`type`属性的大小写不敏感，写成`POST`和`post`效果相同。

```java
// GET请求
@Request(
        url = "http://localhost:5000/hello",
        type = "get"
)
String simpleGet();

// POST请求
@Request(
        url = "http://localhost:5000/hello",
        type = "post"
)
String simplePost();

// PUT请求
@Request(
        url = "http://localhost:5000/hello",
        type = "put"
)
String simplePut();

// HEAD请求
@Request(
        url = "http://localhost:5000/hello",
        type = "head"
)
String simpleHead();

// Options请求
@Request(
        url = "http://localhost:5000/hello",
        type = "options"
)
String simpleOptions();

// Delete请求
@Request(
        url = "http://localhost:5000/hello",
        type = "delete"
)
String simpleDelete();
```

另外，可以用`@GetRequest`, `@PostRequest`等注解代替`@Request`注解，这样就可以省去写`type`属性的麻烦了。

```java
// GET请求
@Get(url = "http://localhost:5000/hello")
String simpleGet();

// GET请求
@GetRequest(url = "http://localhost:5000/hello")
String simpleGetRequest();

// POST请求
@Post(url = "http://localhost:5000/hello")
String simplePost();

// POST请求
@PostRequest(url = "http://localhost:5000/hello")
String simplePostRequest();

// PUT请求
@Put(url = "http://localhost:5000/hello")
String simplePut();

// PUT请求
@PutRequest(url = "http://localhost:5000/hello")
String simplePutRequest();

// HEAD请求
@HeadRequest(url = "http://localhost:5000/hello")
String simpleHead();


// Options请求
@Options(url = "http://localhost:5000/hello")
String simpleOptions();

// Options请求
@OptionsRequest(url = "http://localhost:5000/hello")
String simpleOptionsRequest();

// Delete请求
@Delete(url = "http://localhost:5000/hello")
String simpleDelete();

// Delete请求
@DeleteRequest(url = "http://localhost:5000/hello")
String simpleDeleteRequest();
```

如上所示，请求类型是不是更一目了然了，代码也更短了。

`@Get`和`@GetRequest`两个注解的效果是等价的，`@Post`和`@PostRequest`、`@Put`和`@PutRequest`等注解也是同理。

?> 需要注意的是，`HEAD`请求类型没有对应的`@Head`注解，只有`@HeadRequest`注解。原因是容易和`@Header`注解混淆

## 3.4 HTTP URL

HTTP请求可以没有请求头、请求体，但一定会有`URL`，以及很多请求的参数都是直接绑定在`URL`的`Query`部分上。

基本`URL`设置方法就如[3.1](###_31-简单请求)的例子所示，只要在`url`属性中填入完整的请求地址即可。

除此之外，也可以从外部动态传入`URL`:

```java
/**
 * 整个完整的URL都通过 @DataVariable 注解修饰的参数动态传入
 */
@Request(url = "${myURL}")
String send1(@DataVariable("myURL") String myURL);

/**
 * 通过参数转入的值值作为URL的一部分
 */
@Request(url = "http://${myURL}/abc")
String send2(@DataVariable("myURL") String myURL);
```

### 3.4.1 通过字符串模板传参

HTTP的`URL`不光有协议名、域名、端口号等等基本信息，更为重要的是它能携带各种参数，称为`Query`参数，它通常包含参数名和参数值两部分。

Forest给`URL`的`Query`部分传参也有多种方式，其中最简洁直白的就数字符串拼接了。

```java

/**
 * 直接在url字符串的问号后面部分直接写上 参数名=参数值 的形式
 * 等号后面的参数值部分可以用 ${变量名} 这种字符串模板的形式替代
 * 在发送请求时会动态拼接成一个完整的URL
 */
@Request(url = "http://${myURL}/abc?a=${a}&b=${b}&id=0")
String send2(@DataVariable("a") String a, @DataVariable("b") String b);
```


### 3.4.2 通过 @Query 注解

但把所有`Query`参数直接写在`url`属性的字符串里面是不是也太简单粗暴了，有没有优雅点的方式？有的。

```java

/**
 * 使用 @Query 注解，可以直接将该注解修饰的参数动态绑定到请求url中
 * 注解的 value 值即代表它在url的Query部分的参数名
 */
@Request(url = "http://${myURL}/abc?id=0")
String send2(@Query("a") String a, @Query("b") String b);

```

?> @Query 与 @DataParam 不同，@Query 注解修饰的参数一定会出现在 URL 中，而 @DataParam 修饰的参数则要视情况而定。

若是要传的`URL`参数太多了呢？难道要我在方法上定义十几二十个`@Query`修饰的参数？那也太难看了吧。别急，Forest还是有办法让您变的代码变得优雅的。

```java

/**
 * 使用 @Query 注解，可以修饰 Map 类型的参数
 * 很自然的，Map 的 Key 将作为 URL 的参数名， Value 将作为 URL 的参数值
 * 这时候 @Query 注解不定义名称
 */
@Get(url = "http://${myURL}/abc?id=0")
String send2(@Query Map<String, Object> map);


/**
 * @Query 注解也可以修饰自定义类型的对象参数
 * 依据对象类的 Getter 和 Setter 的规则取出属性
 * 其属性名为 URL 参数名，属性值为 URL 参数值
 * 这时候 @Query 注解不定义名称
 */
@Get(url = "http://${myURL}/abc?id=0")
String send2(@Query UserInfo user);

```

是不是瞬间简洁不少，但用`@Query`注解绑定参数的时候也有需要注意的地方：

!> 注意：<br>
(1) 需要单个单个定义 参数名=参数值 的时候，@Query注解的value值一定要有，比如 @Query("name") String name<br>
(2) 需要绑定对象的时候，@Query注解的value值一定要空着，比如 @Query User user 或 @Query Map map


## 3.5 HTTP Header

在[3.2](###_32-稍复杂点的请求)的例子中，我们已经知道了可以通过`@Request`注解的`headers`属性设置一条 HTTP 请求头。

现在我们来看看如何添加多条请求头

### 3.5.1 通过headers属性

其中`headers`属性接受的是一个字符串数组，在接受多个请求头信息时以以下形式填入请求头:

```java
{
    "请求头名称1: 请求头值1",
    "请求头名称2: 请求头值2",
    "请求头名称3: 请求头值3",
    ...
 }
```

其中组数每一项都是一个字符串，每个字符串代表一个请求头。请求头的名称和值用`:`分割。

具体代码请看如下示例：

```java
public interface MyClient {

    @Request(
            url = "http://localhost:5000/hello/user",
            headers = {
                "Accept-Charset: utf-8",
                "Content-Type: text/plain"
            }
    )
    String multipleHeaders();
}
```

该接口调用后所实际产生的 HTTP 请求如下：

    GET http://localhost:5000/hello/user
    HEADER:
        Accept-Charset: utf-8
        Content-Type: text/plain

如果要每次请求传入不同的请求头内容，可以在`headers`属性的请求头定义中加入`数据绑定`。

```java
public interface MyClient {

    @Request(
            url = "http://localhost:5000/hello/user",
            headers = {
                "Accept-Charset: ${encoding}",
                "Content-Type: text/plain"
            }
    )
    String bindingHeader(@DataVariable("encoding") String encoding);
}
```

如果调用方代码如下所示：

```java
myClient.bindingHeader("gbk");
```

这段调用所实际产生的 HTTP 请求如下：

    GET http://localhost:5000/hello/user
    HEADER:
        Accept-Charset: gbk
        Content-Type: text/plain
        
### 3.5.2 通过 @Header 注解

想必大家都已经了解通过 `headers` 属性设置请求头的方法了。不过这种方式虽然直观，但如要没通过参数传入到请求头中就显得比较啰嗦了。

所以Forest还提供了 `@Header` 注解来帮助您把方法的参数直接绑定到请求体中。

```java

/**
 * 使用 @Header 注解将参数绑定到请求头上
 * @Header 注解的 value 指为请求头的名称，参数值为请求头的值
 * @Header("Accept") String accept将字符串类型参数绑定到请求头 Accept 上
 * @Header("accessToken") String accessToken将字符串类型参数绑定到请求头 accessToken 上
 */
@Post(url = "http://localhost:${port}/hello/user?username=foo")
void postUser(@Header("Accept") String accept, @Header("accessToken") String accessToken);

```

如果有很多很多的请求头要通过参数传入，我需要定义很多很多参数吗？当然不用！

```java
/**
 * 使用 @Header 注解可以修饰 Map 类型的参数
 * Map 的 Key 指为请求头的名称，Value 为请求头的值
 * 通过此方式，可以将 Map 中所有的键值对批量地绑定到请求头中
 */
@Post(url = "http://localhost:${port}/hello/user?username=foo")
void headHelloUser(@Header Map<String, Object> headerMap);


/**
 * 使用 @Header 注解可以修饰自定义类型的对象参数
 * 依据对象类的 Getter 和 Setter 的规则取出属性
 * 其属性名为 URL 请求头的名称，属性值为请求头的值
 * 以此方式，将一个对象中的所有属性批量地绑定到请求头中
 */
@Post(url = "http://localhost:${port}/hello/user?username=foo")
void headHelloUser(@Header MyHeaderInfo headersInfo);

```

!> 注意：<br>
(1) 需要单个单个定义请求头的时候，@Header注解的value值一定要有，比如 @Header("Content-Type") String contentType<br>
(2) 需要绑定对象的时候，@Header注解的value值一定要空着，比如 @Header MyHeaders headers 或 @Header Map headerMap
        

## 3.6 HTTP Body

在`POST`和`PUT`等请求方法中，通常使用 HTTP 请求体进行传输数据。在 Forest 中有多种方式设置请求体数据。

### 3.6.1 通过 data 属性添加请求体

您可以通过`@Request`注解的`data`属性把数据添加到请求体。需要注意的是只有当`type`为`POST`、`PUT`、`PATCH`这类 HTTP Method 时，`data`属性中的值才会绑定到请求体中，而`GET`请求在有些情况会绑定到`url`的参数中。

具体`type`属性和`data`属性数据绑定位置的具体关系如下表：

| type      | `data`属性数据绑定位置 | 支持的`contentType`或`Content-Type`请求头 |
| --------- | ---------------------- | ----------------------------------------- |
| `GET`     | `url`参数部分          | 只有`application/x-www-form-urlencoded`   |
| `POST`    | 请求体                 | 任何`contentType`                         |
| `PUT`     | 请求体                 | 任何`contentType`                         |
| `PATCH`   | 请求体                 | 任何`contentType`                         |
| `HEAD`    | `url`参数部分          | 只有`application/x-www-form-urlencoded`   |
| `OPTIONS` | `url`参数部分          | 只有`application/x-www-form-urlencoded`   |
| `DELETE`  | `url`参数部分          | 只有`application/x-www-form-urlencoded`   |
| `TRACE`   | `url`参数部分          | 只有`application/x-www-form-urlencoded`   |

`data`属性在`POST`请求中绑定请求体

```java
public interface MyClient {

    @Request(
            url = "http://localhost:5000/hello/user",
            type = "post",
            data = "username=foo&password=bar",
            headers = {"Accept:text/plain"}
    )
    String dataPost();
}
```

该接口调用后所实际产生的 HTTP 请求如下：

    POST http://localhost:5000/hello/user
    HEADER:
        Accept:text/plain
    BODY:
        username=foo&password=bar

在`data`属性中进行数据绑定：

```java
public interface MyClient {

    /**
     * 这里 data 属性中设置的字符串内容会绑定到请求体中
     * 其中 ${0} 和 ${1} 为参数序号绑定，会将序号对应的参数绑定到字符串中对应的位置
     * ${0} 会替换为 username 的值，${1} 会替换为 password 的值
     */
    @Request(
            url = "http://localhost:5000/hello/user",
            type = "post",
            data = "username=${0}&password=${1}",
            headers = {"Accept:text/plain"}
    )
    String dataPost(String username, String password);
}
```

?> 其中具体的参数序号绑定内容请参见 [[6.1 参数序号绑定]](###_61-参数序号绑定)

如果调用方代码如下所示：

```java
myClient.dataPost("foo", "bar");
```

实际产生的 HTTP 请求如下：

    POST http://localhost:5000/hello/user
    HEADER:
        Accept: text/plain
    BODY:
        username=foo&password=bar

您可以直接把 JSON 数据加入到请求体中，其中`header`设置为`Content-Type: application/json`

```java
public interface MyClient {

    @Request(
            url = "http://localhost:5000/hello/user",
            type = "post",
            data = "{\"username\": \"${0}\", \"password\": \"${1}\"}",
            headers = {"Content-Type: application/json"}
    )
    String postJson(String username, String password);
}
```

如果调用方代码如下所示：

```java
myClient.postJson("foo", "bar");
```

实际产生的 HTTP 请求如下：

    POST http://localhost:5000/hello/user
    HEADER:
        Content-Type: application/json
    BODY:
        {"username": "foo", "password": "bar"}

把 XML 数据加入到请求体中，其中`header`设置为`Content-Type: application/json`

```java
public interface MyClient {

    @Request(
            url = "http://localhost:5000/hello/user",
            type = "post",
            data = "<misc><username>${0}</username><password>${1}</password></misc>",
            headers = {"Content-Type: application/xml"}
    )
    String postXml(String username, String password);
}
```

如果调用方代码如下所示：

```java
myClient.postXml("foo", "bar");
```

实际产生的 HTTP 请求如下：

    POST http://localhost:5000/hello/user
    HEADER:
        Content-Type: application/xml
    BODY:
        <misc><username>foo</username><password>bar</password></misc>
        

### 3.6.2 通过 @Body 注解

使用`data`属性太麻烦？绑定到`URL`还是`Body`搞不清楚？那您可以使用`@Body`注解修饰参数的方式，将传入参数的数据绑定到 HTTP 请求体中。

`@Body`注解修饰的参数一定会绑定到请求体中，不用担心它会出现在其他地方

```java
/**
 * 默认body格式为 application/x-www-form-urlencoded，即以表单形式序列化数据
 */
@Post(
    url = "http://localhost:${port}/user",
    headers = {"Accept:text/plain"}
)
String sendPost(@Body("username") String username,  @Body("password") String password);
```

### 3.6.3 绑定为JSON格式

要让`@Body`绑定的对象转换成`JSON`格式也非常简单，只要将`contentType`属性或`Content-Type`请求头指定为`application/json`便可。

```java
@Request(
    url = "http://localhost:5000/hello/user",
    contentType = "application/json", 
    type = "post"
)
String send(@Body User user);
```
调用后产生的结果如下：

    POST http://localhost:5000/hello/user
    HEADER:
        Content-Type: application/json
    BODY:
        {"username": "foo", "password": "bar"}


### 3.6.4 请求体为XML格式

`@Body`较为特殊，除了指定`contentType`属性或`Content-Type`请求头为`application/xml`外，还需要设置`@Body`的`filter`属性为`xml`。

```java
@Post(
    url = "http://localhost:5000/hello/user",
    contentType = "application/xml"
)
String send(@Body(filter = "xml") User user);
```

此外，这里的`User`对象也要绑定`JAXB`注解：

```java
@XmlRootElement(name = "misc")
public User {

    private String usrname;

    private String password;

    public String getUsername() {
        return username;
    }

    public void setUsername(String username) {
        this.username = username;
    }

    public String getPassword() {
        return password;
    }

    public void setPassword(String password) {
        this.password = password;
    }
}
```

调用传入`User`对象后的结果如下：

    POST http://localhost:5000/hello/user
    HEADER:
        Content-Type: application/xml
    BODY:
        <misc><username>foo</username><password>bar</password></misc>


## 3.7 @BaseRequest

`@BaseRequest`注解定义在接口类上，在`@BaseRequest`上定义的属性会被分配到该接口中每一个方法上，但方法上定义的请求属性会覆盖`@BaseRequest`上重复定义的内容。
因此可以认为`@BaseRequest`上定义的属性内容是所在接口中所有请求的默认属性。

```java
/**
 * @BaseRequest 为配置接口层级请求信息的注解，
 * 其属性会成为该接口下所有请求的默认属性，
 * 但可以被方法上定义的属性所覆盖
 */
@BaseRequest(
    baseUrl = "http://localhost:8080",     // 默认域名
    headers = {
        "Accept:text/plain"                // 默认请求头
    },
    sslProtocol = "TLS"                    // 默认SSL协议
)
public interface MyClient {
  
    // 方法的URL不必再写域名部分
    @Get(url = "/hello/user")     
    String send1(@Query("username") String username);

    // 若方法的URL是完整包含http://开头的，那么会以方法的URL中域名为准，不会被接口层级中的baseUrl属性覆盖
    @Get(url = "http://www.xxx.com/hello/user")
    String send2(@Query("username") String username);
  

    @Get(
        url = "/hello/user",
        headers = {
            "Accept:application/json"      // 覆盖接口层级配置的请求头信息
        }
    )     
    String send1(@Query("username") String username);

}
```

`@BaseRequest`注解中的所有字符串属性都可以通过[模板表达式](https://dt_flys.gitee.io/forest/#/?id=%E5%8D%81-%E6%A8%A1%E6%9D%BF%E8%A1%A8%E8%BE%BE%E5%BC%8F)引用[全局变量](https://dt_flys.gitee.io/forest/#/?id=_65-%E5%85%A8%E5%B1%80%E5%8F%98%E9%87%8F%E7%BB%91%E5%AE%9A)或方法中的参数。

```java
/** 
 * 若全局变量中已定义 baseUrl 和 accept，
 * 便会将全局变量中的值绑定到 @BaseRequest 的属性中
 */
@BaseRequest(
    baseUrl = "${baseUrl}",     // 默认域名
    headers = {
        "Accept:${accept}"      // 默认请求头
    }
)
public interface MyClient {

    // 方法的URL的域名将会引用全局变量中定义的 baseUrl
    @Get(url = "/hello/user")     
    String send1(@Query("username") String username);

    // @BaseRequest 中的属性亦可以引用方法中的绑定变量名的参数
    @Get(url = "/hello/user")
    String send2(@DataVariable("baseUrl") String baseUrl);
  

}

```

## 3.8 接受数据

Forest请求会自动将响应的返回数据反序列化成您要的数据类型。想要接受指定类型的数据需要完成两步操作：

第一步：定义`dataType`属性

`dataType`属性指定了该请求响应返回的数据类型，目前可选的数据类型有三种: `text`, `json`, `xml`

Forest会根据您指定的`dataType`属性选择不同的反序列化方式。其中`dataType`的默认值为`text`，如果您不指定其他数据类型，那么Forest就不会做任何形式的序列化，并以文本字符串的形式返回给你数据。

```java
/**
 * dataType为text或不填时，请求响应的数据将以文本字符串的形式返回回来
 */
@Request(
    url = "http://localhost:8080/text/data",
    dataType = "text"
)
String getData();
```

若您指定为`json`或`xml`，那就告诉了Forest该请求的响应数据类型为JSON或XML形式的数据，就会以相应的形式进行反序列化。

```java
/**
 * dataType为json或xml时，Forest会进行相应的反序列化
 */
@Request(
    url = "http://localhost:8080/text/data",
    dataType = "json"
)
Map getData();
```

第二步：指定反序列化的目标类型

反序列化需要一个目标类型，而该类型其实就是方法的返回值类型，如返回值为`String`就会反序列成`String`字符串，返回值为`Map`就会反序列化成一个HashMap对象，您也可以指定为自定义的Class类型。

```java
public class User {
    private String username;
    private String score;
    
    // Setter和Getter ...
}
```

如有上面这样的User类，并把它指定为方法的返回类型，而且相应返回的数据这样一段JSON：

```json
{"username":  "Foo", "score":  "82"}
```

那请求接口就应该定义成这样：

```java
/**
 * dataType属性指明了返回的数据类型为JSON
 */
@Get(
    url = "http://localhost:8080/user?id=${0}",
    dataType = "json"
)
User getUser(Integer id)
```

从`1.4.0`版本开始，`dataType` 属性默认为 `auto`（自动判断数据类型）， 也就是说 `dataType` 属性可以完全省略不填，Forest会自行判断返回的数据类型是哪种格式。

```java
/**
 * 省略dataType属性会自动判断返回的数据格式并进行反序列化
 */
@Get(url = "http://localhost:8080/user?id=${0}")
User getUser(Integer id)
```


## 3.9 回调函数

在Forest中的回调函数使用单方法的接口定义，这样可以使您在 `Java 8` 或 `Kotlin` 语言中方便使用 `Lambda` 表达式。

使用的时候只需在接口方法加入`OnSuccess<T>`类型或`OnError`类型的参数：

```java
@Request(
        url = "http://localhost:5000/hello/user",
        headers = {"Accept:text/plain"},
        data = "username=${username}"
)
String send(@DataVariable("username") String username, OnSuccess<String> onSuccess, OnError onError);
```

如这两个回调函数的类名所示的含义一样，`OnSuccess<T>`在请求成功调用响应时会被调用，而`OnError`在失败或出现错误的时候被调用。

其中`OnSuccess<T>`的泛型参数`T`定义为请求响应返回结果的数据类型。

```java
myClient.send("foo", (String resText, ForestRequest request, ForestResponse response) -> {
        // 成功响应回调
        System.out.println(resText);    
    },
    (ForestRuntimeException ex, ForestRequest request, ForestResponse response) -> {
        // 异常回调
        System.out.println(ex.getMessage());
    });
```

?> 提示：
在异步请求中只能通`OnSuccess<T>`回调函数接或`Future`返回值接受数据。
而在同步请求中，`OnSuccess<T>`回调函数和任何类型的返回值都能接受到请求响应的数据。
`OnError`回调函数可以用于异常处理，一般在同步请求中使用`try-catch`也能达到同样的效果。

## 3.10 异步请求

在Forest使用异步请求，可以通过设置`@Request`注解的`async`属性为`true`实现，不设置或设置为`false`即为同步请求。

```java
@Request(
        url = "http://localhost:5000/hello/user?username=${0}",
        async = true,
        headers = {"Accept:text/plain"}
)
void asyncGet(String username， OnSuccess<String> onSuccess);
```

一般情况下，异步请求都通过`OnSuccess<T>`回调函数来接受响应返回的数据，而不是通过接口方法的返回值，所以这里的返回值类型一般会定义为`void`。

?> 关于回调函数的使用请参见 [3.9 回调函数](###_39-回调函数)

```java
// 异步执行
myClient.asyncGet("foo", (result, request, response) -> {
    // 处理响应结果
    System.out.println(result);
});
```

此外，若有些小伙伴不习惯这种函数式的编程方式，也可以用`Future<T>`类型定义方法返回值的方式来接受响应数据。

```java
@Request(
        url = "http://localhost:5000/hello/user?username=foo",
        async = true,
        headers = {"Accept:text/plain"}
)
Future<String> asyncFuture();
```

这里`Future<T>`类就是`JDK`自带的`java.util.concurrent.Future`类, 其泛型参数`T`代表您想接受的响应数据的类型。

关于如何使用`Future`类，这里不再赘述。

```java
// 异步执行
Future<String> future = myClient.asyncFuture();

// 做一些其它事情

// 等待数据
String result = future.get();
```

# 四. 上传下载

Forest从 `1.4.0` 版本开始支持多种形式的文件上传和文件下载功能

## 4.1 上传

```java
/**
 * 用@DataFile注解修饰要上传的参数对象
 * OnProgress参数为监听上传进度的回调函数
 */
@Post(url = "/upload")
Map upload(@DataFile("file") String filePath, OnProgress onProgress);
```

调用上传接口以及监听上传进度的代码如下：

```java
Map result = myClient.upload("D:\\TestUpload\\xxx.jpg", progress -> {
    System.out.println("total bytes: " + progress.getTotalBytes());   // 文件大小
    System.out.println("current bytes: " + progress.getCurrentBytes());   // 已上传字节数
    System.out.println("progress: " + Math.round(progress.getRate() * 100) + "%");  // 已上传百分比
    if (progress.isDone()) {   // 是否上传完成
        System.out.println("--------   Upload Completed!   --------");
    }
});
```

在文件上传的接口定义中，除了可以使用字符串表示文件路径外，还可以用以下几种类型的对象表示要上传的文件:

```java
/**
 * File类型对象
 */
@Post(url = "/upload")
Map upload(@DataFile("file") File file, OnProgress onProgress);

/**
 * byte数组
 * 使用byte数组和Inputstream对象时一定要定义fileName属性
 */
@Post(url = "/upload")
Map upload(@DataFile(value = "file", fileName = "${1}") byte[] bytes, String filename);

/**
 * Inputstream 对象
 * 使用byte数组和Inputstream对象时一定要定义fileName属性
 */
@Post(url = "/upload")
Map upload(@DataFile(value = "file", fileName = "${1}") InputStream in, String filename);

/**
 * Spring Web MVC 中的 MultipartFile 对象
 */
@PostRequest(url = "/upload")
Map upload(@DataFile(value = "file") MultipartFile multipartFile, OnProgress onProgress);

/**
 * Spring 的 Resource 对象
 */
@Post(url = "/upload")
Map upload(@DataFile(value = "file") Resource resource);
```

## 4.2 下载

```java
/**
 * 在方法上加上@DownloadFile注解
 * dir属性表示文件下载到哪个目录
 * filename属性表示文件下载成功后以什么名字保存，如果不填，这默认从URL中取得文件名
 * OnProgress参数为监听上传进度的回调函数
 */
@Get(url = "http://localhost:8080/images/xxx.jpg")
@DownloadFile(dir = "${0}", filename = "${1}")
File downloadFile(String dir, String filename, OnProgress onProgress);
```


调用下载接口以及监听上传进度的代码如下：

```java
File file = myClient.downloadFile("D:\\TestDownload", progress -> {
    System.out.println("total bytes: " + progress.getTotalBytes());   // 文件大小
    System.out.println("current bytes: " + progress.getCurrentBytes());   // 已下载字节数
    System.out.println("progress: " + Math.round(progress.getRate() * 100) + "%");  // 已下载百分比
    if (progress.isDone()) {   // 是否下载完成
        System.out.println("--------   Download Completed!   --------");
    }
});
```

如果您不想将文件下载到硬盘上，而是直接在内存中读取，可以去掉@DownloadFile注解，并且用以下几种方式定义接口:

```java

/**
 * 返回类型用byte[]，可将下载的文件转换成字节数组
 */
@GetRequest(url = "http://localhost:8080/images/test-img.jpg")
byte[] downloadImageToByteArray();

/**
 * 返回类型用InputStream，用流的方式读取文件内容
 */
@GetRequest(url = "http://localhost:8080/images/test-img.jpg")
InputStream downloadImageToInputStream();
```

!>  **注意**：
用File类型定义的文件下载接口方法，一定要加上@DownloadFile注解


# 五. 配置

## 5.1 在 Spring Boot 项目中配置

若您的项目依赖`Spring Boot`，并加入了`spring-boot-starter-forest`依赖，就可以通过 `application.yml`/`application.properties` 方式定义配置。

### 5.1.1 配置后端 HTTP API

```yaml
forest:
  backend: okhttp3 # 配置后端HTTP API为 okhttp3
```

目前 Forest 支持`okhttp3`和`httpclient`两种后端 HTTP API，若不配置该属性，默认为`okhttp3`.
当然，您也可以改为`httpclient`

```yaml
forest:
  backend: httpclient # 配置后端HTTP API为 httpclient
```

### 5.1.2 全局基本配置

在`application.yaml` / `application.properties`中配置的 HTTP 基本参数

```yaml
forest:
  bean-id: config0 # 在spring上下文中bean的id, 默认值为forestConfiguration
  backend: okhttp3 # 后端HTTP API： okhttp3
  max-connections: 1000 # 连接池最大连接数，默认值为500
  max-route-connections: 500 # 每个路由的最大连接数，默认值为500
  timeout: 3000 # 请求超时时间，单位为毫秒, 默认值为3000
  connect-timeout: 3000 # 连接超时时间，单位为毫秒, 默认值为2000
  retry-count: 1 # 请求失败后重试次数，默认为0次不重试
  ssl-protocol: SSLv3 # 单向验证的HTTPS的默认SSL协议，默认为SSLv3
  logEnabled: true # 打开或关闭日志，默认为true
```
!>  **注意**：
这里`retry-count`只是简单机械的请求失败后的重试次数，所以一般建议设置为`0`。
如果一定要多次重试，请一定要在保证服务端的`幂等性`的基础上进行重试，否则容易引发生产事故！

### 5.1.3 全局变量定义

Forest 可以在`forest.variables`属性下自定义全局变量。

其中 key 为变量名，value 为变量值。

全局变量可以在任何模板表达式中进行数据绑定。

```yaml
forest:
  variables:
    username: foo
    userpwd: bar
```

### 5.1.4 配置 Bean ID

Forest 允许您在 yaml 文件中配置 Bean Id，它对应着`ForestConfiguration`对象在 Spring 上下文中的 Bean 名称。

```yaml
forest:
  bean-id: config0 # 在spring上下文中bean的id，默认值为forestConfiguration
```

然后便可以在 Spring 中通过 Bean 的名称引用到它

```java
@Resource(name = "config0")
private ForestConfiguration config0;
```



## 5.2 在非 Spring Boot 项目中配置

若您的项目不是`Spring Boot`项目，或者没有依赖`spring-boot-starter-forest`，可以通过下面方式定义 Forest 配置。

### 5.2.1 创建 ForestConfiguration 对象

`ForestConfiguration`为 Forest 的全局配置对象类，所有的 Forest 的全局基本配置信息由此类进行管理。

`ForestConfiguration`对象的创建方式：调用静态方法`ForestConfiguration.configuration()`，此方法会创建 ForestConfiguration 对象并初始化默认值。

```java
ForestConfiguration configuration = ForestConfiguration.configuration();
```

### 5.2.2 配置后端 HTTP API

```java
configuration.setBackendName("okhttp3");
```

目前 Forest 支持`okhttp3`和`httpclient`两种后端 HTTP API，若不配置该属性，默认为`okhttp3`。

当然，您也可以改为`httpclient`

```java
configuration.setBackendName("httpclient");
```

### 5.2.3 全局基本配置

```java
// 连接池最大连接数，默认值为500
configuration.setMaxConnections(123);
// 每个路由的最大连接数，默认值为500
configuration.setMaxRouteConnections(222);
// 请求超时时间，单位为毫秒, 默认值为3000
configuration.setTimeout(3000);
// 连接超时时间，单位为毫秒, 默认值为2000
configuration.setConnectTimeout(2000);
// 请求失败后重试次数，默认为0次不重试
configuration.setRetryCount(3);
// 单向验证的HTTPS的默认SSL协议，默认为SSLv3
configuration.setSslProtocol(SSLUtils.SSLv3);
// 打开或关闭日志，默认为true
configuration.setLogEnabled(true);
```

!>  **注意**：
这里`setRetryCount`只是简单机械的请求失败后的重试次数，所以一般建议设置为`0`。
如果一定要多次重试，请一定要在保证服务端的`幂等性`的基础上进行重试，否则容易引发生产事故！


### 5.2.4 全局变量定义

Forest 可以通过`ForestConfiguration`对象的`setVariableValue`方法自定义全局变量。

其中第一个参数为变量名，第二个为变量值。

全局变量可以在任何模板表达式中进行数据绑定。

```java
ForestConfiguration configuration = ForestConfiguration.configuration();
...
configuration.setVariableValue("username", "foo");
configuration.setVariableValue("userpwd", "bar");
```

## 5.3 配置层级

上面介绍的`application.yml` / `application.properties`配置以及通过`ForestConfiguration`对象设置的配置都是全局配置。

除了全局配置，Forest 还提供了接口配置和请求配置。

这三种配置的作用域和读取优先级各不相同。

作用域： 配置作用域指的是配置所影响的请求范围。

优先级： 优先级值的是是否优先读取该配置。比如您优先级最高`@Request`中定义了`timeout`为`500`，那么即便在全局配置中定了`timeout`为`1000`，最终该请求实际的`timeout`为优先级配置最高的`@Request`中定义的`500`。

具体的配置层级如图所示：

![avatar](media/config.png)

Forest 的配置层级介绍：

1. 全局配置：针对全局所有请求，作用域最大，配置读取的优先级最小。

2. 接口配置： 作用域为某一个`interface`中定义的请求，读取的优先级最小。您可以通过在`interface`上修饰`@BaseRequest`注解进行配置。

3. 请求配置： 作用域为某一个具体的请求，读取的优先级最高。您可以在接口的方法上修饰`@Request`注解进行 HTTP 信息配置的定义。


# 六. 数据绑定

上面已经介绍了如何创建可以发送 HTTP 请求的接口，并绑定到某个接口方法上，已经可以实现简单请求的发送和接受。

但问题是这些绑定的 HTTP 请求信息如 URL 和 HEAD 信息都是静态的不能动态改变，而我们在业务中大多数时候都需要动态地将数据传入到 HTTP 请求的各个部分（如 URL、参数、HEAD、BODY 等等），并发送到远端服务器。

这时候就需要`数据绑定`来实现这些功能，
Forest 提供多种方式进行`数据绑定`。

## 6.1 参数序号绑定

您可以使用`${数字}`的方式引用对应顺序的参数，其中`${...}`是模板表达式的语法形式。

序号所对应的参数在接口方法调用时传入的值，会被自动绑定到`${数字}`所在的位置。

`注`：参数序号从`0`开始计数。

比如`${0}`表示的就是第一个参数，`${1}`表示的第二个参数，以此类推。

```java

@Request(
    url = "${0}/send?un=${1}&pw=${2}&da=${3}&sm=${4}",
    type = "get",
    dataType = "json"
)
public Map send(
    String base,
    String userName,
    String password,
    String phoneList,
    String content
);
```

如果调用方代码如下所示：

```java
myClient.send("http://localhost:8080", "DT", "123456", "123888888", "Hahaha");
```

实际产生的 HTTP 请求如下：

    GET http://localhost:8080/send?un=DT&pw=123456&da=123888888&sm=Hahaha


## 6.2 @DataVariable 参数绑定

在接口方法中定义的参数前加上`@DataVariable`注解并`value`中输入一个名称，便可以实现参数的`变量名`绑定。

`@DataVariable`注解的`value`的值便是该参数在 Forest 请求中对应的`变量名`。

意思就是在`@Request`的多个不同属性（`url`, `headers`, `data`）中通过`${变量名}`的模板表达式的语法形式引用之前在`@DataVariable`注解上定义的`变量名`，实际引用到的值就是调用该方法时传入该参数的实际值。

`@DataVariable`注解修饰的参数数据可以出现在请求的任意部分（`url`, `header`, `data`），具体在哪个部分取决于您在哪里引用它。如果没有任何地方引用该变量名，该变量的值就不会出现在任何地方。

```java

@Request(
    url = "${base}/send?un=${un}&pw=${pw}&da=${da}&sm=${sm}",
    type = "get",
    dataType = "json"
)
public Map send(
    @DataVariable("base") String base,
    @DataVariable("un") String userName,
    @DataVariable("pw") String password,
    @DataVariable("da") String phoneList,
    @DataVariable("sm") String content
);
```

如果调用方代码如下所示：

```java
myClient.send("http://localhost:8080", "DT", "123456", "123888888", "Hahaha");
```

实际产生的 HTTP 请求如下：

    GET http://localhost:8080/send?un=DT&pw=123456&da=123888888&sm=Hahaha


### 6.3 绑定到请求体

参见 [3.4 HTTP URL](###_34-http-url)    

### 6.4 绑定到请求头

参见 [3.6 HTTP Header](###_35-http-header)    


### 6.5 绑定到请求体

参见 [3.6 HTTP Body](###_36-http-body)    


## 6.5 全局变量绑定

若您已经定义好全局变量，那便可以直接在请求定义中绑定全局变量了。

?> 关于如何定义全局变量请参见[Spring Boot 项目全局变量定义](###_513-全局变量定义)，或[非 Spring Boot 项目全局变量定义](###_524-全局变量定义)

若有全局变量：

forest:
    variables:
        basetUrl: http://localhost:5050
        usrename: foo
        userpwd: bar
        phoneList: 123888888

```java

@Request(
        url = "${basetUrl}/send?un=${usrename}&pw=${userpwd}&da=${phoneList}&sm=${sm}",
        type = "get",
        dataType = "json"
)
Map testVar(@DataVariable("sm") String content);
```

如果调用方代码如下所示：

```java
myClient.send("Xxxxxx");
```

实际产生的 HTTP 请求如下：

    GET http://localhost:5050/send?un=foo&pw=bar&da=123888888&sm=Xxxxxx

# 七. 使用请求接口

若您已有定义好的 Forest 请求接口(比如名为 `com.yoursite.client.MyClient`)，并且一切配置都已准备好，那就可以开始愉快使用它了。

## 7.1 在 Spring Boot 项目中调用接口

只要在`Spring Boot`的配置类或者启动类上加上`@ForestScan`注解，并在`basePackages`属性里填上远程接口的所在的包名

```java
@SpringBootApplication
@Configuration
@ForestScan(basePackages = "com.yoursite.client")
public class MyApp {
 ...
}
```

Forest 会扫描`@ForestScan`注解中`basePackages`属性指定的包下面所有的接口，然后会将符合条件的接口进行动态代理并注入到 Spring 的上下文中。

然后便能在其他代码中从 Spring 上下文注入接口实例，然后如调用普通接口那样调用即可。

```java
@Component
public class MyService {
    @Autowired
    private MyClient myClient;

    public void testClient() {
        Map result = myClient.send("http://localhost:8080", "DT", "123456", "123888888", "Hahaha");
        System.out.println(result);
    }

}
```

## 7.2 在非 Spring Boot 项目中调用接口

通过`ForestConfiguration`的静态方法`createInstance(Class clazz)`实例化接口，然后如调用普通接口那样调用即可。

```java
MyClient myClient = configuration.createInstance(MyClient.class);

...

Map result = myClient.send("http://localhost:8080", "DT", "123456", "123888888", "Hahaha");
System.out.println(result);
```

# 八. HTTPS

为保证网络访问安全，现在大多数企业都会选择使用SSL验证来提高网站的安全性。
 
所以Forest自然也加入了对HTTPS的处理，现在支持单向认证和双向认证的HTTPS请求。

## 8.1 单向认证

如果访问的目标站点的SSL证书由信任的Root CA发布的，那么您无需做任何事情便可以自动信任

```java

public interface Gitee {
    @Request(url = "https://gitee.com")
    String index();
}
```

Forest的单向验证的默认协议为`SSLv3`，如果一些站点的API不支持该协议，您可以在全局配置中将`ssl-protocol`属性修改为其它协议，如：`TLSv1.1`, `TLSv1.2`, `SSLv2`等等。  

```yaml
forest:
  ...
  ssl-protocol: TLSv1.2
```


全局配置可以配置一个全局统一的SSL协议，但现实情况是有很多不同服务（尤其是第三方）的API会使用不同的SSL协议，这种情况需要针对不同的接口设置不同的SSL协议。

```java
/**
 * 在某个请求接口上通过 sslProtocol 属性设置单向SSL协议
 */
@Get(
    url = "https://localhost:5555/hello/user",
    sslProtocol = "SSL"
)
ForestResponse<String> truestSSLGet();
```

在一个个方法上设置太麻烦，也可以在 `@BaseRequest` 注解中设置一整个接口类的SSL协议

```java
@BaseRequest(sslProtocol = "TLS")
public interface SSLClient {

    @Get(url = "https://localhost:5555/hello/user")
    String testSend();

}
```

## 7.2 双向认证

 若是需要在Forest中进行双向验证的HTTPS请求，也很简单。
 
 在全局配置中添加`keystore`配置：
 
 ```yaml
forest:
  ...
  ssl-key-stores:
    - id: keystore1           # id为该keystore的名称，必填
      file: test.keystore     # 公钥文件地址
      keystore-pass: 123456   # keystore秘钥
      cert-pass: 123456       # cert秘钥
      protocols: SSLv3        # SSL协议
```

接着，在`@Request`中引入该`keystore`的`id`即可

```java
@Request(
    url = "https://localhost:5555/hello/user",
    keyStore = "keystore1"
)
String send();
```

另外，您也可以在全局配置中配多个`keystore`：

```yaml
forest:
  ...
  ssl-key-stores:
    - id: keystore1          # 第一个keystore
      file: test1.keystore    
      keystore-pass: 123456  
      cert-pass: 123456      
      protocols: SSLv3       

    - id: keystore2          # 第二个keystore
      file: test2.keystore    
      keystore-pass: abcdef  
      cert-pass: abcdef      
      protocols: SSLv3       
      ...
```

随后在某个具体`@Request`中配置其中任意一个`keystore`的`id`都可以

# 九. 异常处理

发送HTTP请求不会总是成功的，总会有失败的情况。Forest提供多种异常处理的方法来处理请求失败的过程。

## 9.1 try-catch方式

最常用的是直接用`try-catch`。Forest请求失败的时候通常会以抛异常的方式报告错误， 获取错误信息只需捕获`ForestNetworkException`异常类的对象，如示例代码所示：

```java
/**
 * try-catch方式：捕获ForestNetworkException异常类的对象
 */
try {
    String result = myClient.send();
} catch (ForestNetworkException ex) {
    int status = ex.getStatusCode(); // 获取请求响应状态码
    ForestResponse response = ex.getResponse(); // 获取Response对象
    String content = response.getContent(); // 获取请求的响应内容
    String resResult = response.getResult(); // 获取方法返回类型对应的最终数据结果
}
```

## 9.2 回调函数方式

第二种方式是使用`OnError`回调函数，如示例代码所示：

```java
/**
 * 在请求接口中定义OnError回调函数类型参数
 */
@Request(
        url = "http://localhost:5000/hello/user",
        headers = {"Accept:text/plain"},
        data = "username=${username}"
)
String send(@DataVariable("username") String username, OnError onError);
```

调用的代码如下：

```java
// 在调用接口时，在Lambda中处理错误结果
myClient.send("foo",  (ex, request, response) -> {
    int status = response.getStatusCode(); // 获取请求响应状态码
    String content = response.getContent(); // 获取请求的响应内容
    String result = response.getResult(); // 获取方法返回类型对应的最终数据结果
});
```

!> 需要注意的是：加上`OnError`回调函数后便不会再向上抛出异常，所有错误信息均通过`OnError`回调函数的参数获得。


## 9.3 ForestResponse返回类型方式

第三种，用`ForestResponse`类作为请求方法的返回值类型，示例代码如下：

```java
/**
 * 用`ForestResponse`类作为请求方法的返回值类型, 其泛型参数代表实际返回数据的类型
 */
@Request(
        url = "http://localhost:5000/hello/user",
        headers = {"Accept:text/plain"},
        data = "username=${username}"
)
ForestResponse<String> send(@DataVariable("username") String username);
```

调用和处理的过程如下：

```java
ForestResponse<String> response = myClient.send("foo");
// 用isError方法判断请求是否失败, 比如404, 500等情况
if (response.isError()) {
    int status = response.getStatusCode(); // 获取请求响应状态码
    String content = response.getContent(); // 获取请求的响应内容
    String result = response.getResult(); // 获取方法返回类型对应的最终数据结果
}
```

!> 以`ForestResponse`类为返回值类型的方法也不会向上抛出异常，错误信息均通过`ForestResponse`对象获得。


## 9.4 拦截器方式

若要批量处理各种不同请求的异常情况，可以定义一个拦截器, 并在拦截器的`onError`方法中处理异常，示例代码如下：

````java
public class ErrorInterceptor implements Interceptor<String> {

    // ... ...

    @Override
    public void onError(ForestRuntimeException ex, ForestRequest request, ForestResponse response) {
        int status = response.getStatusCode(); // 获取请求响应状态码
        String content = response.getContent(); // 获取请求的响应内容
        Object result = response.getResult(); // 获取方法返回类型对应的返回数据结果
    }
}
````

?> 关于具体如何使用拦截器请参见 [拦截器](###十一-拦截器)


# 十. 模板表达式

在`@Request`的各大属性中大多数都是用`String`字符串填值的，如果要在这些字符串属性中动态地关联参数数据，用Java原生字符串连接(如`+`)是不行的，而且也不够直观。

所以Forest为了帮助您参数数据动态绑定到这些属性上，提供了模板表达式。

##  10.1 表达式Hello World

Forest的模板表达式是在普通的Java字符串中嵌入`${}`来实现字符串和数据的动态绑定.

嵌入的表达式由`$`符 + 左花括号`{`开始，到右花括号`}`结束，在两边花括号中间填写的内容是表达式的本体。

最简单的表达式可以是一个`@DataVariable`标注的变量名，或是一个全局配置中定义的全局变量名。

让我们来看一个最简单的模板表达式Hello World的例子吧

```java
@Request(url = "http://localhost:8080/hello/${name}")
String send(@DataVariable("name") String name);
```

若在调用`send`方法时传入参数为`"world"`，那么这时被表达式绑定`url`属性则会变成：

    http://localhost:8080/hello/world
    
## 10.2 引用数据

模板表达式最原始的目的就是各种各样的数据动态绑定到HTTP请求的各个属性中，要完成这一步就要实现对外部数据的引用。

Forest的模板表达式提供了两种最基本的数据引用方式：

### 10.2.1 变量名引用

如上面Hello World例子所示，表达式中可以直接引用`@DataVariable`所标注的变量名。除此之外也可以直接引用全局配置中定义的全局变量名。

```yaml
forest:
  variables:
    a: foo
    b: bar
```

我们在全局配置中定义了两个全局变量，分别为`a`和`b`。接着就可以在`@Request`中同时引用这两个变量。

```java
@Request(url = "http://localhost:8080/${a}/${b}")
String send();
```

调用`send()`方法后产生的`url`的值为：

    http://localhost:8080/foo/bar

这里因为是全局变量，`${a}`和`${b}`的值分别来自全局配置中的变量`a`和`b`的值，也就是`foo`和`bar`，所以并不需要在方法中传入额外的参数。


### 10.2.2 参数序号引用

直接在`${}`中填入从`0`开始的数字，其中的数字代表方法参数的序号，比如`${0}`代表方法的第一个参数，`${1}`代表第二个参数，第n个参数引用用`${n-1}`表示（这里的n是数字，并不是变量名）

!> 注意：代表参数序号的数字只能是`整数`，不能是小数，并且不能是负数。

```java
@Request(url = "http://localhost:8080/hello?p1=${0}&p2=${1}&p3=${2}")
String send(int a, int b, int c);
```

如调用`send()`方法并传入参数`3`, `6`, `9`, 那么产生的`url`值就是

    http://localhost:8080/hello?p1=3&p2=6&p3=9

以上这种`${数字}`的形式是参数序号的简化语法，而有时候`${}`中的数字如果和其它表达式结合起来参与计算，那此时它就不代表数字所对应的参数了，而只是纯粹的数字。如一下例子：

```java
@Request(url = "http://localhost:8080/hello?p1=${0.toString()}")
String send(int num);
```

如果此时调用方法`send(100)`，那么产生的`url`将是：

    http://localhost:8080/hello?p1=0


?> 这里使用了模板表达式的方法调用语法

这时`${}`中的`0`代表的并不是参数`num`的值，而仅仅就是数字`0`，作为被调用`toString()`方法的整数对象。

若想此时也引用参数序号传入参数`num`的值，并且也参与`toString()`方法调用的运算，也是有办法的。

这时就要用到参数序号的非简化语法`$` + `非负整数`了。

```java
@Request(url = "http://localhost:8080/hello?p1=${$0.toString()}")
String send(int num);
```
如果此时调用方法`send(100)`，那么产生的`url`将是：

    http://localhost:8080/hello?p1=100

这时我们所看到`${$0.toString()}`就我们所期望的`num`参数经过调用`toString()`方法最终返回的结果了。

#### 10.2.2.1 简化与非简化

说到这里，可能我们有些小伙伴就凌乱了。什么简化的？非简化的？不都是参数序号吗？怎么又变成数字了呢？

别急，其实要区分什么时候是数字，什么时候是参数序号，以及什么是简化参数序号，什么是非简化参数序号是很简单的，只要记住以下3条规则即可。

> 1. `${}`中只包含一个`非负整数`时，就是参数序号，且是简化形态的。如：`${1}`, `${5}`等等。
> 
> 2. `${}`中不只包含一个数字，还有其它东西存在时，那此时里面的数字都只是数字。如`${1.toString()}`, `${json(0)}`等等。
> 
> 3. `${}`中的`非负整数`以`$`符号开头，那它就是一个参数序号（非简化的），不管`${}`中只有一个还是有多个都是。如`${$1}`, `${json($0)}`, `$3.compareTo($2)`


!> 还有不要忘了参数序号只能是`整数`，并且`不能是负的`。


#### 10.2.2.1 参数序号总结

用参数序号方式比变量名方式更为简洁，因为不用定义`@DataVariable`注解，也不用引用冗长的变量名，是目前比较推荐的引用方式。

不过它也有缺点，就是在参数较多的时候较难立刻对应起来，不够直观，比较影响代码可读性。所以还请根据场景和入参的多寡来决定用哪种引用方式。

### 10.2.3 引用对象属性

模板表达式中除了可以引用变量和参数序号外，还可以引用它们的属性。 

属性引用和`java`以及`SpringEL`一样，通过在变量名或者参数序号后面跟上点`.`符号，再加上属性名即可。

```java
@Request(url = "http://localhost:8080/user/${user.username}")
String getUser(@DataVariable("user") User user);
```

现在我们调用`getUser()`方法，并传入一个`User`类的对象，那么`${user.username}`得到的结果就是调用user对象的`Getter`方法`getUsername()`所得到的值。

模板表达式支持连续的属性引用

```java
@Request(url = "http://localhost:8080/user/phone_number/${user.phone.number}")
String getUser(@DataVariable("user") User user);
```
这里`${user.phone.number}`的结果就相当于调用`user.getPhone().getNumber()`的结果。

### 10.2.3 对象方法调用

既然模板表示能支持对象属性的引用，那也支持对象方法的调用吗？答案是肯定的，且调用方法的语法与`Java`的一致。

```java
@Request(url = "http://localhost:8080/user/phone_number/${user.getUsername()}")
String getUser(@DataVariable("user") User user);
```

这里的`${user.getUsername()}`的运行结果和`Java`中调用`user.getUsername()`执行效果是一样的。

此外，模板表达式有个特别的语法，即当调用的方法中没有参数时可以把括号`()`省去。

```java
@Request(url = "http://localhost:8080/user/phone_number/${user.getUsername}")
String getUser(@DataVariable("user") User user);
```

这里的`${user.getUsername}`和上面的`${user.getUsername()}`是等价的。

传入参数的形式也和`Java`中的一样：

```java
@Request(url = "http://localhost:8080/user/phone_number/${user.getPhoneList().get(phoneIndex).getNumber()}")
String getUser(@DataVariable("user") User user, @DataVariable("phoneIndex") int phoneIndex);
```

也可以结合参数序号形式：

```java
@Request(url = "http://localhost:8080/user/phone_number/${$0.getPhoneList().get($1).getNumber()}")
String getUser(User user, int phoneIndex);
```

结合属性引用，进一步简化：

```java
@Request(url = "http://localhost:8080/user/phone_number/${$0.phoneList.get($1).number}")
String getUser(User user, int phoneIndex);
```


# 十一. 拦截器

用过Spring MVC的朋友一定对Spring的拦截器并不陌生，Forest也同样支持针对Forest请求的拦截器。

如果您想在很多个请求发送之前或之后做一些事情（如下日志、计数等等），拦截器就是您的好帮手。

## 11.1 构建拦截器

定义一个拦截器需要实现com.dtflys.forest.interceptor.Interceptor接口

````java
public class SimpleInterceptor implements Interceptor<String> {

    private final static Logger log = LoggerFactory.getLogger(SimpleInterceptor.class);

    /**
     * 该方法在被调用时，并在beforeExecute前被调用 
     * @Param request Forest请求对象
     * @Param args 方法被调用时传入的参数数组 
     */
    @Override
    public void onInvokeMethod(ForestRequest request, ForestMethod method, Object[] args) {
        log.info("on invoke method");
        
        // addAttribute作用是添加和Request以及该拦截器绑定的属性
        addAttribute(request, "A", "value1");  
        addAttribute(request, "B", "value2");
    }

    /**
     * 该方法在请求发送之前被调用, 若返回false则不会继续发送请求
     * @Param request Forest请求对象
     */
    @Override
    public boolean beforeExecute(ForestRequest request) {
        log.info("invoke Simple beforeExecute");
        // 执行在发送请求之前处理的代码
        request.addHeader("accessToken", "11111111");  // 添加Header
        request.addQuery("username", "foo");  // 添加URL的Query参数
        return true;  // 继续执行请求返回true
    }

    /**
     * 该方法在请求成功响应时被调用
     */
    @Override
    public void onSuccess(String data, ForestRequest request, ForestResponse response) {
        log.info("invoke Simple onSuccess");
        // 执行成功接收响应后处理的代码
        int status = response.getStatusCode(); // 获取请求响应状态码
        String content = response.getContent(); // 获取请求的响应内容
        String result = data;  // data参数是方法返回类型对应的返回数据结果
        result = response.getResult(); // getResult()也可以获取返回的数据结果
        response.setResult("修改后的结果: " + result);  // 可以修改请求响应的返回数据结果
        
        // 使用getAttributeAsString取出属性，这里只能取到与该Request对象，以及该拦截器绑定的属性
        String attrValue1 = getAttributeAsString(request, "A1");

    }

    /**
     * 该方法在请求发送失败时被调用
     */
    @Override
    public void onError(ForestRuntimeException ex, ForestRequest request, ForestResponse response) {
        log.info("invoke Simple onError");
        // 执行发送请求失败后处理的代码
        int status = response.getStatusCode(); // 获取请求响应状态码
        String content = response.getContent(); // 获取请求的响应内容
        String result = response.getResult(); // 获取方法返回类型对应的返回数据结果
    }

    /**
     * 该方法在请求发送之后被调用
     */
    @Override
    public void afterExecute(ForestRequest request, ForestResponse response) {
        log.info("invoke Simple afterExecute");
        // 执行在发送请求之后处理的代码
        int status = response.getStatusCode(); // 获取请求响应状态码
        String content = response.getContent(); // 获取请求的响应内容
        String result = response.getResult(); // 获取方法返回类型对应的最终数据结果
    }
}

````
`Interceptor`接口带有一个泛型参数，其表示的是请求响应后返回的数据类型。
Interceptor<String>即代表返回的数据类型为`String`。


## 11.2 拦截器与 Spring 集成

若我要在拦截器中注入 Spring 的 Bean 改如何做？

```java

/**
 * 在拦截器的类上加上@Component注解，并保证它能被Spring扫描到
 */
@Component
public class SimpleInterceptor implements Interceptor<String> {

    // 如此便能直接注入Spring上下文中所有的Bean了
    @Resouce
    private UserService userService;
    
    ... ...
}

```

## 11.3 在拦截器中传递数据

在Forest中，拦截器是基于单例模式创建的，也就是说一个拦截器类最多只能对应一个拦截器实例。

那么以下这种通过共享变量的方式就可能造成错误：

```java
public class SimpleInterceptor implements Interceptor<String> {
  
    private String name;
   
    @Override
    public boolean beforeExecute(ForestRequest request) {
        this.name = request.getQuery("name");
    }

    @Override
    public void onSuccess(String data, ForestRequest request, ForestResponse response) {
        System.out.println("name = " + name);
    }
}
```

若有两个请求同时进入该拦截器（请求1 url=...?name=A1, 请求2 url=...?name=A2）, 而最后当请求1进入`onSuccess`方法时，应该打印出 `name = A2`，却因为之前执行了请求2的`beforeExecute`方法，将类变量`name`的值改成了`A2`,
所以最终打印出来的是 `name = A2` （其实应该是 `name = A1`），这明显是错误的。

那该如何做能在传递数据的同时避免这类问题呢？ 

方法也很简单，就是将您要传递的数据与请求对象绑定在一起，比如在 `onSuccess` 中调用`request.getQuery`方法。

```java
System.out.println("name = " + forest.getQuery("name"));
```

虽然这种方法能够解决并发问题，但有个明显的限制：如果要传递的数据不想出现在请求中的任何位置(包括URL、请求头、请求体)，那就无能为力了。

这时候就要使用 `ForestRequest` 的扩展绑定数据的方法了。

### 11.3.1 Attribute

在拦截器中使用`addAttribute`方法和`getAttribute`方法来添加和获取`Attribute`。

`Attribute` 是和请求以及所在拦截器绑定的属性值，这些属性值不能通过网络请求传递到远端服务器。

而且，在使用`getAttribute`方法时，只能获取在相同拦截器，以及相同请求中绑定的`Attribute`，这两个条件缺一不可。

```java
public class SimpleInterceptor implements Interceptor<String> {
  
    @Override
    public void onInvokeMethod(ForestRequest request, ForestMethod method, Object[] args) {
        String methodName = method.getMethodName();
        addAttribute(request, "methodName", methodName); // 添加Attribute
        addAttribute(request, "num", (Integer) args[0]); // 添加Attribute
    }

    @Override
    public void onSuccess(String data, ForestRequest request, ForestResponse response) {
        Object value1 = getAttribute(request, "methodName");  // 获取名称为methodName的Attribute，不指定返回类型
        String value2 = getAttribute(request, "methodName", String.class);  // 获取名称为methodName的Attribute，并转换为指定的Class类型
        String value3 = getAttributeAsString(request, "methodName");  // 获取名称为methodName的Attribute，并转换为String类型
        Integer value4 = getAttributeAsInteger(request, "num");  // 获取名称为num的Attribute，并转换为Integer类型
    }
}
```

### 11.3.2 Attachment

可以使用`ForestRequest`对象的`addAttachment`方法和`getAttachment`方法来添加和获取`Attachment`。

`Attachment` 是和请求绑定的附件属性值，这些值不能通过网络请求传递到远端服务器。

而且，在使用`getAttachment`方法时，只能获取在相同请求中绑定的`Attachment`，但不必是相同的拦截器。


```java
public class SimpleInterceptor1 implements Interceptor<String> {
  
    @Override
    public void onInvokeMethod(ForestRequest request, ForestMethod method, Object[] args) {
        String methodName = method.getMethodName();
        request.addAttachment("methodName", methodName); // 添加Attachment
        request.addAttachment("num", (Integer) args[0]); // 添加Attachment
    }
    ... ...
}

/**
 * Attachment不依赖任何一个拦截器，可以跨拦截器传递数据
 */
public class SimpleInterceptor2 implements Interceptor<String> {
  
    @Override
    public void onSuccess(String data, ForestRequest request, ForestResponse response) {
        Object value1 = request.getAttachment("methodName");  // 获取名称为methodName的Attachment
        Object value2 = request.getAttachment("num");  // 获取名称为num的Attachment
    }
}
```

### 11.3.3 Attribute与Attachment的区别

`Attribute`和`Attachment`都是能通过请求进行绑定的数据传递方式，但也有所不同。

|             | 绑定请求 | 绑定拦截器 |
| ----------- | ---------| ---------- |
| Attribute  | <font color=green>✔</font> | <font color=green>✔</font> |
| Attachment | <font color=green>✔</font> | <font color=red>✘</font> |


## 11.4 配置拦截器

Forest有三个地方可以添加拦截器：`@Request`、`@BaseRequest`、全局配置，这三个地方代表三个不同的作用域。

### 11.4.1 @Request上的拦截器

若您想要指定的拦截器只作用在指定的请求上，只需要在该请求方法的`@Request`注解中设置`interceptor`属性即可。

```java

public interface SimpleClient {

    @Request(
            url = "http://localhost:8080/hello/user?username=foo",
            headers = {"Accept:text/plain"},
            interceptor = SimpleInterceptor.class
    )
    String simple();
}
```

`@Request`中拦截器可以配置多个:

```java
    @Request(
            url = "http://localhost:8080/hello/user?username=foo",
            headers = {"Accept:text/plain"},
            interceptor = {SimpleInterceptor1.class, SimpleInterceptor2.class, ...}
    )
    String simple();
```


?> `@Request`上的拦截器只会拦截指定的请求

### 11.4.2 @BaseRequest 上的拦截器

若您想使一个`interface`内的所有请求方法都指定某一个拦截器，可以在`@BaseRequest`的`interceptor`中设置

```java

@BaseRequest(baseURL = "http://localhost:8080", interceptor = SimpleInterceptor.class)
public interface SimpleClient {

    @Request(url = "/hello/user1?username=foo" )
    String send1();

    @Request(url = "/hello/user2?username=foo" )
    String send2();

    @Request(url = "/hello/user3?username=foo" )
    String send3();
}

```

如以上代码所示，`SimpleClient`接口中的`send1`、`send2`、`send3`方法都被会`SimpleInterceptor`拦截器拦截

`@BaseRequest`也如`@Request`中的`interceptor`属性一样，可以配1到多个拦截器，如代码所示：

```java
@BaseRequest(
    baseURL = "http://localhost:8080", 
    interceptor = {SimpleInterceptor1.class, SimpleInterceptor2.class, ...})
public interface SimpleClient {
    // ... ...
}
```

### 11.4.3 全局拦截器

若要配置能拦截项目范围所有Forest请求的拦截器也很简单，只要在全局配置中加上`interceptors`属性即可

```yaml
forest:
  ...
  interceptors:                   # 可配置1到多个拦截器
     com.your.site.client.SimpleInterceptor1
     com.your.site.client.SimpleInterceptor2
     ...
```


# 十二. 数据转换

Forest支持JSON、XML、普通文本等数据转换形式。不需要接口调用者自己写具体的数据转换代码。

## 12.1 序列化

几乎所有数据格式的转换都包含序列化和反序列化，Forest的数据转换同样如此。

Forest中对数据进行序列化可以通过指定`contentType`属性或`Content-Type`头指定内容格式。

```java

@Request(
        url = "http://localhost:5000/hello/user",
        type = "post",
        contentType = "application/json"    // 指定contentType为application/json
)
String postJson(@Body MyUser user);   // 自动将user对象序列化为JSON格式
```

同理，指定为`application/xml`会将参数序列化为`XML`格式，`text/plain`则为文本，默认的`application/x-www-form-urlencoded`则为表格格式。

## 12.2 反序列化

HTTP请求响应后返回结果的数据同样需要转换，Forest则会将返回结果自动转换为您通过方法返回类型指定对象类型。这个过程就是反序列化，您可以通过`dataType`指定返回数据的反序列化格式。

```java
@Request(
    url = "http://localhost:8080/data",
    dataType = "json"        // 指定dataType为json，将按JSON格式反序列化数据
)
Map getData();               // 请求响应的结果将被转换为Map类型对象
```

## 12.3  更换转换器

在Forest中已定义好默认的转换器，比如JSON的默认转为器为`ForestFastjsonConverter`，即`FastJson`的转换器。你也可以通过如下代码进行更换：

```java
@Autowrired
private ForestConfiguration forestConfiguration;

...

// 更换JSON转换器为FastJson
forestConfiguration.setJsonConverter(new ForestFastjsonConverter());
// 更换JSON转换器为Jackson
forestConfiguration.setJsonConverter(new ForestJacksonConverter());
// 更换JSON转换器Gson
forestConfiguration.setJsonConverter(new ForestGsonConverter());

// 更换XML转换器JAXB
forestConfiguration.getConverterMap().put(ForestDataType.XML, new ForestJaxbConverter());


```


## 12.4 自定义转换器

在Forest中，每个转换类型都对应一个转换器对象，比如`JSON`格式的转换器有`com.dtflys.forest.converter.json.ForestFastjsonConverter`、`com.dtflys.forest.converter.json.ForestGsonConverter`、`com.dtflys.forest.converter.json.ForestJacksonConverter`三种，分别是基于`FastJson`、`Gson`、`Jackson`三种不同的`JSON`序列化框架。

当然，您也可以自定义自己的转换器，以适应自己项目的需要。只需三步便可完成自定义扩展转换器。

第一步. 定义一个转换器类，并实现`com.dtflys.forest.converter.ForestConverter`接口

```java
/**
 *  自定义一个Protobuf的转换器，并实现ForestConverter接口下的convertToJavaObject方法
 */
public class MyProtobufConverter implements ForestConverter {

    <T> T convertToJavaObject(String source, Class<T> targetType) {
        // 将字符串参数source转换成目标Class对象
    }

    <T> T convertToJavaObject(String source, Type targetType) {
        // 将字符串参数source转换成目标Type(可能是一个泛型类型)对象
    }

}
```

第二步. 注册您定义好的转换器类到`ForestConfiguration`

```java
@Autowrired
private ForestConfiguration forestConfiguration;

...

// 设置文本转换器
forestConfiguration.getConverterMap().put(ForestDataType.TEXT, new MyProtobufConverter());
// 设置二进制转换器
forestConfiguration.getConverterMap().put(ForestDataType.BINARY, new MyProtobufConverter());
// 设置二进制转换器
forestConfiguration.getConverterMap().put(ForestDataType.BINARY, new MyProtobufConverter());

```

JSON转换器和XML转换器比较特殊，需要 `implements` 的类不是 `ForestConverter`

```java

/**
 *  JSON转换器需要实现 ForestJsonConverter 接口
 */
public class MyJsonConverter implements ForestJsonConverter {
   ... ...
}


/**
 *  XML转换器需要实现 ForestXmlConverter 接口
 */
public class MyXmlConverter implements ForestXmlConverter {
   ... ...
}


```


注册到配置中

```java

// 设置JSON转换器
forestConfiguration.setJsonConverter(new MyJsonConverter());
// 设置XML转换器
forestConfiguration.getConverterMap().put(ForestDataType.XML, new MyXmlConverter());

```


# 十三. 联系作者

您如有问题可以扫码加入微信的技术交流群

扫描二维码关注公众号，点击菜单中的 <font color=red>进群</font> 按钮即可进群

![avatar](https://dt_flys.gitee.io/forest/media/wx_gzh.jpg)


# 十四. 项目协议

The MIT License (MIT)

Copyright (c) 2016 Jun Gong


