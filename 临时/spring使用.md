[TOC]



## Bean

### @Repository和它的三个注解

@Repository衍生出三个注解：@Component、@Service、@Constroller。这四个注解功能是相同的

为什么 @Repository 只能标注在 DAO 类上呢？**这是因为该注解的作用不只是将类识别为 Bean，同时它还能将所标注的类中抛出的数据访问异常封装为 Spring 的数据访问异常类型。** Spring 本身提供了一个丰富的并且是与具体的数据访问技术无关的数据访问异常结构，用于封装不同的持久层框架抛出的异常，使得异常独立于底层的框架。

通过上述四个注解标识的 Bean，其默认作用域是"singleton"。

如果不想是单例，使用@Scope 注解。只需提供作用域的名称就行了。

```
@Scope("prototype") 
@Repository 
public class Demo { … }
```



#### @Autowired(required = false)找不到也没关系

问：如果属性注入找不到我不想让Spring容器抛出异常，而就是显示null，可以吗？

可以的，其实异常信息里面也给出了提示了，就是将@Autowired注解的required属性设置为false即可

```
    @Autowired(required = false)
    private Car car;
```



#### @Qualifier("BMW")接口有多个实现拿哪一个

问：一个Car接口有两个个实现，类Benz和BMW，要怎么拿？

用Qualifier

```
@Autowired
    @Qualifier("BMW")
    private Car car;
```

###  @Configuration 和 @Bean 进行方法声明

AnnotationConfigApplicationContext 将配置类中标注了 @Bean 的方法的返回值识别为 Spring Bean，并注册到容器中，受 IoC 容器管理。@Bean 的作用等价于 XML 配置中的 <bean/> 标签。示例如下：

```
@Configuration 
public class BookStoreDaoConfig{ 
   @Bean 
   public UserDao userDao(){ return new UserDaoImpl();} 
   @Bean 
   public BookDao bookDao(){return new BookDaoImpl();} 
}
```



## bean回调方法@PostConstruct 和 @PreDestroy

在某些情况下，可能需要我们手工做一些额外的初始化或者销毁操作，这通常是针对一些资源的获取和释放操作。

@PostConstruct 和 @PreDestroy

由于使用了注解，因此需要配置相应的 Bean 后处理器，亦即在 XML 中增加如下一行



[spring注解](https://www.cnblogs.com/xrq730/p/5313412.html)

[IBM，语言比较晦涩](https://www.ibm.com/developerworks/cn/opensource/os-cn-spring-iocannt/index.html)