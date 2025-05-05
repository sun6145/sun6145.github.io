---
layout: post
title: 高并发设计模式
abbrlink: 3a9815be06404faab2dbc6ffec1ee0ef
tags:
  - juc
categories:
  - JDK
  - 多线程编程
date: 1745339269634
updated: 1746417858949
---

介绍在高并发场景常用的几种模式：线程安全的单例模式、ForkJoin 模式、生产者-消费者模式、Master-Worker 模式和 Future 模式。

## <!-- more -->

## 线程安全的单例模式

单例模式是常见的一种设计模式，一般用于全局对象管理，比如 XML 读写实例、系统配置实例、任务调度实例、数据库连接池实例等。

### 从饿汉式单例到懒汉式单

按照单例对象被初始化的时机，单例模式一般分为懒汉式、饿汉式两种。饿汉式单例在类被加载时就直接被初始化，参考代码具体如
下：

```java
public class Singleton1 {
    private Singleton1() {
    } // 私有构造器

    // 静态成员
    private static final Singleton1 single = new Singleton1();

    public static Singleton1 getInstance() {
        return single;
    }
}
```

饿汉单例模式的优点是足够简单、安全。其缺点是：单例对象在类被加载时，实例就直接被初始化了。很多时候，在类被加载时并不需要进行单例初始化，所以需要对单例的初始化予以延迟，一直到实
例使用的时候初始化。**在使用的时候才对单例进行初始化，这就是懒汉单例模式。** 懒汉单例模式的参考代码如下：

```java
public class ASingleton {
    static ASingleton instance; // 静态成员

    // 私有构造器
    private ASingleton() {
    }

    // 获取单例的方法
    static ASingleton getInstance() {
        if (instance == null) //①
        {
            instance = new ASingleton(); //②
        }
        return instance;
    }
}
```

以上懒汉单例模式的实现大家都很熟悉，估计也编写过类似的代码。以上参考实现在单线程场景中是合理的、安全的。在第一次被调用时，getInstance()方法会新建一个 ASingleton 实例，但之后访问时
返回的是第一次新建的 ASingleton 实例。多线程并发访问 getInstance()方法时，问题就出来了：不同的线程有可能同时进入代码 ① 处的条件判断，多次执行代码 ②，从而新建多个 ASingleton 对象。

### 使用内置锁保护懒汉式单例

如何确保单例只创建一次，可以使用 synchronized 内置锁进行单例获取同步，确保同时只能有一个线程进入临界区执行。

```java
// 使用synchronized内置锁进行单例获取同步
public class BSingleton {
    static BSingleton instance; // 保持单例的静态成员

    private BSingleton() {
    } // 私有构造器

    // 获取单例的方法
    static synchronized BSingleton getInstance() {
        if (instance == null) {
            instance = new BSingleton();
        }
        return instance;
    }
}
```

getInstance()方法加 synchronized 关键字之后，可以保证在并发执行时不出错。问题是：每次执行 getInstance()方法都要用到同步，在争用激烈的场景下，内置锁会升级为重量级锁，开销大、性能差，所以不推荐高并发线程使用这种方式的单例模式。

### 双重检查锁单例模式

实际上，单例模式的加锁操作只有单例在第一次创建的时候才需要用到，之后的单例获取操作都没必要再加锁。所以，可以先判断单例对象是否已经被初始化，如果没有，加锁后再初始化，这种模式被
叫作双重检查锁（Double Checked Locking）单例模式。示例代码如下：

```java
// 双重检查的懒汉式单例模式
public class ESingleton {
    static ESingleton instance;// 保持单例的静态成员

    private ESingleton() {
    } // 私有构造器

    static synchronized ESingleton getInstance() {
        if (instance == null) // 检查①
        {
            synchronized (ESingleton.class)// 加锁
            {
                if (instance == null) // 检查②
                {
                    instance = new ESingleton();
                }
            }
        }
        return instance;
    }
}
```

- 检查单例对象是否被初始化，如果已被初始化，就立即返回单例对象。这是第一次检查，对应示例代码中的检查 ①，此次检查不需要使用锁进行线程同步，用于提高获取单例对象的性能。
- 如果单例没有被初始化，就试图进入临界区进行初始化操作，此时才去获取锁
- 进入临界区之后，再次检查单例对象是否已经被初始化，如果还没被初始化，就初始化一个实例。这是第二次检查，对应代码中的检查 ②，此次检查在临界区内进行。
  - 为什么在临界区内还需要执行一次检查呢？
  - 答案是：在多个线程竞争的场景下，可能同时不止一个线程通过了第一次检查（检查 ①），此时第一个通过“检查 ①”的线程将首先进入临界区，而其他通过“检查 ①”的线程将被阻塞，在第一个线程实例化单例对象释放锁之后，其他线程可能获取到锁进入临界区，实际上单例已经被初始化了，所以哪怕进入了临界区，其他线程并没有办法通过“检查 ②”的条件判断，无法执行重复的初始化。

双重检查不仅避免了单例对象在多线程场景中的反复初始化，而且除了初始化的时候需要现加锁外，后续的所有调用都不需要加锁而直接返回单例，从而提升了获取单例时的性能。

### 使用双重检查锁+volatile

表面上，使用双重检查锁机制的单例模式一切看上去都很完美，其实并不是这样的。那么问题出现在哪里呢？下面这行代码实际大有玄机：

```java
//初始化单例
instance = new Singleton()
```

这行初始化单例代码转换成汇编指令（具有原子性的指令）后，大致会细分成三个：

1. 分配一块内存 M。
2. 在内存 M 上初始化 Singleton 对象。
3. M 的地址赋值给 instance 变量。

编译器、CPU 都可能对没有内存屏障、数据依赖关系的操作进行重排序，上述的三个指令优化后可能就变成了这样：

1. 分配一块内存 M。
2. 将 M 的地址赋值给 instance 变量。
3. 在内存 M 上初始化 Singleton 对象。

这里假设两个线程以下面的次序执行：

1. 线程 A 先执行 getInstance()方法，当执行到分配一块内存并将地址赋值给 M 后，恰好发生了线程切换。此时，线程 A 还没来得及将 M 指向的内存初始化。
2. 线程 B 刚进入 getInstance()方法，判断 if 语句 instance 是否为空，此时的 instance 不为空，线程 B 直接获取到了未初始化的 instance 变量。
3. 由于线程 B 得到的是一个未初始化完全的对象，因此访问 instance 成员变量的时候可能发生异常。

如何确保线程 B 获取的是一个完成初始化的单例呢？**可以通过 volatile 禁止指令重排**。双重检查锁+volatile 相结合的单例模式实现大致的代码如下：

```java
public class Singleton {
    // 使用 volatile 关键字保证可见性和禁止指令重排序
    private static volatile Singleton instance;

    private Singleton() {
        // 私有构造函数，防止外部实例化
    }

    public static Singleton getInstance() {
        // 第一次检查，如果 instance 不为 null，直接返回
        if (instance == null) {
            // 同步块，保证多线程环境下只创建一个实例
            synchronized (Singleton.class) {
                // 第二次检查，防止多个线程同时进入同步块时重复创建实例
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

### 静态内部类方式

虽然通过双重检查锁+volatile 相结合的方式能实现高性能、线程安全的单例模式，但是该实现的底层原理比较复杂，写法烦琐。另一种易于理解、编程简单的单例模式的实现为使用静态内部类实例懒汉
式单例模式

```java
public class Singleton {
    // 私有构造函数，防止外部实例化
    private Singleton() {
    }

    // 静态内部类，用于持有单例对象
    private static class SingletonHolder {
        // 使用 final 关键字确保该实例一旦创建就不会被修改
        private static final Singleton INSTANCE = new Singleton();
    }

    // 提供获取单例的方法
    public static Singleton getInstance() {
        return SingletonHolder.INSTANCE;
    }
}
```

使用静态内部类实现懒汉式单例模式只有在 getInstance()被调用时才去加载内部类并且初始化单例，该方式既解决了线程安全问题，又解决了写法烦琐问题。

## Master-Worker 模式

Master-Worker 模式是一种常见的高并发模式，它的核心思想是任务的调度和执行分离，调度任务的角色为 Master，执行任务的角色为 Worker，Master 负责接收、分配任务和合并（Merge）任务结果，
Worker 负责执行任务。

Master-Worker 模式是一种归并类型的模式。举一个例子，在 TCP 服务端的请求处理过程中，大量的客户端连接相当于大量的任务，Master 需要将这些任务存储在一个任务队列中，然后分发给各个 Worker，每个 Worker 是一个工作线程，负责完成连接的传输处理。

### 参考实现

假设一个场景，需要执行 N 个任务，将这些任务的结果进行累加求和，如果任务太多，就可以采用 Master-Worker 模式来实现。Master 持有 workerCount 个 Worker，并且负责接收任务，然后分发给 Worker，最后在回调函数中对 Worker 的结果进行归并求和。

```java
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;

// 定义Worker线程类
class Worker implements Runnable {
    private List<Integer> subTask;

    public Worker(List<Integer> subTask) {
        this.subTask = subTask;
    }

    @Override
    public void run() {
        int subResult = 0;
        for (Integer i : subTask) {
            subResult += i;
        }
        // 假设这里有一个机制可以将结果返回给Master，这里简化处理，直接打印
        System.out.println("Worker result: " + subResult);
    }
}

// 定义Master类
class Master {
    private List<Integer> task;
    private ExecutorService executorService;
    private List<Future<?>> futures;

    public Master(List<Integer> task) {
        this.task = task;
        // 创建一个线程池，这里简单地使用固定大小的线程池
        this.executorService = Executors.newFixedThreadPool(5);
        this.futures = new ArrayList<>();
    }

    public void submitTasks() {
        // 将任务划分为多个子任务，这里简单地将数组划分为大小相近的子数组
        int size = task.size();
        int subTaskSize = size / 5;
        for (int i = 0; i < 5; i++) {
            int startIndex = i * subTaskSize;
            int endIndex = (i == 4)? size : (i + 1) * subTaskSize;
            List<Integer> subTask = task.subList(startIndex, endIndex);
            Worker worker = new Worker(subTask);
            // 提交任务到线程池，并保存Future对象
            Future<?> future = executorService.submit(worker);
            futures.add(future);
        }
    }

    public void collectResults() {
        // 这里只是简单地等待所有任务完成，实际应用中可能需要收集结果并进行汇总
        for (Future<?> future : futures) {
            try {
                future.get();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
        // 关闭线程池
        executorService.shutdown();
    }
}

public class MasterWorkerExample {
    public static void main(String[] args) {
        List<Integer> task = new ArrayList<>();
        for (int i = 1; i <= 100; i++) {
            task.add(i);
        }
        Master master = new Master(task);
        master.submitTasks();
        master.collectResults();
    }
}
```

在这个示例中：

- `Worker` 类实现了 `Runnable` 接口，它的 `run` 方法用于计算分配给它的子任务（子数组元素的和）。
- `Master` 类负责管理任务。它在构造函数中初始化了任务列表和线程池，`submitTasks` 方法用于将任务分解并分发给 `Worker` 线程，`collectResults` 方法用于等待所有任务完成（实际应用中还可以收集和汇总结果）。
- 在 `main` 方法中，创建了一个整数列表作为任务，然后创建 `Master` 对象，提交任务并收集结果。

1. 适用场景
   - **计算密集型任务**：例如科学计算中的矩阵运算、大规模数据的统计分析（如计算海量数据的平均值、方差等）。通过 Master - Worker 模式，可以将复杂的计算任务分解为多个小的计算子任务，利用多核 CPU 的并行计算能力，加速任务的完成。
   - **数据处理任务**：如对大型文件的处理，将文件内容分割为多个部分，每个 Worker 线程处理一部分内容，最后由 Master 汇总处理结果，像是文本文件的词频统计、日志文件的分析等。
2. 优点
   - **提高性能**：通过并行处理任务，能够充分利用系统资源，显著缩短任务的执行时间，特别是在多核处理器环境下。
   - **任务分解和管理清晰**：Master 负责任务的分解和结果的收集，Worker 专注于执行子任务，这种分工使得代码结构清晰，易于理解和维护。
3. 缺点
   - **实现复杂度增加**：相比简单的单线程任务处理，Master - Worker 模式需要考虑任务的分解、线程间的通信和同步、结果的收集等多个方面，增加了代码的复杂性。
   - **资源管理问题**：如果任务划分不合理，可能导致某些 Worker 线程负载过重，而其他线程空闲，影响整体性能。同时，线程的创建和销毁也会消耗一定的系统资源，需要合理地使用线程池来优化。

## ForkJoin 模式

“分而治之”是一种思想，所谓“分而治之”，就是把一个复杂的算法问题按一定的“分解”方法分为规模较小的若干部分，然后逐个解决，分别找出各部分的解，最后把各部分的解组成整个问题的
解。“分而治之”思想在软件体系结构设计、模块化设计、基础算法中得到了非常广泛的应用。许多基础算法都运用了“分而治之”的思想，比如二分查找、快速排序等。

Master-Worker 模式是“分而治之”思想的一种应用，与 MasterWorker 模式不同，ForkJoin 模式没有 Master 角色，其所有的角色都是 Worker，ForkJoin 模式中的 Worker 将大的任务分割成小的任务，一直到任务的规模足够小，可以使用很简单、直接的方式来完成。

ForkJoin 模式先把一个大任务分解成许多个独立的子任务，然后开启多个线程并行去处理这些子任务。有可能子任务还是很大而需要进一步分解，最终得到足够小的任务。ForkJoin 模式借助了现代计算机多核的优势并行处理数据。

通常情况下，ForkJoin 模式将分解出来的子任务放入双端队列中，然后几个启动线程从双端队列中获取任务并执行。子任务执行的结果放到一个队列中，各个线程从队列中获取数据，然后进行局部结果的合并，得到最终结果。

### ForkJoin 框架

JUC 包提供了一套 ForkJoin 框架的实现，具体以 ForkJoinPool 线程池的形式提供，并且该线程池在 Java 8 的 Lambda 并行流框架中充当着底层框架的角色。JUC 包的 ForkJoin 框架包含如下组件：

1. ForkJoinPool：执行任务的线程池，继承了 AbstractExecutorService 类。
2. ForkJoinWorkerThread：执行任务的工作线程（ForkJoinPool 线程池中的线程）。每个线程都维护着一个内部队列，用于存放“内部任务”该类继承了 Thread 类。
3. ForkJoinTask：用于 ForkJoinPool 的任务抽象类，实现了 Future 接口。
4. `RecursiveTask`：带返回结果的递归执行任务，是 ForkJoinTask 的子类，在子任务带返回结果时使用。
5. `RecursiveAction`：不返回结果的递归执行任务，是 ForkJoinTask 的子类，在子任务不带返回结果时使用。

因为 ForkJoinTask 比较复杂，并且其抽象方法比较多，故在日常使用时一般不会直接继承 ForkJoinTask 来实现自定义的任务类，而是通过继承 ForkJoinTask 两个子类 RecursiveTask 或者 RecursiveAction
之一去实现自定义任务类，自定义任务类需要实现这些子类的 compute()方法，该方法的执行流程一般如下：

```java
if 任务足够小
 直接返回结果
else
 分割成N个子任务
 依次调用每个子任务的fork方法执行子任务
 依次调用每个子任务的join方法，等待子任务完成，然后合并执行结果
```

### ForkJoin 框架使用实战

假设需要计算 0 ～ 100 的累加求和，可以使用 ForkJoin 框架完成。首先需要设计一个可以递归执行的异步任务子类。

```java
import java.util.concurrent.RecursiveTask;

public class AccumulateTask extends RecursiveTask<Integer> {
    private static final int THRESHOLD = 2;
    // 累加的起始编号
    private int start;
    // 累加的结束编号
    private int end;

    public AccumulateTask(int start, int end) {
        this.start = start;
        this.end = end;
    }

    @Override
    protected Integer compute() {
        int sum = 0;
        // 判断任务的规模：若规模小则可以直接计算
        boolean canCompute = (end - start) <= THRESHOLD;
        // 若任务已经足够小，则可以直接计算
        if (canCompute) {
            // 直接计算并返回结果，Recursive结束
            for (int i = start; i <= end; i++) {
                sum += i;
            }
            System.out.println(Thread.currentThread().getName() + "执行任务，计算" + start + "到" + end + "的和，结果是：" + sum);
        } else {
            // 任务过大，需要切割，Recursive 递归计算
            System.out.println(Thread.currentThread().getName() + "切割任务：将" + start + "到" + end + "的和一分为二");
            int middle = (start + end) / 2;
            // 切割成两个子任务
            AccumulateTask lTask = new AccumulateTask(start, middle);
            AccumulateTask rTask = new AccumulateTask(middle + 1, end);
            // 依次调用每个子任务的fork()方法执行子任务
            lTask.fork();
            rTask.fork();
            // 等待子任务完成，依次调用每个子任务的join()方法合并执行结果
            int leftResult = lTask.join();
            int rightResult = rTask.join();
            // 合并子任务执行结果
            sum = leftResult + rightResult;
        }
        return sum;
    }
}

```

自定义的异步任务子类 AccumulateTask 继承自 RecursiveTask，每一次执行可以携带返回值。AccumulateTask 通过 THRESHOLD 常量设置子任务分解的阈值，并在它的 compute()方法中进行阈值判断，判断的逻辑如下：

- 若当前的计算规模（这里为求和的数字个数）大于 THRESHOLD，就当前子任务需要进一步分解，若当前的计算规模没有大于 THRESHOLD，则直接计算（这里为求和）。
- 如果子任务可以直接执行，就进行求和操作，并返回结果。如果任务进行了分解，就需要等待所有的子任务执行完毕、然后对各个分解结果求和。如果一个任务分解为多个子任务（含两个），就依
  次调用每个子任务的 fork()方法执行子任务，然后依次调用每个子任务的 join()方法合并执行结果。

使用 ForkJoinPool 调度 `AccumulateTask()`

```java
public class ForkJoinTest {
    public static void main(String[] args) throws Exception, TimeoutException {
        ForkJoinPool forkJoinPool = new ForkJoinPool();
        //创建一个累加任务，计算由1加到10
        AccumulateTask task = new AccumulateTask(1, 10);
        //执行任务
        Future<Integer> result = forkJoinPool.submit(task);
        Integer sum = result.get(1, TimeUnit.SECONDS);
        //预期的结果为5050
        System.out.println("结果为：" + sum+",预期结果为5050");
    }
}

ForkJoinPool-1-worker-9切割任务：将1到10的和一分为二
ForkJoinPool-1-worker-9切割任务：将1到5的和一分为二
ForkJoinPool-1-worker-2执行任务，计算1到3的和，结果是：6
ForkJoinPool-1-worker-4执行任务，计算4到5的和，结果是：9
ForkJoinPool-1-worker-11切割任务：将6到10的和一分为二
ForkJoinPool-1-worker-2执行任务，计算9到10的和，结果是：19
ForkJoinPool-1-worker-4执行任务，计算6到8的和，结果是：21
结果为：55
```

### ForkJoin 框架的核心 API

ForkJoin 框架的核心是 ForkJoinPool 线程池。该线程池使用一个无锁的栈来管理空闲线程，如果一个工作线程暂时取不到可用的任务，则可能被挂起，而挂起的线程将被压入由 ForkJoinPool 维护的栈
中，待有新任务到来时，再从栈中唤醒这些线程。

> ForkJoinPool 的构造器

```java
public ForkJoinPool(int parallelism,
                    ForkJoinWorkerThreadFactory factory,
                    UncaughtExceptionHandler handler,
                    boolean asyncMode) {
    this(checkParallelism(parallelism),
         checkFactory(factory),
         handler,
         asyncMode ? FIFO_QUEUE : LIFO_QUEUE,
         "ForkJoinPool-" + nextPoolId() + "-worker-");
    checkPermission();
}
```

对以上构造器的 4 个参数具体介绍如下：

1. `parallelism`：可并行级别
   ForkJoin 框架将依据 parallelism 设定的级别决定框架内并行执行的线程数量。并行的每一个任务都会有一个线程进行处理，但 parallelism 属性并不是 ForkJoin 框架中最大的线程数量，该属性和
   ThreadPoolExecutor 线程池中的 corePoolSize、maximumPoolSize 属性有区别，因为 ForkJoinPool 的结构和工作方式与 ThreadPoolExecutor 完全不一样。ForkJoin 框架中可存在的线程数量和 parallelism 参数值并不是绝对关联的。
2. `factory`：线程创建工厂
   当 ForkJoin 框架创建一个新的线程时，同样会用到线程创建工厂。只不过这个线程工厂不再需要实现 ThreadFactory 接口，而是需要实现 ForkJoinWorkerThreadFactory 接口。后者是一个函数式接口，只需要实现一个名叫 newThread()的方法。在 ForkJoin 框架中有一个默认的 ForkJoinWorkerThreadFactory 接口实现 `DefaultForkJoinWorkerThreadFactory`。
3. `handler`：异常捕获处理程序
   当执行的任务中出现异常，并从任务中被抛出时，就会被 handle 捕获。
4. `asyncMode`：异步模式
   asyncMode 参数表示任务是否为异步模式，其默认值为 false。**如果 asyncMode 为 true，就表示子任务的执行遵循 FIFO（先进先出）顺序，并且子任务不能被合并；如果 asyncMode 为 false，就表示子任务的执行遵循 LIFO（后进先出）顺序，并且子任务可以被合并。** 虽然从字面意思来看 asyncMode 是指异步模式，它并不是指 ForkJoin 框架的调度模式采用是同步模式还是异步模式工作，仅仅指任务的调度方式。

ForkJoin 框架中为每一个独立工作的线程准备了对应的待执行任务队列，这个任务队列是使用数组进行组合的双向队列。asyncMode 模式的主要意思指的是待执行任务可以使用 FIFO（先进先出）的工作模式，也可以使用 LIFO（后进先出）的工作模式，工作模式为 FIFO（先进先出）的任务适用于工作线程只负责运行异步事件，不需要合并结果的异步任务。

> ForkJoinPool 无参数的、默认的构造器如下：

该构造器的 parallelism 值为 CPU 核数；factory 值为 defaultForkJoinWorkerThreadFactory 默认的线程工厂；异常捕获处理程序 handler 值为 null，表示不进行异常处理；异步模式 asyncMode
值为 false，使用 LIFO（后进先出）的、可以合并子任务的模式。

```java
public ForkJoinPool() {
    // //并行度，默认为CPU数，最小为1
    this(Math.min(MAX_CAP, Runtime.getRuntime().availableProcessors()),
         defaultForkJoinWorkerThreadFactory, null, false);
}
```

> ForkJoinPool 的 common 通用池

很多场景可以直接使用 ForkJoinPool 定义的 common 通用池，调用 `ForkJoinPool.commonPool()` 方法可以获取该 ForkJoin 线程池，该线程池通过 makeCommonPool()来构造，具体的代码如下：

```java
public static ForkJoinPool commonPool() {
    // assert common != null : "static init error";
    return common;
}

private static ForkJoinPool makeCommonPool() {
    int parallelism = -1;
    ForkJoinWorkerThreadFactory factory = null;
    UncaughtExceptionHandler handler = null;
    try {  // ignore exceptions in accessing/parsing properties
        //并行度
        String pp = System.getProperty
            ("java.util.concurrent.ForkJoinPool.common.parallelism");
        //线程工厂
        String fp = System.getProperty
            ("java.util.concurrent.ForkJoinPool.common.threadFactory");
        //异常处理类
        String hp = System.getProperty
            ("java.util.concurrent.ForkJoinPool.common.exceptionHandler");
        if (pp != null)
            parallelism = Integer.parseInt(pp);
        if (fp != null)
            factory = ((ForkJoinWorkerThreadFactory)ClassLoader.
                       getSystemClassLoader().loadClass(fp).newInstance());
        if (hp != null)
            handler = ((UncaughtExceptionHandler)ClassLoader.
                       getSystemClassLoader().loadClass(hp).newInstance());
    } catch (Exception ignore) {
    }
    if (factory == null) {
        if (System.getSecurityManager() == null)
            factory = new DefaultCommonPoolForkJoinWorkerThreadFactory();
        else // use security-managed default
            factory = new InnocuousForkJoinWorkerThreadFactory();
    }
    //默认并行度为cores-1
    if (parallelism < 0 && // default 1 less than #cores
        (parallelism = Runtime.getRuntime().availableProcessors() - 1) <= 0)
        parallelism = 1;
    if (parallelism > MAX_CAP)
        parallelism = MAX_CAP;
    return new ForkJoinPool(parallelism, factory, handler, LIFO_QUEUE,
                            "ForkJoinPool.commonPool-worker-");
}
```

使用 common 池的优点是可以 **通过指定系统属性的方式定义“并行度、线程工厂和异常处理类”**，并且 common 池使用的是同步模式，也就是说可以支持任务合并。

1. 通过系统属性的方式指定 parallelism 值的示例如下：

   ```java
   System.setProperty("java.util.concurrent.ForkJoinPool.common.parallelism", "8");
   ```

2. 通过 Java 指令选项的方式指定 parallelism 值

   ```sh
   -Djava.util.concurrent.ForkJoinPool.common.parallelism=8
   ```

其他的参数值如异常处理程序 handler，都可以通过以上两种方式指定。

> 向 ForkJoinPool 线程池提交任务的方式

1. 外部任务（External/Submissions Task）提交向 ForkJoinPool 提交外部任务有三种方式：
   - 方式一是调用 invoke()方法，该方法提交任务后线程会等待，等到任务计算完毕返回结果；
   - 方式二是调用 execute()方法提交一个任务来异步执行，无返回结果；
   - 方式三是调用 submit()方法提交一个任务，并且会返回一个 ForkJoinTask 实例，之后适当的时候可通过 ForkJoinTask 实例获取执行结果。
2. 子任务（Worker Task）提交
   - 向 ForkJoinPool 提交子任务的方法相对比较简单，由任务实例的 fork()方法完成。当任务被分割之后，内部会调用 ForkJoinPool.WorkQueue.push()方法直接把任务放到内部队列中等待
     被执行。

### 工作窃取算法

ForkJoinPool 线程池的任务分为“外部任务”和“内部任务”，两种任务的存放位置不同：

- 外部任务存放在 ForkJoinPool 的全局队列中。
- 子任务会作为“内部任务”放到内部队列中，ForkJoinPool 池中的每个线程都维护着一个内部队列，用于存放这些“内部任务”。

由于 ForkJoinPool 线程池通常有多个工作线程，与之相对应的就会有多个任务队列，这就会出现任务分配不均衡的问题：有的队列任务多，忙得不停，有的队列没有任务，一直空闲。那么有没有一种机
制帮忙将任务从繁忙的线程分摊给空闲的线程呢？答案是使用工作窃取算法。

工作窃取算法的核心思想是：工作线程自己的活干完了之后，会去看看别人有没有没干完的活，如果有就拿过来帮忙干。**工作窃取算法的主要逻辑：每个线程拥有一个双端队列（本地队列），用于存放**
**需要执行的任务，当自己的队列没有任务时，可以从其他线程的任务队列中获得一个任务继续执行**

在实际进行任务窃取操作的时候，操作线程会进行其他线程的任务队列的扫描和任务的出队尝试。为什么说是尝试？因为完全有可能操作失败，主要原因是并行执行肯定涉及线程安全的问题，假如在窃取过程中该任务已经开始执行，那么任务的窃取操作就会失败。

如何尽量避免在任务窃取中发生的线程安全问题呢？一种简单的优化方法是：在线程自己的本地队列采取 LIFO（后进先出）策略，窃取其他任务队列的任务时采用 FIFO（先进先出）策略。简单来说，**获取自己队列的任务时从头开始，窃取其他队列的任务时从尾开始**。由于窃取的动作十分快速，会大量降低这种冲突，也是一种优化方式

![](/resources/d65948ee1c5146e9b0ac69bb5c4d5311.png)

### ForkJoin 框架的原理

ForkJoin 框架的核心原理大致如下：

1. ForkJoin 框架的线程池 ForkJoinPool 的任务分为“外部任务”和“内部任务”。
2. “外部任务”放在 ForkJoinPool 的全局队列中。
3. ForkJoinPool 池中的每个线程都维护着一个任务队列，用于存放“内部任务”，线程切割任务得到的子任务会作为“内部任务”放到内部队列中。
4. 当工作线程想要拿到子任务的计算结果时，先判断子任务有没有完成，如果没有完成，再判断子任务有没有被其他线程“窃取”，如果子任务没有被窃取，就由本线程来完成；一旦子任务被窃
   取了，就去执行本线程“内部队列”的其他任务，或者扫描其他的任务队列并窃取任务。
5. 当工作线程完成其“内部任务”，处于空闲状态时，就会扫描其他的任务队列窃取任务，尽可能不会阻塞等待。

![](/resources/effc9fe39f6b42759092fc427e72c29a.png)

总之，ForkJoin 线程在等待一个任务完成时，要么自己来完成这个任务，要么在其他线程窃取了这个任务的情况下，去执行其他任务，是不会阻塞等待的，从而避免资源浪费，除非所有任务队列都为
空。

工作窃取算法的优点如下：

1. 线程是不会因为等待某个子任务的执行或者没有内部任务要执行而被阻塞等待、挂起的，而是会扫描所有的队列窃取任务，直到所有队列都为空时才会被挂起。
2. ForkJoin 框架为每个线程维护着一个内部任务队列以及一个全局的任务队列，而且任务队列都是双向队列，可从首尾两端来获取任务，极大地减少了竞争的可能性，提高并行的性能。

ForkJoinPool 适合需要“分而治之”的场景，特别是分治之后递归调用的函数，例如快速排序、二分搜索、大整数乘法、矩阵乘法、棋盘覆盖、归并排序、线性时间选择、汉诺塔问题等。ForkJoinPool
适合调度的任务为 CPU 密集型任务，如果任务存在 I/O 操作、线程同步操作、sleep()睡眠等较长时间阻塞的情况，最好配合使用 ManagedBlocker 进行阻塞管理。总体来说，ForkJoinPool 不适合进行
IO 密集型、混合型的任务调度。

## Future 模式

Future 模式是高并发设计与开发过程中常见的设计模式，它的核心思想是异步调用。对于 Future 模式来说，它不是立即返回我们所需要的数据，但是它会返回一个契约（或异步任务），将来我们可以凭借这个契约（或异步任务）获取需要的结果。

在进行传统的 RPC（远程调用）时，同步调用 RPC 是一段耗时的过程。当客户端发出 RPC 请求后，服务端完成请求处理需要很长的一段时间才会返回，这个过程中客户端一直在等待，直到数据返回后，再进行其他任务的处理。

现有一个 Client 同步对三个 Server 分别进行一次 RPC 调用，假设一次远程调用的时间为 500 毫秒，则一个 Client 同步对三个 Server 分别进行一次 RPC 调用的总时间需要耗费 1500 毫秒。如果要节省这个总时间，可以使用 Future 模式对其进行改造，将同步的 RPC 调用改为异步并发的 RPC 调用。

Future 模式的核心思想是异步调用，有点类似于异步的 Ajax 请求。当调用某个耗时方法时，可以不急于立刻获取结果，而是让被调
用者立刻返回一个契约（或异步任务），并且将耗时的方法放到另外的线程中执行，后续凭契约再去获取异步执行的结果。在具体的实现上，Future 模式和异步回调模式既有区别，又有联系。**Java 的 `Future` 实现类并没有支持异步回调，仍然需要主动获取耗时任务的结果；而 Java 8 中的 `CompletableFuture` 组件实现了异步回调模式。**
