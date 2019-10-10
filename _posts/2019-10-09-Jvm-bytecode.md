---
layout: post
title:  "Jvm系列-深入理解字节码文件（一）"
date:   2019-10-09 20:06:06
categories: 并发
tags: 字节码 Jvm
---

* content
{:toc}
## 源码

```Java
package jvm.bytecode;

/**
 * @author: wyj
 * @date: 2019/9/30
 * @description:
 */
public class ByteCodeTest1 {
    private int a = 1;

    public int getA() {
        return a;
    }

    public void setA(int a) {
        this.a = a;
    }
}
```



## 字节码详细信息

Javap -verbose xxx(class文件)

```
public class jvm.bytecode.ByteCodeTest1
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #4.#20         // java/lang/Object."<init>":()V
   #2 = Fieldref           #3.#21         // jvm/bytecode/ByteCodeTest1.a:I
   #3 = Class              #22            // jvm/bytecode/ByteCodeTest1
   #4 = Class              #23            // java/lang/Object
   #5 = Utf8               a
   #6 = Utf8               I
   #7 = Utf8               <init>
   #8 = Utf8               ()V
   #9 = Utf8               Code
  #10 = Utf8               LineNumberTable
  #11 = Utf8               LocalVariableTable
  #12 = Utf8               this
  #13 = Utf8               Ljvm/bytecode/ByteCodeTest1;
  #14 = Utf8               getA
  #15 = Utf8               ()I
  #16 = Utf8               setA
  #17 = Utf8               (I)V
  #18 = Utf8               SourceFile
  #19 = Utf8               ByteCodeTest1.java
  #20 = NameAndType        #7:#8          // "<init>":()V
  #21 = NameAndType        #5:#6          // a:I
  #22 = Utf8               jvm/bytecode/ByteCodeTest1
  #23 = Utf8               java/lang/Object
{
  public jvm.bytecode.ByteCodeTest1();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: aload_0
         5: iconst_1
         6: putfield      #2                  // Field a:I
         9: return
      LineNumberTable:
        line 8: 0
        line 9: 4
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      10     0  this   Ljvm/bytecode/ByteCodeTest1;

  public int getA();
    descriptor: ()I
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: getfield      #2                  // Field a:I
         4: ireturn
      LineNumberTable:
        line 12: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Ljvm/bytecode/ByteCodeTest1;

  public void setA(int);
    descriptor: (I)V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=2, args_size=2
         0: aload_0
         1: iload_1
         2: putfield      #2                  // Field a:I
         5: return
      LineNumberTable:
        line 16: 0
        line 17: 5
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       6     0  this   Ljvm/bytecode/ByteCodeTest1;
            0       6     1     a   I
}
SourceFile: "ByteCodeTest1.java"
```


## 二进制文件

![image](https://raw.githubusercontent.com/jie3615/blogImages/master/images/bytecode1.png)



字节码说明：

1. 使用Javap -verbose分析一个字节码文件时，会分析字节码的 魔数，版本号，常量池，类信息，类的构造方法，普通方法，类变量，成员变量等；

2. 所有的.class文件的前四个字节都是魔数值，固定为0xcafebabe；

3. 魔数之后的是版本信息：前两个字节是次版本号，后两个字节是主版本号，这个文件上0000 0034 换算成十进制是52；

4. 接下来是常量池，由于常量池长度是不固定的，所以需要描述常量池的长度，什么是常量池，一个Java类定义的很多信息都是由常量池来描述的；可以将常量池比作类的资源仓库，eg:方法，变量信息；常量池主要存储两种常量，一个是字面量，一个是符号引用；字面量：比如文本字符串，Java中声明为final的常量值等，而符号引用：类和接口的全局限定名，字段的名称和描述符，方法的名称和描述符；

5. 常量池中的值不是不变的值，变量的定义也存在常量池中；

6. 常量池的总体结构：Java类所对应的常量池由两部分构成，常量池数量和常量池数组构成，常量池数量紧跟主版本号由两个字节构成，常量池数组紧跟常量池数量之后；

7. 常量池数组和一般的数组不同的是：常量池数组中不同的元素的类型，结构都是不一样的；所以每个元素的占据的空间都不一样；但是每一种元素的第一个数据都是一个u1类型，该字节是一个标志位，占一个字节，jvm 在解析常量池时，根据这个u1类型获取元素的具体类型；

8. 注意：常量池数组中元素的个数= 常量池数量-1（0暂时不使用）；对应上面案例就是24-1=23；为什么减掉1?目的是为了满足某些常量池索引值的数据在特定的情况下要表达【不引用任何一个常量池】的含义，根本原因在于索引为0也是常理，是JVM的保留常量，不存在常量表中，这个常量对应NULL，所以常量池中的索引的开始为1；

9. 每个变量字段都有描述信息：作用：描述字段的数据类型，参数列表，返回值。。。

10. 为了压缩字节码的体积，基本类型和无返回值都是用一个大写字母表示，其他类型使用L全限定类名；

11. 常量池之后：类的访问标志u2->类的名字u2->父类名字u2->接口的个数u2->接口名u2->成员变量的个数u2->成员变量表->方法的个数u2->方法表->附加属性的个数u2-附加属性的表；

    **未完待续。。。**
    
    ---
  本文版权归作者本人拥有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。