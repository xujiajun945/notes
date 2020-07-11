# Spring的声明式事务

​		今天读到了一个关于Spring声明式事务的文章, 里面提到了关于声明式事务失效的问题, 具体如下:

## 1. 当声明式事务所在的方法为非public时, 事务不生效

​		例如如下代码:

```java
@Service
public class XXXServiceImpl implement XXXService {	

	@Transactional
	private void methodName(T params) {
        throw new RuntimeException("test transactional");
    }
}
```



## 2. 当声明式事务所在的方法被当前对象另一个方法内部调用, 事务不生效

​		例如如下代码:


```java		
@Service
public class XXXServiceImpl implement XXXService {
	// 方法A
	public void methodA() {
        this.methodB();
    }

	// 带有声明式事务注解的方法B
	@Transactional
	public void methodB() {
        throw new RuntimeException("test transactional!");
    }
}
```



​		在这两种情况下,  事务不会回滚,  因为我们在使用Spring的时候注入的其实是当前Service实现类的一个代理对象, 事务是根据Spring AOP实现的而AOP的实现原理其实是**动态代理**, 简单的说, 当Spring在初始化创建一些有advice的类的时候, 会触发Spring在AOP相关包下的bean代理生成的相关代码, 生成一个当前bean的代理bean来取代原来的bean对象被注入Spring容器中.

​		**Spring生成的新的代理只会处理被public修饰的方法, 因此, 情况1可以延伸为, Spring中的所有AOP相关操作,  其方法本身必须为public的**

​		同理也可以解释情况2事务不生效的原因: 由于Spring对当前Service生成了新的代理对象去操作原Service类中的methodA与methodB,  代理对象在操作具有@Transaction注解的methodB时会添加自己的AOP相关代码(在这里是控制事务回滚的相关代码), 而方法A中的this指向的其实是原Service类的实例, 因此事务无法生效.

### 那么如何让情况2下methodB的事务生效呢?

请看如下代码:

```java
@Service
public class XXXServiceImpl implement XXXService {	
    
    // 注入本身, 其实是注入Spring对当前接口的实现生成的一个代理对象
	@Autowire
	private XXXService xxxService;

	// 方法A
	public void methodA() {
        // 使用代理对象去调用methodB
        xxxService.methodB();
    }
    
    // 添加的声明式事务注解的方法B
    @Transactional
    public void methodB() {
        throw new RuntimeException("test transactional!");
    }
}
```

​		由于我们注入了代理对象, 此时的调用是通过当前实例的代理对象而不是实例本身, 事务生效

> 在实际开发中, 由于在bean中注入本身这件事很奇怪, 我们可以使用Spring提供的AopContext来获取当前类的代理对象
>
> ``` java
> @Service
> public class XXXServiceImpl implement XXXService {
> 
> 	// 方法A
> 	public void methodA() {
>         // 使用代理对象去调用methodB
>         this.currentProxy().methodB();
>     }
>     
>     // 添加的声明式事务注解的方法B
>     @Transactional
>     public void methodB() {
>         throw new RuntimeException("test transactional!");
>     }
>     
>     private XXXService currentProxy() {
>         return (XXXService) AopContext.currentProxy();
>     }
> }
> ```



## 总结

​		研究Spring的事务, 其实本质上是在研究Spring的AOP机制, Spring在处理一些AOP相关注解时会对当前类自动生成代理对象, 用代理对象去操作原对象中的对应方法

