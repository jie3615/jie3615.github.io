---
layout: post
title:  "Jvm系列-深入理解类加载器ClassLoader"
date:   2019-09-29 20:06:06
categories: 并发
tags: 类加载器 ClassLoader Jvm
---

* content
{:toc}
## 类加载器的种类

1. 根类/启动类加载器
2. 扩展类加载器
3. 系统类/应用类加载器
4. 用户自己定义的类加载器
5. 线程上下文类加载器

前三种是默认的三种类加载器，他们分别负责加载不同路径下的class文件；如下：





```java

Bootstrap 加载路径：
file:/C:/Program%20Files/Java/jdk1.8.0_162/jre/lib/resources.jar
file:/C:/Program%20Files/Java/jdk1.8.0_162/jre/lib/rt.jar
file:/C:/Program%20Files/Java/jdk1.8.0_162/jre/lib/sunrsasign.jar
file:/C:/Program%20Files/Java/jdk1.8.0_162/jre/lib/jsse.jar
file:/C:/Program%20Files/Java/jdk1.8.0_162/jre/lib/jce.jar
file:/C:/Program%20Files/Java/jdk1.8.0_162/jre/lib/charsets.jar
file:/C:/Program%20Files/Java/jdk1.8.0_162/jre/lib/jfr.jar
file:/C:/Program%20Files/Java/jdk1.8.0_162/jre/classes/
ExtClassLoader 加载路径：
sun.misc.Launcher$ExtClassLoader@6d6f6e28
file:/C:/Program%20Files/Java/jdk1.8.0_162/jre/lib/ext/access-bridge-64.jar
file:/C:/Program%20Files/Java/jdk1.8.0_162/jre/lib/ext/cldrdata.jar
file:/C:/Program%20Files/Java/jdk1.8.0_162/jre/lib/ext/dnsns.jar
file:/C:/Program%20Files/Java/jdk1.8.0_162/jre/lib/ext/jaccess.jar
file:/C:/Program%20Files/Java/jdk1.8.0_162/jre/lib/ext/jfxrt.jar
file:/C:/Program%20Files/Java/jdk1.8.0_162/jre/lib/ext/localedata.jar
file:/C:/Program%20Files/Java/jdk1.8.0_162/jre/lib/ext/nashorn.jar
file:/C:/Program%20Files/Java/jdk1.8.0_162/jre/lib/ext/sunec.jar
file:/C:/Program%20Files/Java/jdk1.8.0_162/jre/lib/ext/sunjce_provider.jar
file:/C:/Program%20Files/Java/jdk1.8.0_162/jre/lib/ext/sunmscapi.jar
file:/C:/Program%20Files/Java/jdk1.8.0_162/jre/lib/ext/sunpkcs11.jar
file:/C:/Program%20Files/Java/jdk1.8.0_162/jre/lib/ext/zipfs.jar
SystemClassLoader 加载路径：
sun.misc.Launcher$AppClassLoader@18b4aac2
file:/C:/Program%20Files/Java/jdk1.8.0_162/jre/lib/charsets.jar
file:/C:/Program%20Files/Java/jdk1.8.0_162/jre/lib/deploy.jar
file:/C:/Program%20Files/Java/jdk1.8.0_162/jre/lib/ext/access-bridge-64.jar
file:/C:/Program%20Files/Java/jdk1.8.0_162/jre/lib/ext/cldrdata.jar
file:/C:/Program%20Files/Java/jdk1.8.0_162/jre/lib/ext/dnsns.jar
file:/C:/Program%20Files/Java/jdk1.8.0_162/jre/lib/ext/jaccess.jar
file:/C:/Program%20Files/Java/jdk1.8.0_162/jre/lib/ext/jfxrt.jar
file:/C:/Program%20Files/Java/jdk1.8.0_162/jre/lib/ext/localedata.jar
file:/C:/Program%20Files/Java/jdk1.8.0_162/jre/lib/ext/nashorn.jar
file:/C:/Program%20Files/Java/jdk1.8.0_162/jre/lib/ext/sunec.jar
file:/C:/Program%20Files/Java/jdk1.8.0_162/jre/lib/ext/sunjce_provider.jar
file:/C:/Program%20Files/Java/jdk1.8.0_162/jre/lib/ext/sunmscapi.jar
file:/C:/Program%20Files/Java/jdk1.8.0_162/jre/lib/ext/sunpkcs11.jar
file:/C:/Program%20Files/Java/jdk1.8.0_162/jre/lib/ext/zipfs.jar
file:/C:/Program%20Files/Java/jdk1.8.0_162/jre/lib/javaws.jar
file:/C:/Program%20Files/Java/jdk1.8.0_162/jre/lib/jce.jar
file:/C:/Program%20Files/Java/jdk1.8.0_162/jre/lib/jfr.jar
file:/C:/Program%20Files/Java/jdk1.8.0_162/jre/lib/jfxswt.jar
file:/C:/Program%20Files/Java/jdk1.8.0_162/jre/lib/jsse.jar
file:/C:/Program%20Files/Java/jdk1.8.0_162/jre/lib/management-agent.jar
file:/C:/Program%20Files/Java/jdk1.8.0_162/jre/lib/plugin.jar
file:/C:/Program%20Files/Java/jdk1.8.0_162/jre/lib/resources.jar
file:/C:/Program%20Files/Java/jdk1.8.0_162/jre/lib/rt.jar
file:/F:/gitPro/java_study/out/production/java_study/
file:/F:/gitPro/java_study/lib/mysql-connector-java-5.1.44.jar
file:/F:/gitPro/java_study/lib/junit-4.8.2.jar
file:/D:/Program%20Files/JetBrains/IntelliJ%20IDEA%202017.3.5/lib/idea_rt.jar
sun.misc.Launcher$AppClassLoader@18b4aac2
414493378

```

其中  file:/F:/gitPro/java_study/out/production/java_study/ 是我项目的classpath；

## 什么是双亲委派

在加载某个类的时候，首先交给父类去加载，依次递归，如果成功加载了，就返回。没有成功加载就由自己去加载；

### 双亲委派的好处？

- 可以避免同一个类被重复加载

- 保证Java核心类的安全

-  保证Java核心类库不被自定义类加载器所破坏

  自定义类加载器可以打破双亲委派模型，下面提供一个自定义的类加载器，有兴趣的同学可以去我的github仓库中订阅，可以看到更多的关于类加载器的小案例；

  ```java
  package jvm.classloaderProcessed;
  
  import java.io.*;
  
  /**
   * @author: wyj
   * @date: 2019/9/21
   * @description:
   */
  public class LoaderTest5 extends ClassLoader {
  
      private String classLoaderName;
      private final String fileExtension = ".class";
      private String path;
  
      public void setPath(String path) {
          this.path = path;
      }
  
      public LoaderTest5(String classLoaderName){
          super();//使用系统类加载器作为当前类的父类委托加载器
          this.classLoaderName = classLoaderName;
      }
  
      public LoaderTest5(ClassLoader classLoader,String classLoaderName){
          super(classLoader);//使用自定义的类加载器作为当前类的父类委托加载器
          this.classLoaderName = classLoaderName;
      }
  
      private byte[] loadClassData(String name ){
          InputStream inputStream = null;
          ByteArrayOutputStream baos = null;
          byte [] bytes = null;
  
          try{
              //this.classLoaderName = this.classLoaderName.replace(".","\\");
              name = name.replace(".","\\");//将包名里边的"."替换为路径分隔符
              inputStream = new FileInputStream(new File(this.path + name + this.fileExtension));//文件来自于加载路径path下的包里边的class文件
              baos = new ByteArrayOutputStream();
              int ch = 0;
              while (-1 !=(ch = inputStream.read())){
                  baos.write(ch);
              }
              bytes = baos.toByteArray();
          }catch (Exception e){
              e.printStackTrace();
          }finally {
              try{
                  inputStream.close();
                  baos.close();
              }catch (Exception e){
                  e.printStackTrace();
              }
          }
          return bytes;
      }
  
      @Override
      protected Class<?> findClass(String className) throws ClassNotFoundException {
          System.out.println("findClass invoked "+className);
          System.out.println(" this.classLoaderName : "+ this.classLoaderName);
          byte [] data = loadClassData(className);//中间调用子类的findClass方法
          return defineClass(className,data,0,data.length);
      }
  
      public static void main(String[] args) throws Exception {
          LoaderTest5 myClassLoader = new LoaderTest5("myClassLoader");
          myClassLoader.setPath("E:\\");
  
  
         /* Class<?> clazz = myClassLoader.loadClass("beans.User");
  
          System.out.println("class :"+clazz.hashCode());
          // 需要有无参构造器
          Object object = clazz.newInstance();
          System.out.println(object.getClass().getClassLoader());
          System.out.println(object);*/
  
          System.out.println("###############################");
          LoaderTest5 myClassLoader2 = new LoaderTest5(myClassLoader,"myClassLoader2"); // 传入父  classLoader 会由父类加载
          myClassLoader2.setPath("E:\\");
  
          Class<?> clazz2 = myClassLoader2.loadClass("beans.User");
          System.out.println("class :"+clazz2.hashCode());
          // 需要有无参构造器
          Object object2 = clazz2.newInstance();
          System.out.println(object2.getClass().getClassLoader());
          System.out.println(object2);
  
          System.out.println("#########################");
          LoaderTest5 myClassLoader3 = new LoaderTest5( myClassLoader2,"myClassLoader3"); // 传入父classLoader 会由父类加载
          myClassLoader3.setPath("E:\\");
  
          Class<?> clazz3 = myClassLoader3.loadClass("beans.User");
  
          System.out.println("class :"+clazz3.hashCode());
          // 需要有无参构造器
          Object object3 = clazz3.newInstance();
          System.out.println(object3.getClass().getClassLoader());
          System.out.println(object3);
          /**
           * 结果：###############################
           findClass invoked beans.User
           this.classLoaderName : myClassLoader
           class :1173230247
           jvm.classloaderProcessed.LoaderTest5@6d6f6e28
           User{name='null', age=0}
  
           Process finished with exit code 0
           */
      }
  
      }
  ```


ClassLoader中getSystemClassLoader()是返回一个系统类加载器，他是遵循双亲委派机制的，接下来我们看看源码是怎么实现的？

## 源码角度理解双亲委派模型

```java
initSystemClassLoader();
sclSet 判断是否设置系统类加载器；，如果为假，没有设置那么继续，
在判断scl是否为空，是一个双重判断；，如果不为空抛出异常
通过Lanucher.getLauncher()
这个方法没有开源，可以通过反编译查看
首先是通过构造方法返回launcher;
构造方法：首先创建扩展类加载器；
            var1 = Launcher.ExtClassLoader.getExtClassLoader();
            // 获取扩展类加载器读取目录；
            final File[] var0 = getExtDirs(); 
            // 根据系统属性读取
             String var0 = System.getProperty("java.ext.dirs");
      权限校验doPrivileged 返回run方法中的结果；extClassLoader
      接下来创建AppClassLoader
      this.loader = Launcher.AppClassLoader.getAppClassLoader(var1);
      结果赋给Launcher的一个私有的成员变量loader;
      对于Launcher的loader是赋值的系统类加载器；
      getAppClassLoader(extClassLoader) 
      加载final String var1 = System.getProperty("java.class.path");
      java.class.path中类；
      final File[] var2 = var1 == null ? new File[0] : Launcher.getClassPath(var1);
      获取路径下文件
      return new Launcher.AppClassLoader(var1x, var0);
      第二个参数是extClassLoader 
      AppClassLoader(URL[] var1, ClassLoader var2) {
      super(var1, var2, Launcher.factory);
      this.ucp.initLookupCache(this);
      调用父类的构造函数
      public URLClassLoader(URL[] urls, ClassLoader parent,URLStreamHandlerFactory factory) 
	 会把extClassLoader作为appClassLoder的父类；
     回到Launcher的构造函数
     Thread.currentThread().setContextClassLoader(this.loader);
     把刚刚创建的系统类加载器传进去，为当前的执行线程设置一个上下文类加载器。就是上面的应用类加载器；
     下面就是一些安全管理器相关的逻辑；
     至此getSystemAppClassLoader完毕；
     private static String bootClassPath = System.getProperty("sun.boot.class.path");这个是启动类加载器的路径；
回到initSystemClassLoader()
scl = l.getClassLoader();
将上面创建的系统类加载器赋值给scl;
scl = AccessController.doPrivileged(new SystemClassLoaderAction(scl)); 
     SystemClassLoaderAction是ClassLoader的一个内部类，把scl传入给parent属性，
     会执行其中的run方法，函数式接口
     public ClassLoader run() throws Exception {
   	 // 首先获取自定义的类加载器的属性
    String cls = System.getProperty("java.system.class.loader");
    // 如果为空，那就是我们没有配置系统类加载器，则直接返回；
    if (cls == null) {
        return parent;
    }
     // 否则用户自定义了系统类加载器，进行下面的逻辑处理
     // 先获取cls对应的类对象，（自定义的类加载器）
        获取构造方法，带一个classLoader.class类型的参数； 
    Constructor<?> ctor = Class.forName(cls, true, parent).getDeclaredConstructor(new Class<?>[] { ClassLoader.class });
    通过 构造函数创建应用类加载器，把系统类加载器作为其双亲；
    创建用户自定义的类加载器；
    ClassLoader sys = (ClassLoader) ctor.newInstance(
        new Object[] { parent });
        // 设置线程上下文类加载器为自定义类加载
    Thread.currentThread().setContextClassLoader(sys);
    返回自定义类加载器；
    return sys;
} 
最后经过异常处理，返回sclSet = true; 系统类加载器设置成功的标志；
```
---
  本文版权归作者本人拥有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。