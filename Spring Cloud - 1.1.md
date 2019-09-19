##Basics 

### Java 观察者模式

`java.util.Observable` 发布者

`java.util.Observer` 订阅者

```java
public class ObservableDemo {

    public static void main(String[] args) {
			        //发布者
        Observable observable = new Observable();
        /*
          增加订阅者
         */
        observable.addObserver(new Observer() {
            /**
             *
             * @param o 发布者
             * @param value 订阅的东西
             */
            @Override
            public void update(Observable o, Object value) {
                System.out.println(value);
            }
        });
   			     //唤醒订阅者 
        observable.notifyObservers("Hello World");
    }
}
```

执行代码会发现什么都没有。
究其原因是因为发布者需要通过java.util.Observable#setChanged方法告诉订阅者，它发生了变化。

但它那个方法不是public方法，所以在外边是取不到的，需要继续后重写该方法。

```java
public class ObservableDemo {

    public static void main(String[] args) {

        MyObservable observable = new MyObservable();
        /*
          增加订阅者
         */
        observable.addObserver(new Observer() {
            /**
             *
             * @param o 发布者
             * @param value 订阅的东西
             */
            @Override
            public void update(Observable o, Object value) {
                System.out.println(value);
            }
        });
   						     //通知订阅者 
        observable.setChanged();
   	     //唤醒订阅者
        observable.notifyObservers("Hello World");
    }

    private static class MyObservable extends Observable {

        @Override
        public synchronized void setChanged() {
            super.setChanged();
        }
    }

}
```

> 这种方式也称为推的方式，因为订阅者是被动触发的。
>
> 拉的方式常见的有迭代器
>
> ```java
> private static void echoIterator(){
>     List<Integer> values = Arrays.asList(1,2,3,4,5);
>     Iterator<Integer> iterator = values.iterator();
>     if (iterator.hasNext()){
>    	     //是主动去获取数据
>         Integer next = iterator.next();
>     }
> }
> ```

### Java事件/监听模式

三个组成部分：

​	当事件源触发事件监听器时，会自动执行监听器中相应方法。

- 事件源 ：负责触发
- 事件对象 ：包含数据源的引用
- 事件监听器 ： 提供逻辑处理

`java.util.EventObject` 事件对象（每一个事件对象都绑定着事件源 transient Object  source）

`java.util.EventListener` 事件监听者（标记接口）

```java
package com.guoxiaohui.example.demo;

import java.util.*;

public class EventDemo {
		    //事件源
    public static class EventSource {

        private String name;
   	
        public String getName() {
            return name;
        }

        public void setName(String name) {
            this.name = name;
        }
   
	        //事件源存有事件监听器引用
        private List<EventListener> eventListeners = new ArrayList<>();
	        //通过构造器注册事件监听器
   	     public EventSource(EventListener eventListener) {
            eventListeners.add(eventListener);
        }
        
        //关键操作
        public void setValues(String name) {
            this.name = name;
       	     //如果没有事件监听器，执行正常操作
            if (eventListeners.size() != 0) {
                //如果存在，将当前事件源包装在事件对象中
                MyEventObject myEventObject = new MyEventObject(this);
                //实际环境中这部分内容会根据相应需求进行处理
                MyEventListener myEventListener = (MyEventListener) eventListeners.get(0);
               //执行事件监听器中的方法
                myEventListener.onMyEvent(myEventObject);
            }
            System.out.println("123456789");
        }

    }
    //事件监听器
    public static class MyEventListener implements EventListener {
        // do something
        public void onMyEvent(MyEventObject eventObject) {
            System.out.println(eventObject.getSource());
        }
    }

    //事件对象
    public static class MyEventObject extends EventObject {
        private EventSource source;
        public MyEventObject(EventSource source) {
            super(source.getName());
        }
    }

    public static void main(String[] args) {
        MyEventListener myEventListener = new MyEventListener();
        //注册监听器
        EventSource eventSource = new EventSource(myEventListener);
        //触发操作
        eventSource.setValues();
    }
}
```

### Spring事件/监听模式

`org.springframework.context.ApplicationEvent`  应用事件

`org.springframework.context.ApplicationListener` 应用事件监听器

##### 自定义Spring事件

```java
public class CustomerSpringEventListenerDemo {

    public static void main(String[] args) {
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
   	     //注册监听器
        context.addApplicationListener(new CustomerEventListener());
        //这个方法名字起的，其实很不好，很难让人理解。想不明白为什么要刷新尼。
        context.refresh();
   	     //发布事件
        context.publishEvent(new CustomerEventObject("guoxiaohui"));
        //再发布一个
        context.publishEvent(new CustomerEventObject(1));
    }

    public static class CustomerEventObject extends ApplicationEvent{
        public CustomerEventObject(Object source) {
            super(source);
        }
    }

    public static class CustomerEventListener implements ApplicationListener<CustomerEventObject>{
        //响应事件
        @Override
        public void onApplicationEvent(CustomerEventObject event) {
            System.out.println(event.getSource());
        }
    }
}
```

> 题外话：Spring组织起初有一个负责人秉持的理念是不要重复造轮子，但是当Spring越来越壮大，贡献者越来越多的时候初衷已经发生了改变。并且Spring当初有一个口号是轻量级，现在看来，还可以说是轻量级吗。世间唯一的不变就是变化。
>
> 事件有什么用？   参考它的实现类就可以明白，它可以做很多事情，在你需要的时候。

### Spring Boot核心事件

[官方文档说了几大事件对象](https://docs.spring.io/spring-boot/docs/2.1.8.RELEASE/reference/html/boot-features-spring-application.html#boot-features-application-events-and-listeners)

Spring Boot 事件/监听器

#### 几个比较出名的listener

##### ConfigFileApplicationListener

- `org.springframework.boot.context.config.ConfigFileApplicationListener` 管理配置文件

	`application-{profile}.properties` / `application.yaml`

	profile = dev /  test 

	[官方文档描述的配置文件顺序](https://docs.spring.io/spring-boot/docs/2.1.8.RELEASE/reference/html/boot-features-external-config.html#boot-features-external-config)

###### 支持properties和yaml的原因

- 当这个事件来的时候`ConfigFileApplicationListener` 走`#onApplicationEnvironmentPreparedEvent` 

- `postProcessEnvironment` 它又会调这个方法

- `addPropertySources` 又进入这个方法

- 最终会到这个方法`ConfigFileApplicationListener.Loader#loadDocuments`

- 最终可以找到真正解析配置东西。`PropertySourceLoader#load`

- `PropertySourceLoader`默认提供了两个实现

	- `org.springframework.boot.env.PropertiesPropertySourceLoader`

	- `org.springframework.boot.env.YamlPropertySourceLoader`

		> 那`PropertySourceLoader`什么时候放进去的？和今天说的内容有些冲突，并且目前一两句话没办法概括，后续补上

###### `ConfigFileApplicationListener` 加载时机

no code address :  org/springframework/boot/spring-boot/2.1.8.RELEASE/spring-boot-2.1.8.RELEASE.jar!/META-INF/spring.factories

Spring SPI

```properties
# Application Listeners
org.springframework.context.ApplicationListener=\
org.springframework.boot.ClearCachesApplicationListener,\
org.springframework.boot.builder.ParentContextCloserApplicationListener,\
org.springframework.boot.context.FileEncodingApplicationListener,\
org.springframework.boot.context.config.AnsiOutputApplicationListener,\
org.springframework.boot.context.config.ConfigFileApplicationListener,\
org.springframework.boot.context.config.DelegatingApplicationListener,\
org.springframework.boot.context.logging.ClasspathLoggingApplicationListener,\
org.springframework.boot.context.logging.LoggingApplicationListener,\
org.springframework.boot.liquibase.LiquibaseServiceLocatorApplicationListener

spring会将每一个实现类添加进去，当事件来的时候一个个的调用。
Spring控制顺序的方法：
		       1  实现order接口
		       2  标记@order
		spring中数值越小越优先
```

> 扩展知识：
>
> Java SPI : ` java.util.ServiceLoader`



今晚先搞工作去了，未完待续.....。