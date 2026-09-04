## Spring面试题

Spring框架核心特性包括：

IoC容器：Spring通过控制反转实现了对象的创建和对象间的依赖关系管理。开发者只需要定义好Bean及其依赖关系，Spring容器负责创建和组装这些对象。
AOP：面向切面编程，允许开发者定义横切关注点，例如事务管理、安全控制等，独立于业务逻辑的代码。通过AOP，可以将这些关注点模块化，提高代码的可维护性和可重用性。
事务管理：Spring提供了一致的事务管理接口，支持声明式和编程式事务。开发者可以轻松地进行事务管理，而无需关心具体的事务API。
MVC框架：Spring MVC是一个基于Servlet API构建的Web框架，采用了模型-视图-控制器（MVC）架构。它支持灵活的URL到页面控制器的映射，以及多种视图技术

### IoC 与 AOP

Spring IoC和AOP 区别：

IoC：即控制反转的意思，它是一种创建和获取对象的技术思想，依赖注入(DI)是实现这种技术的一种方式。传统开发过程中，我们需要通过new关键字来创建对象。使用IoC思想开发方式的话，我们不通过new关键字创建对象，而是通过IoC容器来帮我们实例化对象。 通过IoC的方式，可以大大降低对象之间的耦合度。
AOP：是面向切面编程，能够将那些与业务无关，却为业务模块所共同调用的逻辑封装起来，以减少系统的重复代码，降低模块间的耦合度。Spring AOP 基于动态代理：如果被代理对象实现了接口，Spring AOP 默认会使用 JDK Proxy 创建代理对象；如果没有实现接口，则会使用 CGLIB 生成被代理对象的子类作为代理。注意：自 Spring Boot 2.0 起，默认配置 spring.aop.proxy-target-class=true，即无论是否实现接口都优先使用 CGLIB，如需切回 JDK 动态代理需手动设为 false。
在 Spring 框架中，IOC 和 AOP 结合使用，可以更好地实现代码的模块化和分层管理。例如：

通过 IOC 容器管理对象的依赖关系，然后通过 AOP 将横切关注点统一切入到需要的业务逻辑中。
使用 IOC 容器管理 Service 层和 DAO 层的依赖关系，然后通过 AOP 在 Service 层实现事务管理、日志记录等横切功能，使得业务逻辑更加清晰和可维护。

### AOP实现原理

Spring AOP的实现依赖于动态代理技术。动态代理是在运行时动态生成代理对象，而不是在编译时。它允许开发者在运行时指定要代理的接口和行为，从而实现在不修改源码的情况下增强方法的功能。

Spring AOP支持两种动态代理：

- 基于JDK的动态代理：使用java.lang.reflect.Proxy类和java.lang.reflect.InvocationHandler接口实现。这种方式需要代理的类实现一个或多个接口。
- 基于CGLIB的动态代理：当被代理的类没有实现接口时，Spring会使用CGLIB库生成一个被代理类的子类作为代理。CGLIB（Code Generation Library）是一个第三方代码生成库，通过继承方式实现代理。

### AOP 通知类型与执行顺序

**五种通知（Advice）**：

| 通知 | 注解 | 执行时机 |
|---|---|---|
| 前置通知 | @Before | 目标方法执行前 |
| 后置通知 | @After | 目标方法执行后（无论是否抛异常都执行，类似 finally） |
| 返回通知 | @AfterReturning | 目标方法正常返回后（可拿到返回值） |
| 异常通知 | @AfterThrowing | 目标方法抛异常后 |
| 环绕通知 | @Around | 包裹整个方法，可控制目标方法是否执行、修改参数/返回值/异常 |

**@Around 与 ProceedingJoinPoint**：@Around 里通过 `proceed()` 手动调用目标方法，是最强大的通知，可以完全接管方法调用（事务、缓存、限流等都用它实现）。

```java
@Around("execution(* com.demo.service.*.*(..))")
public Object around(ProceedingJoinPoint pjp) throws Throwable {
    long start = System.currentTimeMillis();
    try {
        return pjp.proceed();          // 执行目标方法
    } finally {
        System.out.println("耗时: " + (System.currentTimeMillis() - start) + "ms");
    }
}
```

**同一切面内多个通知的执行顺序**（面试细节）：@Around 前部分 → @Before → 目标方法 → @AfterReturning/@AfterThrowing → @After → @Around 后部分。

**切点表达式（Pointcut）**：`execution(修饰符 返回类型 包.类.方法(参数))`，如 `execution(* com.demo.service.*.*(..))` 匹配 service 包下所有方法；也可用注解切点 `@annotation(com.demo.annotation.Cache)`。

### @Autowired 注入原理

**@Autowired 是 Spring 提供的依赖注入注解，默认按类型（byType）注入**，其实现核心是 **AutowiredAnnotationBeanPostProcessor**（一个 BeanPostProcessor，在属性填充阶段工作）：

```text
1. 扫描 Bean 中标注 @Autowired 的字段/方法（在 postProcessProperties 阶段）
2. 按字段类型从容器中查找候选 Bean
3. 找到唯一一个 → 直接注入
4. 找到多个（类型相同）→ 按名称（字段名）匹配；仍失败则注入失败
5. 找不到 → 默认报错（required=true）；可以 required=false 允许为空
```

**与 @Resource 的区别（常考）**：

| 维度 | @Autowired | @Resource |
|---|---|---|
| 来源 | Spring | JDK 标准（javax.annotation） |
| 默认策略 | 按类型（byType） | 按名称（byName） |
| 多实现处理 | 需配 @Qualifier 指定 | name 属性直接指定 |

**循环依赖与 @Autowired 的关系**：@Autowired 默认支持字段/setter 注入的循环依赖（依赖 Spring 三级缓存），但**构造器注入的循环依赖无法解决**——这也是阿里规范推荐构造器注入却要避免循环依赖的原因。

### BeanFactory 与 ApplicationContext 的区别

| 维度 | BeanFactory | ApplicationContext |
|---|---|---|
| 定位 | 最底层的 IoC 容器接口，最基础 | BeanFactory 的子接口，功能增强版 |
| 加载时机 | 懒加载：getBean() 时才创建 Bean | 预加载：启动时立即初始化单例 Bean |
| 扩展能力 | 基础：创建/获取 Bean | 增加：国际化（MessageSource）、事件发布（ApplicationEventPublisher）、资源加载（ResourceLoader）、AOP 集成、Environment 管理 |
| 实现 | XmlBeanFactory（已废弃） | ClassPathXmlApplicationContext、AnnotationConfigApplicationContext |
| 实际使用 | 几乎不用，Spring 内部使用 | 开发中默认使用 |

**Spring Boot 中的应用**：SpringApplication.run() 内部创建的是 AnnotationConfigApplicationContext（通过 refresh() 完成 Bean 的定义加载、实例化、初始化全流程）。

### Spring Boot 启动流程（简化版）

```text
1. SpringApplication.run()：创建应用上下文
2. 环境准备：加载 application.yml、系统属性、环境变量（Environment 就绪）
3. 创建 ApplicationContext（Servlet 应用 → AnnotationConfigServletWebServerApplicationContext）
4. 执行自动配置：@EnableAutoConfiguration 导入所有自动配置类（见上一节）
5. refresh() 容器：
   ① BeanFactory 准备与注册
   ② BeanDefinition 加载（扫描 @Component/@Configuration + 自动配置类）
   ③ BeanPostProcessor 注册（@Autowired、AOP、事务的基石）
   ④ 单例 Bean 实例化（预加载）
   ⑤ 内嵌 Tomcat 启动（Servlet 容器初始化）
6. 启动完成：执行 ApplicationRunner / CommandLineRunner 回调
```

### 动态代理是什么

Java的动态代理是一种在运行时动态创建代理对象的机制，主要用于在不修改原始类的情况下对方法调用进行拦截和增强。

Java动态代理主要分为两种类型：

- 基于接口的代理（JDK动态代理）： 这种类型的代理要求目标对象必须实现至少一个接口。Java动态代理会创建一个实现了相同接口的代理类，然后在运行时动态生成该类的实例。这种代理的实现核心是java.lang.reflect.Proxy类和java.lang.reflect.InvocationHandler接口。每一个动态代理类都必须实现InvocationHandler接口，并且每个代理类的实例都关联到一个handler。当通过代理对象调用一个方法时，这个方法的调用会被转发为由InvocationHandler接口的invoke()方法来进行调用。

- 基于类的代理（CGLIB动态代理）： CGLIB（Code Generation Library）是一个强大的高性能的代码生成库，它可以在运行时动态生成一个目标类的子类。CGLIB代理不需要目标类实现接口，而是通过继承的方式创建代理类。因此，如果目标对象没有实现任何接口，可以使用CGLIB来创建动态代理。

### 动态代理和静态代理的区别

代理是一种常用的设计模式，目的是：为其他对象提供一个代理以控制对某个对象的访问，将两个类的关系解耦。代理类和委托类都要实现相同的接口，因为代理真正调用的是委托类的方法。

区别：

- 静态代理：由程序员创建或者是由特定工具创建，在代码编译时就确定了被代理的类是一个静态代理。静态代理通常只代理一个类；
- 动态代理：在代码运行期间，由 JVM 根据目标对象自动生成代理类，不需要为每个被代理类手写代理类。JDK 动态代理基于接口（要求目标对象实现接口），CGLIB 动态代理基于继承（通过生成目标类的子类作为代理，不要求目标对象实现接口）。

### spring是如何解决循环依赖的

循环依赖指的是两个类中的属性相互依赖对方，循环依赖问题在Spring中主要有三种情况：

第一种：通过构造方法进行依赖注入时产生的循环依赖问题。
第二种：通过setter方法进行依赖注入且是在多例（原型）模式下产生的循环依赖问题。
第三种：通过 setter 方法或字段（Field）进行依赖注入且是在单例模式下产生的循环依赖问题。

只有【第三种方式】的循环依赖问题被 Spring 解决了，其他两种方式在遇到循环依赖问题时，Spring都会产生异常。

### spring三级缓存

三者都是 Map 类型的缓存，但 value 的类型不同：

- 一级缓存（Singleton Objects）：Map<String, Object> 存储的是已经完全初始化好的 bean，即完全准备好可以使用的 bean 实例。键是 bean 的名称，值是 bean 的实例。对应 DefaultSingletonBeanRegistry 类中的 singletonObjects 属性。
- 二级缓存（Early Singleton Objects）：Map<String, Object> 存储的是早期的 bean 引用，即已经实例化但还未完全初始化的 bean（可能是原始对象，也可能是为解决 AOP 循环依赖而提前生成的代理对象）。这些 bean 已经被实例化，但可能还没有完成属性注入等操作。对应 DefaultSingletonBeanRegistry 类中的 earlySingletonObjects 属性。
- 三级缓存（Singleton Factories）：Map<String, ObjectFactory<?>> 注意 value 不是 bean 实例本身，而是一个 ObjectFactory 工厂函数。当一个 bean 正在创建过程中被其他 bean 依赖时，就会调用这个工厂的 getObject() 方法生成早期引用（必要时会提前生成代理），从而解决循环依赖与 AOP 协同的问题。对应 DefaultSingletonBeanRegistry 类中的 singletonFactories 属性。

### bean的生命周期

Spring 只帮我们管理单例模式 Bean 的完整生命周期，对于 prototype 的 Bean，Spring 在创建好交给使用者之后，则不会再管理后续的生命周期。

Bean作用域（Scope）定义了Bean的生命周期和可见性。不同的作用域影响着 Spring 如何管理这些Bean。

- Singleton（单例）：在整个应用程序中只存在一个 Bean 实例。默认作用域，Spring 容器中只会创建一个 Bean 实例，并在容器的整个生命周期中共享该实例。
- Prototype（原型）：每次请求时都会创建一个新的 Bean 实例。每次从容器中获取该 Bean 时都会创建一个新实例，适用于状态非常瞬时的 Bean。
- Request（请求）：每个 HTTP 请求都会创建一个新的 Bean 实例。仅在 Spring Web 应用程序中有效，每个 HTTP 请求都会创建一个新的 Bean 实例，适用于 Web 应用中需求局部性的 Bean。
- Session（会话）：Session 范围内只会创建一个 Bean 实例。该 Bean 实例在用户会话范围内共享，仅在 Spring Web 应用程序中有效，适用于与用户会话相关的 Bean。
- Application：当前 ServletContext 中只存在一个 Bean 实例。仅在 Spring Web 应用程序中有效，该 Bean 实例在整个 ServletContext 范围内共享，适用于应用程序范围内共享的 Bean。
- WebSocket（Web套接字）：在 WebSocket 范围内只存在一个 Bean 实例。仅在支持 WebSocket 的应用程序中有效，该 Bean 实例在 WebSocket 会话范围内共享，适用于 WebSocket 会话范围内共享的 Bean。
- Custom scopes（自定义作用域）：Spring 允许开发者定义自定义的作用域，通过实现 Scope 接口来创建新的 Bean 作用域。

### Bean 的完整生命周期（回调链路）

上面讲了作用域，这里补完整的创建 → 销毁回调链路（单例 Bean）：

```text
实例化（构造器）
  → 属性填充（依赖注入，populateBean）
  → Aware 回调：BeanNameAware → BeanFactoryAware → ApplicationContextAware
  → BeanPostProcessor.postProcessBeforeInitialization
  → 初始化：@PostConstruct → InitializingBean.afterPropertiesSet() → init-method
  → BeanPostProcessor.postProcessAfterInitialization（★ AOP 代理在此创建）
  → 就绪，可以使用
  → 容器关闭：@PreDestroy → DisposableBean.destroy() → destroy-method
```

**面试关键点**：

- **AOP 代理在 postProcessAfterInitialization 阶段生成**：所以循环依赖 + AOP 时需要三级缓存提前暴露代理（这就是三级缓存存在的原因）。
- **BeanPostProcessor 是 Spring 扩展的基石**：@Autowired、@Resource 注入、AOP、事务都通过它实现。

### Spring 事务（高频必问）

**实现原理**：基于 AOP（动态代理）+ TransactionInterceptor，声明式事务 @Transactional 本质是给目标方法包了一层代理，在方法执行前开启事务、异常时回滚、正常时提交。

**7 种传播行为**（事务在方法调用链上的传递策略）：

| 传播行为 | 说明 |
|---|---|
| REQUIRED（默认） | 有事务就加入，没有就新建。**最常用** |
| SUPPORTS | 有事务就加入，没有就以非事务方式执行 |
| MANDATORY | 必须有事务，否则抛异常 |
| REQUIRES_NEW | 必须新建事务，挂起外层事务（内层回滚不影响外层） |
| NOT_SUPPORTED | 以非事务方式执行，挂起当前事务 |
| NEVER | 必须没有事务，有则抛异常 |
| NESTED | 嵌套事务（Savepoint 机制），内层回滚只回滚内层，外层可继续 |

```java
// 典型场景：A 调 B，B 标注 REQUIRES_NEW
// A 回滚 → B 已提交的数据不回滚；B 回滚 → 不影响 A（除非 A 也捕获了异常）
```

**5 种隔离级别**：

| 隔离级别 | 脏读 | 不可重复读 | 幻读 |
|---|---|---|---|
| READ_UNCOMMITTED | ✅可能 | ✅可能 | ✅可能 |
| READ_COMMITTED | ❌ | ✅可能 | ✅可能 |
| REPEATABLE_READ（MySQL 默认） | ❌ | ❌ | ⚠️InnoDB 靠 MVCC+间隙锁解决 |
| SERIALIZABLE | ❌ | ❌ | ❌（性能最差） |

**@Transactional 失效场景（必背）**：

1. **自调用**：同类内部 this 调用，绕过代理，事务不生效（最常考！解决：注入自身/拆到别的类/AopContext.currentProxy()）。
2. **方法非 public**：Spring 默认只对 public 方法代理。
3. **异常被 catch 吞掉**：方法内部 try-catch 住了异常，事务感知不到，不会回滚。
4. **抛出受检异常**：默认只对 RuntimeException（Error）回滚，受检异常需显式指定 `rollbackFor = Exception.class`。
5. **类没有被 Spring 管理**：没加 @Service/@Component 等注解。
6. **数据库引擎不支持事务**：如 MyISAM（应使用 InnoDB）。

### Spring MVC 请求处理流程

```text
① 请求到达 DispatcherServlet（前端控制器，所有请求的入口）
② HandlerMapping 根据 URL 找到对应的 Handler（Controller 方法）和拦截器链
③ HandlerAdapter 适配并调用 Controller 方法（参数解析、数据绑定、校验）
④ 返回 ModelAndView（或 @ResponseBody 直接序列化 JSON 返回）
⑤ 拦截器 postHandle 执行
⑥ ViewResolver 解析视图名 → 渲染视图（JSP/Thymeleaf 等）
⑦ 拦截器 afterCompletion 执行，响应返回客户端
```

**核心组件**：DispatcherServlet（前端控制器）、HandlerMapping（URL 映射）、HandlerAdapter（方法适配调用）、ViewResolver（视图解析）、HandlerInterceptor（拦截器）。

**拦截器与过滤器区别**：

| 维度 | Filter | Interceptor |
|---|---|---|
| 归属 | Servlet 规范 | Spring MVC |
| 时机 | 请求进入 Servlet 前 | DispatcherServlet 分发后 |
| 作用对象 | 所有 URL | 只对 Controller 方法 |
| 获取 Spring Bean | 不能 | 可以（本身是 Bean） |

### Spring Boot 自动配置原理

**@SpringBootApplication** 是三个注解的组合：

```text
@SpringBootConfiguration  ≈ @Configuration（标记配置类）
@EnableAutoConfiguration    核心：开启自动配置
@ComponentScan              扫描当前包及子包的组件
```

**@EnableAutoConfiguration 的原理**：

```text
1. @EnableAutoConfiguration 导入 AutoConfigurationImportSelector
2. 该 Selector 读取所有 jar 包中 META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
   （Spring Boot 2.7 前是 META-INF/spring.factories）
3. 得到所有自动配置类（如 RedisAutoConfiguration、DataSourceAutoConfiguration）
4. 逐类通过 @Conditional 条件注解判断是否生效（如类路径上存在 RedisTemplate 才生效）
5. 生效的配置类通过 @Bean 注册组件（默认装配）→ 用户自定义 Bean 优先（@ConditionalOnMissingBean）
```

**常用条件注解**：

| 注解 | 条件 |
|---|---|
| @ConditionalOnClass / OnMissingClass | 类路径是否存在某个类 |
| @ConditionalOnBean / OnMissingBean | 容器中是否存在某个 Bean |
| @ConditionalOnProperty | 配置文件中的属性（如 spring.redis.host 存在） |
| @ConditionalOnWebApplication | 是否是 Web 应用 |

**面试核心**：自动配置 = 约定优于配置。Spring Boot 通过"导入选择器 + 条件注解 + 配置绑定（@ConfigurationProperties）"实现"引入依赖即自动可用"，而用户通过 @Bean / 配置文件覆盖默认配置（@ConditionalOnMissingBean 保证用户自定义优先）。