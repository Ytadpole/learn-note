

## SPI梳理

### 0.测试代码

![image-20190821104414654](images/spi梳理/image-20190821104414654.png)

### 1.getExtensionLoader(Protocol.class) 

```java
public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> type) {
    if (type == null) {
        throw new IllegalArgumentException("Extension type == null");
    }
  //type必须为接口
    if (!type.isInterface()) {
        throw new IllegalArgumentException("Extension type (" + type + ") is not an interface!");
    }
  //type必须有@SPI注解
    if (!withExtensionAnnotation(type)) {
        throw new IllegalArgumentException("Extension type (" + type +
                ") is not an extension, because it is NOT annotated with @" + SPI.class.getSimpleName() + "!");
    }

  //主要代码
    ExtensionLoader<T> loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
    if (loader == null) {
      //缓存未命中，实例化一个ExtensionLoader对象
        EXTENSION_LOADERS.putIfAbsent(type, new ExtensionLoader<T>(type));
        loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
    }
    return loader;
}
```


ExtensionLoader类中的 **EXTENSION_LOADERS** 作为一个缓存。存储class为key，ExtendLoader对象为值

```Java
private ExtensionLoader(Class<?> type) {
    this.type = type;
    objectFactory = (type == ExtensionFactory.class ? null : ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension());
}
```

ExtensionLoader构造方法包含两个参数

1. type //对应的对象接口class
2. objectFactory //从上面代码可看出，为AdaptiveExtensionFactory

### 2.getAdaptiveExtension

主要缓存

1. 实例缓存
2. AdaptiveClass缓存
3. WrapperClass缓存
4. 一般Class缓存



在获取ExtensionLoader<Prococol>实例前，他自身的objectFacory对象是通过下面代码获取。

```java
ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension());
```



ExtensionLoader.getExtensionLoader(ExtensionFactory.class)这部分代码同上，再看下getAdaptiveExtension（）逻辑

```java
public T getAdaptiveExtension() {
  //读取缓存
    Object instance = cachedAdaptiveInstance.get();
    if (instance == null) {
      //缓存未命中
        if (createAdaptiveInstanceError == null) {
            synchronized (cachedAdaptiveInstance) {
                instance = cachedAdaptiveInstance.get();
                if (instance == null) {
                    try {
                      //缓存未命中的情况创建 实例
                        instance = createAdaptiveExtension();
                      //存入缓存对象
                        cachedAdaptiveInstance.set(instance);
                    } catch (Throwable t) {
                        createAdaptiveInstanceError = t;
                        throw new IllegalStateException("Failed to create adaptive instance: " + t.toString(), t);
                    }
                }
            }
        } else {
            throw new IllegalStateException("Failed to create adaptive instance: " + createAdaptiveInstanceError.toString(), createAdaptiveInstanceError);
        }
    }

    return (T) instance;
}
```



instance = createAdaptiveExtension();创建实例对象

```java
private T createAdaptiveExtension() {
    try {
      //先加载adaptiveExtension的class，再实例化，最后交给injectExtension
        return injectExtension((T) getAdaptiveExtensionClass().newInstance());
    } catch (Exception e) {
        throw new IllegalStateException("Can't create adaptive extension " + type + ", cause: " + e.getMessage(), e);
    }
}
```

getAdaptiveExtensionClass() 加载class

```java
private Class<?> getAdaptiveExtensionClass() {
    getExtensionClasses();
    if (cachedAdaptiveClass != null) {
        return cachedAdaptiveClass;
    }
    return cachedAdaptiveClass = createAdaptiveExtensionClass();
}
```

getExtensionClasses()
```java
private Map<String, Class<?>> getExtensionClasses() {
    Map<String, Class<?>> classes = cachedClasses.get();
    if (classes == null) {
        synchronized (cachedClasses) {
            classes = cachedClasses.get();
            if (classes == null) {
                classes = loadExtensionClasses();
                cachedClasses.set(classes);
            }
        }
    }
    return classes;
}
```

loadExtensionClasses（）最终逻辑，加载对应路径下的文件

```java
private Map<String, Class<?>> loadExtensionClasses() {
    cacheDefaultExtensionName();//获取@SPI注解给定的名字

  //加载路径下的文件，存储到class缓存map
    Map<String, Class<?>> extensionClasses = new HashMap<>();
    loadDirectory(extensionClasses, DUBBO_INTERNAL_DIRECTORY, type.getName());
    loadDirectory(extensionClasses, DUBBO_INTERNAL_DIRECTORY, type.getName().replace("org.apache", "com.alibaba"));
    loadDirectory(extensionClasses, DUBBO_DIRECTORY, type.getName());
    loadDirectory(extensionClasses, DUBBO_DIRECTORY, type.getName().replace("org.apache", "com.alibaba"));
    loadDirectory(extensionClasses, SERVICES_DIRECTORY, type.getName());
    loadDirectory(extensionClasses, SERVICES_DIRECTORY, type.getName().replace("org.apache", "com.alibaba"));
    return extensionClasses;
}
```



读文件核心代码

```java
private void loadResource(Map<String, Class<?>> extensionClasses, ClassLoader classLoader, java.net.URL resourceURL) {
    try {
        try (BufferedReader reader = new BufferedReader(new InputStreamReader(resourceURL.openStream(), StandardCharsets.UTF_8))) {
            String line;
            while ((line = reader.readLine()) != null) {
                final int ci = line.indexOf('#');//#后面为注释内容，去掉注释
                if (ci >= 0) {
                    line = line.substring(0, ci);
                }
                line = line.trim();
                if (line.length() > 0) {
                    try {
                        String name = null;
                        int i = line.indexOf('=');
                      //解析key和value
                        if (i > 0) {
                            name = line.substring(0, i).trim();
                            line = line.substring(i + 1).trim();
                        }
                        if (line.length() > 0) {
                          //通过全限定名加载class Class.forName(line, true, classLoader)加载class
                          //loadClass在进行处理
                            loadClass(extensionClasses, resourceURL, Class.forName(line, true, classLoader), name);
                        }
                    } catch (Throwable t) {
                        IllegalStateException e = new IllegalStateException("Failed to load extension class (interface: " + type + ", class line: " + line + ") in " + resourceURL + ", cause: " + t.getMessage(), t);
                        exceptions.put(line, e);
                    }
                }
            }
        }
    } catch (Throwable t) {
        logger.error("Exception occurred when loading extension class (interface: " +
                type + ", class file: " + resourceURL + ") in " + resourceURL, t);
    }
}
```



```java
private void loadClass(Map<String, Class<?>> extensionClasses, java.net.URL resourceURL, Class<?> clazz, String name) throws NoSuchMethodException {
    if (!type.isAssignableFrom(clazz)) {
        throw new IllegalStateException("Error occurred when loading extension class (interface: " +
                type + ", class line: " + clazz.getName() + "), class "
                + clazz.getName() + " is not subtype of interface.");
    }

    if (clazz.isAnnotationPresent(Adaptive.class)) {
      //加载这条数据进入这个判断adaptive=org.apache.dubbo.common.extension.factory.AdaptiveExtensionFactory
      //放入cachedAdaptiveClass缓存对象，并检查是否存在多个adaptive设置
        cacheAdaptiveClass(clazz);
    } else if (isWrapperClass(clazz)) {
        cacheWrapperClass(clazz);
    } else {
      //加载spi=org.apache.dubbo.common.extension.factory.SpiExtensionFactory
      //这条数据进入这个判断。 
      //非Adaptive和wrapper
        clazz.getConstructor();
        if (StringUtils.isEmpty(name)) {
            name = findAnnotationName(clazz);
            if (name.length() == 0) {
                throw new IllegalStateException("No such extension name for the class " + clazz.getName() + " in the config " + resourceURL);
            }
        }

        String[] names = NAME_SEPARATOR.split(name);
        if (ArrayUtils.isNotEmpty(names)) {
          //通过activate注解判断 是否 存储activate缓存
            cacheActivateClass(clazz, names[0]);
            for (String n : names) {
              //缓存class的名称
                cacheName(clazz, n);
              //缓存到extensionClasses缓存
                saveInExtensionClass(extensionClasses, clazz, n);
            }
        }
    }
}
```



加载完extensionFactory的class后，回到下面代码，实例化AdaptiveExtensionFactory

```java
private T createAdaptiveExtension() {
    try {
      //先加载adaptiveExtension的class，再实例化，最后交给injectExtension
        return injectExtension((T) getAdaptiveExtensionClass().newInstance());
    } catch (Exception e) {
        throw new IllegalStateException("Can't create adaptive extension " + type + ", cause: " + e.getMessage(), e);
    }
}
```



遍历内部方法，对set方法处理

```java
private T injectExtension(T instance) {
    try {
        if (objectFactory != null) {
            for (Method method : instance.getClass().getMethods()) {
                if (isSetter(method)) {
                    /**
                     * Check {@link DisableInject} to see if we need auto injection for this property
                     */
                    if (method.getAnnotation(DisableInject.class) != null) {
                        continue;
                    }
                    Class<?> pt = method.getParameterTypes()[0];
                    if (ReflectUtils.isPrimitives(pt)) {
                        continue;
                    }
                    try {
                        String property = getSetterProperty(method);
                        Object object = objectFactory.getExtension(pt, property);
                        if (object != null) {
                            method.invoke(instance, object);
                        }
                    } catch (Exception e) {
                        logger.error("Failed to inject via method " + method.getName()
                                + " of interface " + type.getName() + ": " + e.getMessage(), e);
                    }
                }
            }
        }
    } catch (Exception e) {
        logger.error(e.getMessage(), e);
    }
    return instance;
}
```

最后返回实例，并存放到缓存中，完成了AdaptiveExtensionFactory实例的生成。对于protocol同样的逻辑，进行加载文件，生成class，由class产生实例对象。



对于protocol接口，由于没有实现类 有 **@Adaptive**注解，所以他的adaptive类是由程序动态生成的。

```java
private Class<?> getAdaptiveExtensionClass() {
    getExtensionClasses();
    if (cachedAdaptiveClass != null) {//这里由于找不到缓存对象，会执行if后面的代码逻辑
        return cachedAdaptiveClass;
    }
  //protocol没有@Adaptive注解的实现类，会通过程序来生成@Adaptive实现
    return cachedAdaptiveClass = createAdaptiveExtensionClass();
}
```

```java
private Class<?> createAdaptiveExtensionClass() {
  //生成代码
    String code = new AdaptiveClassCodeGenerator(type, cachedDefaultName).generate();
    ClassLoader classLoader = findClassLoader();
  //加载编译器
    org.apache.dubbo.common.compiler.Compiler compiler = ExtensionLoader.getExtensionLoader(org.apache.dubbo.common.compiler.Compiler.class).getAdaptiveExtension();
    return compiler.compile(code, classLoader);
}
```

主要是通过 代码生成java文件内容字符串，用编译器compiler进行编译成class再生成实例。





