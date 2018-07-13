# Spring中的设计模式

## template method
>定义：定义一个操作中的算法的骨架，而将一些步骤延迟到子类中。模板方法使得子类可以不改变一个算法的结构即可重定义该算法的某些特定步骤。

AbstracApplicationContext 在refresh操作中定义了算法的骨架，并将诸如如`onRefresh`，`postProcessBeanFactory`等步骤交由子类去实现。代码如下：
```java
@Override
    public void refresh() throws BeansException, IllegalStateException {
        synchronized (this.startupShutdownMonitor) {
            // Prepare this context for refreshing.
            prepareRefresh();

            // Tell the subclass to refresh the internal bean factory.
            ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

            // Prepare the bean factory for use in this context.
            prepareBeanFactory(beanFactory);

            try {
                // Allows post-processing of the bean factory in context subclasses.
                postProcessBeanFactory(beanFactory);

                // Invoke factory processors registered as beans in the context.
                invokeBeanFactoryPostProcessors(beanFactory);

                // Register bean processors that intercept bean creation.
                registerBeanPostProcessors(beanFactory);

                // Initialize message source for this context.
                initMessageSource();

                // Initialize event multicaster for this context.
                initApplicationEventMulticaster();

                // Initialize other special beans in specific context subclasses.
                onRefresh();

                // Check for listener beans and register them.
                registerListeners();

                // Instantiate all remaining (non-lazy-init) singletons.
                finishBeanFactoryInitialization(beanFactory);

                // Last step: publish corresponding event.
                finishRefresh();
            }

            catch (BeansException ex) {
                if (logger.isWarnEnabled()) {
                    logger.warn("Exception encountered during context initialization - " +
                            "cancelling refresh attempt: " + ex);
                }

                // Destroy already created singletons to avoid dangling resources.
                destroyBeans();

                // Reset 'active' flag.
                cancelRefresh(ex);

                // Propagate exception to caller.
                throw ex;
            }

            finally {
                // Reset common introspection caches in Spring's core, since we
                // might not ever need metadata for singleton beans anymore...
                resetCommonCaches();
            }
        }
    }
```
其子类，比如SpringWeb中常用的`ServletWebServerApplicationContext`覆写了`onRefresh`，实现了启动嵌入式webserver的功能。代码如下：
```java
protected void onRefresh() {
        super.onRefresh();

        try {
            this.createWebServer();//此处启动webserver
        } catch (Throwable var2) {
            throw new ApplicationContextException("Unable to start web server", var2);
        }
}
```

其他应用
+ JdbcTemplate
+ KafkaTemplate
+ RabbitTempalte
+ MongoTemplate


## Proxy
>为其他对象提供一种代理以控制对这个对象的访问

我们都知道ApplicationContext接口继承了BeanFactory接口，理所当然，其实现类`AbstractApplicationContext`也就间接实现了`BeanFactory`接口。其实现方式如下：
```java
    @Override
    public <T> T getBean(Class<T> requiredType) throws BeansException {
        assertBeanFactoryActive();
        return getBeanFactory().getBean(requiredType);
    }
```
说到底，`AbstractApplicationContext`还是将`getBean`的操作委托给了`BeanFactory`。
此处`AbstractApplicationContext`代理了`BeanFactory`。

其他应用：
+ Aop。
+ 声明式事务。


## Strategy
>定义一系列的算法,把它们一个个封装起来, 并且使它们可相互替换。

我们看`SimpleApplicationEventMulticaster`，代码如下：
```java
public class SimpleApplicationEventMulticaster extends AbstractApplicationEventMulticaster {

    @Nullable
    private Executor taskExecutor;

    @Nullable
    private ErrorHandler errorHandler;

    public void multicastEvent(final ApplicationEvent event, @Nullable ResolvableType eventType) {
        ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
        for (final ApplicationListener<?> listener : getApplicationListeners(event, type)) {
            Executor executor = getTaskExecutor(); //此处使用executor
            if (executor != null) {
                executor.execute(() -> invokeListener(listener, event));
            }
            else {
                invokeListener(listener, event);
            }
        }
    }

    protected void invokeListener(ApplicationListener<?> listener, ApplicationEvent event) {
        ErrorHandler errorHandler = getErrorHandler();
        if (errorHandler != null) {
            try {
                doInvokeListener(listener, event);
            }
            catch (Throwable err) {
                errorHandler.handleError(err); //此处进行失败处理
            }
        }
        else {
            doInvokeListener(listener, event);
        }
    }
```

如上代码封装了任务执行策略和错误处理策略
+ 任务执行策略我们可以选择同步执行，也可以选择异步执行。
+ 错误处理策略有`LoggingErrorHandler`也可以自己实现。

## Chain of Responsibility
>让多个对象都有可能接收请求，将这些对象连接成一条链，并且沿着这条链传递请求，直到有对象处理它为止。

我们看`BeanPostProcessor`,spring可以接受多个`BeanPostProcessor`的注册，并将其组成一条链。当初始化bean的时候便可将bean沿着这条链传递，使每个`BeanPostProcessor`都可以对bean进行初始化处理。

其他应用：
- servlet filter
- spring mvc interceptor
- mybatis plugin

## Adaptor
>将一个类的接口转换成客户希望的另外一个接口。适配器模式使得原本由于接口不兼容而不能一起工作的那些类可以一起工作。

spring提供了事件机制，主要接口为`ApplicationListener`。使用时，我们可以实现此接口,也可以使用声明式的接口实现。主要看声明式实现，源码如下：
```java

public class OrderPlacedListener {
    
    @EventListener
    public void onOrderPlaced(PlaceOrderEvent event) {
        sendRabbitMessage(event);
    }
}
```
对于`@EventListener`,spring会通过反射为其生成一个`ApplicationListenerMethodAdapter`,简化后代码如下：
```java
public class ApplicationListenerMethodAdapter implements ApplicationListener {
    private Object bean; //orderPlacedListener
    private Method method;//onOrderPlaced
    
    @Override
    public void onApplicationEvent(ApplicationEvent event) {
        Object[] args = extract(event);
        method.invoke(bean, args);
    }
}
```
原本OrderPlacedListener没有实现ApplicationListener，是不能监听PlaceOrderEvent事件的。但是我们在ApplicationListenerMethodAdapter实现了`ApplicationListener`, 并将`onApplicationEvent`操作通过反射的方式委托给了`OrderPlacedListener.onOrderPlaced`，这样就实现了接口上的兼容。

其他应用：
- @RabbitListener
- @KafkaListener
- @RequestMapping

## Observer 
> 定义对象间的一种一对多的依赖关系，当一个对象的状态发生改变时，所有依赖于它的对象都得到通知并被自动更新。

说的通俗一点，其实就是事件监听机制。对此spring提供了`ApplicationEventPublisher`和`ApplicationListener`接口。任何`ApplicationEventPublisher.publishEvent`的调用，都可以通知对应的多个`ApplicationListener`进行处理。

其他：
- 消息队列
- js中的onClick, onClose等回调函数
