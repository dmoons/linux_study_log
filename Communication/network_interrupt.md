Interrupts

> refs: Understanding Linux Networking internals

The kernel use two main techniques for exchanging data: polling and interrupts.

- polling: kernel constantly checking a memory register on the device about if the device has anything to say.

- interrupts: kernel instructs the device to generate a hardware interrupt when specific event occur. When interrupt happens, kernel will then invoke a handler registered by the driver to take care of the device's needs. Unfortunately, it does not perform well under high traffic loads: forcing an interrupt for each frame received can easily make the CPU waste all of its time handling interrupts.

`receive-livelock`： at some point the input queue will be full, but since the code that is supposed to dequeue and process those frames does not have a change to run due to its lower priority, the system collapses. New frames cannot be queued since there is no space, and the old frame cannot be processed because there is no CPU available for them.

- Timer-Driven interrupts：　The driver generate interrupts at intervals, but only if it has something to say.

Under low load, the pure interrupt model guarantees a low latency, but that under high load it performs terribly.

`interrupt context` : interrupts are disabled for the CPU serving the interrupt.
> 1. The device generates an interrupt and the hardware notifies the kernel
> 2. if the kernel is not serving another interrupt it will see the notification
> 3. The kernel disables interrupts for the local CPU and exectutes the handler associated with the interrupt type received.
> 4. The kernel exist the interrupt handler and re-enables interrupts for the local CPU.
> *interrupt handlers are nonpreemptible and non-reentrant(a function is defined as **non-reentrant** when it cannot be interrupted by another invocation of **itself**)*

Network drvices have a relatively complex job: they need to *allocate a buffer(sk_buff), copy the received data into it*, *initialize a few parameters within the buffer structure to tell the higher-layer protocol handlers what kind of data is coming from the driver* and so on.

Modern interrupt handlers are divided into a top half and a bottom half. The top half consists of everything that haas to be executed before releasing the CPU, to preserve data. The bottom half consists of everything that can be done at relative leisure.

**Concurrency and locking**  
There are *three* kinds of bottom half functions involved:
- old-styple bottom half: can run at any time, regardless of the number oif CPUs
- tasklets : only one instance of each tasklet can run at any time, different tasklets can run concurrently on different CPUs.
- softirq : the same softirq can run on different CPUs concurrently.



**Preemption**  
The following functions control preemption:
- preempt_disable : Disable preemption for the current task. Can be called repreatedly incrementing a reference counter.
- preempt_enable, preemt_enable_no_resched : The reverse of preepmpt_disable, allowing preemption to be enabled again. Preemp_enable, in addition, checks whether the counter is zero and forces a call to schedule() to allow any higher-priority tasks to run.

A counter for each process, named preempt_count and embeded in the thread_info structure, indicates whether a given process allows preemption, can be read by `preempt_count`, preemp_count is split into three components, `each  byte is a counter for a different condition that requires nonpreemption`: hardware interrupts(irq_enter, irq_exit), software interrupts(local_bh_disable, local_bh_enable), and general non prermption(preempt_disable, preempt_enable).


**Handle bottom-half in kenrel 2.2**

bottom-half types are listed in a enum, take NET_BH as example.  
Each bottom-half type is associated with a function handler **init_bh**:  
> init_bh(NET_BH, net_bh)

This initializes the NET_BH bottom-half type to the net_bh handler.   
Whenever an interrupt handler wants to trigger the execution of a bottom half handler, it has to set the corresponding flag with **mark_bh**, it sets a bit into a global bitmap *bh_active*,
 
> extern inline void mark_bh(int nr)  
> {  
> set_bit(nr, &bh_active);   
> };

Every time a network device driver has successfully received a frame, it 1) signals the kernel about it with a call to *netif_rx*, 2)  netif_rx queues the newly received frame into the ingress queue backlog, and 3) marks the NET_BH bottom-half handler flag.

> skb_queue_tail(&backlog, skb);  
> mark_bh(NET_BH);  
> return;  

During several rountine operations, the kernel checks whether any bottom halves are scheduled for execution.
- do_IRQ : what could give them less latency than an invocation right at the end of do_IRQ?
- Returns from interrupts and exceptions
- schedule: This function decides what to execute next on the CPU, checks if any bottom-half handlers are pending and gives them higher priority over other tasks.
- Bottom halves in 2.2 and earilier kernels suffer fomr a ban on concurrency. Only one bottom half can run at any time, regardless of the number of CPUs.


**Softirqs**  
Which can be seen as tthe multithreaded version of bottom half handlers. Not only can manyu softirqs run concurrently, but also the same softirq can run on different CPUs concurrently.
*The only restriction on concurrency is that only one instance of each softirq can run at the same time on a CPU*  

Softirq has only 6 types
- HI_SOFTIRQ  
- TIMER_SOFRIRQ  
- NET_TX_SOFTIRQ  
- NET_RX_SOFTIRQ  
- SCSI_SOFRIRQ  
- TASKLET_SOFTIRQ

Each softirq type can maintain an array of data structures of type softnet_data, one per CPU, to hold stata informatoin about the current sofirq. softirq handlers are registered with the open_sofrirq function. open_softirq simply copies the input parameters into the global array *softirq_vec*, declared in kernel/softirq.c, which holds the associations between types and handlers.

A softirq can be scheduled for execution on the local CPU by the following functions:
- raise_softirq_irqoff : counterpart of mark_bh, simply sets the bit flag associated to the softirq to run, when the flag is checked, the associated handler will be invoked.

**Tasklets**  

Tasklets are built on top of softirqs and are usually kicked off by interrupt handlers.
HI_SOFTIRQ is used to implement high-priority tasklets, and TASKLET_SOFIRQ is used for lower-priority ones, each time a request for a deferred execution is issued, an instance of a tasklet_struct structure is queued onto either a list processed by HI_SOFTIRQ or another one that is instead processed by TASKLET_SOFTIRQ.

Since softirqs are handled indenpendly by each CPU,it should not be a suprise that there are two lists of pending tasklet_structs for each CPU, one associated with HI_SOFRIRQ and  one with TASKLET_SIFTIRQ

