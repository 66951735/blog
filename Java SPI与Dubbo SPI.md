## Java SPI与Dubbo SPI

#### 1. SPI是什么

SPI 全称为 Service Provider Interface，是一种服务发现机制。SPI 的本质是将接口实现类的全限定名配置在文件中，并由服务加载器读取配置文件，加载实现类。这样可以在运行时，动态为接口替换实现类。正因此特性，我们可以很容易的通过 SPI 机制为我们的程序提供拓展功能。

<img src="https://tva1.sinaimg.cn/large/007S8ZIlly1gfjnu3i0nrj314w0bgq58.jpg" alt="image-20200607132802474" style="zoom:50%;" />

#### 2. 一些使用场景

- JDBC加载不同类型的数据库驱动
- SLF4J加载不同的日志实现
- Dubbo SPI：基于Java原生SPI的扩展

#### 3. Java SPI

##### 3.1. 一个简单的例子

前面简单介绍了 SPI 机制的原理，本节通过一个示例演示 Java SPI 的使用方法。首先，我们定义一个接口，名称为 `SPITest`。

```
public interface A {
    void sayHello();
}
```

接下来实现两个实现类，分别为A和B。

```
public class B implements A {

    @Override
    public void sayHello() {
        System.out.println("Hello, I am B!");
    }
}

public class C implements A {

    @Override
    public void sayHello() {
        System.out.println("hello, I am C!");
    }
}
```

接下来 META-INF/services 文件夹下创建一个文件，名称为 A 的全限定名 com.test.A。文件内容为实现类的全限定的类名，如下：

```
com.test.B
com.test.C
```

接下来编写代码进行测试

```
public class JavaSPITest {

    @Test
    public void sayHello() throws Exception {
        ServiceLoader<A> serviceLoader = ServiceLoader.load(A.class);
        System.out.println("Java SPI");
        serviceLoader.forEach(A::sayHello);
    }
}
```

运行结果如下：

<img src="https://tva1.sinaimg.cn/large/007S8ZIlly1gfjooeffzkj30re07c0t3.jpg" style="zoom:50%;" />

##### 3.2 ServiceLoader源码分析

Java SPI实现的核心类为`ServiceLoader`，源代码一共不到600行，读起来比较容易。

我们先看下核心的成员变量：

```
// 要加载的文件路径前缀
private static final String PREFIX = "META-INF/services/";

// 要加载的服务的class对象类型
private final Class<S> service;

// 类加载器
private final ClassLoader loader;

// 上下文的访问控制
private final AccessControlContext acc;

// 缓存服务名和实例化后的对象的映射，避免重复加载
private LinkedHashMap<String,S> providers = new LinkedHashMap<>();

// 一个懒加载的迭代器，这里的懒加载是指只有调用next方法时才会实例化具体的实现类
private LazyIterator lookupIterator;
```

接下来我们从核心入口方法load开始，分析下`ServiceLoader`加载实现类的流程

```
// 这里load方法有两个重载方法
// 要加载的服务Class对象类型必传，另一个可选参数为ClassLoader，如果不指定的话，默认为线程上下文类加载器
// 注：这里的线程上下文类加载器是一个典型的破坏类加载双亲委派模型的场景
public static <S> ServiceLoader<S> load(Class<S> service) {
    ClassLoader cl = Thread.currentThread().getContextClassLoader();
    return ServiceLoader.load(service, cl);
}

public static <S> ServiceLoader<S> load(Class<S> service, ClassLoader loader){
    return new ServiceLoader<>(service, loader);
}
```

然后`ServiceLoader`构造方法的核心逻辑就是调用`reload`方法初始化`LazyIterator`对象。

下面具体看下`LazyIterator`的`hasNextService`和`nextService` 方法

```
// 延迟加载，这里不会实例化具体的实现类
private boolean hasNextService() {
    if (nextName != null) {
        return true;
    }
    if (configs == null) {
        try {
            String fullName = PREFIX + service.getName();
            // 如果loader为空，则在加载系统路径上的META-INF/servcies文件
            if (loader == null)
                configs = ClassLoader.getSystemResources(fullName);
            else
                // 如果指定的ClassLoader为线程上下文类加载，这里就可以加载具体应用路径上配置文件
                configs = loader.getResources(fullName);
        } catch (IOException x) {
            fail(service, "Error locating configuration files", x);
        }
     }
     
     while ((pending == null) || !pending.hasNext()) {
         if (!configs.hasMoreElements()) {
             return false;
         }
         // 读取文件，得到实现类的全称限定名，并缓存在pending中
         pending = parse(service, configs.nextElement());
     }
     nextName = pending.next();
     return true;
}
```

```
private S nextService() {
    if (!hasNextService())
        throw new NoSuchElementException();
    String cn = nextName;
    nextName = null;
    Class<?> c = null;
    try {
        // 加载具体的实现类
        c = Class.forName(cn, false, loader);
    } catch (ClassNotFoundException x) {
        fail(service, "Provider " + cn + " not found");
    }
    if (!service.isAssignableFrom(c)) {
        fail(service, "Provider " + cn  + " not a subtype");
    }
    try {
        // 实例化实现类并缓存
        S p = service.cast(c.newInstance());
        providers.put(cn, p);
        return p;
    } catch (Throwable x) {
        fail(service, "Provider " + cn + " could not be instantiated", x);
    }
    throw new Error();          // This cannot happen
}
```

#### 4. JDBC使用SPI分析

在JDBC中`DriverManager`是管理和注册不同数据库驱动的核心工具类。针对一个数据库，可能会存在着不同的数据库驱动实现，我们在使用特定的驱动实现时，不希望修改现有的代码，而希望通过一个简单的配置就可以达到效果，这恰好就是SPI的应用场景。

接下来我们就分析下源码，看看`DriverManager`是如何加载不同的驱动类，以及如何获取某个确定的驱动类的。

`DriverManager`初始化的时候首先会执行一个静态代码块。

```
static {
    // 加载驱动类
    loadInitialDrivers();
    println("JDBC DriverManager initialized");
}
```

```
// 两种不同加载驱动类的方式
// 一种是通过System.getProperty("jdbc.drivers")获取系统变量，加载
// 一种是SPI机制记载META-INF/services路径下的具体驱动类
private static void loadInitialDrivers() {
     String drivers;
     try {
         drivers = AccessController.doPrivileged(new PrivilegedAction<String>() {
             public String run() {
                 // 或者系统变量的值
                 return System.getProperty("jdbc.drivers");
             }
         });
     } catch (Exception ex) {
         drivers = null;
     }
      
     AccessController.doPrivileged(new PrivilegedAction<Void>() {
         public Void run() {
             // 熟悉的代码，通过SPI的方式加载具体的驱动类
             ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class);
             Iterator<Driver> driversIterator = loadedDrivers.iterator();
             try{
                 while(driversIterator.hasNext()) {
                     // 实例化具体的Driver实现类
                     // 每个实现类初始化的时候会调用DriverManager.registerDriver
                     // 将自己注册到registeredDrivers变量中
                     driversIterator.next();
                 }
             } catch(Throwable t) {
                 // Do nothing
             }
             return null;
        }
     });

     println("DriverManager.initialize: jdbc.drivers = " + drivers);

     // 基于系统变量值的初始化方式，与方法开始的第一部分相对应
     if (drivers == null || drivers.equals("")) {
         return;
     }
     String[] driversList = drivers.split(":");
     println("number of Drivers:" + driversList.length);
     for (String aDriver : driversList) {
         try {
             println("DriverManager.Initialize: loading " + aDriver);
             Class.forName(aDriver, true,
                        ClassLoader.getSystemClassLoader());
         } catch (Exception ex) {
             println("DriverManager.Initialize: load failed: " + ex);
         }
     }
}
```

下面我们看看`DriverManager`中是如何获取确定的驱动类的

```
private static Connection getConnection(
    String url, java.util.Properties info, Class<?> caller) throws SQLException {
      
    ClassLoader callerCL = caller != null ? caller.getClassLoader() : null;
    synchronized(DriverManager.class) {
            // synchronize loading of the correct classloader.
        if (callerCL == null) {
            callerCL = Thread.currentThread().getContextClassLoader();
        }
    }

    if(url == null) {
        throw new SQLException("The url cannot be null", "08001");
    }

    println("DriverManager.getConnection(\"" + url + "\")");

    // Walk through the loaded registeredDrivers attempting to make a connection.
    // Remember the first exception that gets raised so we can reraise it.
    SQLException reason = null;
				
				// 遍历已经加载的驱动类
    for(DriverInfo aDriver : registeredDrivers) {
            // If the caller does not have permission to load the driver then
            // skip it.
            // 判断调用方的类加载器是否能加载这个Driver
        if(isDriverAllowed(aDriver.driver, callerCL)) {
            try {
                println("    trying " + aDriver.driver.getClass().getName());
                    // 获取connection，会校验url规则，每种类型的驱动的连接串协议不同
                Connection con = aDriver.driver.connect(url, info);
                if (con != null) {
                        // Success!
                    println("getConnection returning " + aDriver.driver.getClass().getName());
                    return (con);
                }
            } catch (SQLException ex) {
                if (reason == null) {
                    reason = ex;
                }
            }

        } else {
            println("    skipping: " + aDriver.getClass().getName());
        }
    }

        // if we got here nobody could connect.
    if (reason != null)    {
        println("getConnection failed: " + reason);
        throw reason;
     }

    println("getConnection: no suitable driver found for "+ url);
    throw new SQLException("No suitable driver found for "+ url, "08001");
}
```

下面我们以mysql为例，看看mysql-connector-java-5.1.46.jar源码的一些对应实现。

首先可以看到META-INF/services有java.sql.Driver文件，这种`DriverManager`初始化的时候就可以自动加载对应的实现类`com.mysql.jdbc.Driver`

<img src="https://tva1.sinaimg.cn/large/007S8ZIlly1gfjsjsfb7bj30ow07saan.jpg" alt="image-20200607161107557" style="zoom:50%;" />

在看下`com.mysql.jdbc.Driver`的实现

```
public class Driver extends NonRegisteringDriver implements java.sql.Driver {
    public Driver() throws SQLException {
    }

    static {
        try {
            // 调用registerDriver将自己注册到registeredDrivers
            // 后续调用getConnection方法的时候，会用自己对应的是Driver实现获取Connection
            DriverManager.registerDriver(new Driver());
        } catch (SQLException var1) {
            throw new RuntimeException("Can't register driver!");
        }
    }
}
```

#### 5. SLF4J使用SPI分析

SLF4J主要是为了给Java日志访问提供一个标准、规范的API框架，其主要意义在于提供接口，具体的实现可以交由其他日志框架，例如log4j和logback等。

SLF4J是1.7以后的版本也是通过SPI机制加载不通的日志实现框架的，具体的源码大家可以自己跟踪下，下面仅贴下`ServiceLoader`的核心代码

```
// 类路径org.slf4j.LoggerFactory
private static List<SLF4JServiceProvider> findServiceProviders() {
     ServiceLoader<SLF4JServiceProvider> serviceLoader = ServiceLoader.load(SLF4JServiceProvider.class);
     List<SLF4JServiceProvider> providerList = new ArrayList<SLF4JServiceProvider>();
     for (SLF4JServiceProvider provider : serviceLoader) {
         providerList.add(provider);
     }
     return providerList;
}
```

#### 6. Dubbo SPI

##### 6.1 来源

Dubbo 的扩展点加载从 JDK 标准的 SPI (Service Provider Interface) 扩展点发现机制加强而来。

Dubbo 改进了 JDK 标准的 SPI 的以下问题：

- JDK 标准的 SPI 会一次性实例化扩展点所有实现，如果有扩展实现初始化很耗时，但如果没用上也加载，会很浪费资源。Dubbo SPI可以通过key加载指定的实现类。
- 如果扩展点加载失败，连扩展点的名称都拿不到了。比如：JDK 标准的 ScriptEngine，通过 `getName()` 获取脚本类型的名称，但如果 RubyScriptEngine 因为所依赖的 jruby.jar 不存在，导致 RubyScriptEngine 类加载失败，这个失败原因被吃掉了，和 ruby 对应不起来，当用户执行 ruby 脚本时，会报不支持 ruby，而不是真正失败的原因。
- 增加了对扩展点 IoC 和 AOP 的支持，一个扩展点可以直接 setter 注入其它扩展点。

##### 6.2 约定

在扩展类的 jar 包内，放置扩展点配置文件 `META-INF/dubbo/接口全限定名`，内容为：`配置名=扩展实现类全限定名`，多个实现类用换行符分隔。

##### 6.3 示例

复用3.1节，Java SPI示例的代码，Dubbo SPI 的相关逻辑被封装在了 ExtensionLoader 类中，通过 ExtensionLoader，我们可以加载指定的实现类。Dubbo SPI 所需的配置文件需放置在 META-INF/dubbo 路径下，配置内容如下。

```
B = com.test.B
C = com.test.C
```

与 Java SPI 实现类配置不同，Dubbo SPI 是通过键值对的方式进行配置，这样我们可以按需加载指定的实现类。另外，在测试 Dubbo SPI 时，需要在 A 接口上标注 @SPI 注解。下面来演示 Dubbo SPI 的用法：

```
public class DubboSPITest {

    @Test
    public void sayHello() throws Exception {
        ExtensionLoader<A> extensionLoader = 
            ExtensionLoader.getExtensionLoader(A.class);
        A B = extensionLoader.getExtension("B");
        B.sayHello();
        A C = extensionLoader.getExtension("C");
        C.sayHello();
    }
}
```

测试结果如下：

<img src="https://tva1.sinaimg.cn/large/007S8ZIlly1gfjtofn7aqj316w07mq3e.jpg" alt="image-20200607165011521" style="zoom:50%;" />

##### 6.3 源码分析

版本：**dubbo-2.6.4**

我们首先通过 ExtensionLoader 的 getExtensionLoader 方法获取一个 ExtensionLoader 实例，然后再通过 ExtensionLoader 的 getExtension 方法获取拓展类对象。这其中，getExtensionLoader 方法用于从缓存中获取与拓展类对应的 ExtensionLoader，若缓存未命中，则创建一个新的实例。该方法的逻辑比较简单，本章就不进行分析了。下面我们从 ExtensionLoader 的 getExtension 方法作为入口，对拓展类对象的获取过程进行详细的分析。	

```
public T getExtension(String name) {
    if (name == null || name.length() == 0)
        throw new IllegalArgumentException("Extension name == null");
    if ("true".equals(name)) {
        // 获取默认的拓展实现类
        return getDefaultExtension();
    }
    // Holder，顾名思义，用于持有目标对象
    Holder<Object> holder = cachedInstances.get(name);
    if (holder == null) {
        cachedInstances.putIfAbsent(name, new Holder<Object>());
        holder = cachedInstances.get(name);
    }
    Object instance = holder.get();
    // 双重检查
    if (instance == null) {
        synchronized (holder) {
            instance = holder.get();
            if (instance == null) {
                // 创建拓展实例
                instance = createExtension(name);
                // 设置实例到 holder 中
                holder.set(instance);
            }
        }
    }
    return (T) instance;
}
```

上面代码的逻辑比较简单，首先检查缓存，缓存未命中则创建拓展对象。下面我们来看一下创建拓展对象的过程是怎样的。

```
private T createExtension(String name) {
    // 从配置文件中加载所有的拓展类，可得到“配置项名称”到“配置类”的映射关系表
    Class<?> clazz = getExtensionClasses().get(name);
    if (clazz == null) {
        throw findException(name);
    }
    try {
        T instance = (T) EXTENSION_INSTANCES.get(clazz);
        if (instance == null) {
            // 通过反射创建实例
            EXTENSION_INSTANCES.putIfAbsent(clazz, clazz.newInstance());
            instance = (T) EXTENSION_INSTANCES.get(clazz);
        }
        // 向实例中注入依赖
        injectExtension(instance);
        Set<Class<?>> wrapperClasses = cachedWrapperClasses;
        if (wrapperClasses != null && !wrapperClasses.isEmpty()) {
            // 循环创建 Wrapper 实例
            for (Class<?> wrapperClass : wrapperClasses) {
                // 将当前 instance 作为参数传给 Wrapper 的构造方法，并通过反射创建 Wrapper 实例。
                // 然后向 Wrapper 实例中注入依赖，最后将 Wrapper 实例再次赋值给 instance 变量
                instance = injectExtension(
                    (T) wrapperClass.getConstructor(type).newInstance(instance));
            }
        }
        return instance;
    } catch (Throwable t) {
        throw new IllegalStateException("...");
    }
}
```

createExtension 方法的逻辑稍复杂一下，包含了如下的步骤：

1. 通过 getExtensionClasses 获取所有的拓展类
2. 通过反射创建拓展对象
3. 向拓展对象中注入依赖
4. 将拓展对象包裹在相应的 Wrapper 对象中

以上步骤中，第一个步骤是加载拓展类的关键，第三和第四个步骤是 Dubbo IOC 与 AOP 的具体实现。在接下来的章节中，将会重点分析 getExtensionClasses 方法的逻辑，以及简单介绍 Dubbo IOC 的具体实现。

###### 6.3.1 获取所有的拓展类

我们在通过名称获取拓展类之前，首先需要根据配置文件解析出拓展项名称到拓展类的映射关系表（Map<名称, 拓展类>），之后再根据拓展项名称从映射关系表中取出相应的拓展类即可。相关过程的代码分析如下：

```
private Map<String, Class<?>> getExtensionClasses() {
    // 从缓存中获取已加载的拓展类
    Map<String, Class<?>> classes = cachedClasses.get();
    // 双重检查
    if (classes == null) {
        synchronized (cachedClasses) {
            classes = cachedClasses.get();
            if (classes == null) {
                // 加载拓展类
                classes = loadExtensionClasses();
                cachedClasses.set(classes);
            }
        }
    }
    return classes;
}
```

这里也是先检查缓存，若缓存未命中，则通过 synchronized 加锁。加锁后再次检查缓存，并判空。此时如果 classes 仍为 null，则通过 loadExtensionClasses 加载拓展类。下面分析 loadExtensionClasses 方法的逻辑。

```
private Map<String, Class<?>> loadExtensionClasses() {
    // 获取 SPI 注解，这里的 type 变量是在调用 getExtensionLoader 方法时传入的
    final SPI defaultAnnotation = type.getAnnotation(SPI.class);
    if (defaultAnnotation != null) {
        String value = defaultAnnotation.value();
        if ((value = value.trim()).length() > 0) {
            // 对 SPI 注解内容进行切分
            String[] names = NAME_SEPARATOR.split(value);
            // 检测 SPI 注解内容是否合法，不合法则抛出异常
            if (names.length > 1) {
                throw new IllegalStateException("more than 1 default extension name on extension...");
            }

            // 设置默认名称，参考 getDefaultExtension 方法
            if (names.length == 1) {
                cachedDefaultName = names[0];
            }
        }
    }

    Map<String, Class<?>> extensionClasses = new HashMap<String, Class<?>>();
    // 加载指定文件夹下的配置文件
    loadDirectory(extensionClasses, DUBBO_INTERNAL_DIRECTORY);
    loadDirectory(extensionClasses, DUBBO_DIRECTORY);
    loadDirectory(extensionClasses, SERVICES_DIRECTORY);
    return extensionClasses;
}
```

loadExtensionClasses 方法总共做了两件事情，一是对 SPI 注解进行解析，二是调用 loadDirectory 方法加载指定文件夹配置文件。SPI 注解解析过程比较简单，无需多说。下面我们来看一下 loadDirectory 做了哪些事情。

```
private void loadDirectory(Map<String, Class<?>> extensionClasses, String dir) {
    // fileName = 文件夹路径 + type 全限定名 
    String fileName = dir + type.getName();
    try {
        Enumeration<java.net.URL> urls;
        ClassLoader classLoader = findClassLoader();
        // 根据文件名加载所有的同名文件
        if (classLoader != null) {
            urls = classLoader.getResources(fileName);
        } else {
            urls = ClassLoader.getSystemResources(fileName);
        }
        if (urls != null) {
            while (urls.hasMoreElements()) {
                java.net.URL resourceURL = urls.nextElement();
                // 加载资源
                loadResource(extensionClasses, classLoader, resourceURL);
            }
        }
    } catch (Throwable t) {
        logger.error("...");
    }
}
```

loadDirectory 方法先通过 classLoader 获取所有资源链接，然后再通过 loadResource 方法加载资源。我们继续跟下去，看一下 loadResource 方法的实现。

```
private void loadResource(Map<String, Class<?>> extensionClasses, 
	ClassLoader classLoader, java.net.URL resourceURL) {
    try {
        BufferedReader reader = new BufferedReader(
            new InputStreamReader(resourceURL.openStream(), "utf-8"));
        try {
            String line;
            // 按行读取配置内容
            while ((line = reader.readLine()) != null) {
                // 定位 # 字符
                final int ci = line.indexOf('#');
                if (ci >= 0) {
                    // 截取 # 之前的字符串，# 之后的内容为注释，需要忽略
                    line = line.substring(0, ci);
                }
                line = line.trim();
                if (line.length() > 0) {
                    try {
                        String name = null;
                        int i = line.indexOf('=');
                        if (i > 0) {
                            // 以等于号 = 为界，截取键与值
                            name = line.substring(0, i).trim();
                            line = line.substring(i + 1).trim();
                        }
                        if (line.length() > 0) {
                            // 加载类，并通过 loadClass 方法对类进行缓存
                            loadClass(extensionClasses, resourceURL, 
                                      Class.forName(line, true, classLoader), name);
                        }
                    } catch (Throwable t) {
                        IllegalStateException e = new IllegalStateException("Failed to load extension class...");
                    }
                }
            }
        } finally {
            reader.close();
        }
    } catch (Throwable t) {
        logger.error("Exception when load extension class...");
    }
}
```

loadResource 方法用于读取和解析配置文件，并通过反射加载类，最后调用 loadClass 方法进行其他操作。loadClass 方法用于主要用于操作缓存，该方法的逻辑如下：

```
private void loadClass(Map<String, Class<?>> extensionClasses, java.net.URL resourceURL, 
    Class<?> clazz, String name) throws NoSuchMethodException {
    
    if (!type.isAssignableFrom(clazz)) {
        throw new IllegalStateException("...");
    }

    // 检测目标类上是否有 Adaptive 注解
    if (clazz.isAnnotationPresent(Adaptive.class)) {
        if (cachedAdaptiveClass == null) {
            // 设置 cachedAdaptiveClass缓存
            cachedAdaptiveClass = clazz;
        } else if (!cachedAdaptiveClass.equals(clazz)) {
            throw new IllegalStateException("...");
        }
        
    // 检测 clazz 是否是 Wrapper 类型
    } else if (isWrapperClass(clazz)) {
        Set<Class<?>> wrappers = cachedWrapperClasses;
        if (wrappers == null) {
            cachedWrapperClasses = new ConcurrentHashSet<Class<?>>();
            wrappers = cachedWrapperClasses;
        }
        // 存储 clazz 到 cachedWrapperClasses 缓存中
        wrappers.add(clazz);
        
    // 程序进入此分支，表明 clazz 是一个普通的拓展类
    } else {
        // 检测 clazz 是否有默认的构造方法，如果没有，则抛出异常
        clazz.getConstructor();
        if (name == null || name.length() == 0) {
            // 如果 name 为空，则尝试从 Extension 注解中获取 name，或使用小写的类名作为 name
            name = findAnnotationName(clazz);
            if (name.length() == 0) {
                throw new IllegalStateException("...");
            }
        }
        // 切分 name
        String[] names = NAME_SEPARATOR.split(name);
        if (names != null && names.length > 0) {
            Activate activate = clazz.getAnnotation(Activate.class);
            if (activate != null) {
                // 如果类上有 Activate 注解，则使用 names 数组的第一个元素作为键，
                // 存储 name 到 Activate 注解对象的映射关系
                cachedActivates.put(names[0], activate);
            }
            for (String n : names) {
                if (!cachedNames.containsKey(clazz)) {
                    // 存储 Class 到名称的映射关系
                    cachedNames.put(clazz, n);
                }
                Class<?> c = extensionClasses.get(n);
                if (c == null) {
                    // 存储名称到 Class 的映射关系
                    extensionClasses.put(n, clazz);
                } else if (c != clazz) {
                    throw new IllegalStateException("...");
                }
            }
        }
    }
}
```

如上，loadClass 方法操作了不同的缓存，比如 cachedAdaptiveClass、cachedWrapperClasses 和 cachedNames 等等。除此之外，该方法没有其他什么逻辑了。

到此，关于缓存类加载的过程就分析完了。整个过程没什么特别复杂的地方，大家按部就班的分析即可，不懂的地方可以调试一下。接下来，我们来聊聊 Dubbo IOC 方面的内容。

###### 6.3.2 Dubbo IOC

Dubbo IOC 是通过 setter 方法注入依赖。Dubbo 首先会通过反射获取到实例的所有方法，然后再遍历方法列表，检测方法名是否具有 setter 方法特征。若有，则通过 ObjectFactory 获取依赖对象，最后通过反射调用 setter 方法将依赖设置到目标对象中。整个过程对应的代码如下：

```
private T injectExtension(T instance) {
    try {
        if (objectFactory != null) {
            // 遍历目标类的所有方法
            for (Method method : instance.getClass().getMethods()) {
                // 检测方法是否以 set 开头，且方法仅有一个参数，且方法访问级别为 public
                if (method.getName().startsWith("set")
                    && method.getParameterTypes().length == 1
                    && Modifier.isPublic(method.getModifiers())) {
                    // 获取 setter 方法参数类型
                    Class<?> pt = method.getParameterTypes()[0];
                    try {
                        // 获取属性名，比如 setName 方法对应属性名 name
                        String property = method.getName().length() > 3 ? 
                            method.getName().substring(3, 4).toLowerCase() + 
                            	method.getName().substring(4) : "";
                        // 从 ObjectFactory 中获取依赖对象
                        Object object = objectFactory.getExtension(pt, property);
                        if (object != null) {
                            // 通过反射调用 setter 方法设置依赖
                            method.invoke(instance, object);
                        }
                    } catch (Exception e) {
                        logger.error("fail to inject via method...");
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

在上面代码中，objectFactory 变量的类型为 AdaptiveExtensionFactory，AdaptiveExtensionFactory 内部维护了一个 ExtensionFactory 列表，用于存储其他类型的 ExtensionFactory。Dubbo 目前提供了两种 ExtensionFactory，分别是 SpiExtensionFactory 和 SpringExtensionFactory。前者用于创建自适应的拓展，后者是用于从 Spring 的 IOC 容器中获取所需的拓展。这两个类的类的代码不是很复杂，这里就不一一分析了。

Dubbo IOC 目前仅支持 setter 方式注入，总的来说，逻辑比较简单易懂。

#### 7. 参考

- dubbo源码部分http://dubbo.apache.org/zh-cn/docs/source_code_guide/dubbo-spi.html