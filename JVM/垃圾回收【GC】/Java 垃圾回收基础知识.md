# Java 垃圾回收基础知识

## 1.如何判断对象可以回收

### 1.1 引用计数法

> 因为循环引用的存在，因此 Java 虚拟机不使用引用计数算法

给对象添加一个引用计数器，当对象增加一个引用时计数器加 1，引用失效时计数器减 1。引用计数为 0 的对象可被回收。

两个对象出现**循环引用**的情况下，此时<u>引用计数器永远不为 0</u>，导致无法对它们进行回收

![image-20220422143435579](https://adao-blog-pic.oss-cn-beijing.aliyuncs.com/pic/blog/image-20220422143435579.png)

###  1.2 可达性分析算法

> Java 虚拟机使用该算法来判断对象是否可被回收

通过 GC Roots 作为起始点的引用链进行搜索，能够到达到的对象都是存活的，不可达的对象可被回收。

#### 哪些对象可以是`GC Root`呢？

> 可以借助工具来查看运行程序的GC Root
>
> Eclipse 出品的 Memory Analyzer(MAT),可以用来分析GC Root

- MAT的使用

  1.  借助`jmap`指令生成二进制文件

  ```shell
  jmap -dump:format-b,live,file=1.bin 21384
  ```

  ![image-20220422145047516](https://adao-blog-pic.oss-cn-beijing.aliyuncs.com/pic/blog/image-20220422145047516.png) 2. 将二进制文件导入MAT，进行解析查看

![image-20220422145132100](https://adao-blog-pic.oss-cn-beijing.aliyuncs.com/pic/blog/image-20220422145132100.png)

可以看到：

在 Java 中 GC Roots 一般包含以下内容:

- 系统类： 这些类都是由启动类加载器加载的类
- 本地方法栈中引用的对象
- 活动线程中的引用的对象
- Monitor ，锁对象。锁对象不能被清除，锁对象如果被清除，那无法解锁，线程就无法释放锁，进程无法执行

### 1.3 四种引用

> 引用的类型 一说是四种，一说是五种。
>
> 四种是指：强引用，软引用，弱引用，虚引用
>
> 五种是指：（除上四种以外）终结器引用

#### 1.强引用

- 只有所有GC Roots 对象都不通过【强引用】引用该对象，该对象才能被垃圾回收

正常的创建对象，建立的就是强引用。

被强引用关联的对象不会被回收

```java
Object obj = new Object();
```

#### 2.软引用

- 仅有软引用引用该对象时，在垃圾回收后，内存扔不足时会再次发出法及回收，回收软引用对象

被软引用关联的对象，如果在**垃圾回收后发现<u>内存仍然不够使用</u>**，则会将该关联对象回收释放。

```java
/**
 * 软引用演示
 * -Xmx20M -XX:+PrintGCDetails
 */
public class SoftRef {

    private static final int _4MB = 4*1024*1024;

    public static void main(String[] args) throws IOException {
//        reference();
        softReference();
    }
    /**
     * 强引用
     */
    public static void reference() throws IOException {
        ArrayList<byte[]> list = new ArrayList<>();
        for (int i = 0; i < 5; i++) {
            list.add(new byte[_4MB]);
        }
        System.in.read();
    }

    /**
     * 软引用
     */
    public static void softReference(){
        List<SoftReference<byte[]>> list = new ArrayList<>();
        for (int i = 0; i < 5; i++) {
            SoftReference<byte[]> softReference = new SoftReference<>(new byte[_4MB]);
            list.add(softReference);
            System.err.println("加入第"+(i+1)+"个对象："+softReference.get());
            for (int j = 0; j < list.size(); j++) {
                System.out.print(list.get(j).get()+" | ");
            }
            System.out.println();
        }
    }
}

```

执行强引用方法的结果打印如下

```java
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
	at com.chenayin.jvm.reference.SoftReference.reference(SoftReference.java:24)
	at com.chenayin.jvm.reference.SoftReference.main(SoftReference.java:15)
```

执行软引用方法结果如下

```java
加入第1个对象：[B@1b6d3586
加入第2个对象：[B@4554617c
加入第3个对象：[B@74a14482
加入第4个对象：[B@1540e19d
[B@1b6d3586 | 
[B@1b6d3586 | [B@4554617c | 
[B@1b6d3586 | [B@4554617c | [B@74a14482 | 
 // 此处前三个对象都能顺利加入，此时堆内存够用
 // 放入第四个对象之前，发生了一次MinorGC ，将第四个对象放入
[GC (Allocation Failure) [PSYoungGen: 1864K->488K(6144K)] 14152K->12960K(19968K), 0.0007951 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
  // 此时list中已有四个元素
[B@1b6d3586 | [B@4554617c | [B@74a14482 | [B@1540e19d | 
  // 放入四个元素之后，要放入第五个元素，内存不足 发生了一次MinorGc,
[GC (Allocation Failure) --[PSYoungGen: 4696K->4696K(6144K)] 17168K->17248K(19968K), 0.0008782 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
  // 内存仍然不够，发生Full GC
[Full GC (Ergonomics) [PSYoungGen: 4696K->4524K(6144K)] [ParOldGen: 12552K->12492K(13824K)] 17248K->17016K(19968K), [Metaspace: 3306K->3306K(1056768K)], 0.0051148 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
  // 再次尝试放入发现内存仍然不够，触发Minor GC
[GC (Allocation Failure) --[PSYoungGen: 4524K->4524K(6144K)] 17016K->17032K(19968K), 0.0005696 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
  // 内存还是不够，触发Full GC，将软引用对象释放
[Full GC (Allocation Failure) [PSYoungGen: 4524K->0K(6144K)] [ParOldGen: 12508K->614K(8704K)] 17032K->614K(14848K), [Metaspace: 3306K->3306K(1056768K)], 0.0061536 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
 // 四个软引用对象释放之后，内存够用。
null | null | null | null | [B@677327b6 | 
…………
加入第5个对象：[B@677327b6
```

#### 3.弱引用

被弱引用关联的对象，**不管内存是否充足足够使用**，都会将该对象回收释放。

```java
	/**
     * 弱引用
     */
    public static void weakReference(){
        List<WeakReference<byte[]>> list = new ArrayList<>();
        for (int i = 0; i < 5; i++) {
            WeakReference<byte[]> weakReference = new WeakReference<>(new byte[_4MB]);
            list.add(weakReference);
            System.err.println("加入第"+(i+1)+"个对象："+weakReference.get());
            for (int j = 0; j < list.size(); j++) {
                System.out.print(list.get(j).get()+" | ");
            }
            System.out.println();
        }
    }
```

为了更好的看到效果，将堆内存大小设置为更小的值 15M

执行结果如下：

```java
[B@1b6d3586 | 
[B@1b6d3586 | [B@4554617c | 
// 只要触发GC，都会释放掉弱引用内存
[GC (Allocation Failure) [PSYoungGen: 1730K->488K(4608K)] 9922K->8864K(15872K), 0.0013371 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[GC (Allocation Failure) [PSYoungGen: 488K->488K(4608K)] 8864K->8896K(15872K), 0.0006080 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[Full GC (Allocation Failure) [PSYoungGen: 488K->0K(4608K)] [ParOldGen: 8408K->605K(6144K)] 8896K->605K(10752K), [Metaspace: 3258K->3258K(1056768K)], 0.0090827 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
 // 经历GC之后，不管内存是否够用，弱引用对象都被释放
null | null | [B@74a14482 | 
null | null | [B@74a14482 | [B@1540e19d | 
[GC (Allocation Failure) [PSYoungGen: 245K->192K(4608K)] 9043K->8997K(15872K), 0.0006770 secs] [Times: user=0.02 sys=0.00, real=0.00 secs] 
[GC (Allocation Failure) [PSYoungGen: 192K->96K(4608K)] 8997K->8901K(15872K), 0.0004742 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[Full GC (Allocation Failure) [PSYoungGen: 96K->0K(4608K)] [ParOldGen: 8805K->608K(8192K)] 8901K->608K(12800K), [Metaspace: 3293K->3293K(1056768K)], 0.0055623 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
null | null | null | null | [B@677327b6 | 
```

> 不论是软、弱引用都可以配合引用队列来使用，软、弱引用对象本身也是占据空间的，当需要释放时可以借助引用队列，遍历释放。

```java
    /**
     * 软引用[关联引用队列]
     */
    public static void softReferenceWithQueue(){
        ReferenceQueue<byte[]> queue = new ReferenceQueue<>();
        List<SoftReference<byte[]>> list = new ArrayList<>();
        for (int i = 0; i < 5; i++) {
            // 关联了引用队列，当软引用所关联的byte[]被回收，软引用自己会加入到引用队列queue
            SoftReference<byte[]> softReference = new SoftReference<>(new byte[_4MB],queue);
            list.add(softReference);
            System.err.println("加入第"+(i+1)+"个对象："+softReference.get());
        }
        // 获取队列首个元素【也就是第一个放入队列的元素】
        Reference<? extends byte[]> poll = queue.poll();
        while (poll!=null){
            // 元素不为空，就将该元素从list集合中移除
            list.remove(poll);
            // 并继续获取队列中的元素，直到队列元素为空。
            poll = queue.poll();
        }
        for (SoftReference<byte[]> softReference : list) {
            System.out.println(softReference.get());
        }
    }

```

未关联引用队列之前的执行结果

```java
加入第1个对象：[B@1b6d3586
加入第2个对象：[B@4554617c
加入第3个对象：[B@74a14482
加入第4个对象：[B@1540e19d
加入第5个对象：[B@677327b6
null
null
null
null
[B@677327b6
```

关联引用队列后执行结果

```java
加入第1个对象：[B@1b6d3586
加入第2个对象：[B@4554617c
加入第3个对象：[B@74a14482
加入第4个对象：[B@1540e19d
加入第5个对象：[B@677327b6
[B@677327b6
```

#### 4.虚引用

- 必须配合引用队列使用，主要配合`ByteBuffer`使用，被引用对象回收时，会将虚引用入队，由`Reference Handler`线程调用虚引用相关方法释放直接内存

虚引用必须借助引用队列来实现。当被虚引用关联的对象被回收时，虚引用对象本身加入到引用队列，由`ReferenceHandler`定时执行任务，去执行真正的方法。



以`ByteBuffer`为例，当虚引用`Cleaner`关联的对象`ByteBuffer`被回收释放，则会触发`Cleaner`的clean方法，执行`unsafe.freeMemory()`方法

![image-20220422174424128](https://adao-blog-pic.oss-cn-beijing.aliyuncs.com/pic/blog/image-20220422174424128.png)

![image-20220422174558551](https://adao-blog-pic.oss-cn-beijing.aliyuncs.com/pic/blog/image-20220422174558551.png)

#### 5.终结器引用

- 无需手动编码，但其内部配合引用队列使用，在垃圾回收时，终结器引用入队（被引用对象暂时不被回收），再由`Finalizer`线程通过终结器引用找到被引用对象并调用它的`finalize()`方法，第二次`GC` 才能回收被引用对象。

当一个对象可被回收时，如果需要执行该对象的 finalize() 方法；

那该终结器引用对象会加入到引用队列，被关联对象第一次被回收，优先级很低的线程查看引用队列，如果发现了终结器引用，则会执行被关联对象的finalize方法，下次垃圾回收才会释放掉该对象。

## 2.垃圾回收算法

> 垃圾回收中提到的标记，标记的是活动的对象，未被标记的是非活动对象。未被标记的对象会被回收。

### 2.1标记-清除

![image-20220422181343877](https://adao-blog-pic.oss-cn-beijing.aliyuncs.com/pic/blog/image-20220422181343877.png)

优点：速度快

缺点：内存碎片多，连续的空间少。

### 2.2标记-整理

![image-20220422181358239](https://adao-blog-pic.oss-cn-beijing.aliyuncs.com/pic/blog/image-20220422181358239.png)

优点：内存碎片少

缺点：速度慢

### 2.3复制

![image-20220422181406226](https://adao-blog-pic.oss-cn-beijing.aliyuncs.com/pic/blog/image-20220422181406226.png)

优点：内存碎片少

缺点：需要双倍的内存空间，复制，移动过程效率下降。

### 2.4分代

现在的商业虚拟机采用分代收集算法，它根据对象存活周期将内存划分为几块，不同块采用适当的收集算法。

一般将堆分为新生代和老年代。

- 新生代使用: 复制算法
- 老年代使用: 标记 - 清除 或者 标记 - 整理 算法