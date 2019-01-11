# <center>深入理解Java虚拟机笔记</center>

---

## 1. 自动内存管理机制

### **1.1. 运行时数据区**

<center>

![img](https://images2015.cnblogs.com/blog/1182497/201706/1182497-20170616192740650-1039521219.png)
图1 java运行时数据区域
</center>

- **程序计数器（PC，<font color=red>线程私有</font>）**
    &emsp;&emsp;为了线程恢复后能切换到正确的位置，每条线程都有一个PC，互不影响。
    &emsp;&emsp;若线程正在执行java方法，pc记录正在执行的虚拟机字节码指令的地址；若正在执行native方法，则pc为空（undefined）。
    &emsp;
- **java虚拟机栈（<font color=red>线程私有</font>）**
    &emsp;&emsp;用于存储**局部变量表、操作数栈、动态链接、方法出口**等信息。
    &emsp;&emsp;局部变量表存放：**基本数据类型**（int, long, short, byte, float, double, char, boolean. 64位长度的long和double占用两个局部空间（slot），其他的都只占用一个）、**对象引用**（reference类型，不等同于对象本身）、**returnAddress类型**（指向一条字节码指令的地址）。
    &emsp;&emsp;<font color=red>局部变量表所需内存在编译时完成。运行期间不会改变局部变量表的大小。</font>
    &emsp;&emsp;这个区域有两种异常状况：StackOverflowError（请求的栈深度大于虚拟机所允许的深度）, OutOfMemeryError（当虚拟机可以动态扩展，扩展时申请不到足够的内存）。
    &emsp;
- **本地方法栈**
    &emsp;&emsp;同java虚拟机栈。java虚拟机栈为虚拟机执行java方法服务，本地方法栈为虚拟机执行native方法服务。有的虚拟机将二者合二为一。
    &emsp;
- **Java堆（所有线程共享，<font color=red>虚拟机启动时创建</font>）**
    &emsp;&emsp;目的：<font color=red>存放对象实例</font>。
    &emsp;&emsp;java堆是垃圾收集器管理的主要区域，也被叫做GC堆。堆还可分为：新生代，老年代；Eden空间，From Survior空间，To Survivor空间。
    &emsp;&emsp;<font color=red>java堆中可能划分出多个线程私有的分配缓冲区（TLAB），目的是为了更好地回收内存或者更快的分配内存。</font>
    &emsp;&emsp;Java堆只需逻辑上是连续的即可，可通过-Xmx和-Xms进行扩展。
    &emsp;&emsp;如果在堆中没有内存完成实例分配，且堆无法再扩展时，将抛出OutofMemoryError异常。
    &emsp;
- **方法区（线程共享）**
    &emsp;&emsp;用于存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译完成后的代码等数据。（别名：<font color=red>Non-Heap</font>）
    &emsp;&emsp;可用多种方法实现方法区，比如堆中的永久代。但是永久代有-XX:MaxPermSize的限制，容易遇到内存溢出问题。也需要实现垃圾回收机制。
    &emsp;&emsp;当方法区无法满足内存分配需求时，将抛出OutOfMemory异常。
    &emsp;
- **运行时常量池（方法区的一部分）**
    &emsp;&emsp;Class文件中除了有类的版本、字段、方法、接口等描述信息外,还有一项信息是**常量池(Constant Pool Table)**，用于存放编译期生成的各种字面量和符号引用，这部分内容将在类加载后进入方法区的运行时常量池中存放。
    &emsp;&emsp;运行时常量池相对于Class文件常量池的另外一个重要特征是具备动态性, Java语言并不要求常量一定只有编译期才能产生，<font color=red>也就是并非预置入Class文件中常量池的内容才能进入方法区运行时常量池，运行期间也可能将新的常量放入池中</font>，这种特性被开发人员利用得比较多的便是String类的intern()方法。
    &emsp;
- **直接内存**
    &emsp;&emsp;并不是虚拟机运行时数据区的一部分，也不是Java虚拟机规范中定义的内存区域。
    &emsp;&emsp;JDK1.4加入了NIO，引入一种基于通道与缓冲区的I/O方式，它可以使用Native函数库直接分配堆外内存，然后通过一个存储在Java堆中的DirectByteBuffer对象作为这块内存的引用进行操作。因为避免了在Java堆和Native堆中来回复制数据，提高了性能。
    &emsp;&emsp;当各个内存区域总和大于物理内存限制，抛出OutOfMemoryError异常。

### **1.2. 虚拟机对象**

- **Java对象的创建过程**

<center>

![img](https://images2015.cnblogs.com/blog/592743/201603/592743-20160319235423381-1926278401.png)
图2 java对象的创建过程
</center>

&emsp;&emsp;为对象分配内存，有两种方式：
&emsp;&emsp;<font color=red>指针碰撞</font>：用过的内存放在一边，空闲内存放另一边，中间一个作为分界点的指针。分配内存时就把指针左右移动。（需要堆中的内存是规整的）
&emsp;&emsp;<font color=red>空闲列表</font>：若堆内存不是规整的，需要维护一个表来记录哪些内存是可用的。分配时从列表中找到足够的内存空间划分给对象。
&emsp;&emsp;堆内存是否规整，取决于垃圾收集器是否带有压缩整理功能。
&emsp;&emsp;上述两种内存分配方式还需考虑<font color=red>线程安全性</font>。实际上虚拟机采用<font color=red>CAS配上失败重试的方式保证更新操作的原子性</font>。另一种是：<font color=red>引入TLAB（本地线程分配缓冲）</font>，将内存分配的动作按照线程划分到不同的空间进行。虚拟机是否使用TLAB，通过<font color=red>-XX:+/-UseTLAB</font>参数来设定。

- **对象的内存布局**

&emsp;&emsp;在HotSpot中，对象在内存中存储的布局可以分为3块区域：<font color=red>对象头（Header）</font>、<font color=red>实例数据（Instance Data）</font>、<font color=red>对齐填充（Padding）</font>。
&emsp;&emsp;对象头包括两部分，一部分用于存储对象自身的运行时数据，如哈希码、GC分代年龄、锁状态标志、线程持有的锁、偏向线程ID、偏向时间戳等。另一部分是类型指针，即对象指向它的类元数据的指针，虚拟机通过这个指针来确定这个对象是哪个类的实例。
&emsp;&emsp;实例数据部分是对象真正存储的有效信息。存储顺序受虚拟机分配策略参数和字段在Java源码中定义顺序的影响。
&emsp;&emsp;对齐填充不是必然存在的。起占位符的作用。HotSpot VM要求对象起始地址必须是8字节的整数倍。

- **对象的访问定位**

&emsp;&emsp;方式：<font color=red>使用句柄和直接指针</font>。
