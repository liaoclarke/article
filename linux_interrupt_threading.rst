=========================
Linux Interrupt Threading
=========================

Revision:
=========

v1.0.0 Liao Chang
    2021.11.18, 创建文件框架，介绍soft_irq机制

1. 背景介绍
===========

中断是一种硬件事件，当CPU接收到中断时，正常的指令执行流将会被打断，
CPU将从特定的中断处理入口开始取指执行，当中断事件被软件处理完毕后，
通过特殊返回指令将CPU上下文恢复到被打断点再继续执行。

所以，中断一旦产生就能立刻被软件响应，一般来说，当CPU跳转到中断处理入口后就应该被软件处理掉，
在ARM架构下，内核通过读取ICC IAR寄存器来获取中断请求号，然后调用注册在该请求号上的处理函数完成中断的处理，
即中断响应和中断处理是在同一个线程上下文里完成的。

中断线程化顾名思义就是：当中断产生时，内核先在当前线程上下文里响应中断，但在其他线程上下文里完成中断的处理，
中断线程化可以带来两个收益:

#. 降低对正常业务干扰：处理中断会打断CPU的正常执行，当中断处理流程耗时较长，比如，等待设备状态翻转，处理大量数据处理，
   将会干扰正常业务，此时就将中断处理放到其他线程上下文里延迟执行(deferrable execution)，尽量保证对正常业务的影响比较小。
#. 改善中断的响应性：一般而言，在中断处理完毕之前都需要将关闭中断关闭(disable)，这会降低中断响应的及时性，
   通过将中断放到其他线程上下文执行，可以保证关闭中断的时间尽可能短，从而改善中断响应的及时性。

按照Linux Kernel的概念而言，响应中断的过程称之为中断上半部(Top half)，处理中断的过程称之为中断下半部(Bottom half)，
目前Linux Kernel里有好几套中断线程化的机制，他们的技术原理和适用场景各个不相同，包括：

- **softirq** : 虽然名字里有“irq"，其实softirq是一套通用的deferrable execution机制，
  内核里需要延迟处理的事件都可以通过softirq机制来执行。

- **tasklet** : tasklet是基于softirq实现的，但它在使用场景上和通用的softirq略有不同。

  +---------+---------------------------------------------------------------------+
  | softirq | | 编译阶段静态创建，可以在多CPU上并发处理，所以softirq的处理函数需  |
  |         | | 要设计成可重入的，关键数据的修改需加自旋锁保护。                  |
  +---------+---------------------------------------------------------------------+
  | tasklet | | 运行时动态创建，通过tasklet执行的deferrable function都是串行执行，|
  |         | | 相同类型的tasklet任务不会在多个CPU上并发执行。                    |
  +---------+---------------------------------------------------------------------+

- **workqueue** : TODO

- **threaded IRQ** : TODO

- **IRQ resend** : TODO

- **RT_PREEMPT分支的中断线程方案** : 一旦中断产生，所有任务包括RT任务通通都要给中断处理程序让路，如果Top half处理任务较多就会
  对RT任务造成很大的影响，并且这种影响存在较大的不确定性，因此难以准确评估。为了解决这些实时性相关问题，Linux RT_PREEMPT
  补丁也引入了中断线程化的机制。

2. Softirq
==========

**2.1 延迟执行对象**

softirq是内核执行deferrable function的一套机制，deferrable function由struct softirq_action来实现:

.. code-block:: c

    struct softirq_action {
        void (*action)(struct softirq_action *);
    };
..

一般而言，每个deferrable function都会支持以下四种操作：

**Initialization**
    定义一个新的softirq_action对象，该操作往往是在内核初始化阶段，或者模块加载时执行。

**Activation**
    将一个softirq_action对象设置为“pending”状态，当内核启动下一轮deferable function的执行时，
    将把标记为“pending”状态的softirq_action对象全部处理掉，该操作可以在任何时刻下执行，比如，
    响应中断的上下文里。

**Masking**
    将一个softirq_action对象标记为“disabled”，被标记的deferrable function即使处于“pending”状态,
    也将得不到执行，内核提供了一套完整机制和接口来支持deferrable function masking操作，后续章节
    将详细介绍。

**Execution**
    执行所有处于“pending”状态的softirq_action对象，因为是延迟执行的， 所以尽量不要对这些deferrable 
    function的执行时机有任何期望，不过现在内核为了追求一定的可预见性，也在内核一些特定点上允许插入
    了deferrable function的调用点，后续章节将详细介绍。

.. note::

    **开放性话题:**

    执行 **Activation** 和 **Exectution** 的CPU是否必须相同?，一方面，deferrable function
    使用的数据往往会在激活前保留在CPU L1$，另外在本CPU上执行也可以避免和其他CPU的调度队列
    造成竞争，还能避免IPI唤醒的开销。但另一方面，将执行和激活绑定在同一个CPU上，也会造成某些
    CPU执行deferrable function过载的问题。
..

**2.2 数据结构**

目前内核的softirq只支持9种deferrable function，他们都是在内核里静态定义的，并且在内核初始化
阶段完成initialization。

+------------------+------+-------------------------------+
| softirq          | 编号 | 描述                          |
+==================+======+===============================+
| HI_SOFTIRQ       | 0    | 用于延迟执行高优先级tasklet   |
+------------------+------+-------------------------------+
| TIMER_SOFTIRQ    | 1    | 用于延迟执行软件注册的定时器  |
+------------------+------+-------------------------------+
| NET_TX_SOFTIRQ   | 2    | 用于延迟执行往网卡发生报文    |
+------------------+------+-------------------------------+
| NET_RX_SOFTIRQ   | 3    | 用于延迟执行从网卡读取报文    |
+------------------+------+-------------------------------+
| BLOCK_SOFTIRQ    | 4    | 用于延迟执行块设备数据处理    |
+------------------+------+-------------------------------+
| IRQ_POLL_SOFTIRQ | 5    |                               |
+------------------+------+-------------------------------+
| TASKLET_SOFTIRQ  | 6    | 用于延迟指向普通优先级tasklet |
+------------------+------+-------------------------------+
| SCHED_SOFTIRQ    | 7    |                               |
+------------------+------+-------------------------------+
| HRTIMER_SOFTIRQ  | 8    |                               |
+------------------+------+-------------------------------+
| RCU_SOFTIRQ      | 9    |                               |
+------------------+------+-------------------------------+

.. code-block:: c

   // include/linux/interrupt.h
    enum
    {
        HI_SOFTIRQ=0,
        TIMER_SOFTIRQ,
        NET_TX_SOFTIRQ,
        NET_RX_SOFTIRQ,
        BLOCK_SOFTIRQ,
        IRQ_POLL_SOFTIRQ,
        TASKLET_SOFTIRQ,
        SCHED_SOFTIRQ,
        HRTIMER_SOFTIRQ,
        RCU_SOFTIRQ,    /* Preferable RCU should always be the last softirq */

        NR_SOFTIRQS
    };
..

内核每一轮执行defeerable function过程中，都是先从编号小的softirq_action开始执行。
所有这些defeerable function通过对应的编号和softirq_vec来访问。

.. code-block:: c

    // kernel/softirq.c
    static struct softirq_action softirq_vec[NR_SOFTIRQS] __cacheline_aligned_in_smp;
..

**2.3 激活softirq**

内核通过接口函数open_softirq()来初始化defeerable function，这个过程很简单就是初始化
softiq_vec数组里对应softirq_action对象。内核通过接口函数raise_softirq()来激活defeera
ble function，这两个接口都是通过softirq_actino的编号来进行操作。

.. code-block:: c

    // kernel/softirq.c
    open_softirq(int nr, void (*action)(struct softirq_action *));
    raise_softirq(unsigned int nr);
..

其中raise_softirq的核心流程如下：

1. local_irq_save(): 让本CPU屏蔽所有中断，因为该函数需要操作一个per-cpu的全局变量，
   防止操作过程中被中断打断导致的数据不一致问题。
2. __raise_softirq_irqoff(): 现在内核选择的策略是activiation和execution发生在相同
   CPU上，所以激活操作也是通过将softirq编号标记到per-cpu的pending位掩码实现的。

.. code-block:: c

   // include/asm-generic/hardirq.h
    typedef struct {
        unsigned int __softirq_pending;
        ... 
    } ____cacheline_aligned irq_cpustat_t;
    DECLARE_PER_CPU_ALIGNED(irq_cpustat_t, irq_stat);

    // include/linux/interrupt.h
    #define local_softirq_pending_ref irq_stat.__softirq_pending
    #define or_softirq_pending(x)	(__this_cpu_or(local_softirq_pending_ref, (x)))
..

3. in_interrupt()和should_wake_ksoftirqd(): 目前内核的deferrable function的主要执行
   点都是在内核线程ksoftirqd，但由于ksoftirqd的调度策略是SCHED_NORMAL，这就导致执行
   时机的不确定，为了改善这些不可预测性，内核还在top halft返回时部署了一个执行点，所
   以如果内核还处于top half时，in_interrupt返回true就表明无需唤醒ksoftirqd来执行soft
   irq。另外，如果activiation操作是在ksoftirq线程上下文里发起的，即should_wake_ksoft
   irqd返回false，也无需再次唤醒ksoftirq，否则就执行#5。

4. wakeup_softirqd(): 唤醒本CPU上的ksoftirqd内核线程来执行所有pending状态的deferrabl
   e function。

5. local_irq_restore: 恢复本本CPU的所有中断屏蔽状态。

**2.4 softirq状态检查**

内核只有在检查到有pending的softirq才会尝试处理ksoftirq，但如果太频繁的检查又会影响内
核性能，所以不同版本的内核都在一些关键流程点里部署了检测点，以5.13.10为例：

- 当内核通过local_bh_enable()函数允许本CPU处理softirq的过程里。

.. code-block:: c

   // kernel/softirq.c
    void __local_bh_enable_ip(unsigned long ip, unsigned int cnt)
    {
        ...
        pending = local_softirq_pending();
        if (!pending || ksoftirqd_running(pending))
            goto out;
        ...
        __do_softirq();
    out:
        ...
    }
..

- 当内核通过do_IRQ执行完中断top half后，调用irq_exit的过程中。

.. code-block:: c

    static inline void __irq_exit_rcu(void)
    {
        ...
        if (!in_interrupt() && local_softirq_pending())
            invoke_softirq();
        ...
    }
..

- 当CPU执行hotplug下线时，会将自己runqueue队列里的任务都迁移到其他CPU的runqueue里，
  这时候也会检查是否有pending的softirq。

.. code-block:: c

    /*
     * migration_cpu_stop - this will be executed by a highprio stopper thread
     * and performs thread migration by bumping thread off CPU then
     * 'pushing' onto another runqueue.
    */
    static int migration_cpu_stop(void *data)
    {
        ...
        flush_smp_call_function_from_idle();
        ...
    }
..

- 在CPU准备从idle切换到其他任务前，也会检查是否有pending的softirq。

.. code-block:: c

    static void do_idle() 
    {
        ...
        flush_smp_call_function_from_idle();
        schedule_idle();
        ...
    }

    void flush_smp_call_function_from_idle(void) {
        ...
        if (local_softirq_pending())
            do_softirq();
        ...
    }
..

- 在内核线程ksoftirqd的线程里。

.. code-block:: c

    static void run_ksoftirqd() {
        ...
        if (local_softirqd_pending()) {
            __do_softirq();
        }
        ...
    }
..

**2.5 执行softirq**

内核主要通过两个接口函数来执行softirq，他们分别适用于不同的常见。

- do_softirq(): 这个接口使用在除ksoftirqd内核线外的大部分场景下，往往配合local_soft
  irqd_pending()来使用。所以为了保证数据一致性，还会屏蔽本CPU的所有中断，并且检查函
  数上下文对执行softirq来说是否安全，判断上下文是否安全的原则和raise_softirq基本相同
  ，参考前面章节。

.. code-block:: c

    void do_softirq(void) {
        if (in_interrupt())
            return;

        local_irq_save(flags);

        if (local_softirq_pending() && !ksoftirqd_running(pending))
            __do_softirq();

        local_irq_restore(flags);
    }
..

- _do_softirq(): 这个接口可以用在所有执行softirq的场景下，不管是ksoftirqd和内核里
  固定的softirq执行点。该函数在执行过程中允许有新softirq被激活，所以函数会尽量将当前
  前CPU上处于pending状态的softirq全部处理完毕才退出，但如果这个函数执行时间过长也会
  导致其他用户态任务饿死，所以该函数有一个迭代次数上限，在每次迭代里会将9种softirq里
  处于pending的deferrable function都处理一遍，如果迭代次数超过上限，就会唤醒ksoftirq
  d来完成执行剩余的softirq。

.. code-block:: c

    asmlinkage __visible void __softirq_entry __do_softirq(void)
    {
        ...
        unsigned long end = jiffies + MAX_SOFTIRQ_TIME;
        uint32_t pending = local_softirq_pending();
        int max_restart = MAX_SOFTIRQ_RESTART;

        softirq_handle_begin();
    restart:
        /* Reset the pending bitmask before enabling irqs */
        set_softirq_pending(0);
        local_irq_enable();

        h = softirq_vec;
        while ((softirq_bit = ffs(pending))) {
            h += softirq_bit - 1;
            h->action(h);
            h++;
            pending >>= softirq_bit;
        }

        local_irq_disable();

        pending = local_softirq_pending();
        if (pending) {
            if (time_before(jiffies, end) && !need_resched() &&
                --max_restart)
                goto restart;

            wakeup_softirqd();
        }

        softirq_handle_end();
        ...
    }
..

这个函数的核心流程是：

#. 将本CPU上的pending状态的softirq的掩码拷贝到本地变量，因为在执行deferrable functio
   n的过程中是允许有新的softirq被激活。

#. 将当前CPU的运行状态标记为处于softirq的执行过程中，原因也是因为执行deferrable func
   tion的过程中断时允许响应新的中断，而在中断处理过程又可能执行softirq，但这就会导致
   执行deferrable function的出现了嵌套了，但内核期望它的执行应该是串行的，并保证编号
   小softirq一定要在编号大softirq之前处理完，所以通过本步骤防止嵌套执行的问题。

#. 将pending的softirq掩码清零。

#. 执行local_irq_enable()使能本CPU的中断。

#. 将所有处于pending的deferrable function都执行一遍。

#. 执行local_irq_disable()屏蔽本CPU的中断。

#. 再次获取pending的softirq掩码快照，并且将迭代次数减1。

#. 如果pending的softirq非空，就意味着有新的softirq被激活了，还有再重新执行一次迭代，
   但在迭代之前要检查迭代次数是否超过上限，该函数的执行时间是否超过上限，如果没有超
   过上限就再次从第3步开始执行。

#. 否则，该函数执行时间已经超过上限了，还未处理完毕的deferrable function就要通过kso
   ftirqd来执行。

.. note::

    **开放性话题:**

    当前内核将__do_softirq()里处理deferrable function的迭代上限设置为10，执行时间设置
    为2ms，这两个值其实应该根据系统负载动态调整，另外，当__do_softirq无法在有限时间内
    将所有softirq全部处理掉，它才会唤醒ksoftirqd线程来继续处理，当驱动程序中断处理函数
    频繁激活软中断的情况下，比如，网络收到大量RX报文，仍然会导致大量的系统时间被消耗在
    __do_softirq(ksoftirqd线程也是通过该函数处理softirq)里，内核目前并没有特别有效的手
    段来监控这种问题，是一个潜在可开发的特效，即如何提高内核处理softirq的韧性。
..

**2.6 ksoftirqd内核线程**

直到这里，才可以说真正涉及到中断的线程化，每个CPU上都会创建一个名字为 **ksoftirqd/<cpu-id>**
的内核线程，这些内核线程的调度策略都是SHCED_NORMAL，这些内核线程都是在内核初始化阶段创建的，
他们的功能主体是：如果存在pending的softirq，如果有就调用__do_softirq处理这些softirq，否则就
将状态设置为TASK_INTERRUPTIBLE，然后调度其他任务运行。

.. code-block:: c

    static void run_ksoftirqd() {
        ksoftirqd_run_begin();
        if (local_softirq_pending()) {
            __do_softirq();
            ksoftirqd_run_end();
            cond_resched();
        }
        ksoftirqd_run_end();
    }
..

ksoftirqd的作用实际上类似一种"safe net"，还是考虑上述，网卡收到大量RX报文的情况，此时__do_softirq
的带宽已经不足以在有限时间内将所有softirq处理完毕，但为了保证这些softirq最终有机会得到处理，将唤醒
ksoftirqd来逐步消耗这些deferrable function。

**2.7 tasklet机制**

TODO

**2.8 使用场景**

TODO

3. Workqueue
============

TODO

4. Threaded IRQ
===============

TODO

5. IRQ resend
=============

TODO

6. RT_PREEMPT的中断线程方案
=============================

TODO

7. 参考资料
=============

#. Understanding the Linux Kernel, 3rd edition
#. kernel/Documentation/admin-guide/kernel-per-CPU-kthreads.rst
#. RT-Linux: https://wiki.linuxfoundation.org/realtime/documentation/start
