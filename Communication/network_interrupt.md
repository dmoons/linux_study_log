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





