===========================
X86 User Interrupts Summary
===========================

Revision
========
V1.0 Liao Chang:
    2021.11.10 总结X86 User IPI/User Interrupt方案。

本文介绍Intel在2021.5月发布的文档architecture-instruction-set-extension-programming-reference
里引入的User IPI和User Interrupts硬件特性，以及基于该特性开发的内核补丁。

- 第一章介绍X86 User Interrupts的潜在使用场景。
- 第二章介绍X86 User Interrupts的硬件架构。
- 第三章介绍X86 User Interrupts在用户态的软件模型和API。
- 第四章介绍User Interrupts的内核补丁(Uintr)。
- 第五章介绍User Interrupts在LKML讨论中反映的问题。
- 第六章是参考资料。

1. 潜在场景
===========

这个补丁利用X86 User-IPI和User Interrupt机制来改善用户态进程的通知性能，
并且和传统的IPC机制(eventfd, signal，pipe)进行了比较。

**性能微观数据**

利用了不同手段测试了1M Ping-Pong IPC的时延，下表反映User IPI/Interrupt相比其他纯软件IPC时延有x9的改善。

+-------------------+----------------------------+
| IPC type          | |   Relative Latency       |
|                   | | (normalized to User IPI) |
+-------------------+----------------------------+
| User IPI          | 1.0                        |
+-------------------+----------------------------+
| User IPI(blocked) | 8.9                        |
+-------------------+----------------------------+
| Signal            | 14.8                       |
+-------------------+----------------------------+
| Eventfd           | 9.7                        |
+-------------------+----------------------------+
| Pipe              | 16.3                       |
+-------------------+----------------------------+
| Domain            | 17.3                       |
+-------------------+----------------------------+

主要使用了以下benchmark来收集数据：

#. https://github.com/goldsborough/ipc-bench
#. https://github.com/intel/uintr-ipc-bench/tree/linux-rfc-v1



Greg KH也抛出了一个问题是：该特性除了能耐提高"one userspace task waking up another userspace"的性能，
是否在其他更加真实的负载也有收益。(what real workload can actually use this)

Jens Axboe提到一个场景是：liburing/io_uring当前利用eventfd来做通知，可以使用该机制避免任务必须陷入到内核态来接收通知。

其他参与讨论者对User IPI和User Interrup的实际使用场景也表示一定质疑。

2. 硬件架构
===========

2.1 介绍
--------

软件可以在用户态直接触发一个硬件事件，硬件提供一套机制保证某个特定线程来响应该事件，
且软件直接在用户态响应中断，且不必发生特权级切换。

软件通过一个6比特的中断向量区分事件，当CPU陷入用户态中断时，硬件将中断向量被压入栈里，
并跳转到用户态中断处理函数入口，当中断处理函数结束后，通过特殊指令 **UIRET** 返回，
CPU回到被打断现场继续执行。

上述机制围绕两个关键的内存数据对象来实现：

- **UPID(user posted-interrupt descriptor)**
- **UITT(user interrupt target table)**

硬件会提供一组MSR对用户态中断进行管理，由于用户态中断投递目标是OS管理的线程，
所以OS在线程上下文切换时，还需要同步更新这些MSR。OS管理的线程必须被分配一个UPID对象，
才能具备响应用户态中断的能力。

硬件提供了一条指令 **SENDUIPI** ，在ring3下执行用于触发用户态中断，
该指令会使用UITT组中断路由，保证中断事件能发送到目标线程所在CPU上。

2.2 硬件概念和接口
------------------

用户态中断相关的硬件概念有三类：

- 全局配置和查询
- 用户态中断的发布: 包括触发和路由
- 用户态中断响应: 中断上下文配置，中断陷入

表-1 User-Interrupts全局控制接口：

+-----------+-------------------------------------------------------------------+
| CR4.UINTR | 系统级别的User-Interrupts使能开关。                               |
+-----------+-------------------------------------------------------------------+
| CPUID     | CPUID指令返回值保存在EDX[5]，表明该CPU支持User-Interrupts特性。   |
+-----------+-------------------------------------------------------------------+

表-2 和用户态中断响应相关概念

(注意：这些概念都实现在LAPIC上，所以是Per-CPU粒度)

+---------------+-------------------------------------------------------------------+
| UIRR          | | user-interrupt request register.                                |
|               | | MSR: IA32_UINTR_RR                                              |
|               | | 记录线程待处理的用户态中断状态。 每位代表一个中断向量的Pending  |
|               | | 状态，对应位置1表明对应中断向量处于Pending。如果UIRR有多个中断  |
|               | | 向量都处于Pending状态，将选择UIRR里的MSB对应的中断向量优先处理  |
+---------------+-------------------------------------------------------------------+
| UIF           | | user-interrupt flags.                                           |
|               | | MSR: IA32_XSS                                                   |
|               | | 对投递用户态中断事件起开关作用。                                |
|               | | 当IA32_XSS.UIF==0时，当通过SETUI触发用户态中断后，中断将被阻塞, |
|               | | 不能投递到目标线程，当IA32_XSS.UIF==1时，用户态中断事件将被投递 |
|               | | 目标线程。                                                      |
|               | | 注意：当用户态中断被成功投递后，硬件将IA32_XSS.UIF设置为0，直到 |
|               | | UIRET指令被执行后才会重新设置为1，这种机制应该是防止用户态中断  |
|               | | 嵌套。                                                          |
+---------------+-------------------------------------------------------------------+
| UIHANDLER     | | user-interrupt handler.                                         |
|               | | MSR: IA32_UINTR_HANDLER                                         |
|               | | 保存用户态中断处理函数的线性地址，当CPU陷入中断时，将UIHANDLER  |
|               | | 值填入RIP里。                                                   |
+---------------+-------------------------------------------------------------------+
| UISTACKADJUST | | user-interrupt stack adjustment.                                |
|               | | MSR: IA32_UINTR_STACKADJUST                                     |
|               | | 当CPU陷入中断时，利用该MSR里的值对RSP进行调整。                 |
|               | | 如果UISTACKADJUST[0]==1，将IA32_UINTR_STACKADJUST值加载到RSP后，|
|               | | 对其按照16字节对齐处理，如果UISTACKADJUST[0]==0，将RSP的值减去  |
|               | | IA32_UINTR_STACKADJUST的值后，对其按照16字节对齐处理。          |
+---------------+-------------------------------------------------------------------+
| UINV          | | use-interrupt notification vector.                              |
|               | | MSR: IA32_UINTR_MISC                                            |
|               | | 用户态中断通知向量，当LAPIC收到一个中断消息后，会从中断消息里提 |
|               | | 取的中断向量和UINV的值进行比较，如果相同就意味着，LAPIC收到了一 |
|               | | 个User-Interrupts。                                             |
+---------------+-------------------------------------------------------------------+

表-3 和用户态中断发送相关概念：

+---------------+-------------------------------------------------------------------+
| UPIDADDR      | | user posted-interrupt descriptor address.                       |
|               | | MSR: IA32_UINTR_PD                                              |
|               | | 每个需要处理用户态中断的线程都需要关联一个UIPD对象，这个对象由  |
|               | | OS分配保存在内存里，对应的线性地址通过UPIDADDR接口访问。        |
|               | | 当用户态中断触发时，硬件会根据UPID里的信息发送用户态中断。      |
+---------------+-------------------------------------------------------------------+
| UITTADDR      | | user-interrupt target table address.                            |
|               | | MSR: IA32_UINTR_TT                                              |
|               | | 当软件通过SENDUIPI指令触发用户态中断时，会从UITT里查询中断对应  |
|               | | UPID对象，然后在发送中断。                                      |
|               | | UITT由OS分配，对应的线性地址通过UITTADDR访问。                  |
+---------------+-------------------------------------------------------------------+
| UITTSZ        | | user-interrupt target table size                                |
|               | | MSR: IA32_UINTR_MISC                                            |
|               | | UITT的表条目数量。                                              |
+---------------+-------------------------------------------------------------------+

表-4 用户态中断相关指令：

+----------+-------------------------------------------------------------------+
| senduipi | | 用法：senduipi <index>                                          |
|          | | 在用户态给特定目标线程发送user-ipi。                            |
+----------+-------------------------------------------------------------------+
| clui     | | 用法：clui                                                      |
|          | | 清零本CPU的UIF，以到达mask本CPU用户态中断效果。                 |
+----------+-------------------------------------------------------------------+
| stui     | | 用法：stui                                                      |
|          | | 设置本CPU的UIF，以达到unmask本CPU用户态中断效果。               |
+----------+-------------------------------------------------------------------+
| testui   | | 用法：testui                                                    |
|          | | 检测本CPU的UIF的值。                                            |
+----------+-------------------------------------------------------------------+
| uiret    | | 用法：uiret                                                     |
|          | | 从用户态中断处理上下文恢复                                      |
+----------+-------------------------------------------------------------------+

上述硬件概念间的逻辑关系见下图：

.. figure:: images/UINTR_concept.png
   :scale: 40 %
   :align: center

   图-1 用户态中断硬件模型
..

上图所示的用户态中断的逻辑模型：

1. 软件通过“SENDUIPI reg”触发用户态中断(可以在Ring3操作)，硬件利用UITTSIZE检测reg的有效性，
   确保reg不超过UITT条目数量，然后利用UITTADDR获取UITT条目线性地址。

.. note::

  UITT起到中断路由表的作用，可以类比为GIC ITS的device table和interrupt translation table。

2. UITT条目包含两种信息：“UIV” 和“UPIDADDR”，其中“UIV”就是中断ID，用户态中断处理函数根据UIV调用对应ISR。
   “UPIDADDR”用于访问目标线程关联的UPID对象，硬件将“UIV”记录到UPID.PIR里，并且根据UPID.Notification_destination
   选择中断路由目标LAPIC。

.. note::

  UITT条目将用户态中断号和目标线程做一层映射，并且利用线程的UPID选择中断路由目标。
  可以把UPID类比GIC ITS的VPE table和vpending table。

3. 硬件从线程关联的UPID取出UPID.Notification_vector，将其发往UPID.Notification_destination对应的LAPIC，
   目标LAPIC收到notifcation vector后，会和本地UINV进行比较，如果相同就任务收到了一个用户态中断，
   然后通过本地UPIDADDR的访问线程的UPID对象，将UPID.PIR的值填充到UIRR里，然后将RIP设置为UIHANDLER，
   RSP设置为USTACKADJUST，最后保存现场开始处理中断。

.. note::

  需要区别user interrupt notification vector和user interrupt vector，硬件往目标LAPIC投递中断的向量是前者，
  我理解这个概念用于判断“目标CPU上运行的线程是否该中断的真正处理者”，硬件将后者记录到内存的UPID里，
  我理解这种做法是防止，目标线程在CPU间迁移也能保证不会丢失中断。

4. 所以，当用户态中断触发线程被OS调入时，需要更新本地的UITTADDR，UITTSIZE。
   当用户态中断处理线程被OS调入时，需要更新本地的UPIDADDR，UIHANDLER，UISTACKADJUST，UINV。

2.3 硬件语义
------------

CPU响应用户态中断主要由四个环节构成：

- **中断确认**:

  #. LAPIC收到了一个notificatio vector后，和MSR IA32_UINTR_MISC.UINV进行比较，如果相同就表示，当前CPU上的线程可以处理该用户态中断。
  #. 然后通过IA32_UINTR_PD访问内存里的UPID，将UPID.Outstanding_notification字段清零，
  #. 然后将UPID.PIR保存到一个临时寄存器里，并将UPID.PIR清零。
  #. 如果临时寄存器里的PIR是非零，就该值OR到IA32_UINTR_RR寄存器里。

- **识别中断**:

  当CPU检测到IA32_UINTR_RR值非0，就认为有用户态中断处于Pending状态，将会启动中断投递操作，

- **投递中断**:

  当满足如下条件时，用户态中断将投递到CPU上。
  
  #. UIF==1，即允许投递用户态中断。
  #. CPL==3，即CPU运行在Ring3。
  #. IA32_EFER.LMA=CS.L=1，即CPU运行在64位模式。
  #. 目标线程不在enclave里执行。

  如果CPU通过TPAUSE和UNWAIT指令进行低功耗模式，投递的用户态中断可以将CPU唤醒，
  但如果CPU进入的是'shutdown'或者'wait-for-SIPI'状态，无法被用户态中断唤醒。

- **中断陷入**:

  当满足投递中断的条件后，CPU将执行完下列操作后陷入中断处理函数：

  #. 根据IA32_UINTR_STACKADJUST的值，设置RSP寄存器的值，用户态中断处理函数使用该栈。
  #. 从IA32_UINTR_RR提取MSB对应的中断向量，即UIRRV，这个值就是待处理的中断向量，并将UIRRV压入中断栈。
  #. 将中断现场的RSP，RFLAGS，RIP压入中断处理栈。
  #. 将UIRRV在IA32_UINTR_RR对应的位清0，表示该中断向量已经被响应了。
  #. 将UIF清零，防止保存用户态中断现场时，又发生中断陷入。
  #. RFLAGS.TF和RFLAGS.RF清0，
  #. 将RIP设置为IA32_UINTR_HANDLER的值，CPU将跳转到用户态中断处理函数入口。

3. 用户态模型和API
==================

3.1 软件协作模型
----------------

.. figure:: images/uintr_sequence.png
   :scale: 70 %
   :align: center

   图-2 用户态IPI处理流程
..

3.2 用户态接口
--------------

- **uintr_register_handler(handler, flags)：**

  用户态线程通过该syscall将自己注册为用户态中断接收线程，并设置用户态中断处理函数，
  每个线程只能注册一个用户态中断处理函数，只有成功调用了该syscall，用户态线程才可以响应用户态中断。

- **uintr_create_fd(vector, flags)：**

  用户态线程注册了中断处理函数后，该线程通过该syscall注册用户态中断向量，每个中断处理函数可以处理64种中断向量，
  该syscall还会返回一个fd，这个fd关联调用线程注册的uintr_handler和uintr_vector，其他线程可以利用该fd往线程发送用户态中断。
  中断对应的向量是uintr_vector，响应中的处理函数是uintr_handler。

- **uintr_register_sender(uintr_fd, flags)：**

  用户态线程通过该syscall将自己注册为用户态中断发送线程，其中uintr_fd就是来自用户态中断接收线程，
  该syscall返回一个index，用户态中断发送线程利用指令“senduipi <index>”发送用户态中断，
  中断发送线程可以多次调用该syscall，返回不同的index，就能给不同线程发送不同用户态中断。

.. note::

  理论上，内核可以直接利用uintr_fd触发用户态中断，但该RFC还未支持。

- **uintr_wait()：**

  用户态中断接收线程可以调用该系统调用阻塞到内核态，一旦内核接收到用户态中断后，
  可以立即唤醒阻塞在uintr_wait上的线程。但如果线程不是通过uintr_wait阻塞在内核态，
  比如阻塞在read，write上，就必须等待被条件满足的情况下，比如，IO条件满足的情况下由内核唤醒。

- **senduipi <index>：**

  用户态中断发送线程通过senduipi指令给特定用户态线程发送中断，index是通过uintr_register_sender返回值。

- **uintr_unregister_handler(flags)：**

  用户态线程注销中断处理函数，调用该syscall后，线程将不再具备响应用户态中断能力。


- **uintr_unregister_sender(uintr_fd, flags)：**

  用户态中断发送线程主动解除发送特定中断向量的能力。

3.3 工具链的支持
----------------

GCC-11.1和Binutils-2.36.1支持了用户态中断指令和“-muintr”编译选项。

- “-muintr”编译选项可以将为中断处理函数自动生成，保存和恢复上下文指令，通用中断入口和返回值指令，uiret返回打断点。

- 支持clui, stui, testui, uiret, senduipi指令。

3.4 开放性问题
--------------

- 用户态中断是否提供类似signal类似的语义，比如，允许用户态中断打断sleep，read，poll这些系统调用，
  并提供SA_RESTART来重启这些被打断系统调用。

- 对于UITT(中断路由表)的共享范围还不确定，该RFC提供的实现里UITT是一个线程私有数据，
  是否支持：同进程内多线程间共享，在多进程间共享，都待讨论。

- 用户态IPI依赖的senduipi是一个Ring3(user mode)指令，图-1模型所示，该指令会产生一个潜在的supervisor
  mode的内存访问(UITT和UPID)，这个要求关闭KPTI特性(解决meltdwown漏洞)，
  如果要保留KPTI功能，就需要保存UITT和UPID的内存映射到用户态(计划在下个版本的RFC实现)

4. 内核补丁
===========

.. code-block:: bash

    Sohil Mehta (13):
      x86/uintr/man-page: Include man pages draft for reference
      Documentation/x86: Add documentation for User Interrupts
      x86/cpu: Enumerate User Interrupts support
      x86/fpu/xstate: Enumerate User Interrupts supervisor state
      x86/irq: Reserve a user IPI notification vector
      x86/uintr: Introduce uintr receiver syscalls
      x86/process/64: Add uintr task context switch support
      x86/process/64: Clean up uintr task fork and exit paths
      x86/uintr: Introduce vector registration and uintr_fd syscall
      x86/uintr: Introduce user IPI sender syscalls
      x86/uintr: Introduce uintr_wait() syscall
      x86/uintr: Wire up the user interrupt syscalls
      selftests/x86: Add basic tests for User IPI
..

上述补丁的结构如下：

- 手册和内核文档: patch 1, 2
- 硬件特性枚举和定义中断XSAVE对象: patch 3, 4
- 分配用户态中断的notification vector：patch 5
- 用户态中断相关的系统调用实现：patch 6~12
- 测试用例：patch 13

这个RFC期望达成的主要目标是：

#. 往业界和社区推广X86 User Interrupt硬件架构。

#. 探讨User Interrupt的应用场景，Intel已经在libevent和liburing这两个库上挖掘了一些应用场景，见2.1节,
   希望社区能反馈更多潜在场景。

#. 请社区检视User Interrupt软件栈的整体框架和实现方案。

#. 有些实现策略仍处于开放状态，需要社区帮忙一起进行决策，间2.3节。

**核心功能分析**

- **[Patch 6/13] x86/uintr: Introduce uintr receiver syscalls：**

  uintr_register_handler系统调用流程包括，
  分配UPID对象，初始化UPID对象，根据UPID对象设置中断响应相关的MSR(见表2)
  并将UPID对象登记到task_struct结构体里，每个task_struct对象只能和一个UPID对象关联起来。

.. note::

   UPID对象比较特殊，中断接收线程(读取pending vector)和中断发送线程(设置pending vector)
   都会使用，所以使用refcount来管理它的生命周期。
..

  uintr_unregister_handler系统调用流程包括，减少UPID对象引用计数，重置用户态中断相关的MSR，
  如果存在中断发送线程，此时并不会直接释放UPID对象。

- **[Patch 7/13] x86/process/64: Add uintr task context switch support：**

  用户态中断相关MSR(UIRR, UIF, UIHANDLER, UISTACKADJUST)属于任务上下文，
  中断接收线程schedule-out时，要通过XSAVES指令保存这些MSR，同时设置UPID.suppress_notification字段标记中断处理线程已下线；
  中断接收线程schedule-in时，要通过SRSTORS指令恢复这些MSR，同时清理UPID.suppress_notification字段标记中断处理线程已上线，
  在多核环境下，还需要更新UPID对象里中断路由相关的字段。
  
.. note::

   中断接收线程比调入后，还要检测UPID.PIR字段是否非0，如果非0表示调出期间收到了中断，
   内核还需要更新UIRR，否则回到用户态也无法响应中断。

- **[Patch 8/13] x86/process/64: Clean up uintr task fork and exit paths：**

  在中断接收线程的exit流程里，需要减少UPID对象的引用计数。在fork流程里，新创建出来的task_struct对象不能继承父线程的UPID对象。

- **[PATCH 9/13] x86/uintr: Introduce vector registration and uintr_fd syscall：**

  为了支持用户态IPI，中断接收线程首先需要将中断信息，包括中断UPID和中断向量，通过uintr_fd的形式发布出去，
  其他线程就可以利用uintr_fd发送中断。

  uintr_create_fd系统调用流程包括：创建uintrfd_ctx对象，分配一个匿名的fd，将uintfd_ctx对象设置为fd操作对象，
  uintrfd_ctx对象会关联一个用户态中断向量。中断接收线程调用该接口注册一个中断向量，另外，
  线程可以对uintr_fd执行close注销用户态中断向量。

.. note::

  1. 考虑到执行中断向量注销的线程可能不是中断接收线程，所以对uintrfd_ctx对象的释放是利用task_work机制异步完成的。

  2. uintr_fd被创建出来后，可以将这个fd共享给同线程组其他线程，或者子进程的线程，甚至其他进程使用(通过pidfd_getfd(2)和sendmsg(2))。

- **[PATCH 10/13] x86/uintr: Introduce user IPI sender syscalls：**

  如果线程想具备发送用户态IPI的能力，需要调用uintr_register_sender，
  uintr_register_sender系统调用流程包括：从task_struct关联的UITT里分配一个可用的条目，
  并且返回该条目在UITT的编号。每次调用该系统调用，线程就具备了发送一个特定中断向量的能力，
  所有线程可以多次调用该接口从而具备发送多种中断的能力，不过需要提前获取uintr_fd。

  uintr_unregister_sender系统调用是uintr_register_sender的反向操作。

- **[PATCH 11/13] x86/uintr: Introduce uintr_wait() syscall：**

  中断接收线程调用uintr_wait后会主动放弃CPU，并且将对应UPID.notification_vector字段改成KERNEL_VECTOR，
  当中断发送线程给该接收线程发送的IPI，这个IPI将会由KERNEL_VECTOR对应的内核态中断处理函数进行响应，
  响应的策略就是唤醒调用uintr_wait的线程。

5. 技术讨论
===========

Thomas Gleixner, GregKH，Andy Lutomirski，Dave Hansen，Jens Axboe，Stefan Hajnoczi
参与了该内核补丁集在LKML里的讨论，主要集中在以下话题：

- 通过uintr_register_handler注册的中断处理函数，是否需要对地址的合法性进行校验？
  另外，如果传递了一个非法的地址，硬件跳转到这个非法地址将会触发一个GP异常，
  内核需要对这个异常进行合理处理。

- 对X86 User Interrupt里notification vector的价值和作用有疑问，以及在context
  switch场景下，更新UINV的代码逻辑有疑问。

- 几个关键对象的生命周期管理缺乏详细建模，代码检视者不好理解。

- 对于uintr_wait这个系统调用，要考虑CPU hotplug场景下的正确性。

- 引入uintr_wait的必要性也有待考虑，完全可以利用现有的read/poll等机制来提供类似等待唤醒语义。

- 当目标线程不在运行时，如何更新UIRR，UINV的代码逻辑也有质疑，作者也表示当前实现确实有问题。

6. 参考资料
===========

#. https://software.intel.com/sites/default/files/managed/c5/15/architecture-instruction-set-extensions-programming-reference.pdf
#. https://lkml.org/lkml/2021/9/13/2153
