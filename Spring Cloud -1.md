## Spring Cloud 服务注册与发现

### 注册中心

多机中心化注册中心

去中心化注册中心	 example：区块链

### 高可用

上线时间作为基准。以一年内的上线时间与自然时间的比率来描述可用性

基于原则：

1. 消灭单点故障
2. 可靠性交迭
3. 故障探测

## Spring Cloud Netflix Eureka

### 客户端：Eureka Client

注册模式：异步

###### Eureka客户端配置api

	`EurekaClientConfigBean`

###### Eureka实例配置api

	`EurekaInstanceConfigBean`

[EUREKA REST API](https://github.com/Netflix/eureka/wiki/Eureka-REST-operations)

> Greenwich.SR3存在的bug：
>
> ​当启动的多个client端口采用0的时候，后启动的会覆盖掉之前启动的项目，无论有多少个，只有一个提供服务。

> zookeeperAutoServiceRegistration
>
> eurekaAutoServiceRegistration
>
> 两个只有一个可以注册，但发现可以发现多个。解决这问题需要关闭自动注册

| 注册中心  | CAP特性 | 推荐规模  |
| --------- | ------- | --------- |
| Eureka    | Ap      | <30K      |
| Zookeeper | Cp      | <30K      |
| Consul    | Ap/Cp   | <5K       |
| Nacos     | Ap/Cp   | 100K-200K |

## 服务调用与熔断

### 核心概念

...

### Spring Cloud Feign

[Feign](https://github.com/openfeign/feign)

JAX-RS-JERSEY

| REST框架                | 使用场景            | 请求映射注解                                      |               |
| ----------------------- | ------------------- | ------------------------------------------------- | ------------- |
| Feign                   | 客户端声明          | @RequestLine("POST /repos/{owner}/{repo}/issues") | @Param        |
| Spring Cloud Open Feign | 客户端声明          | @RequestMapping                                   | @RequestParam |
| JAX-RS                  | 客户端/服务端双声明 | @Path                                             | @*Param       |
| Spring Web MVC          | 服务端声明          | @RequestMapping                                   | @RequestParam |

Spring Cloud Open Feign利用feign高扩展性，使用标准Spring Web MVC来声明客户端接口。Spring Cloud Open Feign通过Java接口的方式来声明REST服务提供者的元信息，通过调用Java接口的方式来实现http/rest通讯。

> 实现细节猜想：
>
> 1 Java接口与rest提供者如何映射？
>
> 2 @FeignClient注解所指定的应用（服务）名称可能使用到了服务发现
>
> 3 @EnableFeignClients注解是如何感知或加载标注了@FeignClient的配置类
>
> 4 Feign请求与相应的内容是如何序列化和反序列化到对应的pojo

#### Feign

1. 注解扩展性-Feign
2. HTTP请求处理-Feign
3. REST请求元信息解析-feign

#### OpenFeign

1. 提供spring web mvc 注解处理
2. 提供feign自动装配

基本步骤：

1  add org.springframework.cloud:spring-cloud-starter-openfeign

2 激活客户端

```java
@SpringBootApplication
@EnableFeignClients
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }

}
```

3 定义feign接口

```java
@FeignClient("stores") //应用(服务)名称
public interface StoreClient {
    @RequestMapping(method = RequestMethod.GET, value = "/stores")
    List<Store> getStores();

    @RequestMapping(method = RequestMethod.POST, value = "/stores/{storeId}", consumes = "application/json")
    Store update(@PathVariable("storeId") Long storeId, Store store);
}
```

> 常见问题
>
> @EnableAutoConfiguration 和 @SpringBootApplication还是有一些区别的，@EnableAutoConfiguration没有@ComponentScan,它是不会去自动扫包的。
>
> 接口中的参数需要采用Spring web mvc注解标注。

坑一：Spring Cloud + Netflix Ribbon 有一个30秒延迟的操作

​	`com.netflix.loadbalancer.DynamicServerListLoadBalancer`

​	`com.netflix.loadbalancer.DynamicServerListLoadBalancer#updateListOfServers`

​		Eureka实现`com.netflix.niws.loadbalancer.EurekaNotificationServerListUpdater`

​		非Eureka实现`com.netflix.loadbalancer.PollingServerListUpdater`

坑二：server.port不靠谱解决方案：

方案一：

​	 @LocalServerPort	也不一定靠谱，因为在注入阶段不一定存在

方案二：

```java
private final Environment environment;

    public UserController(Environment environment) {
        this.environment = environment;
    }

    private String getServerPort(){
        return environment.getProperty("local.server.port");
    }
```

#### Spring Cloud 服务调用

`DiscoveryClient` 服务发现

`Ribbon` 负载均衡

`Feign ` 

> @Enable模块驱动.  三种实现方式
>
> 1 ImportBeanDefinitionRegistrar
>
> ​	example：`@EnableFeignClients`
>
> 2 ImportSelector
>
> ​	example：`@EnableAsync`
>
> 3 @Configuration
>
> ​	example：`@EnableWebMvc`

####源码解析

`@EnableFeignClients`

```java
@Import({FeignClientsRegistrar.class})
public @interface EnableFeignClients {
    String[] value() default {};

    String[] basePackages() default {};

    Class<?>[] basePackageClasses() default {};

    Class<?>[] defaultConfiguration() default {};

    Class<?>[] clients() default {};
}
```

`FeignClientsRegistrar`

```java
@Override
	public void registerBeanDefinitions(AnnotationMetadata metadata,BeanDefinitionRegistry registry) {
		//注册默认配置
		registerDefaultConfiguration(metadata, registry);
		//注册所有标注@FeignClient配置类
	        registerFeignClients(metadata, registry);
	}
```

##### 注册默认配置

```java
private void registerDefaultConfiguration(AnnotationMetadata metadata,BeanDefinitionRegistry registry) {
		      Map<String, Object> defaultAttrs = metadata.getAnnotationAttributes(
                                         EnableFeignClients.class.getName(), true);
			      if (defaultAttrs != null && defaultAttrs.containsKey("defaultConfiguration")) {
           			String name;
			           if (metadata.hasEnclosingClass()) {
				               name = "default." + metadata.getEnclosingClassName();
			           }else {
					                name = "default." + metadata.getClassName();
			           }
           			registerClientConfiguration(	registry,name,
                                       defaultAttrs.get("defaultConfiguration"));
		      }
	}
```

##### 注册标注了@FeignClient配置类

```java
//importingClassMetadata.getClassName() == 标注了@EnableFeignClients类的包名
//org.springframework.cloud.openfeign.FeignClientsRegistrar#getBasePackages
if (basePackages.isEmpty()) {
	basePackages.add(ClassUtils.getPackageName(importingClassMetadata.getClassName()));
}
//过滤掉除org.springframework.cloud.openfeign.FeignClientl类型的BeanDefinition
Set<BeanDefinition> candidateComponents = scanner.findCandidateComponents(basePackage);
//读取@FeignClient注解的元数据（既@FeignClient注解的属性值）
//example： @FeignClient("gxh-service-provider")
            //"value" -> "gxh-service-provider"
            //"name" -> "gxh-service-provider"
            //"path" -> ""
            //"contextId" -> ""
            //"fallback" -> {Class@4367} "void"
            //"qualifier" -> ""
            //"decode404" -> {Boolean@4371} false
            //"fallbackFactory" -> {Class@4367} "void"
            //"serviceId" -> ""
            //"configuration" -> {Class[0]@4376} 
            //"url" -> ""
            //"primary" -> {Boolean@4380} true
Map<String, Object> attributes = annotationMetadata.getAnnotationAttributes(FeignClient.class.getCanonicalName());
//注册
registerFeignClient(registry, annotationMetadata, attributes);

流程整理：
	      ·通过ClassPathScanningCandidateComponentProvider扫描指定basePackages中的@FeignClient
	      ·通过AnnotationMetadata获取@FeignClient属性元数据
	      ·再重新注册一个 FeignClientFactoryBean 的 Bean
	      ·生成一个被标注接口的代理对象
	      		    ·在此过程中有个方法FeignClientFactoryBean#getTarget构建了一个Feign.Builder 
	      		    基本原理：它负责
	      		    		   1 请求拦截器
	      		    		   2 client（类似http的客户端）
	      		    		   3 重试策略
	      		    		   4 解码/编码
	      		    		   5 序列化/反序列化  Decoder(Spring是采用HttpMessageConverters进行类型转换处理)
	      		    		   6 错误采取措施
	      		    		   7 参数（options）
	      		    		   8 InvocationHandlerFactory(创建代理的工厂)
	                                    （默认实现ReflectiveFeign(可反射的)，FeignInvocationHandler(代理)）
	                                   9 最终会走到feign.SynchronousMethodHandler#invoke
	                                   //通信来了，接下来就是构建请求，构建响应喽
	             	                   RequestTemplate template = buildTemplateFromArgs.create(argv)
				           //Spring Cloud 是通过桥接到SpringMvc上进行的Decoder操作
调用时机：
      ·注入的时候
       调用FeignClientFactoryBean#getObject
       既通过FactoryBeanRegistrySupport#doGetObjectFromFactoryBean取
```

> 扩展知识：
>
> ​	所有被ComponentScanProvider扫描出来的类，它的BeanDefinition都是AnnotatedBeanDefinition类型。
>
> 两种获取元信息的方式
>
> 	·AnnotationMetadataReadingVisitor  CGLB方式
>
>	·StandardAnnotationMetadata Java标准反射方式
>
> Spring Boot Actuator 默认会开启jmx和web endpoint
>
> Bean注入的实现（DI的过程）
>
> ​	第一种：普通Bean（编码/XML/注解/BeanDefinition生成的 和直接注册的）（细分四种）
>
> ​	第二种：FactoryBean生成的
>
> ​	FactoryBean当作Bean注入原理：
>
> ​		 基础知识：BeanFactory依赖查找
>
> ​			1通过名称查找
>
> ​				·getBean(String)
>
> ​		    2通过类型查找
>
> ​				·getBean(String,Class)
>
> ​				·getBean(Class)
>
> ​		    3通过注解查找（低层次BeanFactory是没有的）
>
> ​				·getBeanWithAnnotation(Annotation)
>
> ​	FactoryBean两种功效（或者说是语义）：
>
> ​	       1返回Bean的类型。getObjectType()
>
> ​            2返回Bean的对象。getObject()	
