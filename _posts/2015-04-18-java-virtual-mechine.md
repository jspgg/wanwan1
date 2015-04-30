---
layout:     post
title:      Java虚拟机简介
keywords:   java
category:   java 
tags:	[javas, 虚拟机]
---

* 目录
{:toc}

Java虚拟机是整个Java平台的基础，是Java语言用于实现与硬件、操作系统无关的关键，Java虚拟机类似一个微型的计算机，它有自己的指令集和运行时的内存区域。java虚拟机和java语言并没有必然的联系，它只与特定的二进制文件--class文件相关联，只要符合java虚拟机规范的class文件都能在虚拟机上运行。

下面讲到虚拟机的特性时，都只限于 SE 7，因为前不久出了SE8，虚拟机的有些特性发生了改变，以后再来阐述。
这个系列将分为几个部分来阐述：

1. Java虚拟机结构简介
2. Java虚拟机编译器
3. Class文件格式
4. 加载、链接和初始化
5. 虚拟机指令集
6. SE 8 的新特性
 
#Java虚拟机结构简介

##Class文件格式

编译后Java虚拟机采用了一种与平台无关的二进制格式来表示，为了确保class文件能在不同的平台上运行，所以虚拟机也约定了一些惯例，比如字节序。具体的请参考第三部分。

##数据类型

与Java语言的数据类型相似，Java虚拟机可以操作的数据类型分为2类：原生类型(primitive type)和引用类型，可用于变量复制、参数传递、方法返回和运算操作。

Java虚拟机希望在程序运行之前尽可能多的进行类型检查，使得虚拟机在运行期间无需进行这些操作。

**原生类型**

整数类型：


* byte类型：8位有符号整数，默认为0
* char类型：16位**无符号**整数，默认为Unicode的null（'\u0000'）
* short类型：16位有符号整数，默认为0
* int类型：32位有符号整数，默认为0
* long类型：64位有符号整数，默认为0

浮点数类型：

* float单精度浮点数：默认值为正数0
* double双精度浮点数：默认为正数0

特殊的类型：

* boolean类型： 默认为false，虚拟机对boolean类型没有提供任何专用的字节码指令，在java语言中有关boolean类型的运算在编译之后都转换成int类型来代替，对于boolean类型的数组，虚拟机的newarray指令可以创建这种数组，在Oracle公司的虚拟机里，boolean数组会编译成byte数组，每个boolean元素占8位。
* 返回地址：作为一条字节码指令的操作数，这是唯一一个在虚拟机支持的类型当中不能直接与java语言的数据类型相对应的。

**引用类型**

Java虚拟机有三种引用类型：类类型、数组类型和接口类型，这些引用类型的值分别由类实例、数组实例和实现某种接口的类实例创建。

##运行时数据区

###PC程序计数器

每一个虚拟机线程都有自己的pc，在任何时刻，一个虚拟机线程只会执行一个方法的代码，这个方法称为该线程的当前方法，如果这个方法不是native的，pc寄存器就保存虚拟机正在执行的字节码指令的地址，如果方法是native的，那么pc寄存器的值是undefined,pc寄存器的容量至少应当保存一个returnAddress类型的数据或者一个与平台相关的本地指针的值。

###虚拟机栈

每一个虚拟机线程都有自己的虚拟机栈，这个栈与线程同时创建，用于存储栈帧(frame),虚拟机栈的作用就是用于存储一些局部变量和过程调用的返回结果，在方法调用和返回中起了重要的作用。栈容量只能由-Xss参数指定，由于Java虚拟机栈会出现StackOverflowError和OutOfMemoryError两种异常，所以分别使用两个例子演示这两种情况：

* java虚拟机栈深度溢出：

单线程的环境下，无论是由于栈帧太大，还是虚拟机栈容量太小，当内存无法再分配的时候，虚拟机总抛出StackOverflowError异常。使用-Xss128k将java虚拟机栈大小设置为128kb，例子代码如下：

    {% highlight java %}
    
    public class JavaVMStackOF{  
        private int stackLength = 1;  
        public void stackLeak(){  
            statckLength++;  
            stackLeak();  
    }  
    
    public static void main(String[] args){  
        JavaVMStackOF oom = new JavaVMStackOF();  
    oom.stackLeak();  
    }  
    
    } 
    
    {% endhighlight %} 
    
运行一段时间后，产生StackOverflowError异常。Java虚拟机栈溢出一般会产生在方法递归调用过多而java虚拟机栈内存不够的情况下。

* java虚拟机栈内存溢出：

多线程环境下，能够创建的线程最大内存=物理内存-最大堆内存-最大方法区内存，在java虚拟机栈内存一定的情况下，单个线程占用的内存越大，所能创建的线程数目越小，所以在多线程条件下很容易产生java虚拟机栈内存溢出的异常。

使用-Xss2m参数设置java虚拟机栈内存大小为2MB，例子代码如下：

    {% highlight java %}
    
    public class JavaVMStackOOM{  
        private void dontStop(){  
        while(true){  
    }  
    }  
    public void stackLeakByThread(){  
        while(true){  
            Thread t = new Thread(new Runnable(){  
        public void run(){  
        dontStop();  
    }  
    });  
    t.start();  
    }  
    }   
    public static void main(String[] args){  
        JavaVMStackOOM oom = new JavaVMStackOOM();  
        oom. stackLeakByThread();.  
    }  
    }  
    
    
    {% endhighlight %} 
 

运行一段时间之后，java虚拟机栈就会因为内存太小无法创建线程而产生OutOfMemoryError。

###Java堆

在java虚拟机中，堆是各个线程共享的运行时内存区域，所有的对象都是在堆中创建。Java堆在虚拟机启动的时候被创建，它存储了垃圾收集器来管理对象，这些受管理的对象无需也无法显式的销毁，你想一个对象尽快被销魂，只能通过把所有的对象引用设置为null，等内存不足的时候垃圾收集器会标记这个不再被引用的对象然后回收该对对象占有的内存。当对象数量达到堆最大容量时产生OutOfMemoryError异常。

想要方便快速地产生堆溢出，要使用如下java虚拟机参数：-Xms10m(最小堆内存为10MB)，-Xmx10m(最大堆内存为10MB，最小堆内存和最大堆内存相同是为了避免堆动态扩展)，-XX:+HeapDumpOnOutOfMemoryError可以让java虚拟机在出现内存溢出时产生当前堆内存快照以便进行异常分析。

例子代码如下：

  {% highlight java %}
  
    public class HeapOOM{  
        static class OOMObject{  
    }  
    public static void main(String[] args){  
        List<OOMObject> list = new ArrayList<OOMObject>();  
        while(true){  
        list.add(new OOMObject());  
    }  
    }  
    }  
    
    {% endhighlight %} 
    
###方法区&&运行时常量池

方法区也是供各个线程共享的内存区域，用于存储加载的类的结构信息，比如：运行时常量池、字段、方法数据等。方法区在虚拟机启动时创建，简单的虚拟机实现可以选择在这个区域不进行垃圾收集。运行时常量池用于保存加载的class文件的数字字面量和符号引用，在加载类和接口到虚拟机之后，就创建对应的运行时常量池。可以使用-XX:PermSize=10m和-XX:MaxPermSize=10m将永久代最大内存和最小内存设置为10MB大小，并且由于永久代最大内存和最小内存大小相同，因此无法扩展。

String的intern()方法用于检查常量池中如果有等于此String对象的字符串存在，则直接返回常量池中的字符串对象，否则，将此String对象所包含的字符串添加到运行时常量池中，并返回此String对象的引用。因此String的intern()方法特别适合演示运行时常量池溢出，例子代码如下：

    {% highlight java %}

    public class RuntimeConstantPoolOOM{  
        public static void main(String[] args){  
    List<String> list = new ArrayList<String>();  
            int i = 0;  
            while(true){  
            list.add(String.valueOf(i++).intern());  
    }  
    }  
    }  
    
     {% endhighlight %} 
     
运行一段时间，永久代内存不够，运行时常量池因无法再添加常量而产生OutOfMemoryError。

##栈帧

栈帧是用来存储数据和部分过程调用结果的数据结构，有时也会用来处理动态链接、方法返回值和异常分派。

栈帧随着方法调用而创建，随着方法结束而销毁--无论是正常结束还是异常完成都算方法结束，每一个栈帧都有自己的局部变量表和操作数栈(operand stack)和**指向当前方法所属类的运行时常量池的引用**。

局部变量表和操作数栈的容量在编译器确定，保存在方法的code属性提供给栈帧使用，在给定的一个线程中，只有目前那个正在执行的方法的栈帧是活动的，这个栈帧称为当前栈帧，这个栈帧对应的方法称为当前方法，定义这个方法的类称为当前类。对局部变量表和操作数栈的各种操作，都是指对当前栈帧的局部变量表和操作数栈进行的操作。

如果当前方法调用了其他方法时，一个新的栈帧随之创建，随着程序的控制权转移交到新的方法而成为新的当前栈帧，当方法返回之时，当前栈帧把执行结果返回给前一个栈帧，随之丢弃当前栈帧，前一个栈帧重新成为当前栈帧。

###局部变量表

一个局部变量可以保存一个类型为boolean、byte、char、short、int、float、refrence或返回地址的数据，两个局部变量可以保存一个类型为long或者double的数据。

局部变量表使用索引来定位访问，long和double占用两个连续的局部变量，这两种类型的数据使用两个局部变量中较小的索引来访问，Java虚拟机使用局部变量表来完成方法调用时的参数传递，当调用一个方法时，他的参数会传递至从0开始的连续的局部变量表的位置上。当调用的是实例方法时，第0个局部变量一定是用来存储被调用方法所在对象的引用(java语言的this关键字)。

###操作数栈

每个栈帧内部都包含一个称为操作数栈的先进后出的结构，同样操作数栈的长度由编译器决定，并且通过方法的code属性保存及提供给栈帧使用。
栈帧在刚刚创建的时候，操作数栈是空的，java虚拟机提供一些字节码指令来从局部变量表或者对象的字段中复制常量或者变量值到操作数栈中，也提供一些指令用于从操作数栈中取走数据、操作数据以及把操作结果重新入栈，每个操作数栈的位置报以保存一个Java虚拟机定义的任何数据类型的值，包括long和double。在操作数栈中的数据必须正确操作，不可以入栈两个int类型的数据然后当成long类型区操作，也不能入栈两个float类型的数据然后使用iadd指令对他们求和。

###动态链接

每个栈帧内部都包含一个指向当前方法所属的类的运行时常量池的引用，动态链接的作用就是将这些符号引用所表示的方法转换成实际方法的引用，类加载的过程将变量访问转换为这些变量的存储结构所在的运行时内存的位置的正确偏移量。

###方法正常调用完成

方法正常调用完成是指在方法的执行过程中，没有抛出任何异常，如果当前方法正常完成，它很可能会返回一个值给他的调用者，使用哪一个返回指令取决于方法返回值的数据类型。在这样的情况下，当前栈帧(被调用者)承担着回复调用者状态的责任，其状态包括调用者的局部变量表，操作数栈以及PC，使得调用者的代码在被调用者返回后能够继续执行。

##对象的表示

Java虚拟机并不强制规定对象的内部结构应该如何表示。在Oracle的某些虚拟机实现中，指向对象的引用实际上一个指向句柄(Handler)的指针，这个句柄包含两部分，一部分是指向在堆中分配的对象数据，另一部分是指向常量池中该对象所属类的相关信息。

##字节码指令

大部分的指令都没有支持整数类型byte、char和short，甚至没有任何指令支持boolean类型，编译器会在编译器或者运行期把byte和short类型的数据进行符号扩展成int类型数据，把boolean和char类型的数据进行零位扩展成int类型数据。

Java虚拟机支持数值类型之间进行相互转换，包括宽化类型转换和窄化类型转换，这里的宽和窄是指该类型能表示的数值范围大小，比如float的范围比int大。

从int转换成long或者double不会丢失精度，但是从int或者long转换成float，或者long转换成double可能会丢失精度(可能丢失最低几个有效位的数值)。窄化类型的转换可能导致转换结果产生不同的正负号，这种转换仅仅是把数据的高位丢弃，正数int转换成short就可能变成了负数。

将浮点数转换成整数，很有可能浮点数的范围超过了整数能表示的范围，这时候就转换成整数类型所能表示的最大或者最小值，NaN转换成int或者long类型的0。
    