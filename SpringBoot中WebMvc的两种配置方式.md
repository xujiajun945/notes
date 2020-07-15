# SpringBoot中配置WebMvc

​		SpringBoot为我们提供了开箱即用的自动化配置, 省去了我们大量的配置, 但在实际使用中, 我们还是会对消息转换器, 拦截器, 跨域设置等mvc相关设置做自定义配置

​		常见的配置方法有两种

## 1. 继承WebMvcConfigurationSupport类, 重写相关方法

​		通过继承该类, 并在该继承类上添加@Configuration注解, 可以通过重写父类相关方法完成配置, WebMvcConfigurationSupport 中提供了一些内容为空的方法, 这些方法主要是用来给子类自定义时实现, 根据方法名称的分类, 有以下几种类型:

1. addXxx()

   为xxx类型的处理器添加用户自定义的处理器

2. configureXxx()

   重新配置xxx类型的处理器
   
3. extendXxx()

   扩展xxx类型的处理器, 该方法一般放在addXxx和configureXxx后面
   
4. getXxx()

   将xxx类型的处理器完全由子类控制

## 2. 实现WebMvcConfigurer接口, 实现相关方法

​		通过实现该接口, 可以实现相关的方法完成配置(注: Spring 5.0之前, 该接口具有一个抽象实现类WebMvcConfigurerAdapter, 我们需要对该抽象类进行实现, 否则我们必须实现WebMvcConfigurer的全部方法, 不利于定制化配置, spring5.0之后, 使用java8新机制, 接口可以有默认实现, 因此WebMvcConfigurerAdapter抽象类被废弃)

​		WebMvcConfigurer接口中包含了全部的WebMvcConfigurationSupport 中的配置方法, 其默认实现都为空方法



## Mvc配置类的配置原理

​		这里需要介绍两个新的类: DelegatingWebMvcConfiguration 和 WebMvcConfigurerComposite 

DelegatingWebMvcConfiguration 类 简称【代理类】通过继承 WebMvcConfigurationSupport, 所以拥有了 customize 自定义mvc 配置的权利. 该类通过持有的对象 WebMvcConfigurerComposite 简称【聚合类】则是一个聚合了所有实现了 WebMvcConfigurer 接口的子类, 如此一来, 【代理类】所有实现 WebMvcConfigurationSupport 的方法调用时, 则是将【聚合类】中的所有WebMvcConfiguer 接口的实现类的该方法依次进行调用:

```java
class WebMvcConfigurerComposite implements WebMvcConfigurer {
    
    // 持有应用程序中所有实现 WebMvcConfigurer 接口的子类的一个集合
    private final List<WebMvcConfigurer> delegates = new ArrayList<WebMvcConfigurer>();

    public void addWebMvcConfigurers(List<WebMvcConfigurer> configurers) {
        if (!CollectionUtils.isEmpty(configurers)) {
            this.delegates.addAll(configurers);
        }
    }

    // 被父类调用的回调方法，来自 customize 各个子类的配置
    @Override
    public void configurePathMatch(PathMatchConfigurer configurer) {
        for (WebMvcConfigurer delegate : this.delegates) {
            delegate.configurePathMatch(configurer);
        }
    }
        ......
```



## 注意事项

​		**WebMvcConfigurationSupport 在整个程序中只会有一个生效, 如果用户已经实现了WebMvcConfigurationSupport , 则DelegatingWebMvcConfigurationSupport 不会生效(因为DelegatingWebMvcConfigurationSupport 继承WebMvcConfigurationSupport ), 而所有实现WebMvcConfigurer接口的实例被DelegatingWebMvcConfigurationSupport 持有, 因此WebMvcConfigurer的所有实现类都不会生效**, 在Spring中, 当WebMvcConfigurationSupport 实例不存在时(用户没有继承它的实例), Spring会默认初始化DelegatingWebMvcConfigurationSupport 

![WebMvcAutoConfiguration的触发条件](.\img\WebMvcAutoConfiguration的触发条件.png)

而如果我们也没有WebMvcConfigurer的实例, 则Spring会启用一个默认的实现类:

![WebMvcConfigurer默认实现](.\img\WebMvcConfigurer默认实现.png)



## 结语

​		因此, 两种默认的配置方法只能同时生效其一, 一般我们会采用方式2, 也就是实现接口的形式来进行配置, 因为通过继承的方式, SpringBoot的自动配置类将不生效, 我们需要对mvc的全部配置进行个性化定制, 且由于WebMvcConfigurationSupport的实例只能存在一个, 当多个实例存在时会被覆盖, 配置起来不灵活, 如果采用方式2来配置, 多个WebMvcConfigurer实例都会生效, 更加的灵活