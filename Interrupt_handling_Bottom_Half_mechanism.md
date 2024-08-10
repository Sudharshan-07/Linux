<div align="center">
  <h1> Interrupt Handling - Bottom Half Mechanisms </h1>
</div>

The interrupt subsystem has an important design mechanism known as the **"Top-Half" and "Bottom-Half"**. This mechanism handles urgent tasks in the Top-half(handling the interrupt) and defers time-consuming tasks to the Bottom-half. This design ensures that the Top-half processing is completed as quickly as possible. But why is this design necessary? Let's break it down with an example:

When the ARM processor handles an interrupt, it switches to an exception mode, during which interrupts are disabled. After processing, the interrupts are re-enabled. If there were no separation between the Top-half and Bottom-half, the next interrupt could only be handled after the previous one was fully processed. If an interrupt takes a long time to handle, other interrupts could be missed, which is unacceptable. For instance, clock interrupts, which act as the system's pulse, must be handled promptly.

Dividing interrupt handling into Top-half and Bottom-half improves the system's ability to respond to interrupts. After the Top-half is processed (which should be done as quickly as possible), interrupts are re-enabled, allowing other interrupts to be handled. The Bottom-half processing is then performed after the interrupt exits.

The Bottom-half mechanism includes **softirq, tasklets, workqueues, and interrupt threading**. These components work together to handle deferred tasks, which is the focus of this discussion.

Understanding various contexts is crucial for comprehending interrupt handling. Different contexts are distinguished by the state of the **task_struct structure and thread_info.preempt_count.**

**struct task_struct:** structure's "thread_info.preempt_count" field is used to track the current task's context status. <br>

**PREEMPT_BITS:** This field records how many times preemption has been disabled. The value increments by 1 each time preemption is disabled and decrements by 1 when it is enabled. <br>

**SOFTIRQ_BITS:** This field is used for synchronization during the Bottom-half processing. It increments by 1 when the Bottom-half is disabled and decrements by 1 when it is enabled. <br>

**HARDIRQ_BITS:** This field indicates when the system is in a hardware interrupt context. <br>

With this background covered, let's dive into the details. <br>

