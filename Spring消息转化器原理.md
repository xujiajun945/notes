

# Spring的消息转换器

​		最近在工作当中, 发现一个关于spring消息转换器引发的问题. 事情是这样的, 自己负责的某项目在上传文件时, 会有如下操作:

 1. 计算该json文件的md5

 2. 上传该json文件到oss服务器

​		而客户端在使用该文件的时候会:

 1. 查询该json文件的url与md5

 2. 根据url下载文件, 并计算对比下载文件的md5与数据库中的md5值

 3. 当md5相等时, 使用该文件, 否则报错

## 出现的问题

​		刚刚上传上去的json文件, 下载下来后md5计算与上传前一直不相等, 造成上传的文件无法使用, 客户端大量报错. 经过排查后发现并非cdn缓存的问题

## 分析

  1. 公司上传文件的接口是在一个基础服务的项目中, 其他子项目上传文件需要调用该基础服务项目的文件上传接口

  2. 基础服务的上传接口已经存在很久, 排除该接口的bug

  3. 自己负责的项目中的业务代码很久之前已经存在, 一直没有改动, 在没有改代码的情况下突然出现bug, 原因不在接口里

  4. 最近我们对公司全部的后台项目进行了技术迁移, 从dubbo迁移到了Spring cloud

​		那么问题多半出在了这次技术迁移上, dubbo的服务之间的调用方式是rpc, 使用的是dubbo协议, 而切换成Spring Cloud后, 服务之间的调用方式为http, 使用Spring Cloud后, 再从基础服务调用接口时, **需要经历自己项目的消息转换器(发消息)以及基础服务项目的消息转换器(接受消息)**

## 消息转换器的作用

​		在SpringMVC中消息转换器用来处理请求参数以及返回参数, 因为在http协议的网络传输中, 各种类型的参数最后都会被处理成字符串, 而java是强类型语言, 需要对请求参数进行处理, 转化成为特定类型(**反序列化**), 需要对特定类型的参数转化为字符串(**序列化**)

## 问题解决

​		通过排查发现, 基础服务项目中, 对Spring的消息转换器进行了自动配置, 如图:

![微信截图_20200715181721](D:\notes\img\一个错误的消息转换器自动配置类.png)

可以看到, 自己创建一个HttpMessageConverters对象, 并交给了Spring容器管理, 我们翻看SpringBoot的源码, 找一下它的消息转化器自动配置类, 如下:![](D:\notes\img\SpringBoot消息转换器自动配置类.png)

当容器中不存在HttpMessageConverters时, 会创建一个HttpMessageConverters, 我们继续进到它的构造器里:

![HttpMessageConverters构造器](D:\notes\img\HttpMessageConverters构造器.png)

注意这里的getDefaultConverters()方法, 我在进到这个方法里一路进去后, 发现最终Spring创建很多默认的消息转换器:![添加默认消息转换器](D:\notes\img\添加默认消息转换器.png)

最后最关键的一步来了:

![消息转换器的添加顺序](D:\notes\img\消息转换器的添加顺序.png)

现在, 我们知道了消息转换器的添加, 以及知道了消息转换器在添加的时候是有顺序的, 那么跟我们的bug有什么关系呢, 我们在对比了上传后的json和原始json后, 发现经过基础服务接口上传的json, 内容中出现了一些**""**, <u>即原本应该为: {"id": "1"}的地方, 变成了: {"\"id\"": ""1""}</u>, 

## String与jsonString

​		这个是很多人都会忽略的一个点, 因为jsonString是对象序列化的字符串, 因此, 它必须满足一定的格式要求, 造成上面的原因, 就是本来应该被当成String处理的字符串, 被当成了jsonString处理

```java
ObjectMapper objectMapper = new ObjectMapper();
// 普通字符串
String json = "";
// jsonString
String jsonString = objectMapper.writeValueAsString(json);  // jsonString = "\"\"";
```

## 消息转换器的处理机制

​		从DispatcherServlet开始, 一直找请求参数是怎么被消息转换器处理的, 最后看到源码处:

![消息转换器的处理逻辑](D:\notes\img\消息转换器的处理逻辑.png)

由于消息转换器是按照顺序被取出的, 因此第一个被取出的就是我们配置的FastJsonHttpMessageConverter, 它是用来处理json字符串的, <u>**本质上是一个特殊的字符串消息转换器**</u>, 而Spring本身是提供了处理普通字符串的StringHttpMessageConverter类的, 由于我们的配置,  FastJsonHttpMessageConverter跑到了数组的最前面, 来不及被StringHttpMessageConverter处理, 就被当做了json字符串处理, 因此, 我们向基础服务传参的全部参数, 都被当做json字符串处理了

## 结语

​		最后我们调整FastJsonHttpMessageConverter的顺序到StringHttpMessageConverter后面, 成功解决了这个问题, 通过这个问题, 了解了Spring中消息转换器的处理原理

