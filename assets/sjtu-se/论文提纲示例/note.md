#Espresso:Brewing Java For ...
##Definition
###NVM
* byte-addressable non-volatile memory
* near-DRAM latency 动态随机存取存储器一样的延迟时间
* disk-like persistence 硬盘般的持久性

###NVML
* 一个用C写的类库
* 提供了在NVM中获取数据的ACID语义

##Espresso
* 既有粗粒度也有细粒度
* 兼容绝大多数java程序中的数据结构

##PJH
###简介
* 扩展已有的Java堆，新增了一个额外的持久化堆。基于*Parallel Scavenge Heap*，是openJDK默认的堆实现
    - 原本PSHeap的实现包括两个部分：新生代和老年代。类创建之后会在新生代中，当存活过几次gc后，将被移到老年代中
    - 垃圾收集器PSGC有两种垃圾收集算法：
        + young gc：只收集新生代中的垃圾，经常发生，很快结束。
        + old gc：整个堆回收，偶尔发生，花多一些的时间

* 在Java中，每一个对象都有一个类指针，指向类相关的元数据*Klass*，类指针存储在对象的头部，就在真实数据域旁边。JVM维持一个元数据区来管理Klasses。Klass存储了对象的layout信息。一旦类指针坏了、或者Klass丢失，对象中的数据就会变的无法翻译。
* PJH与PSHeap之间是独立的，设计为不分代的堆，因为我们相信里面的数据都是长久存活的数据（持久化保障和相比于DRAM性能低。使用的类似老年代的垃圾回收算法。

* PJH由Metadata Area、Name Table、Klass segment，data heap组成。他们全部持久化在NVM中来保证crash-consistency。
    - Klass segment，Data heap：需要持久化的对象存在数据堆中，对象头结构layout与在普通堆中是一样的，所以还是有类指针指向类相关数据。所有的Klass存储在Klass segment中，与原始的meta space分离开。
    - Name table：存储从字符串常量到两个不同类型条目（Klass entry/root entry）的映射。Klass entry存储了Klass在Klass segment中的起始地址，这是JVM在对象创建时，如果发现Klass不在Klass segment中，就会设定的。root entry存储了root对象的信息，这是由用户设定并管理的。该对象在系统重启的时候十分重要，因为只有他们是已知的入口来获取数据堆中的对象。
    - Metadata Area：用于存储堆相关的元数据（resuable/creash-consistent),从上到下
        + address hint：存储堆的虚拟地址开始地址，为了堆重载
        + heap size：PJH最大可占用的地址空间
        + top address：用于计算PJH可分配的字节
        + GC-related：用于实现可恢复的垃圾回收器

###语言扩展：pnew
* 语法和new是一样的，只是新建的对象在NVM上。修改了javac，来把pnew变成四种不同的字节码因为不同的语法：pnew(普通实例)/panewarray(对象数组)/pnewarray(原始数组)/pmultianewarray(多维数组)。
这些字节码会不管类型的把对象放进PJH。因为pnew将对象放在NVM中不考虑它的域。如果用户想要让某个特定的域值持久化，需要用pnew来实现新的构造器。

* 没有Alias Klasses之前：有着同样类型的对象可以既存在DRAM中也存在NVM中，这违反了Java运行环境的假设。
    - 在Java中每一个Klass里都有一个常量池，池中存储了一些关键符号会在运行期解析。
    - 对于每一个类符号，常量池在初始化的时候会分一个slot，存储指向它名字的引用。一旦符号被解析，slot将会替换为存储相关Klass的地址
    - 示例：new和pnew的同一个对象之间不能互相cast，因为有着不同的Klass。常量池中每一个类符号都只有一个slot，当一开始JVM在DRAM中分配了该对象的Klass，在NVM的Klass segment中分配了该对象的Klass，因为这两个分配的地址不同，常量池会用第二个Klass的地址代替之前的地址，这就会在类型转化的时候，发现常量池中的与实际的Klass不一样。

* 别名的解决方法：如果两个类逻辑上一样，只是存在了不同的地方，那么他们的Klass是互相的别名。实现方式是，定义一个新的域值指向它的别名。
    * 别名们会共享元数据如静态成员和方法来保证正确性。
    * 在JVM的类型检查中增加别名检查
    * 在openJDK服务编译器中扩展类型网络，为了在JIT优化中考虑别名。
    * 扩展了子类型检查和classloader检查。比如检查A和B之间是否有子类型关系，A'和B'是别名，那么在AB/AB'/A'B/A'B'中有任意一对存在子类型关系，那么检查通过。虽然算法变得有点麻烦，但大多数情况下，没有别名。

###堆管理
* 要求定义root对象作为句柄，来在系统重启后获得持久化对象。
* 堆相关API
    - createHeap：创建PJH实例，赋予它名字和大小。因为已经实现了一个额外的名称管理器来负责名称与真实数据之间的映射，这里会给名称管理器插入一条新映射。除此之外，虚拟地址首地址也会被存储为*address hint*
    - loadHeap：加载已经存在的PJH实例到现在的JVM中。调用时，额外的名称管理器会定位PJH的实例，并通过获取address hint返回它的首地址。接下来，JVM会将整个PJH从首地址开始映射，如果因为地址被普通堆占用而映射失败，就需要把整个PJH移到新的虚拟地址。此时，堆里面所有的指针都会变成垃圾，会授权进行完全扫描来更新指针。这个重新映射的过程开销很大，但因为虚拟地址空间非常大所以很少会发生。如果映射操作成功，接下来就是一个类的重新初始化阶段。
    - 每个类初始化的时候，JVM会给它分配一个新的Klass数据结构在Meta Space中。但我们不能直截了当地在Klass Segment中新建所有的Klass，这样会使得PJH中的类指针变成垃圾。为了防止这样，我们需要在PJH中给所有的Klass一个占位符，这样他们就能在正确的位置上初始化了。
    - 加载阶段很快，因为加载的速度与类的个数成正比，而不是对象的个数。同时类的个数一般都很少

* root相关API 
    - root对象标记了一些已知的持久化对象的位置，可以作为获取PJH的入口，尤其是重载的时候。
    - root对象的类型是不存的，所以获取之后要进行类型转化

###引用完整性
PJH的设计解耦了对象与其值域的持久性：一个对象可以存储在NVM里而引用指向DRAM。允许程序员分配对象在不同的地方，比如数据缓存和锁都应该放在DRAM中。但这违反了引用完整性，用于保证所有的引用从crash中恢复后都指向正确的数据。如果用户想在堆重载后，获取指向易失性内存的引用，这个指针可能指向任何地方。相反，一个非常严格的不变量引用可以保证正确性，但很难使用DRAM。为此，我们提供了三种不同的安全等级，来应对可用性和安全性上的不同需求。

* User-guaranted safety：用户要有意识哪些指针指向易失性对象，并在重载堆后，有意识的避免使用。这给程序员很大的压力，但是性能最好。
* Zeroing safety：PJH实例会在加载前先进行检查，把所有指向外面的指针置为无效。这样通过检查null就可以很方便的检验是否有Java执行上下文的缺失，再不济也只是NullPointerException。默认的安全模式。
    - 缺点：检查会横跨整个堆，减慢堆堆加载速度。为了减少横跨开销，PJH维护一个card table来分门别类的存储指到外面的指针，这样在扫描的时候只需要检查这些指针。用户也可以额外启动一个帮助的Java进程来分离load的过程，减少时间。

* Type-based safety：用简单的注解来定义类，只有这些类的对象才会被持久化进PJH。这些安全的指针保证了PJH中没有指针会指到外面。最简单

###持久化保证
主流计算机架构只有易失性内存，所以必须要flush操作来把数据持久化进NVM。

* 为了保护持久化顺序性，引入了内存围栏指令sfence，提供API
* 如果不在意对象值持久化的顺序，可以把他们一起flush后，加上一个hence指令。
* API平台独立，可以被用到各种框架上。如果一个框架有持久化缓存，flush就会被无视。如果框架有严格的内存模型，fence也会没有。

##Crash-consistent Heap
###Crash-consistent分配
* 持久化堆维护一个top变量来存储多少内存资源已经被分配，top变量在PJH中复制了一遍，用于堆重载。
* 分配过程分为三步
    - 从常量池获取Klass指针
    - 分配内存，更新top值
    - 初始化对象头

* 因为Java编译器保证了对象在头部初始化之前不可见，所以不存在线程之间的相互影响。
    * top值的复制一定要在第二步top值一产生在易失性内存中，就持久化到PJH中。(flush/fence)
    * 第三步中的Klass指针的更新也要持久化，避免出现初始化对象指向了腐朽的Klass元数据。

###Crash-consistent Garbage Collection
因为持久化对象的生命周期都很长，我们重用了PSGC中的老年代垃圾回收算法。但原始算法需要增强来应对crash-consistency的问题。

* PSGC中，整个堆被分为了很多小的区域叫region，GC有三个阶段
    - 标记阶段：从所有根标记活着的对象，维护一个只读的位图叫*mark bitmap*来用高效的方式存储所有活跃对象。
    - summary阶段：总览堆空间，产生基于region的指标来存储每一个活着的对象的目的地址。因为这个阶段是幂等的，指标只能从位图中推断出来，所以只要位图完好，总结的结果就是一样的。
    - 压缩阶段：GC线程挑拣出未经处理的region，复制所有活着的对象到它们目标地址。region是并发处理的，但是一个region只会有一个线程处理。对于每一个对象，GC线程会先通过查询基于region的indice来获取它的目标地址，然后把它的内容复制过去。最后在indice帮助下，把复制过去的对象的引用调整为正确的引用。

* Crash-consistent GC，整个压缩阶段都是inconsistnet，所以一个可行的方法用于crash恢复是，让GC从crash的位置继续压缩直到结束。
    - 在压缩阶段开始前拍一个snapshot，并持久化。即持久化标记位图。因为summary是幂等的，所以位图就够了。拍完后，Espresso标记整个堆“正在进行垃圾回收”。
    - 如果一个对象有两个属性，但是crash的时候只复制了一半，那么snapshot只提供了目的地址，却无法判断哪些属性已经被复制了
    - 基于timestamp的算法，提示crash发生的时候对象的状态。重用了java对象头部的元数据位。每个对象头部有一个本地时间戳，全局有一个时间戳在PJH元数据区。一开始两个时间戳是一样的，在压缩阶段开始前，增加全局时间戳。只有当对象的全部内容复制过去之后，才会同步对象的时间戳与全局的一致，更新并持久化。

###Recovery
如果堆被标记为正在进行垃圾回收，那么在*loadHeap*的时候恢复阶段会被激活。这个阶段主要有三步

* 重载一致的snapshot，即位图
* 重新进行summary阶段，即重新生成基于region的指标来计算目的地址。
* 在所有的region中，通过读取时间戳，找出所有腐朽的对象。
* 除此之外，如果安全等级是*zeroing safety*，在最后一步的时候，无效的引用会被置为null

##PJO
因为Java程序和JVM之间的语义代沟sematic gap，提供高标准的如ACID语义会很有挑战性。让Java运行环境给程序管理log空间又麻烦又低效。相反即便一个在Java之上的持久层如JPA能够提供一个很方便的持久化编程模型，但是在NVM上开销仍然很大。为了不要这样，Espresso提供了基于PJH的PJO作为持久化编程的另一种选择。

* 允许如数据库的应用像存储普通Java对象一样的存储它们的数据。通过重用JPA的接口和注解提供了向后兼容，并且收获了PJH提供的便利。
* 依旧用EntityManager来持久化对象，但是一旦调用persist()方法，对象就会直接被移到后端数据库，而不会经过SQL转化的过程。
* 在事务中使用的时候，代码由事务相关的指令包围，这些语义操作类似原子块，PJO隐藏了持久化数据管理的复杂度，用户可以直接用*new*创建对象，用*persist()*来持久化。
* Espresso提供了一个新的轻量级的抽象叫*DBPersistable*来支持所有对象实际存储在NVM中。DBPersistable对象与Persistable对象相似，除了关于PJO Provider的控制域被去掉了。
* PJO Provider会增强对象，让每个对象保存一个叫*StateManager*的域，用于管理元数据和控制获取。实现了一个重复数据删除优化，即下面第三步。
    - 现在DRAM里新增一个DBPerson对象，将它的引用指针指向Person的所有域。
    - 将DBPerson对象持久化到NVM中，等于复制进去。
    - 删除DRAM中的DBPerson对象，及其域，保留State Manager在DRAM中。
    - 将Person对象的属性指针指向NVM中的域。

PJO Provider是基于DataNucleus修改的，它提供了和JPA差不多的API，所以程序不需要做什么修改。程序员可以在面向对象编程中利用API来取出、处理、更新数据。类似JPA，PJO也提供了各种各样的类型，如继承类、集合、外键引用。

## JPA
###特点
* 事务化API
* 粗粒度抽象
* 可以声明自己的类、子类和有注解的集合
* 提供ACID事务的抽象，保证所有关于持久化数据的更新都会在事务commit后持久化

###缺点（测试方式：breakdown）
* 用户操作24%，对象与sql语句之间的转化41.9%
* Java对象和本地序列化数据之间有许多不必要的转化

## DataNucleus（JPA provider）
* 所有需要持久化的类要实现 *Persistable* 的接口，或者使用注解 *@persistable*
* 使用字节码工具 *enhancer* 来将有注解的转化为实现接口的类
* 给持久化对象插入控制域和工具辅助方法，来便于管理。

###缺点
* 初始化一个转化阶段来把所有的更新翻译在SQL中，通过JDBC来把更新实现在关系型数据库中。
* 即使关系型数据库是用java实现的，但是只传输了sql，所以没有办法直接将对象保存在里面，只能通过sql保存更新

##PCJ
###特点
* 细粒度编程模型
* 从对象层面操纵持久化数据

###缺点
* 独立的类型系统，不能兼容已有的Java程序，可移植性差，因为强制使用它自己定义的集合。只有PersistentObject的子类才能被持久化到NVM中。PersistentInteger, PersistentString
* 没有利用堆进行数据管理。没有利用java已有的优点，而是借助NVML来管理数据（C库）。导致必须声明一个特殊层给本地对象来进行**序列化**和**垃圾回收**，这都要自己实现。

    * 实验证明：声明200000个PersistentLong对象，其中真实数据操作占1.8%。
    * 元数据更新36.8%，大部分由类型信息存储导致。这个在java中只是一个引用存储，非常简单。
    * 垃圾回收关于新建对象的信息，占14.8%。PCJ使用**引用计数**的算法，每一次初始化的时候都要更新GC相关的信息。Java堆使用更成熟的垃圾回收器，花更少的时间来分门别类的记录对象。
    * 事务化，主要由**同步原语**和**log记录**。这部分可以由Java中的**对象头部的保留字**和**事务库**优化
    * 总之，这一切都是因为PCJ没有使用*堆设计*

##共同的缺点
* JPA和PCJ目标的场景不同，开发者没法统一的使用一种方法来满足所有需求
* 额外开销都特别大

##新的需求
* 统一的持久化：应该粗细粒度一起支持，支持更多应用
* 高性能：额外开销越小越好，充分利用NVM的性能优势
* 向后兼容：不要有过多的主要的数据库变化，这样已有的应用只需要一点努力就可以在上面运行。

##评估
###环境准备
* 在openJDK 8上实现JPH：7000行c++ && 300行Java
* 在DataNuclcus上实现PJO：1500行Java
* 数据库H2：600行。与之对比PCJ用了3000行。

###与PCJ的比较
* 类型系统
    - PCJ提供了独立的类型系统
    - PJH也有类似的数据结构

* ACID语义
    - PCJ提供了ACID语义给所有的操作
    - PJH通过提供简单的undo log，来给予ACID的保障

* 实验过程：在这些数据类型上进行百万级的原始操作(create/get/set)，然后计算执行时间。
    - PJH在set/create操作中优越感很明显，因为PCJ不在堆上存储数据，导致这些操作需要复杂的元数据更新。
    - PJH在get操作中优越感一般，因为这个操作不需要太多的元数据更新。

* 实验过程：分割评估阶段，来确定性能提升的来源。set Tuple
    - 手动移除PCJ中的GC管理代码后，PJH的优越感降了一半
    - 移除事务相关的远程调用后，优越感几乎就没了。
    - 由此推断，PCJ的开销主要来源于GC和事务相关的操作，这些开销都可以在堆上得到缓和
    - 之前提到的元数据更新在这里微不足道，是因为类型信息存储只发生在PCJ对象create的时候。

###与JPA比较
* 使用的是JPA Benchmark来比较PJO与JPA。JPAB包含了对create/read/update/delete操作的测试。H2-JPA & H2-PJO
    - *BasicTest*测试基础的用户定义的类
    - *ExtTest*测试有继承关系的类
    - *CollectionTest*测试有集合成员的类
    - *NodeTest*测试有外键引用的类

* 对于直接在JVM的普通堆上使用PJO，即没有持久化，在DRAM上处理数据也进行了测试。这个性能比H2-PJO还要好一点，所以要想将应用移植到NVM上，需要付出一点性能代价。H2-PJO & H2-PJO-v
* 用*Basic Test*进行细节分析，将性能分成三块：H2数据库的执行，SQL语句的转化和其他。
    - 可以看出因为PJO，转化开销显著减少了。
    - 因为不使用JDBC的接口，而是使用DBPersistable的抽象，H2的执行时间也减少了。

###基准程序
* 堆加载时间：创建了大量的对象（20万～200万）基于20个不同的类，在两个不同的安全等级user-guaranted/zeroing，进行测试。
    - UG的时间不随对象的增加而变化，因为它的类加载时间取决于类的个数。
    - Zero的时间随对象的增加而线性增加，因为这个安全等级需要全堆扫描来确认所有对象。但就算这样，加载时间（几十毫秒）还是远远小于JVM的热身时间（几秒）。

* 可恢复的GC堆收集效率：分配了一个数组在PJH上，里面有几百万的对象（1千万～2千万），其中三分之一的引用会之后被移去，接着强制调用*System.gc()*来收集PJH。
    - 1G-PSGC和可恢复GC，因为后者是单代堆组织，所以相比于PSGC还需要重新初始化新生代，后者的GC时间比较少。
    - 4G-PSGC和可恢复GC，此时PSGC因为足够大，不再需要新生代初始化，所以两者的性能差不多。

* 可恢复的GC的可恢复性：通过在GC期间，发送SIGKILL信号，随机加入crash事件。为了让不一致性问题更加尖锐，数组大小开到2千万来涉及到更多的region。测试了50次，每次都能很好的恢复，时间2～4秒不等，这与老年代GC速度差不多。

##相关工作
###不易失性内存堆
可恢复的虚拟内存的出现，促进了构建存放用户定义的对象的持久化、可恢复、高效堆的研究。持久化Java的提出了一个有着不易失性堆的全系统持久化的Java运行环境。然而这需要解决*System*类的问题。后面的工作转向于实践持久化对象存储和实体关系映射。

最近，因为NVM技术的发展，持久化内存堆的概念被重新炒热。

* NV-Heaps首先遇到了堆中交叉指针的问题。它通过用在编译期间检查，直接不允许不易失性指向易失性的指针，避免了潜在的内存泄漏。但这些限制阻止了最近很常见的基于NVM系统中，应用既想用DRAM又想用NVM的场景。
* NV-Heaps提供了一个简单的引用计数的垃圾回收器，不得不引入弱引用的概念来手动避免内存泄漏。
* PJH提供了灵活的指针，和基于JVM的自动垃圾回收器。
* Makalu是一个基于Atlas编程模型的持久化内存分配器。它提供了分配元数据的持久化和可恢复的保证。并且提供一个可恢复的垃圾回收器。

###Java运行环境优化
* 减少类加载时间：
    - HotTub发现类加载是在JVM热身期间一个重要的低效来源。它引入虚拟机器池来缓和类加载时间。
    - 我们也想到要减少类加载时间，不过用的是另一种适用NVM的方式。

* 优化垃圾回收器：
    - NumaGiC发现Java中的老年代垃圾回收器性能因为没有利用NUMA，所以有可扩展性问题。于是它就提出了一个NUMA-friendly算法。
    - Yu etal通过缓存之前的结果，去掉了老年代PSGC在目的地址计算中的性能瓶颈。
    - Yak GC通过使用基于region的算法，来避免不必要的对象复制。

###基于NVM的系统中的事务
* Mnemosyne实现了sematic-free的原始字记录(raw word log)来支持事务。
* Atlas使用同步变量，如锁和复原代码，来提供像事务一个的ACID特性和logging协议。
* 其他工作指出，持久化操作不应该在事务的必须执行路径中，然后提供了各种把持久化操作移到后台的解决方案。

_wjq@sjtu-ipads_
