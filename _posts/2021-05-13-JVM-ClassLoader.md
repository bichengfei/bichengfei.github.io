---
title: JVM 类加载机制
author: BiChengfei
date: 2021-05-13 15:53:00 +0800
categories: [JAVA]
tags: [JAVA, JVM]
pin: true
---

参考：  

1. [《深入理解java虚拟机》 --周志明](https://baike.baidu.com/item/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3Java%E8%99%9A%E6%8B%9F%E6%9C%BA/10749828?fr=aladdin)

### 思考几个小问题

1. 这三种都是在什么时候被初始化和赋值，在类加载中有什么不同？
   
   ```
   final static int a = 1;
   static int a = 1;
   int a = 1;
   ```

2. 现在我们有一个项目--Hello，打成jar包，成功部署并正常运行，有一天我们要修改Test01.java(不包含内部类)，打算修改完编译后只把Hello.jar中的Test01.class进行替换，运行，这样是否可以？
   
   + Test01.java中有`final`修饰的常量是否可以？
   + Test01.java中不包含`final`修饰的常量是否可以？

3. 给出下面输出:

```
public class Parent {  
    public static String value = "123";  

    static {  
        System.out.println("parent");  
    }  
}  
```

```
public class Sub extends Parent {  

    static {  
        System.out.println("sub");  
    }  
}  
```

```
public class Main {  

    public static void main(String[] args) {  
        System.out.println(Sub.value);  
    }  
}  
```

Sub对象初始化了吗？Parent对象初始化了吗？

4.`new Sub()`时，`base`,`parent`,`sub`，哪些会被赋值

`Sub sub = new Sub();sub.base;`时，`base`,`parent`,`sub`，哪些会被赋值

```
public interface Base {
    static int base = 1;
}
```

```
public class Parent implements Base {
    static int parent = 2;
}
```

```
public class Sub extends Parent {
    static int sub = 3;
}
```

## 概述

虚拟机把描述类的数据从Class文件加载到内存，并对数据进行检验、转换解析和初始化，最终形成可以被虚拟机直接使用的Java类型，这就是虚拟机的类加载机制。

1. 虚拟机是如何加载Class文件的？  
2. Class文件中的信息进入到虚拟机后发生什么变化？

## 类加载的时机

![图-01](/assets/img/blogs/jvm/classLoader/lifecycle.jpg 'Class文件在JVM内存中的生命周期')类从加载到内存（虚拟机内存）开始，到卸载出内存为止，生命周期包括上面七个阶段，验证、准备、解析，统称为连接。类加载的过程也只包括了前5步。  
第一阶段也就是加载发生的时间虚拟机规范没有要求，不同实现不同，连接、使用、卸载，这三个没有分析必要，一般对类加载实际的分析集中在"初始化"上。

## 类加载的过程

### 加载

加载过程主要完成三件事情：

1. 通过类的全限定名来获取定义此类的二进制字节流
2. 将这个类字节流代表的静态存储结构转为方法区的运行时数据结构
3. 在堆中生成一个代表此类的java.lang.Class对象，作为访问方法区这些数据结构的入口

### 连接

#### 验证

此阶段主要确保Class文件的字节流中包含的信息符合当前虚拟机的要求，并且不会危害虚拟机的自身安全

1. 文件格式验证：基于字节流验证
2. 元数据验证：基于方法区的存储结构验证
3. 字节码验证：基于方法区的存储结构验证
4. 符号引用验证：基于方法区的存储结构验证

#### 准备

为类变量分配内存，并将其初始化为默认值  
例如：  
`static int a = 5;`  
准备阶段会为类变量a分配内存，并赋值为0

#### 解析

把类型中的符号引用转换为直接引用。

符号引用与虚拟机实现的布局无关，引用的目标并不一定要已经加载到内存中。各种虚拟机实现的内存布局可以各不相同，但是它们能接受的符号引用必须是一致的，因为符号引用的字面量形式明确定义在Java虚拟机规范的Class文件格式中。  

直接引用可以是指向目标的指针，相对偏移量或是一个能间接定位到目标的句柄。如果有了直接引用，那引用的目标必定已经在内存中存在

主要有以下四种：

1. 类或接口的解析
2. 字段解析
3. 类方法解析
4. 接口方法解析

### 初始化

虚拟机规范严格规定有且只有5种情况必须对类进行“初始化”

1. 遇到new、getstatic、ptstatic、invokestatic 4条字节码指令时
   + 使用new
   + 读取或设置一个类的静态字段
   + 调用一个类的静态方法
2. 反射调用
3. 初始化一个类时，当父类未被初始化，需先初始化父类
4. 虚拟机启动，主类初始化（main()方法）
5. 当使用JDK 1.7的动态语言支持时，如果一个java.lang.invoke.MethodHandle实例最后的解析结果REF_getStatic、REF_putStatic、REF_invokeStatic的方法句柄，并且这个方法句柄所对应的类没有过初始化，则需要先触发其初始化。

### 使用和卸载

## 类加载器

JVM系统自带的类加载器有三种：

1. 启动类加载器(Bootstrap ClassLoader)
2. 扩展类加载器(Extension ClassLoader)
3. 应用程序类加载器(Application ClassLoader)

### 双亲委派机制

![JVM类加载器模型](/assets/img/blogs/jvm/classLoader/ClassLoader.png 'JVM类加载器模型')

双亲委派机制工作过程：

如果一个类加载器收到了类加载的请求.它首先不会自己去尝试加载这个类.而是把这个请求委派给父加载器去完成.每个层次的类加载器都是如此.因此所有的加载请求最终都会传送到Bootstrap类加载器(启动类加载器)中.只有父类加载反馈自己无法加载这个请求(它的搜索范围中没有找到所需的类)时.子加载器才会尝试自己去加载。

作用：  

+ 可以避免重复加载，父类已经加载了，子类就不需要再次加载 
+ 更加安全，很好的解决了各个类加载器的基础类的统一问题，如果不使用该种方式，那么用户可以随意定义类加载器来加载核心api，会带来相关隐患。