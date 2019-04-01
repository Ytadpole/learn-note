## 深入浅出SpringBoot2-AOP笔记

#### 开始

AOP即为约定编程。对某一个执行方法进行增强操作（某一个方法执行添加前置操作和后置操作）。

常见的AOP有拦截器，spring的事物处理。

比如@Transcation，将数据库的连接，执行sql，提交，回滚，关闭等操作，通过AOP进行约定编排。将一些重复的操作，如连接，提交，回滚，关闭。这些操作统一由AOP管理。不同的SQL执行由开发者这行定义，然后嵌入约定流程中。从而减少重复代码，开发者只需要关心自身的SQL执行部分。

#### JDK动态代理 Proxy

```java
public static Object newProxyinstance (ClassLoader classLoader, 
                                       Class<?>[] interfaces,
                                       InvocationHandler invocationHandler) 
  throws IllegalArguπ1entException
```

1. classLoader 类加载器

2. Interfaces 需要绑定到的接口

3. invocationHandler 代理类逻辑实现

##### 解析

   通过传入**需要代理对象**的*类加载器*，*接口类型*，和*代理类逻辑实现*，创建一个代理对象。

   如：

   ```java
   //需要代理的对象
   NeedProxy needProxy = new NeedProxy();
   //通过JDK代理方法 创建needProxy
   Proxy.newProxyInstance(needProxy.getClass().getClassLoader(),//需要代理对象的类加载器
                         needProxy.getClass().getInterfaces(),//需要代理对象的接口
                         InvocationHandler实现);
   ```

##### InvocationHandler

   ```java
   public Object invoke(Object proxy , Method method , Object[] args);
   ```

​	实现InvocationHander的唯一invoke方法。可以通过反射，发起对受代理对象指定方法的调用。同时，可以添加额外的逻辑处理进行额外操作。

​	如在《深入浅出SpringBoot2》中的列子，实现一个自定义的拦截器。通过在invoke中添加拦截器方法调用的逻辑，实现指定的执行约定。

​	通过Interceptor指定具体方法的实现，通过InvocationHandler指定Interceptor方法的执行约定。再通过JDK动态代理获取代理对象进行触发InvocatonHandler的invoke方法，执行具体约定流程。

#### AOP理论点

1. 连接点 join point:
2. 切点 pint cut:

   