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

.. note::
  1. 和线程化中断相关的irqflags是IRQ_NESTED_THREAD，IRQF_NO_BALANCING，IRQF_ONESHOT
  2. 全局配置变量force_irqthreads，内核启动参数“threadirqs”。
  3. 在irq_flow_handler里调用handle_irq_event，并根据irq_action->handler的返回值，
     选择是否唤醒irq_action的thread_fn。(函数irq_thread)。
  4. 在内核线程里调用irq_thread，进一步调用irq_action->thread_fn。
  5. 线程化之后的中断，需要和其他任务进行同步。
  6. 中断线程的销毁。
  7. 中断线程的亲和性。
..

Linux内核中断框架原生支持在内核线程上下文里处理中断的机制，即Threaded IRQ，
这种机制支持将 **整个中断处理代码全部放入线程上下文里执行** ，
在硬件中断上下文里(即top-half)，内核只需要完成基本硬件操作，比如获取中断ID，
中断ACK和中断EOI，然后根据中断源唤醒对应内核线程，这个机制是Thomas Glexiner在
2009引入到内核(commit: 3aa551c9b4c4)。

**4.1 中断处理框架回顾**

.. figure:: images/hardware_irq_topology.png
   :scale: 80 %
   :align: center

   中断系统物理拓扑
..

中断系统的物理拓扑大致上可以分为四种模型：

- **基本模型:** 最简单模型，系统里所有中断源都连接到一个中断控制器部件，
  中断控制器将中断源转换成INTID再投递到CPU上，中断源的INTID互不相同，
  软件通过中断控制器获取INTID就可以直接调用对应的ISR，
  比如单片机系统，一块MCU搭配一个8259中断控制器。

- **并联模型:** 系统里有不同类型的中断控制器，中断源连接到不同的中断控制器，
  中断控制器将中断源转变成INTID再投递到CPU上，
  连接到相同中断控制器的中断源的INTID互不相同，
  连接到不同中断控制器的中断源的INTID可以相同(或互不相同)，
  软件通过不同中断控制器获取INTID也可以调用到正确的ISR，
  比如，Arm GIC由GICR, GICD，ITS三类中断控制器部件组成，
  GICD投递到CPU的INTID为32~1019，GICR投递到CPU的INTID为0~31，
  ITS投递到CPU的INTID为8192~。

- **级联模型:** 系统里有不同类型的中断控制器，他们之间通过级联的方式联接起来，
  比如，X86的Local-APIC只支持255个中断源，但可以将8259中断控制器级联到Local-APIC
  上扩展中断数量，或者两个8259级联起来扩展单片机的中断源数量。

- **共享中断模型:** 如果中断控制器的中断引脚数量不足，
  多个中断源需要共享同一根中断引脚，
  软件通过中断控制器获取INTID后还需要进一步确定是哪个设备触发了中断。
  比如，PC主板上的多个PCIe设备的INTx中断会共享同一个中断线。

.. figure:: images/software_irq_topology.png
   :scale: 80 %
   :align: center

   Linux中断软件拓扑
..

Linux内核通过上图描绘的中断架构来支持这几种物理拓扑，这个中断架构有几个特点：

#. 每个物理中断号(hwirq)都对应内核的一个软件中断号(virq)，
   软件通过各个中断控制器部件读取hwirq，然后将其映射成一个全局唯一的virq。
   内核是通过virq来识别中断源并调用对应ISR，而不是通过hwirq。

   这样的收益是：(1) 如果中断源修改了亲和性，那么它的hwirq很有可能发生改变，
   但内核只需要重新将hwirq映射到virq，(2) 允许不同的中断控制器部件返回相同的hwirq，
   比如，X86上所有Local-APIC返回的hwirq范围都是0~255。

   设备驱动注册中断源时，首先请求内核分配一个全局唯一virq，
   再请求中断控制器驱动动态分配hwirq，或者从dts里解析出静态定义的hwirq，
   最后将请求内核为hwirq和virq建立映射关系。

#. 当系统存在多种不同的中断控制器，并且它们既可以采用并联，也可采用级联方式来联接，
   这些中断控制器对接入的中断都会分配一个hwirq，这样就会为软件带来很多问题：
   (1) 两个并联的中断控制器会对不同中断源分配相同hwirq，
   (2) 两个级联的中断控制器，父控制器会给子控制器分配一个hwirq，
   那么所有连接到子控制器的中断源对父控制器来说就共享同一个hwirq。

   所以，当软件获取了hwirq后，就需要在一个特定的“空间”里才能将其映射成正确的virq，
   为此，Linux内核引入了一个irq_domain的概念，每个irq_domain构成了一个中断ID空间，
   每个中断控制器驱动都可以根据开发者需要为其绑定一个独立的irq_domain对象，
   这个irq_domain对象维护了关联的中断源hwirq和virq间的映射关系，
   **注意：virq是全局唯一的，但hwirq只能保证是中断控制器内唯一。**
   当内核通过操作中断控制器获取一个hwirq时，还要利用该中断控制器对应irq_domain对象
   将其映射为virq，Linux内核所有中断源都归属于某个irq_domain对象，
   这些irq_domain对象会按照实际中断控制器拓扑来组织成一颗domain tree。

#. 每个中断源的处理函数也是分为两层：flow handler和action handler，
   内核调用virq对应的中断处理函数时都是先调用对应的flow handler，再调用对应action handler。
   设备驱动注册中断源时只提供action handler，这action handler包含中断源处理的核心功能，
   而flow_handler则由中断源关联的irq_domain负责配置，
   每个irq_domain关联的所有中断源的flow handler可以配置成不同的flow handler。

   Linux内核引入flow handler的目的至少有两个：
   (1) 将中断处理的基本操作从ISR里剥离，比如，中断ACK，中断EOI，
   针对边缘或者电平模式的硬件操作流程，tracing等，这些操作大多是硬件操作相关，
   暴露给驱动开发者既会带来开发负担，也会导致各种问题。
   (2) 如果中断控制器采用级联的方式，可以利用flow handler来获取真正的hwirq和virq。
   内核通过父irq_domain获取子irq_domain的hwirq，根据hwirq获取virq并调用flow handler，
   在flow handler读取子irq_domain的获取触发中断源的hwirq，最终获取对应的virq，
   所以flow handler为irq_domain级联模式提供了支持。

   +-------------------------+----------------------------------------------------+
   | handle_simple_irq       | 调用action handler时，不需要额外的硬件操作。       |
   +-------------------------+----------------------------------------------------+
   | handle_level_irq        | 处理电平触发模式的中断事件，需要mask中断。         |
   +-------------------------+----------------------------------------------------+
   | handle_fasteoi_irq      | 调用中断action handler前，需要执行中断eoi。        |
   +-------------------------+----------------------------------------------------+
   | handle_edge_irq         | 处理边缘触发模式的中断事件，需要ack中断。          |
   +-------------------------+----------------------------------------------------+
   | handle_edge_eoi_irq     | 处理边缘触发模式的中断事件，需要ack中断和eoi中断。 |
   +-------------------------+----------------------------------------------------+
   | handle_fasteoi_ack_irq  | 调用action handler时，需要ack中断和eoi中断。       |
   +-------------------------+----------------------------------------------------+
   | handle_fasteoi_mask_irq | 调用action handler时，需要mask中断和eoi中断。      |
   +-------------------------+----------------------------------------------------+

#. 支持多个中断源共享同一个物理中断号(hwirq)，Linux内核通过在virq上注册多个action 
   handler形成一个chain的机制来支持中断号共享的问题，
   内核可以在virq上调用多个action handler，在每个action
   handler里再检测本次中断是否应该被处理，
   **而action handler也是Threaded IRQ机制被线程化执行的代码。**

.. code-block:: c

    static void __handle_irq_event_percpu(struct irq_desc *desc, ...) {
        int res;
        struct irqaction *action;

        for_each_action_of_desc(desc, action) {
            res = action->handler(irq, action->dev_id);
            switch (res) {
            case IRQ_WAKE_THREAD:
                __irq_wake_thread(desc, action);
            case IRQ_HANDLED:
                ...
            }
        }
    }

..

**4.2 注册threaded IRQ**

**4.3 唤醒threaded IRQ**

**4.4 处理theaded IRQ**

**4.5 Threaded IRQ使用约束**

**4.6 Threaded IRQ亲和性**

**4.7 Theaded IRQ相关配置**

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
