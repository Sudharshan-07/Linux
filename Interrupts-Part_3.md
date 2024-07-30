<div align="center">
  <h1> INTERRUPTS-PART 3  </h1>
</div>

## Objective:
In [Interrupts-Part_2](https://github.com/Sudharshan-07/Linux/blob/Linux-driver-model/Interrupts-Part_2.md), we explored Layer 1 and Layer 2 of the Linux kernel's interrupt subsystem software layers, discussing the underlying hardware GIC controller driver and architecture-specific interrupt code in the kernel. In this part, we will focus on the generic, hardware-independent interrupt-handling framework of the Linux kernel, covering the L3 and L4 layers.

##### This article will address two questions:
1. How do device drivers in the L4 level register interrupt?
2. When a peripheral triggers an interrupt signal, how is the interrupt handler ultimately called? Though this was partially covered in part 2, we will now delve into the underlying call flows with visual representations.

### IRQ Data Structure Links Analysis:
Let's start by examining the overall IRQ data structure, with the core centered around **struct irq_desc:**

**Fig:1**

![IRQ-DS-flow](https://github.com/user-attachments/assets/ea6ec013-11f6-434c-b1d8-b3928dc85b49)

<br>

The Linux kernel's interrupt processing revolves around the interrupt descriptor structure **struct irq_desc**. The kernel offers two ways to organize these interrupt descriptors:

**Fig:2**

![sparse-irq-1](https://github.com/user-attachments/assets/d646814f-dda9-481e-8074-7ab438b790e5)

<br>

#### Sparse IRQ:
- If the kernel config option **CONFIG_SPARSE_IRQ** is enabled, dynamically allocate struct irq_desc structures for required IRQ numbers and manage them using Radix Tree.
- arm64 based systems enables the CONFIG_SPARSE_IRQ kernel config by default.
- Functions used to create/destroy an irq descriptor:
  - irq_alloc_desc*()
  - irq_free_desc*()

##### > Allocation flow during early kernel init:

![early_irq_init-1](https://github.com/user-attachments/assets/589b7728-1782-4b48-b1d0-b08ab91b2744)

<br>

#### Flat IRQ:
- If you do not use kernel options, an array of irq_dest structures equal to the max IRQ number is statically allocated and used at compile time.
Regardless of the above ways, the corresponding interrupt descriptor in Linux can ultimately be found using the Linux IRQ number.

##### > Allocation flow during early kernel init:

![early_irq_init-2](https://github.com/user-attachments/assets/e24c7fc7-7cbd-427d-a667-a839c321ceca)

<br>


Therefore, from **Fig 1**, The grey section on the left represents the L2 software layer, where the interrupt controller driver initializes struct irq_chip and irq_domain. The grey area at the top of the figure depicts the creation of the interrupt descriptor, accomplished during the process of acquiring device interrupt information. The remaining parts of the figure are configured during L4 software layer device driver initialization, including setting up "struct irqaction" to link to the specific interrupt handler function.

##### Below is the structure declaration of irq_desc:
```
/**
 * struct irq_desc - interrupt descriptor
 * @irq_common_data:    per irq and chip data passed down to chip functions
 * @kstat_irqs:         irq stats per cpu
 * @handle_irq:         highlevel irq-events handler
 * @preflow_handler:    handler called before the flow handler (currently used by sparc)
 * @action:             the irq action chain
 * @status:             status information
 * @core_internal_state__do_not_mess_with_it: core internal status information
 * @depth:              disable-depth, for nested irq_disable() calls
 * @wake_depth:         enable depth, for multiple irq_set_irq_wake() callers
 * @tot_count:          stats field for non-percpu irqs
 * @irq_count:          stats field to detect stalled irqs
 * @last_unhandled:     aging timer for unhandled count
 * @irqs_unhandled:     stats field for spurious unhandled interrupts
 * @threads_handled:    stats field for deferred spurious detection of threaded handlers
 * @threads_handled_last: comparator field for deferred spurious detection of theraded handlers
 * @lock:               locking for SMP
 * @affinity_hint:      hint to user space for preferred irq affinity
 * @affinity_notify:    context for notification of affinity changes
 * @pending_mask:       pending rebalanced interrupts
 * @threads_oneshot:    bitfield to handle shared oneshot threads
 * @threads_active:     number of irqaction threads currently running
 * @wait_for_threads:   wait queue for sync_irq to wait for threaded handlers
 * @nr_actions:         number of installed actions on this descriptor
 * @no_suspend_depth:   number of irqactions on a irq descriptor with
 *                      IRQF_NO_SUSPEND set
 * @force_resume_depth: number of irqactions on a irq descriptor with
 *                      IRQF_FORCE_RESUME set
 * @rcu:                rcu head for delayed free
 * @kobj:               kobject used to represent this struct in sysfs
 * @request_mutex:      mutex to protect request/free before locking desc->lock
 * @dir:                /proc/irq/ procfs entry
 * @debugfs_file:       dentry for the debugfs file
 * @name:               flow handler name for /proc/interrupts output
 */

struct irq_desc {
        struct irq_common_data  irq_common_data;
        struct irq_data         irq_data;
        unsigned int __percpu   *kstat_irqs;
        irq_flow_handler_t      handle_irq;
#ifdef CONFIG_IRQ_PREFLOW_FASTEOI
        irq_preflow_handler_t   preflow_handler;
#endif
        struct irqaction        *action;        /* IRQ action list */
        unsigned int            status_use_accessors;
        unsigned int            core_internal_state__do_not_mess_with_it;
        unsigned int            depth;          /* nested irq disables */
        unsigned int            wake_depth;     /* nested wake enables */
        unsigned int            tot_count;
        unsigned int            irq_count;      /* For detecting broken IRQs */
        unsigned long           last_unhandled; /* Aging timer for unhandled count */
        unsigned int            irqs_unhandled;
        atomic_t                threads_handled;
        int                     threads_handled_last;
        raw_spinlock_t          lock;
        struct cpumask          *percpu_enabled;
        const struct cpumask    *percpu_affinity;
#ifdef CONFIG_SMP
        const struct cpumask    *affinity_hint;
        struct irq_affinity_notify *affinity_notify;
#ifdef CONFIG_GENERIC_PENDING_IRQ
        cpumask_var_t           pending_mask;
#endif
#endif
        unsigned long           threads_oneshot;
        atomic_t                threads_active;
        wait_queue_head_t       wait_for_threads;
#ifdef CONFIG_PM_SLEEP
        unsigned int            nr_actions;
        unsigned int            no_suspend_depth;
        unsigned int            cond_suspend_depth;
        unsigned int            force_resume_depth;
#endif
#ifdef CONFIG_PROC_FS
        struct proc_dir_entry   *dir;
#endif
#ifdef CONFIG_GENERIC_IRQ_DEBUGFS
        struct dentry           *debugfs_file;
        const char              *dev_name;
#endif
#ifdef CONFIG_SPARSE_IRQ
        struct rcu_head         rcu;
        struct kobject          kobj;
#endif
        struct mutex            request_mutex;
        int                     parent_irq;
        struct module           *owner;
        const char              *name;
} ____cacheline_internodealigned_in_smp;
```

The following figure shows how one irq descriptor is allocated and initialized:

![alloc_desc-1](https://github.com/user-attachments/assets/5b92fdc3-9c8c-4c3a-80b8-4aa924dd89d6)

<br>

#### Interrupt processing primarily involves the following functional components:

1. Mapping hardware interrupt numbers(hwirq) to Linux IRQ(virq) numbers and allocating struct irq_desc interrupt descriptor based on above mentioned methods(Flat or Sparse).
   
2. During interrupt registration(i.e, while calling request_irq()), the device's interrupt number is retrieved(via DTS or ACPI), the matching irq_desc is identified, and the device's interrupt handler is linked to the corresponding irq_desc's struct irqaction's handler field.

3. When an interrupt is triggered, the L2 Layer determines the IRQ number from the hardware interrupt number, finds the associated irq_desc, and calls the corresponding interrupt handler function.


### How do device drivers in the L4 level register interrupt?

Linux Software engineers familiar with device drivers will know that drivers often call the **request_irq() or request_threaded_irq()** interfaces to register the interrupt handling function of the device. An **"irq"** parameter, which represents the Linux interrupt number, is required in these interfaces. This raises the question: where does this Linux interrupt number come from? How is the Linux IRQ number mapped to the interrupt number of a specific hardware device's interrupt pin?.

How is the Linux IRQ number mapped to the interrupt number of a specific hardware device?

<br>

![retrive-interrupt-number-DTS](https://github.com/user-attachments/assets/62bd2e6a-ea50-4726-a7d3-05ac7510b910)

<br>

The interrupt information of hardware devices is described in the device tree(DTS). During system startup, this information is loaded into memory and parsed. The L4 level device driver typically uses the **platform_get_irq() or irq_of_parse_and_map()** interfaces to create a mapping relationship (hardware interrupt number to Linux IRQ number) based on the information in the device tree.

**[Interrupts-Part_2 mentioned that "struct irq_domain" is used for this mapping](https://github.com/Sudharshan-07/Linux/blob/Linux-driver-model/Interrupts-Part_2.md#-l_223-irq_domain-analysis)**. In the **irq_create_fwspec_mapping()** interface, the function first finds the matching IRQ domain and then calls back the function assigned to it. This IRQ domain is usually initialized in the interrupt controller driver. For example, consider ARM GICv2, it calls back to the function **"gic_irq_domain_hierarchy_ops"**.

If the mapping has already been created, the Linux IRQ number can be returned directly. Otherwise, **irq_domain_alloc_irqs()** needs to generate the mapping relationship. This function completes two tasks:
1. Creates an irq_desc interrupt descriptor for the Linux IRQ number.
2. Calls domain->ops->alloc to complete the mapping. In the ARM GICv2 driver, this corresponds to the gic_irq_domain_alloc function, which is crucial and introduced below.





















