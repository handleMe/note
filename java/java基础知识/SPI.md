# 1. 开始感受SPI机制的作用

SPI(Service Provider Interface)，翻译成为中文叫做"服务提供接口"。它的作用是什么呢？它的作用其实就是根据一个接口，找到项目中要使用的实现类，用户可以根据实际需要启用或者是扩展原来的默认策略。

下面先以jdk提供的`ServiceLoader`这个类库去进行举例吧。

通过`ServiceLoader.load`方法中传入指定的接口，它会加载这个接口下的指定实现类，那么这个指定实现类具体是指什么？需要在模块的`resources/META-INF/services/`下创建一个文件，文件名为接口的全类名，文件中的内容为需要通过SPI拿到的实现类的全类名(可以指定多个，每行一个)。

我们编写如下这样一个接口



```java
public interface SPIService {
    public void serve();
}
```

再编写两个它的实现类吧



```java
public class SPIServiceImpl1 implements SPIService {
    @Override
    public void serve() {
        System.out.println("SPIServiceImpl1 is serve...");
    }
}
//.....................................分隔符..........................................
public class SPIServiceImpl2 implements SPIService {
    @Override
    public void serve() {
        System.out.println("SPIServiceImpl2 is serve...");
    }
}
```

在`resources`目录下建立`META-INF/services/`目录，并创建一个文件`com.wanna.spi.SPIService`指定的是服务接口的全类名，在文件中配置的内容是这个接口的类的实现类`com.wanna.spi.SPIServiceImpl1`和`com.wanna.spi.SPIServiceImpl2`。

![img](https:////upload-images.jianshu.io/upload_images/24412352-0d66105f8576c5df.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

24412352ccf0bbed551b74b9.png

预备工作完成，下面我们要做的，就是使用SPI机制，获取这两个实现类？编写如下的测试代码：



```java
public class SpiTest {
    public static void main(String[] args) {
        ServiceLoader<SPIService> serviceLoader = ServiceLoader.load(SPIService.class);
        serviceLoader.stream().forEach(e -> e.get().serve());
    }
}
```

得到运行结果：



```java
SPIServiceImpl1 is serve...
SPIServiceImpl2 is serve...
```

也就是说，通过ServiceLoader，成功地加载到了我们编写的两个实现类。

# 2. 开始了解ServiceLoader的源码？

直接打开`ServiceLoader.load`方法的源码



```java
    public static <S> ServiceLoader<S> load(Class<S> service) {
        ClassLoader cl = Thread.currentThread().getContextClassLoader();
        return new ServiceLoader<>(Reflection.getCallerClass(), service, cl);
    }
```

获取当前线程的上下文加载器、调用方类和Service类，从而去创建一个ServiceLoader对象，然后在构造器中就没做什么。

接着我们主要关心它的迭代器的创建：



```java
    public Iterator<S> iterator() {

        // create lookup iterator if needed
        if (lookupIterator1 == null) {
            lookupIterator1 = newLookupIterator();
        }

        return new Iterator<S>() {

            // record reload count
            final int expectedReloadCount = ServiceLoader.this.reloadCount;

            // index into the cached providers list
            int index;

            /**
             * Throws ConcurrentModificationException if the list of cached
             * providers has been cleared by reload.
             */
            private void checkReloadCount() {
                if (ServiceLoader.this.reloadCount != expectedReloadCount)
                    throw new ConcurrentModificationException();
            }

            @Override
            public boolean hasNext() {
                checkReloadCount();
                if (index < instantiatedProviders.size())
                    return true;
                return lookupIterator1.hasNext();
            }

            @Override
            public S next() {
                checkReloadCount();
                S next;
                if (index < instantiatedProviders.size()) {
                    next = instantiatedProviders.get(index);
                } else {
                    next = lookupIterator1.next().get();
                    instantiatedProviders.add(next);
                }
                index++;
                return next;
            }

        };
    }
```

最主要的组件在于它创建的`lookupIterator1`字段：



```java
    private Iterator<Provider<S>> newLookupIterator() {
        assert layer == null || loader == null;
        if (layer != null) {
            return new LayerLookupIterator<>();
        } else {
            Iterator<Provider<S>> first = new ModuleServicesLookupIterator<>();
            Iterator<Provider<S>> second = new LazyClassPathLookupIterator<>();
            return new Iterator<Provider<S>>() {
                @Override
                public boolean hasNext() {
                    return (first.hasNext() || second.hasNext());
                }
                @Override
                public Provider<S> next() {
                    if (first.hasNext()) {
                        return first.next();
                    } else if (second.hasNext()) {
                        return second.next();
                    } else {
                        throw new NoSuchElementException();
                    }
                }
            };
        }
    }
```

主要创建了两个迭代器，主要关注`LazyClassPathLookupIterator`，我们就可以猜测它是负责加载`META-INF/services/`下的Provider的加载。



```java
    private final class LazyClassPathLookupIterator<T>
        implements Iterator<Provider<T>>
    {
        static final String PREFIX = "META-INF/services/";

        Set<String> providerNames = new HashSet<>();  // to avoid duplicates
        Enumeration<URL> configs;
        Iterator<String> pending;

        Provider<T> nextProvider;
        ServiceConfigurationError nextError;
        //---------------------------------------------------------------------

        private Class<?> nextProviderClass() {
            if (configs == null) {
                try {
                    String fullName = PREFIX + service.getName();
                    if (loader == null) {
                        configs = ClassLoader.getSystemResources(fullName);
                    } else if (loader == ClassLoaders.platformClassLoader()) {
                        // The platform classloader doesn't have a class path,
                        // but the boot loader might.
                        if (BootLoader.hasClassPath()) {
                            configs = BootLoader.findResources(fullName);
                        } else {
                            configs = Collections.emptyEnumeration();
                        }
                    } else {
                        configs = loader.getResources(fullName);
                    }
                } catch (IOException x) {
                    fail(service, "Error locating configuration files", x);
                }
            }
            while ((pending == null) || !pending.hasNext()) {
                if (!configs.hasMoreElements()) {
                    return null;
                }
                pending = parse(configs.nextElement());
            }
            String cn = pending.next();
            try {
                return Class.forName(cn, false, loader);
            } catch (ClassNotFoundException x) {
                fail(service, "Provider " + cn + " not found");
                return null;
            }
        }
  }
```

我们还可以看到，最终就是使用`ClassLoader.getResources`去完成的相关实现类的URL的获取，最后保存到pending队列中，然后使用`Class.forName`返回一个加载到的Class。

# 3. SPI机制在JdbcDriver中的使用？

我们打开mysql的驱动jar包，我们发现也是在`META-INF/services/`下存放了一个`java.sql.Driver`文件，里面放入了一个Driver的实现类`com.mysql.cj.jdbc.Driver`。

![img](https:////upload-images.jianshu.io/upload_images/24412352-c252111538ab6a93.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

image.png

在`DriverManager.getDriver`方法中，会有如下一步，确保Driver被初始化完成。



```java
public static Driver getDriver(String url) throws SQLException {
        ensureDriversInitialized();
        //-------------以下代码省略-------------------
}
```

在这个方法中有如下两行代码，就是使用到的`ServiceLoader`去进行加载`Driver`的实现类。



```java
ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class);
Iterator<Driver> driversIterator = loadedDrivers.iterator();
```

# 4. SPI机制在Spring中的使用？

在Spring Framework的`spring-web`模块中，有如下的文件，主要是用来找到Spring的Servlet容器的初始化器。

![img](https:////upload-images.jianshu.io/upload_images/24412352-8d61d2e98ac6f327.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

image.png

我们可以手写一个`SpringServletContainerInitializer`，Spring容器在启动的过程中，就会回调它的onStartup方法，从而完成Servlet等配置工作，这个过程也就是传统的SpringMVC的注解版的使用方法。



```java
public class MyMvcStarter implements WebApplicationInitializer {
    @Override
    public void onStartup(ServletContext servletContext) throws ServletException {
        //Spring会给我们传入servletContext

        AnnotationConfigWebApplicationContext context = new AnnotationConfigWebApplicationContext();
        context.register(AppConfig.class);

        DispatcherServlet servlet = new DispatcherServlet(context);
        final ServletRegistration.Dynamic app = servletContext.addServlet("app", servlet);
        app.setLoadOnStartup(1);
        app.addMapping("/");  //指定servlet的映射路径
    }
}
```

还有一个典型应用是在SpringBoot中自定义场景启动器时，需要在`META-INF/`目录下配置一个`spring.factories`文件，去进行相关的配置从而往容器中添加合适的组件，不过这个使用的不是ServiceLoader，而是自定义了一个路径，并且自定义了自己的加载器去完成相关组件的加载



