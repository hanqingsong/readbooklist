<!-- MarkdownTOC -->

- 自动内存管理机制
    - 2 java内存区域与内存溢出异常
    - 垃圾收集器与内存分配策略
    - 虚拟机性能监控与故障处理工具
    - 调优案例分析与实战
- 虚拟机执行子系统
    - 类文件结构
    - 字节码指令简介
- 虚拟机类加载机制

<!-- /MarkdownTOC -->

## 自动内存管理机制
### 2 java内存区域与内存溢出异常
#### 2.2 运行时数据区域
>
1. 程序计数器：当前程序所执行的字节码的行号指示器。各线程直接计数器互不影响，独立存储。线程私有的。如果执行的是Native方法，则计数器为空。这是唯一一个在java虚拟机规范中没有任何outOfMemaryError的情况的区域。
>
2. java虚拟机栈：也是线程私有的，它的生命周期与线程相同。描述的是java方法执行的内存模型：每个方法在执行的同时都会创建一个栈帧用于存储局部变量表，操作数栈、动态链接、方法出口等信息。每一个方法从调用直至执行完成的过程，就对应着一个栈帧在虚拟机栈中入栈到出栈的过程。（StackOverflowError、OutOfMemoryError异常）
>
3. 本地方法栈：与虚拟机栈类似，只不过是本地方法服务。（StackOverflowError、OutOfMemoryError异常）
>
4. java堆：是虚拟机中管理的内存中最大的一块。被所哟线程共享的一块内存区域，在虚拟机启动时创建。此内存区域唯一目的是存放对象实例。可以分为：新生代和老年代。采用分代收集算法。OutOfMemoryError异常
>
5. 方法区：和java堆一样，是各个线程共享的内存区域，用于存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据。也被称为永久代。官方有放弃永久代改为采用Native Memory来实现方法区的规划。在jdk7中已经把原本放在永久代的字符串常量池移出。OutOfMemoryError异常
>
6. 运行时常量池：是方法区的一部分。在类加载后放入运行时常量。具备动态性，运行期间可以将新的常量放入池中。OutOfMemoryError异常
>
7. 直接内存：并不是虚拟机运行时数据区的一部分，也不是虚拟机规范定义的内存区域。可能会OutOfMemoryError异常。jdk1.4，加入NIO类引入了一种基于通道channel和缓冲区Buffer的I/O方式，可以使用Native函数库直接分配堆外内存，然后通过一个存储在java堆中的DirectByteBuffer对象作为这块内存的引用进行操作。避免在Java堆和Native堆中来回复制数据。

#### hotspot虚拟机对象探秘

>
1. 对象的创建：虚拟机遇到new指令时，首先将去检查这个指令的参数是否能在常量池中定位到一个类的符号引用，并且检查这个符号引用代表的类是否已被加载、解析和初始化过。如果没有则执行相应的类加载过程。
1.1 创建过程：
		1. 在常量池检查是否有符号引用，是否已被加载、解析和初始化。
		2. 为新生对象分配内存
		3. 分配的内存空间初始化为零值，保证对象的实例字段可以不赋值就直接使用。
		4. 对对象进行必要的设置，是哪个类的实例，如何找到类的元数据信息、对象的哈希码、对象的GC分代年龄等信息。这些信息存放在对象的对象头之中。
		5. 执行init方法

>
2. 对象的内存布局，对象在内存中布局可分为3块区域：对象头、实例数据和对齐填充。
	1. 虚拟机对象头包括两部分信息，第一部分用于存储对象自身运行时数据，如哈希码、GC分代年龄、锁状态标志、线程持有的锁、偏向线程id、偏向时间戳等，这部分数据长度在32位和64位的虚拟机中分别为32bit和64bit;第二部分是类型指针，即对象指向它的类元数据的指针，通过这个指针来确定这个对象是哪个类的实例。
	2. 实例数据是对象存储的真正有效信息，在代码中定义的各种类型的字段内容。无论是在父类中继承下来的，还是在子类中定义的，都需要记录起来。

>
3.对象的访问定位
建立对象是为了使用对象，我们的java程序需要通过栈上的reference数据来操作堆上的具体对象。由于reference类型在java虚拟机规范中只规定了一个指向对象的引用，并没有定义这个引用应该通过何种方式去定位、访问堆中的对象具体位置，所以对象访问方式取决于虚拟机实现。主流的访问方式有使用句柄和直接指针 

#### 实战：OutOfMemoryError异常
>
1. java堆溢出 -Xms -Xmx
不断创建对象，并且保证GC Roots到对象之间有可达路径来避免垃圾回收机制清楚这些对象，那么在数据量到达最大堆的容量限制后就会产生内存溢出异常。
通过参数设置在出现异常时Dump出当前内存转储快照以便时候进行分析。先分清是内存泄露还是内存溢出。
java.lang.OutOfMemoryError: java heap space

>
2. 虚拟机栈和本地方法栈溢出 -Xss
在hotspot中不区分虚拟机栈和本地方法栈。在java虚拟机规范中描述两种异常StackOverflowError和OutOfMemoryError异常
java.lang.StackOverflowError
java.lang.OutOfMemoryError: unable to create new native thread


>
3. 方法区和运行时常量池溢出 -XX:PermSize 和 -XX:MaxPermSize
jdk1.6 java.lang.OutOfMemoryError:PermGen space
jdk1.6 String.intern方法会把首次遇到的字符串实例复制到永久代中，返回的是永久代这个字符串实例的引用。
jdk1.7 strin.intern方法不会再复制实例，只是在常量池中记录首次出现的实例引用。
方法区用于存放class相关信息，如类名，访问修饰符、常量池、字段描述、方法描述等。


>
4. 本机直接内存溢出 -XX

### 垃圾收集器与内存分配策略

- 哪些内存需要回收
- 什么时候回收
- 如何回收

#### 对象已死么？
    
>
1. 引用计数算法
很难解决对象之间的互相循环引用的问题。

2. 可达性分析算法
通过GC Roots的对象作为起始点，从这些节点开始向下搜索，搜索所走过的路径称为引用链，当一个对象到GC Roots没有任何引用链相连时，则证明对象是不可用的。
GC Roots对象包括以下几种：
- 虚拟机栈中引用的对象
- 方法区中类静态属性引用的对象
- 方法区中常量引用的对象
- 本地方法栈中JNI引用的对象

3. 再谈引用

4. 生存还是死亡

5. 回收方法区

#### 垃圾收集算法

>
1. 标记清除算法
先标记后统一回收，最基础算法。效率问题和空间问题。

2. 复制算法
解决效率问题，内存分为大小相等两块，当一块内存使用完了，将还存活的对象复制到另一块上面，然后清理使用的内存。

3. 标记整理算法
和标记清除算法一样，但后续步骤是让所有存活对象都移向一端，润喉直接清理掉边界以外的内存。

4. 分代收集算法
当前商业虚拟机都采用分代算法，吧内存分为不同的代，根据不同的代采用不同的算法。 

#### HotSpot算法的实现

>
1. 枚举根节点
类加载完成的时候，使用OopMap的数据结构，把对象什么偏移量上是什么类型的数据计算出来。在JIT编译工程也会记录栈和寄存器哪些位置是引用。

2. 安全点


3. 安全区域

#### 垃圾收集器

>
1. serial收集器
2. parnew收集器
3. parallel scavenge收集器
是一个新生代收集器，也是复制算法收集器，并行的多线程收集器
4. serial old收集器
单线程收集器，使用标记整理算法。
5. parallel old收集器
使用多线程和标记整理算法。
6. cms收集器
以获取最短回收停顿时间为目标的收集器。基于标记清除算法实现的。
7. g1收集器
当今收集器技术发展最前沿的成果之一。
8. 理解gc日志

#### 内存分配与回收策略

1. 对象有限在eden分配
2. 大对象直接进入老年代
3. 长期存活的对象将进入老年代
4. 动态对象年龄判定
5. 空间分配担保

### 虚拟机性能监控与故障处理工具
数据：运行日志，异常堆栈，GC日志，线程快照，堆转储快照等。

1. jdk命令行工具
    1.1 jps JVM process status tool 显示虚拟机进程
    1.2 jstat JVM statistics Monitoring Tool 收集虚拟机各方面运行数据。
    1.3 jinfo Configuration Info for Java显示虚拟机配置信息
    1.4 jmap Memory Map for java 虚拟机内存转储快照
    1.5 jhat JVM Heap dump browser 用于分析heapdump文件
    1.6 jstack stack trace for java 显示虚拟机的线程快照

2. jdk可视化工具
    2.1 jconsole java见识与管理控制台
    2.2 visualVM 多合一故障处理工具

### 调优案例分析与实战
 
1. 案例分析
2. 实战：eclipse运行速度调优


## 虚拟机执行子系统
### 类文件结构
>
class文件是一组以8位字节为基础单位的二进制流。
使用伪结构体来存储数据，伪结构体只有两种数据类型：无符号数和表，无符号数属于基本的数据类型，以u1，u2，u4，u8来表示1个字节，2个字节，4个字节和8个字节的无符号数，可以用来描述数字，索引引用，数量值或者按照utf-8编码构成的字符串值。
表是由多个无符号数或者其他表作为数据项构成的复合数据类型，习惯以_info结尾。整个class文件本质上就是一张表。_

>
1. 魔数与class文件的版本
每个Class文件的头4个字节称为魔数，他的唯一作用是确定这个文件是否为一个能被虚拟机接受的class文件。class的魔数值为：0xCAFEBABE（咖啡宝贝）
紧接着魔数的4个字节存储的是class文件的版本号：第5和第6是此版本，7和8是主版本。

2. 常量池
紧接着主次版本号之后的是常量池入口，常量池可以理解为class文件中的资源仓库。
在常量池入口需要放置一项u2类型的数据，代表常量池容量计数值。
常量池中主要存放两大类常量：字面量和符号引用。

3. 访问标志
紧接着两个字节是访问标志，标识一些类或接口层次的访问信息，Class是类还是接口，是否定义为public类型，是否定义abstract类型，是否被声明为final等。

4. 类索引、父类索引和接口索引集合


5. 字段表集合
6. 方法表集合
7. 属性表集合

### 字节码指令简介

## 虚拟机类加载机制
虚拟机把描述类的数据从class文件加载到内存，并对数据进行校验，转换解析和初始化，最终形成可以被虚拟机直接使用的java类型

#### 类加载的时机
类从被加载到虚拟机内存中开始，到卸载出内存为止，它的整个生命周期包括：加载loading、验证verification、准备preparation、解析resolution、初始化initalization、使用using和卸载unloading7个阶段。

#### 类加载的过程
##### 加载
加载阶段需要完成3件事情：
1. 通过一个类的全限定名来获取定义此类的二进制字节流
2. 将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构
3. 在内存中生成一个代表这个类的java.lang.Class对象，作为方法区这个类的各种数据的访问入口。
加载完成后，虚拟机外部的二进制字节流就按照虚拟机所需的格式存储在方法区之中，然后在内存中实例化一个java.lang.Class类的对象，这个对象存放在方法区里，这个对象将作为程序访问方法区中的这些类型数据的外部接口。

##### 验证
验证是连接阶段的第一步，这一阶段的目的是为了确保Class文件的字节流中包含的信息符合当前虚拟机的要求，并且不会危害虚拟机自身的安全。

##### 准备
准备阶段是正式为类变量分配内存并设置类变量初始值的阶段，这些变量所使用的内存都将在方法区中进行分配。这个阶段中仅是类变量被static修饰的变量，不包括实例变量，实例变量将会在对象实例化时随着对象一起分配在java堆中。这里的初始值是数据类型0值。赋值为真实值是在初始化阶段才会执行。如果类字段的字段属性表中存在ConstantValue属性，那在准备阶段变量value就会被初始化为ConstantValue属性所指定的值，比如final修饰的类变量，编译时就会为value生成ConstantValue属性，在准备阶段虚拟机就会根据ConstantValue的设置将value设置为真实值。

##### 解析
解析阶段是虚拟机将常量池内的符号引用替换为直接引用的过程。

##### 初始化
类初始化是类加载过程的最后一步，前面的类在加载过程中，除了在加载阶段用户应用程序可以通过自定义类加载器参与之外，其余动作完全由虚拟机主导和控制。到了初始化阶段，才真正开始执行类中定义的java程序代码。初始化阶段是执行类构造器方法<init>()的过程。
<init>()方法是由编译器自动收集类中所有的类变量的赋值动作和静态语句块中的语句合并残生的。
静态语句块只能访问到定义在静态语句块之前的变量，定义在之后的变量在前面的静态语句块可以赋值，但是不能访问。
<init>()方法与类的构造器不同。
<init>()方法对于类或接口不是必需的，如果没有静态语句块也没有对变量赋值，那编译器可以不生成<init>()方法

#### 类加载器

在虚拟机外部实现“通过一个类的全限定名来获取描述此类的二进制字节流”，以便让应用程序自己决定如何去获取所需要的类。

类加载器只用于类的加载动作。
3种系统提供的类加载器：启动类加载器、扩展类加载器和应用程序类加载器。