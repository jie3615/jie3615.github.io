---
layout: post
title:  "Jvm系列-深入理解字节码文件-常量池之后（二）"
date:   2019-10-11 20:06:06
categories: Jvm
tags: 字节码 Jvm
---

[TOC]
**常量池之后，接上篇分析**

## 类相关的字节码分析






类的访问标志符号，0021 ，对比如下访问标志对应关系表；

| 标记名         | 值     | 含义                                                |
| -------------- | ------ | --------------------------------------------------- |
| ACC_PUBLIC     | 0x0001 | 可以被包的类外访问。                                |
| ACC_FINAL      | 0x0010 | 不允许有子类。                                      |
| ACC_SUPER      | 0x0020 | 当用到invokespecial指令时，需要特殊处理的父类方法。 |
| ACC_INTERFACE  | 0x0200 | 标识定义的是接口而不是类。                          |
| ACC_ABSTRACT   | 0x0400 | 不能被实例化。                                      |
| ACC_SYNTHETIC  | 0x1000 | 标识并非Java源码生成的代码。                        |
| ACC_ANNOTATION | 0x2000 | 标识注解类型                                        |
| ACC_ENUM       | 0x4000 | 标识枚举类型                                        |

找不到直接对应的访问标志，实际上是通过0x0001和0x0020并集得到的；

类的名字：00 03 常量池中 // jvm/bytecode/ByteCodeTest1

父类的名字：00 04  常量池中 // java/lang/Object

1. 接口数量：00 00 没有；
2. 成员变量的数量： 00 01

变量表：字段表结构

```json
filedinfo {
             u2 access_flags:00 02 private
             u2 name_index;00 05 常量池中 a
             u2 descripor_index;00 06 常量池中 I
             u2 attributes_count; 00 00 没有属性
             attribute_info attributes[attributes_count]; 无
         }
```

下面表格字段访问属性取值范围；

| 标记名        | 值     | 说明                                |
| ------------- | ------ | ----------------------------------- |
| ACC_PUBLIC    | 0x0001 | public，表示字段可以从任何包访问。  |
| ACC_PRIVATE   | 0x0002 | private，表示字段仅能该类自身调用。 |
| ACC_PROTECTED | 0x0004 | protected，表示字段可以被子类调用。 |
| ACC_STATIC    | 0x0008 | static，表示静态字段。              |
| ACC_FINAL     | 0x0010 | final，表示字段定义后值无法修改。   |
| ACC_VOLATILE  | 0x0040 | volatile，表示字段是易变的。        |
| ACC_TRANSIENT | 0x0080 | transient，表示字段不会被序列化。   |
| ACC_SYNTHETIC | 0x1000 | 表示字段由编译器自动产生。          |
| ACC_ENUM      | 0x4000 | enum，表示字段为枚举类型。          |

## 方法相关字节码分析

方法数量：00 03 三个方法，如下：

---
layout: post
title:  "Jvm系列-深入理解字节码文件-常量池之后（二）"
date:   2019-10-11 20:06:06
categories: Jvm
tags: 字节码 Jvm
---

[TOC]
**常量池之后，接上篇分析**

## 类相关的字节码分析






类的访问标志符号，0021 ，对比如下访问标志对应关系表；

| 标记名         | 值     | 含义                                                |
| -------------- | ------ | --------------------------------------------------- |
| ACC_PUBLIC     | 0x0001 | 可以被包的类外访问。                                |
| ACC_FINAL      | 0x0010 | 不允许有子类。                                      |
| ACC_SUPER      | 0x0020 | 当用到invokespecial指令时，需要特殊处理的父类方法。 |
| ACC_INTERFACE  | 0x0200 | 标识定义的是接口而不是类。                          |
| ACC_ABSTRACT   | 0x0400 | 不能被实例化。                                      |
| ACC_SYNTHETIC  | 0x1000 | 标识并非Java源码生成的代码。                        |
| ACC_ANNOTATION | 0x2000 | 标识注解类型                                        |
| ACC_ENUM       | 0x4000 | 标识枚举类型                                        |

找不到直接对应的访问标志，实际上是通过0x0001和0x0020并集得到的；

类的名字：00 03 常量池中 // jvm/bytecode/ByteCodeTest1

父类的名字：00 04  常量池中 // java/lang/Object

1. 接口数量：00 00 没有；
2. 成员变量的数量： 00 01

变量表：字段表结构

```json
filedinfo {
             u2 access_flags:00 02 private
             u2 name_index;00 05 常量池中 a
             u2 descripor_index;00 06 常量池中 I
             u2 attributes_count; 00 00 没有属性
             attribute_info attributes[attributes_count]; 无
         }
```

下面表格字段访问属性取值范围；

| 标记名        | 值     | 说明                                |
| ------------- | ------ | ----------------------------------- |
| ACC_PUBLIC    | 0x0001 | public，表示字段可以从任何包访问。  |
| ACC_PRIVATE   | 0x0002 | private，表示字段仅能该类自身调用。 |
| ACC_PROTECTED | 0x0004 | protected，表示字段可以被子类调用。 |
| ACC_STATIC    | 0x0008 | static，表示静态字段。              |
| ACC_FINAL     | 0x0010 | final，表示字段定义后值无法修改。   |
| ACC_VOLATILE  | 0x0040 | volatile，表示字段是易变的。        |
| ACC_TRANSIENT | 0x0080 | transient，表示字段不会被序列化。   |
| ACC_SYNTHETIC | 0x1000 | 表示字段由编译器自动产生。          |
| ACC_ENUM      | 0x4000 | enum，表示字段为枚举类型。          |

## 方法相关字节码分析

方法数量：00 03 三个方法，如下：


```
<init>

setA()

getA()
```


> 方法属性结构<init>

```json
method_info { 
    u2 access_flags; 00 01 方法是public 的
    u2 name_index; 00 07 名字索引类型，常量池中  <init>         
    u2 descriptor_index;00 08 描述符索引类型  常量池中 ()V 表示不带参数，返回值是void类型的方法
    u2 attributes_count; 00 01 属性的个数为1，根据代码计算出来的
    attribute_info attributes[attributes_count]; 
}
```

下面表格方法访问属性access_flags取值范围

| 标记名           | 值     | 说明                                 |
| ---------------- | ------ | ------------------------------------ |
| ACC_PUBLIC       | 0x0001 | public，方法可以从包外访问           |
| ACC_PRIVATE      | 0x0002 | private，方法只能本类中访问          |
| ACC_PROTECTED    | 0x0004 | protected，方法在自身和子类可以访问  |
| ACC_STATIC       | 0x0008 | static，静态方法                     |
| ACC_FINAL        | 0x0010 | final，方法不能被重写（覆盖）        |
| ACC_SYNCHRONIZED | 0x0020 | synchronized，方法由管程同步         |
| ACC_BRIDGE       | 0x0040 | bridge，方法由编译器产生             |
| ACC_VARARGS      | 0x0080 | 表示方法带有变长参数                 |
| ACC_NATIVE       | 0x0100 | native，方法引用非java语言的本地方法 |
| ACC_ABSTRACT     | 0x0400 | abstract，方法没有具体实现           |
| ACC_STRICT       | 0x0800 | strictfp，方法使用FP-strict浮点格式  |
| ACC_SYNTHETIC    | 0x1000 | 方法在源文件中不出现，由编译器产生   |

下面是分析attribute_info 的数据结构代表方法的属性的信息

```json
attribute_info {
    u2 attribute_name_index; 00 09 常量池中 Code，就是代表了这个方法的执行代码
    u4 attribute_length; 00 00 00 38 十进制56 ，会占用56个字节长度的空间
    u1 info[attribute_length];56个长度
}
```

上面这个属性表是个通用的结构，在ClassFile、field_info、method_info、code_attribute中都有使用；通过属性名区分，下图代表着各种场景的对应关系；

| 属性名                               | Java SE | Class文件 |
| ------------------------------------ | ------- | --------- |
| ConstantValue                        | 1.0.2   | 45.3      |
| Code                                 | 1.0.2   | 45.3      |
| StackMapTable                        | 6       | 50.0      |
| Exceptions                           | 1.0.2   | 45.3      |
| InnerClasses                         | 1.1     | 45.3      |
| EnclosingMethod                      | 5.0     | 49.0      |
| Synthetic                            | 1.1     | 45.3      |
| Signature                            | 5.0     | 49.0      |
| SourceFile                           | 1.0.2   | 45.3      |
| SourceDebugExtension                 | 5.0     | 49.0      |
| LineNumberTable                      | 1.0.2   | 45.3      |
| LocalVariableTable                   | 1.0.2   | 45.3      |
| LocalVariableTypeTable               | 5.0     | 49.0      |
| Deprecated                           | 1.1     | 45.3      |
| RuntimeVisibleAnnotations            | 5.0     | 49.0      |
| RuntimeInvisibleAnnotations          | 5.0     | 49.0      |
| RuntimeVisibleParameterAnnotations   | 5.0     | 49.0      |
| RuntimeInvisibleParameterAnnotations | 5.0     | 49.0      |
| AnnotationDefault                    | 5.0     | 49.0      |
| BootstrapMethods                     | 7       | 51.0      |

接着，如果是code属性，那么存的是code_attribute类型的数据结构

下面是code的数据结构：保存该方法的结构

```Java
Code_attribute {
    u2 attribute_name_index; 00 09 常量池中 Code，就是代表了这个方法的执行代码
    u4 attribute_length; 00 00 00 38 十进制56 ，会占用56个字节长度的空间
                        表示attribute包含的字节数，不包含attribute_name_index、attribute_length
    u2 max_stack;00 02
    u2 max_locals;00 01
    u4 code_length; 00 00 00 0a >10 该方法包含的字节码的字节数，具体的指令10个
    u1 code[code_length]; 2a b7 00 01 2a 04 b5 00 02 b1  ：16进制的表示形式表示<init>方法的执行指令
                          对应助记符
                           2a: 0 aload_0
                           b7: 1 invokespecial #1 <java/lang/Object.<init>> 父类的构造方法（00 01 代表了这个指令的参数，在常量池中寻找）
                           2a: 4 aload_0
                           04: 5 iconst_1
                           b5: 6 putfield #2 <jvm/bytecode/ByteCodeTest1.a>：带参数的指令，后面两个字节 00 02 常量池中
                           b1: 9 return ：返回

    u2 exception_table_length; 00 00 没有异常抛出
    {   u2 start_pc;
        u2 end_pc; 
        u2 handler_pc; 
        u2 catch_type; 
    } exception_table[exception_table_length]; 
    u2 attributes_count; 00 02 两个属性
    attribute_info attributes[attributes_count];
      第一个u2 00 0a 常量池中#10 LineNumberTable 
              00 00 00 0a属性长度10个字节；               
              00 02 表示有两对映射；
              00 00 > 00 08
              00 04 > 00 09
      第二个u2 00 0b 常量池中#11 LocalVariableTable  
            00 00 00 0c :12个字节代表局部不变量的信息
            00 01 局部变量的个数：1
            00 00 局部变量的开始位置0
            00 0a ....结束位置
            00 0c 局部变量的索引 #12 = Utf8               this
            00 0d 描述#13 = Utf8               Ljvm/bytecode/ByteCodeTest1;
            00 00 校验检查
                 
}
```

方法的属性结构 u1 info  code的结构

 LineNumberTable 的属性结构如下

```json
 LineNumberTable_attribute {

                u2 attribute_name_index;
                u4 attribute_length; 
                u2 line_number_table_length; 
                { 
                    u2 start_pc; 
                    u2 line_number; 
                } line_number_table[line_number_table_length]; 
            }
```

接下来就是第二个方法的结构：方法属性结构getA()

```json
method_info { 
    u2 access_flags; 00 01 方法是public 的
    u2 name_index; 00 0e 名字索引类型，常量池中  getA       
    u2 descriptor_index;00 0f 描述符索引类型  常量池中 ()I 
    u2 attributes_count; 00 01 属性的个数为1，根据代码计算出来的
    attribute_info attributes[attributes_count]; 
}
```

接下来是Code属性的结构

```json
Code_attribute {
    u2 attribute_name_index; 00 09 常量池中 Code，就是代表了这个方法的执行代码
    u4 attribute_length; 00 00 00 2f 十进制47 ，会占用47个字节长度的空间
                        表示attribute包含的字节数，不包含attribute_name_index、attribute_length
    u2 max_stack;00 01
    u2 max_locals;00 01
    u4 code_length; 00 00 00 05 >5 该方法包含的字节码的字节数，具体的指令5个
    u1 code[code_length]; 2a b4 00 02 ac  ：16进制的表示形式表示getA方法的执行指令
                          对应助记符
                         2a: 0 aload_0
                         b4: 1 getfield #2 <jvm/bytecode/ByteCodeTest1.a> 参数：00 02:常量池中
                         ac: 4 ireturn

    u2 exception_table_length; 00 00 没有异常抛出
    {   u2 start_pc;
        u2 end_pc; 
        u2 handler_pc; 
        u2 catch_type; 
    } exception_table[exception_table_length]; 
    u2 attributes_count; 00 02 两个属性
    attribute_info attributes[attributes_count];
      第一个u2 00 0a 常量池中#10 LineNumberTable 
              00 00 00 06 属性长度6；
              00 01 表示有一对映射；
              00 00 > 00 0c :表示0对应12；
              
      第二个u2 00 0b 常量池中#11 LocalVariableTable  
            00 00 00 0c :12个字节代表局部不变量的信息
            00 01 局部变量的个数：1
            00 00 局部变量的开始位置0
            00 05 ....结束位置
            00 0c 局部变量的索引 #12 = Utf8               this
            00 0d 描述#13 = Utf8               Ljvm/bytecode/ByteCodeTest1;
            00 00 
                 
}
```

代码最终都会形成指令保存在方法表中

接下来就是第三个方法的结构：方法属性结构SetA，分析方式同上

```json
method_info { 
    u2 access_flags; 00 01 方法是public 的
    u2 name_index; 00 0e 名字索引类型，常量池中  getA       
    u2 descriptor_index;00 0f 描述符索引类型  常量池中 ()I 
    u2 attributes_count; 00 01 属性的个数为1，根据代码计算出来的
    attribute_info attributes[attributes_count]; 
}......省略
```

最后是源文件信息对应常量池中的#19

  ---
  本文版权归作者本人拥有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。