## Java Config

- 内建
	- Java系统环境  java.lang.System#getProperties
	- os环境变量配置     java.lang.System#getenv(java.lang.String)
- 应用相关
  - properties/yml/.ini
  - csv
  - Hard Code
  - xml
  - json
  - ......
- JDK扩展内容
	- java.lang.Integer#getInteger(java.lang.String, java.lang.Integer)
	- java.lang.Long#getLong(java.lang.String, java.lang.Long)
	- 诸如此类...

```java
//捕获了异常，并提供了便捷的方式
public static Long getLong(String nm, Long val) {
        String v = null;
        try {
       			     //!
            v = System.getProperty(nm);
        } catch (IllegalArgumentException | NullPointerException e) {
        }
        if (v != null) {
            try {
                return Long.decode(v);
            } catch (NumberFormatException e) {
            }
        }
        return val;
    }
```

> Java只是提供了底层配置源的处理方式，并没有具体的抽象配置api。
>
> ConfigFileApplicationListener
>
> org.springframework.boot.context.config.ConfigFileApplicationListener.Loader#propertySourceLoaders 两个实现

## Apache Commons Config

[链接点]:<http://commons.apache.org/proper/commons-configuration/>

Apache将配置源抽象为org.apache.commons.configuration.Configuration接口，每种配置提供了相应的子类去加载解析

常见子类：

- org.apache.commons.configuration.SystemConfiguration
- org.apache.commons.configuration.PropertiesConfiguration
- org.apache.commons.configuration.XMLConfiguration
- org.apache.commons.configuration.DatabaseConfiguration
- org.apache.commons.configuration.CompositeConfiguration 

其中CompositeConfiguration 采用组合模式，并且通过其实例变量

private List<Configuration> configList = new LinkedList<Configuration>();处理顺序问题，以及配置覆盖。

> 组合模式的特性：
>
> ​	被组合的对象必须与容器是相同类型

前段时间Spring Cloud Netflix的新框架 Netflix Archaius，在commons configuration的基础上进行了扩展。

核心api : `com.netflix.config.DynamicConfiguration`  提供了动态更新和写入的操作。

## Spring Framework Environment

核心api ：`org.springframework.core.env.Environment` 

核心api ：`org.springframework.core.env.ConfigurableEnvironment` 写入

> 前缀模式

Spring 将配置源抽象为 `org.springframework.core.env.PropertySources`

配置源支持的方式：     

- 注解方式
	- @PropertySource
	- @PropertySources
- API方式
	- `PropertySource`   单配置源 org.springframework.core.env.PropertySource
	- `PropertySources` 多配置源 org.springframework.core.env.PropertySources

它关注点是配置和源之间的源数据

组合模式子类 `org.springframework.core.env.MutablePropertySources`

```java
private final List<PropertySource<?>> propertySourceList = new CopyOnWriteArrayList<>();
```

和commons项目极其相似，

最重要的不同点是！！！：

CompositeConfiguration只关心配置的顺序，不关系配置的名字。而PropertySource中必须有name。

PropertySource中存在name属性，Spring这样做的目的是因为在Spring中[配置源](https://docs.spring.io/spring-boot/docs/2.1.8.RELEASE/reference/html/boot-features-external-config.html#boot-features-external-config)的绝对路径其实是没有意义的，它的相对路径才有意义，它提供了如下一系列的根据名字去添加配置源的操作org.springframework.core.env.MutablePropertySources#addBefore。在扩展的时候要根据名字进行插入，官方提供的顺序是没有什么意义的。

其次不同的是提供了可变的，动态的类型转换,不像commons那样将所能处理的，转换的类型全部都列了出来。按需调用，那种设计不灵活，但基本够用了。

Spring类型转换过程：

- org.springframework.core.env.PropertySource#getProperty   从配置源中拿到的是原始数据类型
- org.springframework.core.env.PropertyResolver#getProperty(java.lang.String, java.lang.Class<T>)
- PropertySource，PropertyResolver。  配置源和配置解析器    怎么知道的呐？ 
	- 名字猜想
	- 同包
	- debug
- org.springframework.core.env.PropertySourcesPropertyResolver子类来做具体工作

```java
	protected <T> T getProperty(String key, Class<T> targetValueType, boolean resolveNestedPlaceholders) {
				          if (this.propertySources != null) {
				          	    for (PropertySource<?> propertySource : this.propertySources) {
				          	    	    ...
				          	    	    //找到
				          	        Object value = propertySource.getProperty(key);
				          	        ...
				          	        //解析后返回（如果支持类型转换）
				          	        return convertValueIfNecessary(value, targetValueType);
				          	    }
				          }

	protected <T> T convertValueIfNecessary(Object value, @Nullable Class<T> targetType) {
				         		ConversionService conversionServiceToUse = this.conversionService;
	         ...
         //根据类型转换，核心API · ConversionService
         		return conversionServiceToUse.convert(value, targetType);
}

```

[官方说明了一下如何自定义类型解析器](https://docs.spring.io/spring-boot/docs/2.1.8.RELEASE/reference/html/boot-features-external-config.html#boot-features-external-config-conversion)

三种自定义类型转换的方法：

- 提供ID为`conversionService`的`ConversionService` Bean
- 通过CustomerEditorConfig Bean 自定义 Editor
- 自定义Converter Bean 并且标注`@ConfigurationPropertiesBinding`

> @Value
>
> public Integer value;
>
> 转换api会去工作。
>
> org.springframework.core.convert.converter.Converter

总结：Environment所做的工作 

`org.springframework.core.env.AbstractEnvironment`

1   管理profile，它提供了读和写的方法

- `private final Set<String> activeProfiles = new LinkedHashSet<>();`
- `private final Set<String> defaultProfiles = new LinkedHashSet<>(getReservedDefaultProfiles());`

2   维护配置源

- `private final MutablePropertySources propertySources = new MutablePropertySources();`

3   将配置源中的数据转换为相应类型

- `private final ConfigurablePropertyResolver propertyResolver =
				new PropertySourcesPropertyResolver(this.propertySources);`

> 扩展知识
>
> ConversionService是怎么做的呐？
>
> 可以参考它的默认实现
>
> org.springframework.core.convert.support.DefaultConversionService
>
> ```java
> //添加了N多N多
> public static void addDefaultConverters(ConverterRegistry converterRegistry) {
>    addScalarConverters(converterRegistry);
>    addCollectionConverters(converterRegistry);
> 
>    converterRegistry.addConverter(new ByteBufferConverter((ConversionService) converterRegistry));
>    converterRegistry.addConverter(new StringToTimeZoneConverter());
>    converterRegistry.addConverter(new ZoneIdToTimeZoneConverter());
>    converterRegistry.addConverter(new ZonedDateTimeToCalendarConverter());
> 
>    converterRegistry.addConverter(new ObjectToObjectConverter());
>    converterRegistry.addConverter(new IdToEntityConverter((ConversionService) converterRegistry));
>    converterRegistry.addConverter(new FallbackObjectToStringConverter());
>    converterRegistry.addConverter(new ObjectToOptionalConverter((ConversionService) converterRegistry));
> }
> //源类型是否可以转换
> org.springframework.core.convert.support.GenericConversionService#canConvert(java.lang.Class<?>, java.lang.Class<?>)
> //转换过程
> org.springframework.core.convert.support.GenericConversionService#convert(java.lang.Object, org.springframework.core.convert.TypeDescriptor, org.springframework.core.convert.TypeDescriptor)
> ```

## Spring Cloud Config

Bootstrap配置来源：

​	核心：`org.springframework.cloud.bootstrap.BootstrapApplicationListener`

```java
org.springframework.cloud.bootstrap.BootstrapApplicationListener#onApplicationEvent

ConfigurableEnvironment environment = event.getEnvironment();
		if (!environment.getProperty("spring.cloud.bootstrap.enabled", Boolean.class,
				true)) {
			    return;
		}
		// don't listen to events in a bootstrap context
//可以看到它是通过名字去 组合模式的propertysource 中检查是否存在
		if (environment.getPropertySources().contains(BOOTSTRAP_PROPERTY_SOURCE_NAME)) {
    			return;
		}

```

