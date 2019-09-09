---
layout: post
title:  "Jvm系列-深入理解类加载过程"
date:   2019-09-09 20:06:06
categories: JVM
tags: JVM 类加载
---

[TOC]
## 类加载器主要流程：

加载>连接（校验，准备，解析）>初始化>使用>卸载



## 类的使用方式：

- ​          主动使用

- ​           被动使用

所有的java 虚拟机实现必须是在Java程序首次主动使用类、接口的时候才初始化它们；

换句话说被动使用就不会初始化

首次初始化：也就是只会初始化一次；

## 什么情况是主动使用：

1.    创建类的实例
2.    访问某个类或者接口的静态变量，或者对这个静态变量赋值（取值赋值）
3. ​    调用类的静态方法：  在字节码层面 >助记符   getstatic putstatic invokestatic
4. ​    反射，通过全包类名获取类的对象；
5. ​     初始一个类的子类。如果Child extends Parent 当初始化Child 的时候Parent也会被初始化；
6. ​     Java启动的启动类，带有main函数的类；
7. ​     在jvm 上使用动态语言；







##  被动使用：

​    除了以上七种，都不会进行类的初始化，但是可以进行加载连接...

​    **类的加载:**把类class的二进制文件加载内存中，放到运行时数据区的方法区，在内存中创建java.lang.Class对象，用来封装类的Class对象，唯一只有一份。

可以看成一面镜子，可以反映出类的所有内容。是用来描述这个class对象的数据结构。

   **加载后放在哪？** jvm规范都没有规定放置的位置，hotspot是放在方法区；

   **类加载来源：**类的class文件可以从各种途径去加载，本地系统，还可以通过网络下载，jar，zip，从专有数据库中获取。。。

​    还有一种从Java源文件中动态的编译class ，动态代理。。。在web开发中jsp页面转化成servlet对应的class文件。

​      jvm并没有规定具体来源。

## 举例：主动使用/被动使用：

### 案例一

```java
package jvm.classloader;

import org.junit.Test;

/**
 * @author: wyj
 * @date: 2019/9/9
 * @description:
 */
public class ClassLoaderTest1 {
     @Test
    public void test001() {
        /**
         * 子类主动get父类静态属性
         */
        System.out.println(MyChild1.str);
       
    }

    @Test
    public void test002() {
        System.out.println(MyChild1.str2);  
    }

}

 class  MyParent1{
    //静态成员变量
     public static String str = "hello world";

     static {
         System.out.println("MyParent1 静态代码块执行");
     }
 }
 class MyChild1 extends  MyParent1{
     //静态成员变量
     public static String str2 = "hello world child";
     static {
         System.out.println("MyChild1 静态代码块执行");
     }
 }
```

**结果分析：**

test001：

```
MyParent1 静态代码块执行
hello world
```

> 子类主动调用父类静态属性，父类执行初始化，并打印属性信息；

那么子类有没有进行初始化？可以通过打印类加载过程信息查看；



> ```html
> [Loaded jvm.classloader.MyParent1 from file:/F:/gitPro/java_study/out/production/java_study/] 
> [Loaded jvm.classloader.MyChild1 from file:/F:/gitPro/java_study/out/production/java_study/]  MyParent1 静态代码块执行    hello world .......**  
> 可以看出即使MyChild1没有初始化也是先加载了的
> ```

test002：

```
MyParent1 静态代码块执行
MyChild1 静态代码块执行
hello world child
```

> 子类调用自己的静态属性，子类执行初始化，并且在初始化之前初始化所有的父类；

### 案例二

```java
package jvm.classloader;

import org.junit.Test;

/**
 * @author: wyj
 * @date: 2019/9/9
 * @description:
 */
public class ClassLoaderTest2 {

    @Test
    public void test001(){
       System.out.println(MyParent2.str);
    }
}

class MyParent2{
    //静态成员变量
    public final static String str = "hello world";

    static {
        System.out.println("MyParent2 静态代码块执行");
    }


}
```

**结果分析：**

  只打印出   hello world，在引用了父类的静态常量之后，只打印了属性信息，并没有父类初始化。

为什么？

> final :本身表示常量，编译器也知道，所以在编译阶段这个常量就会被存入到调用这个方法所在类的常量池当中；   也就是 "hello world" 在编译阶段就在ClassLoaderTest2中的类的常量池中，本质上并没有引用到定义常量的这个类，因此也不会触发这个类的初始化；那么静态代码块也就不会执行；*注意： 在这个字符串在放入到ClassLoaderTest2的常量池中之后就跟MyParent2没有任何关系了，甚至可以删除编译后的MyParent2的字节码文件（在out路径下删除字节码文件依然可以运行）

### 案例三

```java
package jvm.classloader;

import org.junit.Test;

import java.util.UUID;

/**
 * @author: wyj
 * @date: 2019/9/9
 * @description:
 */
public class ClassLoaderTest3 {

    @Test
    public void test001() {
        System.out.println(MyParent3.str);
    }
}

class MyParent3{
    public final static String str = UUID.randomUUID().toString();
    static {
        System.out.println("MyParent3 静态代码块执行");
    }

}
```

结果分析：

```html
MyParent3 静态代码块执行
b2a4e712-4df1-4bfd-a667-6d13ead5eaba
```

 跟案例二的区别在于静态变量的值是在运行期间才能确定；

**静态成员变量,编译阶段是无法知道str的值的，怎么办？**

那就只能在运行期间进行类初始化并执行方法；

> 如果像上面把MyParent3编译后的类字节码删除会抛出classNotFound 异常在一个常量在编译期间不能确定的话就不会放在调用类的常量池中,从class文件的对比情况也可以发现
---
  本文版权归作者本人拥有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。
