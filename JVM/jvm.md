# 一、运行时数据区域

## 私有内存区域

### 程序计数器

记录字节码指令的行号。

该区域无OOM

### Java 虚拟机栈

**每一个线程一个栈，栈中一个栈帧代表一个调用的方法，每次方法执行过程对应入栈出栈过程。栈帧用于存放局部变量、操作数、动态链接、方法出口等信息。对于执行引擎来说，只有活动线程栈顶的栈帧才是有效的。**

栈深度超过最大值会抛 StackOverflowError，

可以通过 -Xss 这个虚拟机参数来指定每个线程的 Java 虚拟机栈内存大小，在 JDK 1.4 中默认为 256K，而在 JDK 1.5+ 默认为 1M：

```
java -Xss2M HackTheJava
```

该区域可能抛出以下异常：

- 当线程请求的栈深度超过最大值，会抛出 StackOverflowError 异常；
- 如果允许动态扩展，栈进行动态扩展时无法申请到足够内存，会抛出 OutOfMemoryError 异常。

### 本地方法栈

本地方法栈与 Java 虚拟机栈类似，它们之间的区别只不过是本地方法栈为本地方法服务。

本地方法一般是用其它语言（C、C++ 或汇编语言等）编写的，并且被编译为基于本机硬件和操作系统的程序，对待这些方法需要特别处理。

## 共享内存区域

### 堆

堆是对象分配内存的区域，是垃圾收集的主要区域（"GC 堆"）。

现代的垃圾收集器基本都是采用分代收集算法，其主要的思想是针对不同类型的对象采取不同的垃圾回收算法。可以将堆分成两块：

- 新生代（Young Generation）
- 老年代（Old Generation）

当堆没有内存分配实例，且无法扩展时，就会抛出 OutOfMemoryError 异常。

可以通过 -Xms 和 -Xmx 这两个虚拟机参数来指定一个程序的堆内存大小，第一个参数设置初始值，第二个参数设置最大值。

```
java -Xms1M -Xmx2M HackTheJava
```

### 方法区

**方法区用于存放已被加载的类信息、常量、静态变量等数据。**

**方法区是 JVM的一个 规范。JDK1.7以前通过永久代实现，使用堆内存空间。在 JDK 1.8 之后，使用元空间实现，位于直接内存中。原来永久代的数据被分到了堆和元空间中。元空间存储类的元信息，静态变量和常量池等放入堆中。**

和堆一样不需要连续的内存，并且可以动态扩展，动态扩展失败一样会抛出 OutOfMemoryError 异常。

对这块区域进行垃圾回收的主要目标是对常量池的回收和对类的卸载，但是一般比较难实现。

### 运行时常量池

**运行时常量池是方法区的一部分，存放字面量和符号引用。字面量是常量，比如字符串常量池，final修饰的常量。而符号引用编译时无法确定实际地址，就用符号引用代替，包括全类名、方法名，字段名。**

### 直接内存

**直接内存是JVM外的内存区域， NIO 类，它可以使用 Native 函数库直接分配堆外内存，然后通过 Java 堆里的 DirectByteBuffer 对象作为这块内存的引用进行操作。**

从数据流的角度，非直接内存是下面这样的作用链：

```css
本地IO-->直接内存-->非直接内存-->直接内存-->本地IO
```

而直接内存是：

```css
本地IO-->直接内存-->本地IO
```

# 二、垃圾收集

垃圾收集主要是针对共享内存区域堆和方法区进行。程序计数器、虚拟机栈和本地方法栈这三个区域属于线程私有的，只存在于线程的生命周期内，线程结束之后就会消失，因此不需要对这三个区域进行垃圾回收。

### 引用计数法

为对象添加一个引用计数器，当对象增加一个引用时计数器加 1，引用失效时计数器减 1。引用计数为 0 的对象可被回收。

在两个对象出现循环引用的情况下，此时引用计数器永远不为 0，导致无法对它们进行回收。正是因为循环引用的存在，因此 Java 虚拟机不使用引用计数算法。

```
public class Test {

    public Object instance = null;

    public static void main(String[] args) {
        Test a = new Test();
        Test b = new Test();
        a.instance = b;
        b.instance = a;
        a = null;
        b = null;
        doSomething();
    }
}
```

在上述代码中，a 与 b 引用的对象实例互相持有了对象的引用，因此当我们把对 a 对象与 b 对象的引用去除之后，由于两个对象还存在互相之间的引用，导致两个 Test 对象无法被回收。

### 可达性分析法

以 GC Roots 为起始点进行搜索，可达的对象都是存活的，不可达的对象可被回收。

Java 虚拟机使用该算法来判断对象是否可被回收，GC Roots 一般包含以下内容：

- 虚拟机栈中局部变量表中引用的对象
- 本地方法栈中 JNI 中引用的对象
- 方法区中类静态属性引用的对象
- 方法区中的常量引用的对象

[![img](https://camo.githubusercontent.com/cb90e2d0a060faebea7b237b57902247bc0ecbb1/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f38336439303964322d333835382d346665312d386666342d3136343731646230623138302e706e67)](https://camo.githubusercontent.com/cb90e2d0a060faebea7b237b57902247bc0ecbb1/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f38336439303964322d333835382d346665312d386666342d3136343731646230623138302e706e67)



### 方法区回收

因为方法区回收率比较低，回收性价比不高。

**主要是对常量池的回收和对类的卸载。**

为了避免内存溢出，**在大量使用反射和动态代理的场景都需要虚拟机具备类卸载功能**。

类的卸载条件很多，需要满足以下三个条件，并且满足了条件也不一定会被卸载：

- 该类所有的实例都已经被回收，此时堆中不存在该类的任何实例。
- 加载该类的 ClassLoader 已经被回收。
- 该类对应的 Class 对象没有在任何地方被引用，也就无法在任何地方通过反射访问该类方法。

safepoint：

## 引用类型

Java 有4种引用类型。

**1. 强引用**

被强引用关联的对象不会被回收。

new 对象的方式来创建强引用。

```
Object obj = new Object();
```

**2. 软引用**

被软引用关联的对象只有在内存不够的情况下才会被回收。

使用 SoftReference 类来创建软引用。

```
Object obj = new Object();
SoftReference<Object> sf = new SoftReference<Object>(obj);
obj = null;  // 使对象只被软引用关联
```

**3. 弱引用**

被弱引用关联的对象一定会被回收，也就是说它只能存活到下一次垃圾回收发生之前。

使用 WeakReference 类来创建弱引用。

**4. 虚引用**

又称为幽灵引用或者幻影引用，一个对象是否有虚引用的存在，不会对其生存时间造成影响，也**无法通过虚引用得到一个对象。**

为一个对象设置虚引用的唯一目的是能在这个对象被回收时收到一个系统通知。

使用 PhantomReference 来创建虚引用。

```
Object obj = new Object();
PhantomReference<Object> pf = new PhantomReference<Object>(obj, null);
obj = null;
```

## 垃圾收集算法

### 分代收集

现在的商业虚拟机采用分代收集算法，它根据对象存活周期将内存划分为几块，不同块采用适当的收集算法。

一般将堆分为新生代和老年代。

- 新生代使用：复制算法
- 老年代使用：标记 - 清除 或者 标记 - 整理 算法

### 1. 标记 - 清除

**分为标记清楚两阶段，标记阶段对活动对象打标记，清楚阶段回收对象，取消标志位，合并连续空闲块**

在标记阶段，程序会检查每个对象是否为活动对象，如果是活动对象，则程序会在对象头部打上标记。

在清除阶段，会进行对象回收并取消标志位，另外，还会判断回收后的分块与前一个空闲分块是否连续，若连续，会合并这两个分块。回收对象就是把对象作为分块，连接到被称为 “空闲链表” 的单向链表，之后进行分配时只需要遍历这个空闲链表，就可以找到分块。

在分配时，程序会搜索空闲链表寻找空间大于等于新对象大小 size 的块 block。如果它找到的块等于 size，会直接返回这个分块；如果找到的块大于 size，会将块分割成大小为 size 与 (block - size) 的两部分，返回大小为 size 的分块，并把大小为 (block - size) 的块返回给空闲链表。

**不足：**

- 标记和清除过程效率都不高；
- 会产生大量不连续的内存碎片，导致无法给大对象分配内存。

### 2. 标记 - 整理

让所有存活的对象都向一端移动，然后直接清理另一端的内存。

优点:

- 不会产生内存碎片

不足:

- 需要移动大量对象，处理效率比较低。

### 3. 复制

复制算法将新生代划分为Eden 和两块 Survivor ，大小为8：1：1.每次使用 Eden 和其中一块 Survivor。回收的时候，将 Eden 和 Survivor 中的存活对象复制到另一块 Survivor 上，最后清理 Eden 和使用过的那一块 Survivor。

如果回收时多于 10% 的对象存活，一块 Survivor 不够用，也就是老年代存储。

## 垃圾收集器

新生代一般是复制算法，老年代是标记清楚标记整理

除了 CMS 和 G1 之外，其它垃圾收集器都是以串行的方式执行。

1. **Serial** 

Serial 翻译为串行，也就是说它以串行的方式执行。

它是单线程的收集器，只会使用一个线程进行垃圾收集工作。

它的优点是简单高效，在单个 CPU 环境下，由于没有线程交互的开销，因此拥有最高的单线程收集效率。

它是 Client 场景下的默认新生代收集器，因为在该场景下内存一般来说不会很大。它收集一两百兆垃圾的停顿时间可以控制在一百多毫秒以内，只要不是太频繁，这点停顿时间是可以接受的。

2. **ParNew** 

它是 Serial 收集器的多线程版本。

它是 Server 场景下默认的新生代收集器，除了性能原因外，主要是因为除了 Serial 收集器，只有它能与 CMS 收集器配合使用。

3. **Parallel Scavenge** 

与 ParNew 一样是多线程收集器。

其它收集器目标是尽可能缩短垃圾收集时用户线程的停顿时间，而它的目标是达到一个可控制的吞吐量，因此它被称为**“吞吐量优先”收集器**。吞吐量= 程序运行时间/(程序运行时间 + 垃圾收集时间)

停顿时间越短就越适合需要与用户交互的程序，良好的响应速度能提升用户体验。而高吞吐量则可以高效率地利用 CPU 时间，尽快完成程序的运算任务，适合在后台运算而不需要太多交互的任务。

缩短停顿时间是以牺牲吞吐量和新生代空间来换取的：新生代空间变小，垃圾回收变得频繁，导致吞吐量下降。

可以通过一个开关参数打开 GC 自适应的调节策略（GC Ergonomics），就不需要手工指定新生代的大小（-Xmn）、Eden 和 Survivor 区的比例、晋升老年代对象年龄等细节参数了。虚拟机会根据当前系统的运行情况收集性能监控信息，动态调整这些参数以提供最合适的停顿时间或者最大的吞吐量。

4. **Serial Old** 

**是 Serial 收集器的老年代版本**，也是给 Client 场景下的虚拟机使用。如果用在 Server 场景下，它有两大用途：

- 在 JDK 1.5 以及之前版本（Parallel Old 诞生以前）中与 Parallel Scavenge 收集器搭配使用。
- **作为 CMS 收集器的备用方案**，在并发收集发生 Concurrent Mode Failure 时使用。

5. **Parallel Old** 

是 Parallel Scavenge 收集器的老年代版本。

在注重吞吐量以及 CPU 资源敏感的场合，都可以优先考虑 Parallel Scavenge 加 Parallel Old 收集器。

### 6. CMS 收集器（必问）

CMS（Concurrent Mark Sweep）目标是缩短停顿时间，提高交互体验，是老年代收集器。

分为四个流程：

- 初始标记：标记 GC Roots 能直接关联到的对象，需要Stop the world。
- 并发标记：进行 GC Roots 可达性分析。不需要Stop the world。
- 重新标记：为了修正并发标记期间因用户程序继续运作而导致标记产生变动的那一部分对象的标记记录，需要Stop the world。
- 并发清除：不需要Stop the world。

在整个过程中耗时最长的并发标记和并发清除过程中，收集器线程都可以与用户线程一起工作，不需要进行停顿。

具有以下缺点：

- 虽然不会Stop the world，但是会占用CPU资源，导致程序变慢
- 并发清除阶段由于用户线程继续运行而产生的垃圾（浮动垃圾），这部分垃圾只能到下一次 GC 时才能进行回收。由于浮动垃圾的存在，因此需要**预留内存**。如果预留的内存不够存放浮动垃圾，就会出现 Concurrent Mode Failure，这时虚拟机将临时启用 Serial Old 来替代 CMS。
- 标记清除算法导致的空间碎片，可能会无法找到足够大连续空间来分配当前对象，不得不提前触发一次 Full GC。

### 7. G1 收集器（必问）

G1的目标是在给定停顿时间收集最多的垃圾。它把堆划分成多个大小相等的独立区域（Region），以Region为回收单位，每个Region都可以是Eden，Survivor，老年代。通过记录每个 Region 垃圾回收的时间空间（这两个值是通过过去回收的经验获得），并维护一个优先列表，每次根据允许的收集时间，优先回收价值最大的 Region。

G1 收集器回收步骤和CMS有相似之处：

- 一开始是初始标记，标记GCROOTs。Stop the world。
- 然后并发标记，进行 GC Roots 可达性分析。
- 然后最终标记：修正在并发期间改变引用关系的对象的标记。Stop the world。
- 最后，回收会对各个 Region 中的回收价值和时间进行排序，根据设定的停顿时间来回收。把要回收的Region复制到空Region中。Stop the world。

具备如下特点：

- 整体来看是基于“标记 - 整理”算法实现的收集器，从局部（两个 Region 之间）上来看是基于“复制”算法实现的，这意味着运行期间不会产生内存空间碎片。
- 用户可以设置停顿时间

**跨Region引用怎么解决/如果避免全堆扫描？**

G1每个 Region 会有一个 Remembered Set，来记录对自己有引用的 Region。可达性分析的时候就可以避免全堆扫描。

CMS只有一个卡表维护跨代指针，跨代就会全堆扫描

# 三、内存分配与回收策略

## Minor GC 和 Full GC

- Minor GC：回收新生代，因为新生代对象存活时间很短，因此 Minor GC 会频繁执行，执行的速度一般也会比较快。
- Full GC：回收老年代和新生代，老年代对象其存活时间长，因此 Full GC 很少执行，执行速度会比 Minor GC 慢很多。

## 内存分配策略

1. **对象优先在 Eden 分配**

大多数情况下，对象在新生代 Eden 上分配，当 Eden 空间不够时，发起 Minor GC。

**2. 大对象直接进入老年代**

大对象是指需要连续内存空间的对象，最典型的大对象是那种很长的字符串以及数组。

经常出现大对象会提前触发垃圾收集以获取足够的连续空间分配给大对象。

-XX:PretenureSizeThreshold，大于此值的对象直接在老年代分配，避免在 Eden 和 Survivor 之间的大量内存复制。

**3. 长期存活的对象进入老年代**

为对象定义年龄计数器，对象在 Eden 出生并经过 Minor GC 依然存活，将移动到 Survivor 中，年龄就增加 1 岁，增加到一定年龄则移动到老年代中。

-XX:MaxTenuringThreshold 用来定义年龄的阈值。

**4. 动态对象年龄判定**

虚拟机并不是永远要求对象的年龄必须达到 MaxTenuringThreshold 才能晋升老年代，如果在 Survivor 中相同年龄所有对象大小的总和大于 Survivor 空间的一半，则年龄大于或等于该年龄的对象可以直接进入老年代，无需等到 MaxTenuringThreshold 中要求的年龄。

**5. 空间分配担保**

在发生 Minor GC 之前，虚拟机先检查老年代最大可用的连续空间是否大于新生代所有对象总空间，如果条件成立的话，那么 Minor GC 可以确认是安全的。

如果不成立的话虚拟机会查看 HandlePromotionFailure 的值是否允许担保失败，如果允许那么就会继续检查老年代最大可用的连续空间是否大于历次晋升到老年代对象的平均大小，如果大于，将尝试着进行一次 Minor GC；如果小于，或者 HandlePromotionFailure 的值不允许冒险，那么就要进行一次 Full GC。

## Full GC 的触发条件

Minor GC Eden 区满时，就触发一次 。Full GC：回收老年代和新生代，

**1. 调用 System.gc()**

有些NIO框架使用分配堆外内存会显式的调用System.gc(),导致频繁fullgc，可以设置参数ExplicitGCInvokesConcurrent

**2. 老年代空间不足**

老年代空间不足的常见场景为前文所讲的大对象直接进入老年代、长期存活的对象进入老年代等。

为了避免以上原因引起的 Full GC，应当尽量不要创建过大的对象以及数组。除此之外，可以通过 -Xmn 虚拟机参数调大新生代的大小，让对象尽量在新生代被回收掉，不进入老年代。还可以通过 -XX:MaxTenuringThreshold 调大对象进入老年代的年龄，让对象在新生代多存活一段时间。

**3.promotion failed 。minor gc时surviror二区装不下，老年代也没有空间。可能是老年代空间不足或者碎片太多。解决方法是增加survivor区比例，或者设置参数每隔几次CMS做一次复制整理**

**4 concurrent mode failureCMS并发清除阶段老年代空间不足。可以降低触发CMS的百分比**

**5直接内存fullgc**

**fullgc排查：**

先用top看一下整体cpu占用情况，如果有一个进程的cpu占用率特别高，就看他的日志。同时就用top -HP pid看一下该进程各个线程的情况，找到占用率最高的线程。

如果是java进程用jstack pid看一下。线程 id 都转换成了十六进制形式。找到对应的nid，就是对应线程。jstack是java虚拟机自带的一种堆栈跟踪工具。

然后查看是哪些对象占用了内存？用jmap命令来生成head dump文件。然后用eclipse的jmat看下哪个对象消耗内存

## 内存泄漏

1被集合类引用的对象无法被GC，对象属性被修改后，也无法用remove()删除。

像HashMap、Vector等的使用最容易出现内存泄露，这些静态变量的生命周期和应用程序一致，他们所引用的所有的对象Object也不能被释放，因为他们也将一直被Vector等引用着。

2监听器没有删除，或者各种连接没有close；

3ThreadLocalMap，key是虚引用，key被回收，value却还在

4单例模式持有外部引用

内部类的引用是比较容易遗忘的一种，而且一旦没释放可能导致一系列的后继类对象没有释放。此外程序员还要小心外部模块不经意的引用，例如程序员A负责A 模块，调用了B 模块的一个方法如：

public void registerMsg(Object b);

这种调用就要非常小心了，传入了一个对象，很可能模块B就保持了对该对象的引用，这时候就需要注意模块B 是否提供相应的操作去除引用。

# 四、类加载机制

类是在运行期间第一次使用时动态加载的，而不是一次性加载所有类。因为如果一次性加载，那么会占用很多的内存。

包括以下 7 个阶段：

- **加载（Loading）**
- **验证（Verification）**
- **准备（Preparation）**
- **解析（Resolution）**
- **初始化（Initialization）**
- 使用（Using）
- 卸载（Unloading）

## 类加载过程

包含了加载、验证、准备、解析和初始化这 5 个阶段。

**1. 加载**

加载是类加载的一个阶段，注意不要混淆。

加载过程完成以下三件事：

- 通过全类名获取定义该类的二进制字节流。
- 将该字节流表示的静态存储结构转换为方法区的运行时存储结构。
- 在内存中生成一个代表该类的 Class 对象，作为方法区中该类各种数据的访问入口。

其中二进制字节流可以从以下方式中获取：

- 从 ZIP 包读取，成为 JAR、EAR、WAR 格式的基础。
- 从网络中获取，最典型的应用是 Applet。
- 运行时计算生成，例如动态代理技术，在 java.lang.reflect.Proxy 使用 ProxyGenerator.generateProxyClass 的代理类的二进制字节流。
- 由其他文件生成，例如由 JSP 文件生成对应的 Class 类。

**2. 验证**

验证字节流中包含的信息是否符合JVM要求，并且不会危害JVM安全。

**3. 准备**

**准备阶段为静态变量（static）分配内存并设置初始值（final修饰不为0，其他为0），使用的是方法区的内存。（jdk8，存放在堆中）**

而实例变量会在对象实例化时随着对象一起被分配在堆中。应该注意到，实例化不是类加载的一个过程，类加载发生在所有实例化操作之前，并且类加载只进行一次，实例化可以进行多次。

初始值一般为 0 值，例如下面的类变量 value 被初始化为 0 而不是 123。

```
public static int value = 123;
```

如果类变量是常量，那么它将初始化为表达式所定义的值而不是 0。例如下面的常量 value 被初始化为 123 而不是 0。

```
public static final int value = 123;
```

**4. 解析**

**将常量池的符号引用替换为直接引用的过程。**

在类加载的解析阶段，会将其中的一部分符号引用转化为直接引用，这种解析能成立的前提是：方法在程序真正运行之前就有一个可确定的调用版本，并且这个方法的调用版本在运行期是不可改变的。编译期可知，运行期不可变”的方法，主要包括静态方法和私有方法两大类，

**5初始化**

**调<clinit>()方法 给静态变量赋值，执行静态代码块**

初始化阶段才真正开始执行类中定义的 Java 程序代码。初始化阶段是虚拟机执行类构造器 <clinit>() 方法的过程。在准备阶段，类变量已经赋过一次系统要求的初始值，而在初始化阶段，根据程序员通过程序制定的主观计划去初始化类变量和其它资源。

<clinit>() 是由编译器自动收集类中所有类变量的赋值动作和静态语句块中的语句合并产生的，编译器收集的顺序由语句在源文件中出现的顺序决定。特别注意的是，静态语句块只能访问到定义在它之前的类变量，定义在它之后的类变量只能赋值，不能访问。例如以下代码：

```
public class Test {
    static {
        i = 0;                // 给变量赋值可以正常编译通过
        System.out.print(i);  // 这句编译器会提示“非法向前引用”
    }
    static int i = 1;
}
```

由于父类的 <clinit>() 方法先执行，也就意味着父类中定义的静态语句块的执行要优先于子类。例如以下代码：

```
static class Parent {
    public static int A = 1;
    static {
        A = 2;
    }
}

static class Sub extends Parent {
    public static int B = A;
}

public static void main(String[] args) {
     System.out.println(Sub.B);  // 2
}
```

接口中不可以使用静态语句块，但仍然有类变量初始化的赋值操作，因此接口与类一样都会生成 <clinit>() 方法。但接口与类不同的是，执行接口的 <clinit>() 方法不需要先执行父接口的 <clinit>() 方法。只有当父接口中定义的变量使用时，父接口才会初始化。另外，接口的实现类在初始化时也一样不会执行接口的 <clinit>() 方法。

虚拟机会保证一个类的 <clinit>() 方法在多线程环境下被正确的加锁和同步，如果多个线程同时初始化一个类，只会有一个线程执行这个类的 <clinit>() 方法，其它线程都会阻塞等待，直到活动线程执行 <clinit>() 方法完毕。如果在一个类的 <clinit>() 方法中有耗时的操作，就可能造成多个线程阻塞，在实际过程中此种阻塞很隐蔽。

## 类初始化时机

**主动引用**

虚拟机规范中并没有强制约束何时进行加载，但是规范严格规定了有且只有下列五种情况必须对类进行初始化（加载、验证、准备都会随之发生）：

- 遇到 new、getstatic、putstatic、invokestatic 这四条字节码指令时，如果类没有进行过初始化，则必须先触发其初始化。最常见的生成这 4 条指令的场景是：**使用 new 实例化对象的时候**；读取或设置一个**类的静态字段**（被 final 修饰、已在编译期把结果放入常量池的静态字段除外）的时候；以及调用一个**类的静态方法**的时候。
- 使用 java.lang.reflect 包的方法对类进行**反射**调用的时候，如果类没有进行初始化，则需要先触发其初始化。
- 当初始化一个类的时候，如果发现其**父类**还没有进行过初始化，则需要先触发其父类的初始化。
- 当虚拟机启动时，用户需要指定一个要执行的主类（包含 main() 方法的那个类），虚拟机会先初始化这个主类；
- 当使用 JDK 1.7 的动态语言支持时，如果一个 java.lang.invoke.MethodHandle 实例最后的解析结果为 REF_getStatic, REF_putStatic, REF_invokeStatic 的方法句柄，并且这个方法句柄所对应的类没有进行过初始化，则需要先触发其初始化；

**被动引用**

以上 5 种场景中的行为称为对一个类进行主动引用。除此之外，所有引用类的方式都**不会触发初始化，称为被动引用。**被动引用的常见例子包括：

- 通过子类引用父类的静态字段，不会导致子类初始化。

```
System.out.println(SubClass.value);  // value 字段在 SuperClass 中定义
```

- 通过数组定义来引用类，不会触发此类的初始化。该过程会对数组类进行初始化，数组类是一个由虚拟机自动生成的、直接继承自 Object 的子类，其中包含了数组的属性和方法。

```
SuperClass[] sca = new SuperClass[10];
```

- 常量在编译阶段会存入调用类的常量池中，本质上并没有直接引用到定义常量的类，因此不会触发定义常量的类的初始化。

```
System.out.println(ConstClass.HELLOWORLD);
```

## 类相等条件

条件是，**类本身相等，而且使用同一个类加载器进行加载。**这是因为每一个类加载器都拥有一个独立的类名称空间。

这里的相等，包括类的 Class 对象的 equals() 方法、isAssignableFrom() 方法、isInstance() 方法的返回结果为 true，也包括使用 instanceof 关键字做对象所属关系判定结果为 true。

## 类加载器分类

从 Java 虚拟机的角度来讲，只存在以下两种不同的类加载器：

- 启动类加载器（Bootstrap ClassLoader），使用 C++ 实现，是虚拟机自身的一部分；
- 所有其它类的加载器，使用 Java 实现，独立于虚拟机，继承自抽象类 java.lang.ClassLoader。

从 Java 开发人员的角度看，类加载器可以划分得更细致一些：

- **启动类加载器（Bootstrap ClassLoader）由C++编写。加载JAVA核心类库。**此类加载器负责将存放在 <JRE_HOME>\lib 目录中的，或者被 -Xbootclasspath 参数所指定的路径中的，并且是虚拟机识别的（仅按照文件名识别，如 rt.jar，名字不符合的类库即使放在 lib 目录中也不会被加载）类库加载到虚拟机内存中。启动类加载器无法被 Java 程序直接引用，用户在编写自定义类加载器时，如果需要把加载请求委派给启动类加载器，直接使用 null 代替即可。
- **扩展类加载器（Extension ClassLoader）主要加载JAVA中的一些拓展类**这个类加载器是由 ExtClassLoader（sun.misc.Launcher$ExtClassLoader）实现的。它负责将 <JAVA_HOME>/lib/ext 或者被 java.ext.dir 系统变量所指定路径中的所有类库加载到内存中，开发者可以直接使用扩展类加载器。
- **应用程序类加载器（Application ClassLoader）加载ClassPath下自己些的类**这个类加载器是由 AppClassLoader（sun.misc.Launcher$AppClassLoader）实现的。由于这个类加载器是 ClassLoader 中的 getSystemClassLoader() 方法的返回值，因此一般称为系统类加载器。它负责加载用户类路径（ClassPath）上所指定的类库，开发者可以直接使用这个类加载器，如果应用程序中没有自定义过自己的类加载器，一般情况下这个就是程序中默认的类加载器。

## 双亲委派模型

**类加载器加载类时，先让父类加载器加载，依次向上递归，只有父类加载器无法加载时，才自己去加载。**

除了顶层的启动类加载器，其它的类加载器有父类加载器。

**好处**

每个类只被加载一次，防止一个类出现多个.CLASS文件。

**实现**

以下是抽象类 java.lang.ClassLoader 的代码片段，其中的 loadClass() 方法运行过程如下：先检查类是否已经加载过，如果没有则让父类加载器去加载。当父类加载器加载失败时抛出 ClassNotFoundException，此时尝试自己去加载。

```
public abstract class ClassLoader {
    // The parent class loader for delegation
    private final ClassLoader parent;

    public Class<?> loadClass(String name) throws ClassNotFoundException {
        return loadClass(name, false);
    }

    protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
        synchronized (getClassLoadingLock(name)) {
            // First, check if the class has already been loaded
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                try {
                    if (parent != null) {
                        c = parent.loadClass(name, false);
                    } else {
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }

                if (c == null) {
                    // If still not found, then invoke findClass in order
                    // to find the class.
                    c = findClass(name);
                }
            }
            if (resolve) {
                resolveClass(c);
            }
            return c;
        }
    }

    protected Class<?> findClass(String name) throws ClassNotFoundException {
        throw new ClassNotFoundException(name);
    }
}
```

**自定义类加载器实现**

以下代码中的 FileSystemClassLoader 是自定义类加载器，继承自 java.lang.ClassLoader，用于加载文件系统上的类。它首先根据类的全名在文件系统上查找类的字节代码文件（.class 文件），然后读取该文件内容，最后通过 defineClass() 方法来把这些字节代码转换成 java.lang.Class 类的实例。

java.lang.ClassLoader 的 loadClass() 实现了双亲委派模型的逻辑，自定义类加载器一般不去重写它，但是需要重写 findClass() 方法。

```
public class FileSystemClassLoader extends ClassLoader {

    private String rootDir;

    public FileSystemClassLoader(String rootDir) {
        this.rootDir = rootDir;
    }

    protected Class<?> findClass(String name) throws ClassNotFoundException {
        byte[] classData = getClassData(name);
        if (classData == null) {
            throw new ClassNotFoundException();
        } else {
            return defineClass(name, classData, 0, classData.length);
        }
    }

    private byte[] getClassData(String className) {
        String path = classNameToPath(className);
        try {
            InputStream ins = new FileInputStream(path);
            ByteArrayOutputStream baos = new ByteArrayOutputStream();
            int bufferSize = 4096;
            byte[] buffer = new byte[bufferSize];
            int bytesNumRead;
            while ((bytesNumRead = ins.read(buffer)) != -1) {
                baos.write(buffer, 0, bytesNumRead);
            }
            return baos.toByteArray();
        } catch (IOException e) {
            e.printStackTrace();
        }
        return null;
    }

    private String classNameToPath(String className) {
        return rootDir + File.separatorChar
                + className.replace('.', File.separatorChar) + ".class";
    }
}
```

## 对象实例化顺序

1，父类的静态成员变量和静态代码块加载
2，子类的静态成员变量和静态代码块加载
3，父类成员变量和方法块加载
4，父类的构造函数加载
5，子类成员变量和方法块加载
6，子类的构造函数加载

# 五 调优

xmx堆最大值，xms为堆初始值，xmn为新生代内存，xss为栈内存大小。survivorratio为一个survivor区的比例，一般是8，也就是eden和from，to比例为8:1:1

jps 查看java进程信息

jinfo -flag PrintGCDetails 查看参数，打印Gc细节

jstat 查看JVM各个分区的使用情况，gc次数

![img](https://images2017.cnblogs.com/blog/1139681/201710/1139681-20171030182156699-1474991178.png)

jstack pid 用来生成线程快照，查看线程正在执行的方法

https://www.cnblogs.com/yangzhixue/p/11989863.html

jmap -dump:format=b,file=文件名 [pid] 导出JVM中内存信息

**系统慢怎么排查？**

**先用top看一下整体cpu占用情况，如果有一个进程的cpu占用率特别高，就看他的日志。同时就用top -HP pid看一下该进程各个线程的情况，找到占用率最高的线程。**

**用jstack pid看一下。把线程 id 转为十六进制，找到对应的nid，就是对应线程，然后排查代码栈**。



**OOM排查？**

OOM主要有四种 Java heap space  堆内存不足 GC overhead limit exceeded GC时间超过98%，回收空间不足2% Metaspace 加载的类过多，可能是反射用多了   unable to create native thread 线程创的太多了。

**pe -ef | frep 找到java进程pid，然后用jstat 查看jvm各个分区使用情况，gc次数**

**然后查看是哪些对象占用了内存？用jmap命令来生成head dump文件。然后用eclipse的jmat看下哪个对象消耗内存多，有没有大对象持续存活。**

https://blog.csdn.net/weixin_42447959/article/details/81637909

# 问题

**分区**

说一下JVM的内存结构

JVM分区？

说说jvm内存模型 

Jvm的内存分配---各大区域的内容和区别---

**垃圾回收**

jvm垃圾回收，具体问了g1的细节（很细）

jvm的回收算法

垃圾回收算法？

说一下 JVM 的垃圾回收器  CMS G1 

说一下 CMS 的优缺点 

垃圾回收机制，详细问了 CMS

Java虚拟机的垃圾回收算法G1与CMS的区别

回收的机制是什么？凭什么判断一个对象会被回收？ 

说一下 GC Roots 包含哪些内容？ 

新生代与老生代

什么情况下会发生新生代 gc？ 

FULLGC在什么条件下会发生

Eden 区满了之后会怎么样呢？说一下这个处理流程 

Eden 区 和 From Survivor 区中经过 gc 后还能存活的对象移动到 To Survivor 区后，那第二次 GC 时是取 Eden 区和 From Survivor 进行 gc 还是说取 Eden 区和 To Survivor 区？ 

垃圾回收有哪些算法，他们的区别和应用场景---新生代和老年代分别使用哪些回收算法

**什么情况下会出发重量级的全局的gc，如何调优避免**

【分析】

1.minor gc很频繁，但时间短所以问题不大，触发原因基本都是申请空间失败。

 

2.偶尔有System.gc()，时间大概1分钟。代码中没有显式调用，基本确定是监控程序RMI访问触发的。可以加参数禁用 -XX:+DisableExplicitGC 。

 

3.时常有promotion failed，即在minor gc时年轻代的存活区空间不足而进入老年代，老年代又空间不足而触发full gc。时间大概3分钟。解决思路一种是增大存活区，一种则相反是去掉存活区增大老年代。相关参数一般有：

-XX:SurvivorRatio=32 存活区除以伊甸区的比率，存活区有from和to两个，所以这里的意思是单个存活区占年轻代的1/34；

-XX:OldSize=60M 老年代大小；

-XX:MaxTenuringThreshold=15 即多少次minor gc后存活的年轻代对象会晋升老年代。

 

4.经常有concurrent mode failure，即CMS执行过程中老年代空间不足，这时会变成Serial Old收集器导致更长时间的停顿。时间大概5分钟。其中引发这一问题的情况可能是浮动垃圾太多、可能是CMS收集器本身也占用堆空间、也可能是老年代太多碎片，但都是CMS收集器的特性导致的。相关配置一般有：

-XX:CMSInitiatingOccupancyFraction=80 即老年代满80%时触发CMS(full gc)，调高则full gc相对减少，调低则full gc处理得比较快；

-XX:UseCMSCompactAtFullCollection 或 -XX:CMSFullGCsBeforeCompaction=5 即full gc前或后做碎片整理。

\7. 如何查看gc的情况

 

详细的 GC 日志

借助 Linux 平台下的 iostat、vmstat、netstat、mpstat 等命令监控系统情况

使用 GCHisto 这个 GC 图形用户界面工具，可以统计出 Minor GC 及 Full GC 频率及时长分布，可参考:http://blog.csdn.net/wenniuwuren/article/details/50760259

查看 GC 日志中是否出现了上述的典型内存异常问题（promotion failed, concurrent mode failure），整体来说把上述两个典型内存异常情况控制在可接受的发生频率即可，对 CMS 碎片问题来说杜绝以上问题似乎不太可能，只能靠 G1 来解决了

是不是 JVM 本身的 bug 导致的

如果程序没问题，参数调了几次还是不能解决，可能说明流量太大，需要加机器把压力分散到更



**类加载**

类加载机制：为什么需要双亲委派模型，有哪些类加载器，项目中有用过类加载器吗，如何做的。

问下基础吧，类加载

**实战**

查看堆栈内存分配等用哪些工具或者指令，平时使用了什么，如果发现CPU占用很大，而内存使用很少，用什么工具或者指令查看，什么原因

# 参考资料

- 周志明. 深入理解 Java 虚拟机 [M]. 机械工业出版社, 2011.
- [Chapter 2. The Structure of the Java Virtual Machine](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html#jvms-2.5.4)
- [Jvm memory](https://www.slideshare.net/benewu/jvm-memory) [Getting Started with the G1 Garbage Collector](http://www.oracle.com/webfolder/technetwork/tutorials/obe/java/G1GettingStarted/index.html)
- [JNI Part1: Java Native Interface Introduction and “Hello World” application](http://electrofriends.com/articles/jni/jni-part1-java-native-interface/)
- [Memory Architecture Of JVM(Runtime Data Areas)](https://hackthejava.wordpress.com/2015/01/09/memory-architecture-by-jvmruntime-data-areas/)
- [JVM Run-Time Data Areas](https://www.programcreek.com/2013/04/jvm-run-time-data-areas/)
- [Android on x86: Java Native Interface and the Android Native Development Kit](http://www.drdobbs.com/architecture-and-design/android-on-x86-java-native-interface-and/240166271)
- [深入理解 JVM(2)——GC 算法与内存分配策略](https://crowhawk.github.io/2017/08/10/jvm_2/)
- [深入理解 JVM(3)——7 种垃圾收集器](https://crowhawk.github.io/2017/08/15/jvm_3/)
- [JVM Internals](http://blog.jamesdbloom.com/JVMInternals.html)
- [深入探讨 Java 类加载器](https://www.ibm.com/developerworks/cn/java/j-lo-classloader/index.html#code6)
- [Guide to WeakHashMap in Java](http://www.baeldung.com/java-weakhashmap)
- [Tomcat example source code file (ConcurrentCache.java)](https://alvinalexander.com/java/jwarehouse/apache-tomcat-6.0.16/java/org/apache/el/util/ConcurrentCache.java.shtml)