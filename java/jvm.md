[toc]

# 1. jvm类加载机制

## 1. 类加载器

在jvm的类加载机制中，主要存在三类默认的类加载器，BootstrapClassLoader，ExtClassLoader，AapplicationClassLoader，并且存在父子关系。

- 根加载器（BootstrapClassLoader，由c/c++语言，java不可见）：负责装载jre的核心类型，如jre目录下的rt.jar，charsets.jar等。

  ![image-20191211191927892](jvm.assets/image-20191211191927892.png)

- 扩展类加载器（ExtClassLoader）：负责装载jre下扩展jar包，ext/**.jar。

  ![image-20191211191820374](jvm.assets/image-20191211191820374.png)

- 应用程序类加载器（ApplicationClassLoader）：负责装载应用程序的class或者依赖的jar。

## 2. 双亲委派机制

下图就是双亲委派机制（父类委托）。

 ![image-20191210211721667](jvm.assets/image-20191210211721667.png)

源码分析。

```java
// java.lang.ClassLoader#loadClass(java.lang.String, boolean)
protected Class<?> loadClass(String name, boolean resolve)
    throws ClassNotFoundException
{
    synchronized (getClassLoadingLock(name)) {
        // 判断当前类是否已经加载（先从缓存开始查找）
        // First, check if the class has already been loaded
        Class<?> c = findLoadedClass(name);
        if (c == null) {// 没有加载
            long t0 = System.nanoTime();
            try {
                if (parent != null) {
                    // 调用父类的loadClass
                    c = parent.loadClass(name, false);
                } else {
                    c = findBootstrapClassOrNull(name);
                }
            } catch (ClassNotFoundException e) {
                // ClassNotFoundException thrown if class not found
                // from the non-null parent class loader
            }
			// 如果从父加载器中获取的为null
            if (c == null) {
                // If still not found, then invoke findClass in order
                // to find the class.
                long t1 = System.nanoTime();
                // 从当前类加载器中加载
                c = findClass(name);
                // this is the defining class loader; record the stats
                sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                sun.misc.PerfCounter.getFindClasses().increment();
            }
        }
        if (resolve) {
            resolveClass(c);
        }
        return c;
    }
}
```



## 3. 全盘负责委托机制

是指当一个ClassLoader装载一个类时，除非显示的指定使用另外一个ClassLoader，该类所依赖及引用的所有类也有这个ClassLoader加载，同时使用委托机制，从父ClassLoader开始装载。

```java
// 复写系统String类
package java.lang;

public class String {
	public void method()
	{
	}
	public static void main(String[] args) {
		new String().method();
	}
}
```

执行报错。这是因为当前ApplicationClassLoader在加载java.lang.String类时，先委托ExtClassLoader，再委托BootstrapClassLoader进行加载，而根装载器可以装载到jre的java.lang.String类，而该类没有main方法，所有导致程序报错。

```txt
错误: 在类 java.lang.String 中找不到 main 方法, 请将 main 方法定义为:
   public static void main(String[] args)
否则 JavaFX 应用程序类必须扩展javafx.application.Application

Process finished with exit code 1
```

> 注意：不同类加载器就算加载相同的类，也会生成不同的对象，这是因为加载时机不同，或者路径不同，导致类的元数据不同，使得类对象由不同的代码或者行为。

## 4. 自定义类加载器

自定义类加载器可以打破双亲委派机制，可以通过自定义类加载器实现热加载替换class文件。

```java
public class CustomerClassLoader extends ClassLoader {
	private static final String rootPath = CustomerClassLoader.class.getResource("/").getPath();;
	private static final String path = "org/xxx/HotDeploy";
	@Override
	public Class<?> loadClass(String name) throws ClassNotFoundException {
		Class<?> c = null;
        // 每次只加载包含xxx的自定义类，其它类还是通过双亲委派由其它系统默认类加载器加载。
		if(name.contains("xxx"))
		{
			c = findLoadedClass(name, true);
		}
		if(null == c)
		{
			c = getSystemClassLoader().loadClass(name);
		}
		return c;
	}

	public Class<?> findLoadedClass(String name, boolean resovler){
		String fullPath = rootPath + path + ".class";
		File classFile = new File(fullPath);
		byte[] bytes = new byte[(int) classFile.length()];
		try {
			InputStream inputStream = new FileInputStream(classFile);
			inputStream.read(bytes);
			inputStream.close();
		} catch (FileNotFoundException e) {
			e.printStackTrace();
		} catch (IOException e) {
			e.printStackTrace();
		}

		Class<?> defineClass = defineClass(name, bytes, 0, bytes.length);
		return defineClass;
	}
	public static void main(String[] args) throws ClassNotFoundException, InterruptedException {

		String fullPath = rootPath + path + ".class";
		System.out.println(fullPath);
		while(true) {
            // 每次循环必须new一个全新的类加载器，否则同一个类加载器重复加载会报异常（loader (instance of  xxx): attempted  duplicate class definition for name）
			CustomerClassLoader loader = new CustomerClassLoader();
			loader.loadClass("org.xxx.HotDeploy");
			new HotDeploy().method();
			TimeUnit.SECONDS.sleep(5);
		}
	}
}
// 通过自定义加载器，定时刷新加载该类的class文件（修改完打印信息后，重新编译）
public class HotDeploy {
	public void method()
	{
		System.out.println("version 3333333333333333333333");
	}
}
```

```txt
version 22222222222222
version 22222222222222
version 22222222222222
version 22222222222222
version 22222222222222
version 3333333333333333333333
```

通过以上代码可以实现项目的热部署加载（可以借助第三方框架来监听class文件的变化，有了变化再重新加载）。



