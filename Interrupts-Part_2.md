<div align="center">
  <h1> INTERRUPTS-PART 2  </h1>
</div>

## Objective:
  In [Interrupts part 1](https://github.com/Sudharshan-07/Linux/blob/Linux-driver-model/Interrupts-Part_1.md), we covered the basic hardware overview and list of structures associated in the Linux kernel. In this part, we will delve deeper into both the hardware aspects and the software implementation involved in processing an interrupt.
  
  This part will explain the interrupt flow from a peripheral to the CPU and describe the corresponding software actions and responses from the CPU back to the peripheral. It is recommended to review Part 1 for a better understanding before proceeding with Part 2.

## Approach to understand the kernel interrupt subsystem:
 As we explore the fundamentals of the interrupt subsystem, we'll divide the concepts into two categories:
 1. Hardware view.
 2. Software layered view.

## Hardware view:

Each architecture has a different interrupt controller configuration, so let's briefly understand that they have different processing methods as shown in the figure below:

**x86 with PIC**
-  In the early x86 systems that used only one CPU, one or more 8259 Programmable Interrupt Controllers (PICs) were used.
  
**x86 with APIC**
-  When configuring SMP in x86, the APIC (Advanced Programmable Interrupt Controller) is divided into two chips, one is used for the core processor chip with the CPU, and the APIC is configured in the south chipset that is in charge of I/O.

**ARM SoC**
-  ARM has a very diverse configuration. Each company that manufactures SoC uses its own interrupt controller logic as an IP block of the SoC. Still, there is a tendency to gradually use ARM's GIC (Generic Interrupt Controller).

<br>

![interrupt-1a](https://github.com/user-attachments/assets/f41971c3-b870-433e-8537-7d88d8104ed7)

Reviewing the above depictions, it is natural to be unclear(from a software point of view) about how the CPU determines the source of an interrupt from different peripherals. This will be addressed in the upcoming sections, where we will explore the data structures used to identify the interrupt source, as well as the roles of an architecture-dependent and hardware-independent interrupt framework implementation in the Linux kernel.

From the above information, it is evident that interrupt controllers have evolved(so did the Linux interrupt subsystem framework) in response to the increasing need for more peripheral connections.


## Software layered view:

Before we delve into the specifics, let us first gain an understanding of the software layer of the Linux kernel related to the interrupt subsystem. Here is the hierarchical view of the software layers:

<br>

![layered_view](https://github.com/user-attachments/assets/934b860e-d28b-4c5c-82a6-f1e10b51efd0)

**Hardware layer(L1 level)**
- The low-level layer corresponds to the hardware connection between peripherals and the SoC. The interrupt signal originates from the peripheral, is sent to the interrupt controller, is managed uniformly by the interrupt controller, and is then routed to the processor. This was discussed earlier in the [Hardware View](https://github.com/Sudharshan-07/Linux/edit/Linux-driver-model/Interrupt-part-2.md#hardware-view) section.

**Architecture-dependent layer(L2 level)**
- This layer consists of two components:
  1. One part involves architecture-specific code, such as interrupt processing for the ARM64 processor.
  2. The other part includes the driver code for the interrupt controller.

- The dotted links shown in orange illustrate the relationship between the code in the L2 layer and the hardware in the L1 layer. Architecture-specific code processes incoming interrupts according to the hardware architecture, and a dedicated interrupt controller driver operates the actual interrupt controller in the L1 layer.

**Architecture-Independent layer(L3 level)**
- This is the interrupt framework layer, which is hardware-independent and common across hardware platforms, and also contains the generic Linux kernel IRQ subsystem implementation.

**Device driver layer(L4)**
- This part is the user of the interrupt pin(device drivers), which registers through interrupt-related interfaces, and finally calls interrupt handlers for processing when the peripheral triggers an interrupt.

To better understand the concept, we need to examine the software stack from the bottom to the top(i.e. from level L1 to L4). 

### What problem are we examining here?
  Let's clearly outline the objectives of our analysis:
- When a peripheral generates an interrupt signal, how is the interrupt handler eventually invoked when multiple interrupt controllers are involved? As depicted in the block diagram below, although there are multiple interrupt sources, the CPU receives only one INTR signal. How does the CPU/Kernel determine the source of the interrupt in this scenario?

![flow3](https://github.com/user-attachments/assets/38f830d5-a18e-47da-94ea-d37114b069d6)

<br>

#### Now, To understand how the kernel uses the software layers to identify the source of an interrupt, we need to examine the significance of each layer in the process.

### Hardware layer (L1):

In addition to the [Hardware view](https://github.com/Sudharshan-07/Linux/edit/Linux-driver-model/Interrupt-part-2.md#hardware-view), let's take an example of ARM-based interrupt controller's(GIC-v2) design. 

Here is a functional block diagram:

<img width="596" alt="ARM-Intrctrlr-design" src="https://github.com/user-attachments/assets/2c129e3f-2242-45e8-9f9d-d374b6640d0b">

Let's touch on how the GIC-v2 operates:

GIC-V2 supports three types of interrupts:

**1. SGI (Software-Generated Interrupts):** These interrupts are used primarily for inter-core communication. The kernel's Inter-Processor Interrupts (IPI) are based on SGIs, with interrupt numbers ranging from ID0 to ID15 designated for SGIs.

**2. PPI (Private Peripheral Interrupts):** Each CPU has its own set of private interrupts, typically used for local timers. The interrupt numbers ID16 to ID31 are assigned for PPIs.

**3. PI (Shared Peripheral Interrupts):** These interrupts can be routed to a specific CPU after they occur. Interrupt numbers ID32 to ID1019 are used for SPIs, with ID1020 to ID1023 reserved for special purposes.

The process of GIC interrupt detection is, The GIC captures the interrupt signal, asserts it, and marks it as pending. The distributor determines the target CPU and sends the interrupt signal to it. Concurrently, for each CPU, the distributor selects the highest priority interrupt from the pending signals and sends it to the CPU interface. The CPU interface in the GIC decides whether to forward the interrupt signal to the target CPU. After the CPU completes interrupt processing, it sends a completion signal (EOI - End of Interrupt) to the GIC.


### Architecture-dependent Layer(L2):
In this layer, we will look into two aspects that uncover the underlying mechanism that processes and manages the interrupt controllers in an SoC. 

Before delving into the actual data flow, let's first understand the process involved when the kernel locates and initializes an interrupt controller. During the initialization of the interrupt controller, various data structures come into play to manage and control the interrupt controllers. 

Breaking down the analysis of the L2 layer into two steps:

- L_2.1: How does the kernel locate information about the interrupt controller?
- L_2.2: Analysis of the GIC controller's driver.
  - L_2.2.1: GIC's private data structure analysis.
  - L_2.2.2: irq_chip analysis.
  - L_2.2.3: irq_domain analysis.

#### > L_2.1: How does the kernel locate information about the interrupt controller?

The ARM platform device information is incorporated through the device tree(DTS). This information is placed in the "arch/arm64/boot/dts/" directory.

The following figure illustrates the device tree information for an interrupt controller:

![DTS](https://github.com/user-attachments/assets/e679706c-7fea-4b25-bcca-b0cddbe8ea67)

**compatible:** Used to match a specific driver. For instance, in the example of "arm,gic-400," this field helps in matching the appropriate driver based on the compatible string.

**interrupt-cells:** Specifies the number of units required to describe an interrupt source. 
For example, a value of 3 is used. In the device tree, you might see an entry like interrupts = <0 23 4> in a device node that uses the "gic" as the parent interrupt controller, where the first unit (0) indicates the interrupt type (1 for PPI, 0 for SPI), the second unit (23) denotes the interrupt number(or the interrupt pin) used by the driver, and the third unit (4) represents the interrupt trigger type.

**reg:** Describes the address and address range of the interrupt controller. For example, it specifies the address information for the GIC Distributor (GICD) and the GIC CPU Interface (GICC) which can be used to memory map in the controller driver to communicate with the device.

**interrupt-controller:** Indicates that the device is an interrupt controller to which peripherals can be connected.

An important point to consider here is that within the device tree, a logical interrupt tree exists that mimics the hierarchy and routing of interrupts in an SoC. While generically referred to as an interrupt tree it is more technically a directed acyclic graph. The physical wiring of an interrupt source to an interrupt controller is represented in the device tree with the help of the "interrupt-parent" property. Device nodes that represent interrupt-generating devices contain an "interrupt-parent" property which has a "phandle" value that points to the device to which the device’s interrupts are routed, typically an interrupt controller. If an interrupt-generating device does not have an interrupt-parent property, its interrupt parent is assumed to be an interrupt controller device node that resides in the root of the device tree.

###### Below is the overview of how the interrupt routing is done in a device tree by mimicking the hierarchy and routing of interrupts in an SoC:
<br>

<img width="744" alt="DTS-interrupt-tree-logic" src="https://github.com/user-attachments/assets/64e499f1-4804-4ed5-a46f-72c9e62e42d2">

##### ***Reference Spec:https://github.com/devicetree-org/devicetree-specification/releases/download/v0.3/devicetree-specification-v0.3.pdf***
<br>

After all, How is the device tree information processed by the kernel?. The Device Tree Source(DTS) is compiled into a DTB file and passed to the kernel through the bootloader. When the kernel starts, it parses and converts the DTB file into a "struct device_node" data structure which maintains a hierarchical tree in the memory. Devices and drivers are matched and bound based on the compatible string.

###### Here is a diagram that provides a brief overview of the call flow on how a DTB is unflattened and converted into "struct device_node" data structures:
![DTS_Parsing](https://github.com/user-attachments/assets/b4e85f74-f407-4297-9bf6-f5d2f1e1cf53)

<br>

#### > L_2.2: Analysis of the GIC controller's driver

The operation of the GIC is fundamentally driven by interrupt signals. Therefore, the GIC driver's task is to initialize various data and register the appropriate callback functions to ensure they are executed when an interrupt signal is received.

##### The following figure illustrates the execution flow of the GIC driver initialization, detailing the kernel initialization and how the device tree-based IRQ controller is initialized:

![GIC-STARTUP-FLOW](https://github.com/user-attachments/assets/63e8e393-ec44-4572-b3a7-5cd6de95f40f)

First, it's important to understand the **"vmlinux.lds"** link script. This script defines a segment called **__irqchip_of_table**. In the GIC's device driver, the **IRQCHIP_DECLARE** macro eventually defines [struct of_device_id](https://github.com/torvalds/linux/blob/master/include/linux/of.h#L1523), including compatible fields and callback functions. This macro places the "struct of_device_id" into the __irqchip_of_table segment.

This table also has the device ID information of all the interrupt controllers that the kernel supports(for each architecture). the information about the interrupt controllers (used to match with the device node) is stored in the **.init.ramfs** section of the kernel image. The "of_irq_init" function scans the device tree for matching interrupt controller nodes and invokes their corresponding init functions in sequence, starting with the parent nodes.

##### Below is a snippet from the **System.map** file, generated after linking all the kernel objects. It includes the symbol **__irqchip_of_table** and its starting base address, which contains entries for the interrupt controller nodes defined in the device tree:
<br>
<img width="1106" alt="system_map_irq" src="https://github.com/user-attachments/assets/28b36aa0-5c35-4a78-b3a8-cfe11b4563d9">
<br>
<br>

During kernel startup initialization, the "of_irq_init" function searches for device node information. The parameter passed to this function is the "__irqchip_of_table" segment. Since the IRQCHIP_DECLARE macro has filled in the necessary information, the of_irq_init function will find the corresponding device node for "arm,gic-400" and obtain the device information. The interrupt controller cascading is also handled within the of_irq_init function(which we will analyze in another example that has multiple interrupt controllers).

Within the of_irq_init function, the callback function declared by IRQCHIP_DECLARE will eventually be called, which is gic_of_init. This function serves as the initialization entry point for the GIC driver.

**From the call flow represented above we can understand that:**

- The function pointer **__smp_cross_call** is assigned to **gic_raise_softirq** by **set_smp_process_call**(as per kernel v4.10). This setup triggers the GIC using a software-generated SGI interrupt for inter-core communication.

- The **cpuhp_setup_state_nocalls** function sets the GIC callback function for CPU hot-plug events, allowing appropriate processing when the CPU is hot-plugged.
 
- The setting of the **set_handle_irq** function is crucial. It assigns the global function pointer **handle_arch_irq** to **gic_handle_irq**. When the processor encounters an interrupt exception, it jumps to ***handle_arch_irq*** for execution, making it the entry point for interrupt processing.

The driver registers various functions and initializes structures such as **struct irq_chip** and **struct irq_domain**, which will be analyzed further below in detail. Finally, it completes the initialization settings of the GIC hardware module and handles the registrations related to power management.

### > L_2.2.1: GIC private data structure analysis

Let's delve into the GIC's private data structure and examine the fields it contains, below is the illustration:

<br>

![GIC-DS-1](https://github.com/user-attachments/assets/4839da0c-bbd9-4a1f-9fb8-142b7c38a462)

<br>

In the GIC driver, the struct gic_chip_data structure contains information about the GIC controller. The GIC driver focuses on initializing this structure and setting up function pointers. The driver’s operations are activated by interrupt signals, causing the callback functions to be invoked when an interrupt occurs.

The **struct irq_chip** structure defines the set of low-level operations for the interrupt controller, which ultimately controls the hardware.

The **struct irq_domain** structure maps Hardware IRQ numbers to Linux/Software IRQ numbers (virq, virtual interrupt numbers).

If we closely examined this [section](https://github.com/Sudharshan-07/Linux/blob/Linux-driver-model/Interrupts-Part_1.md#interrupt-controller-abstraction-from-kernel-viewpoint) in [Interrupts-part_1](https://github.com/Sudharshan-07/Linux/blob/Linux-driver-model/Interrupts-Part_1.md#-interrupts--), we observed that there are six main IRQ data structures interconnected within the kernel's IRQ subsystem. The question arises: why does the GIC's private data structure only include **struct irq_chip** and **struct irq_domain**, and not the other IRQ data structures such as **irq_desc, irq_data, and irqaction**? Why can't these additional structures be included as well?. Before delving into an in-depth analysis of the irq_chip and irq_domain use cases, let's briefly discuss this topic now.

##### What is the need for irq_chip ?
**[struct irq_chip](https://elixir.bootlin.com/linux/v4.10/source/include/linux/irq.h#L340)** provides three main advantages to the kernel drivers(of L3 and L4 level):
1. **Abstraction:** struct irq_chip indeed acts as an abstraction layer for the interrupt controllers. It provides a unified interface for the kernel to interact with different interrupt controllers.

2. **Standardized Interface:** By using struct irq_chip, the kernel can handle interrupt pins in a standardized manner. This abstraction allows the kernel to interact with different interrupt pins consistently.

3. **Simplified Handling:** With struct irq_chip, the kernel avoids the complexity of implementing separate, detailed handling functions for each interrupt controller. Instead, the kernel can rely on the standardized methods provided by struct irq_chip to manage interrupts.

The GIC driver defines the irq_chip to work with the interrupt lines of the GIC. It provides the low-level operations needed to manage interrupts.

```
/drivers/irqchip/irq-gic.c

static struct irq_chip gic_chip = {
	.irq_mask		= gic_mask_irq,
	.irq_unmask		= gic_unmask_irq,
	.irq_eoi		= gic_eoi_irq,
	.irq_set_type		= gic_set_type,
	.irq_get_irqchip_state	= gic_irq_get_irqchip_state,
	.irq_set_irqchip_state	= gic_irq_set_irqchip_state,
	.flags			= IRQCHIP_SET_TYPE_MASKED |
				  IRQCHIP_SKIP_SET_WAKE |
				  IRQCHIP_MASK_ON_SUSPEND,
};

Initialization of struct irq_chip for GIC controller:
=====================================================
static void gic_init_chip(struct gic_chip_data *gic, struct device *dev,
			  const char *name, bool use_eoimode1)
{
	/* Initialize irq_chip */
	gic->chip = gic_chip; <<<<<<<<<<<<<<<<<<<<<<<<<<<<<< Initialize irq_chip for GIC
	gic->chip.name = name;
	gic->chip.parent_device = dev;

	if (use_eoimode1) {
		gic->chip.irq_mask = gic_eoimode1_mask_irq;
		gic->chip.irq_eoi = gic_eoimode1_eoi_irq;
		gic->chip.irq_set_vcpu_affinity = gic_irq_set_vcpu_affinity;
	}

#ifdef CONFIG_SMP
	if (gic == &gic_data[0])
		gic->chip.irq_set_affinity = gic_set_affinity;
#endif
}

```

**[struct irq_domain](https://elixir.bootlin.com/linux/v4.10/source/include/linux/irqdomain.h#L125)**, provides a way to map and manage the hierarchy and translation of interrupts numbers between the hardware IRQ to Software/Linux irq numbers.

```
/drivers/irqchip/irq-gic.c

static const struct irq_domain_ops gic_irq_domain_hierarchy_ops = {
	.translate = gic_irq_domain_translate, <<<< hw-irq to sw-irq translation function
	.alloc = gic_irq_domain_alloc,
	.free = irq_domain_free_irqs_top,
};

static const struct irq_domain_ops gic_irq_domain_ops = {
	.map = gic_irq_domain_map,
	.unmap = gic_irq_domain_unmap,
};
```

The irq_chip and irq_domain structures are focused on providing an interface for managing and mapping interrupts at the hardware level. They handle the core functions of interacting with the interrupt controller and managing the mapping of interrupts between the hardware and the kernel. Therefore, ***struct irq_chip and struct irq_domain*** are handled at the L2 level of the software layer. Since they manage interactions at the controller level, they are incorporated into the interrupt controller's private data structures.

on the contrary, **struct irq_desc, irq_data, and irqaction** focus on managing and processing individual IRQs, which are distinct from handling the core functionality of the interrupt controller. These structures are concerned with higher-level interrupt management and processing and are used elsewhere in the kernel's interrupt subsystem once they are mapped and managed by the irq_chip and irq_domain during the init of the main interrupt controller.

However, it is crucial to understand that despite their distinct roles, **struct irq_chip, irq_domain, irq_desc, irq_data, and irqaction** need to be linked together to enable proper interrupt processing and handling. We will delve deeper into each structure to explore how they contribute to interrupt handling and management.

<br>

### > L_2.2.2: irq_chip analysis:

Let's go in-depth and understand and visualize irq_chip with some illustrations. Below is the structure declaration of irq_chip:

```
/**
 * struct irq_chip - hardware interrupt chip descriptor
 *
 * @parent_device:      pointer to parent device for irqchip
 * @name:               name for /proc/interrupts
 * @irq_startup:        start up the interrupt (defaults to ->enable if NULL)
 * @irq_shutdown:       shut down the interrupt (defaults to ->disable if NULL)
 * @irq_enable:         enable the interrupt (defaults to chip->unmask if NULL)
 * @irq_disable:        disable the interrupt
 * @irq_ack:            start of a new interrupt
 * @irq_mask:           mask an interrupt source
 * @irq_mask_ack:       ack and mask an interrupt source
 * @irq_unmask:         unmask an interrupt source
 * @irq_eoi:            end of interrupt
 * @irq_set_affinity:   Set the CPU affinity on SMP machines. If the force
 *                      argument is true, it tells the driver to
 *                      unconditionally apply the affinity setting. Sanity
 *                      checks against the supplied affinity mask are not
 *                      required. This is used for CPU hotplug where the
 *                      target CPU is not yet set in the cpu_online_mask.
 * @irq_retrigger:      resend an IRQ to the CPU
 * @irq_set_type:       set the flow type (IRQ_TYPE_LEVEL/etc.) of an IRQ
 * @irq_set_wake:       enable/disable power-management wake-on of an IRQ
 * @irq_bus_lock:       function to lock access to slow bus (i2c) chips
 * @irq_bus_sync_unlock:function to sync and unlock slow bus (i2c) chips
 * @irq_cpu_online:     configure an interrupt source for a secondary CPU
 * @irq_cpu_offline:    un-configure an interrupt source for a secondary CPU
 * @irq_suspend:        function called from core code on suspend once per
 *                      chip, when one or more interrupts are installed
 * @irq_resume:         function called from core code on resume once per chip,
 *                      when one ore more interrupts are installed
 * @irq_pm_shutdown:    function called from core code on shutdown once per chip
 * @irq_calc_mask:      Optional function to set irq_data.mask for special cases
 * @irq_print_chip:     optional to print special chip info in show_interrupts
 * @irq_request_resources:      optional to request resources before calling
 *                              any other callback related to this irq
 * @irq_release_resources:      optional to release resources acquired with
 *                              irq_request_resources
 * @irq_compose_msi_msg:        optional to compose message content for MSI
 * @irq_write_msi_msg:  optional to write message content for MSI
 * @irq_get_irqchip_state:      return the internal state of an interrupt
 * @irq_set_irqchip_state:      set the internal state of a interrupt
 * @irq_set_vcpu_affinity:      optional to target a vCPU in a virtual machine
 * @ipi_send_single:    send a single IPI to destination cpus
 * @ipi_send_mask:      send an IPI to destination cpus in cpumask
 * @irq_nmi_setup:      function called from core code before enabling an NMI
 * @irq_nmi_teardown:   function called from core code after disabling an NMI
 * @flags:              chip specific flags
 */

struct irq_chip {
        struct device   *parent_device;
        const char      *name;
        unsigned int    (*irq_startup)(struct irq_data *data);
        void            (*irq_shutdown)(struct irq_data *data);
        void            (*irq_enable)(struct irq_data *data);
        void            (*irq_disable)(struct irq_data *data);

        void            (*irq_ack)(struct irq_data *data);
        void            (*irq_mask)(struct irq_data *data);
        void            (*irq_mask_ack)(struct irq_data *data);
        void            (*irq_unmask)(struct irq_data *data);
        void            (*irq_eoi)(struct irq_data *data);

        int             (*irq_set_affinity)(struct irq_data *data, const struct cpumask *dest, bool
force);
        int             (*irq_retrigger)(struct irq_data *data);
        int             (*irq_set_type)(struct irq_data *data, unsigned int flow_type);
        int             (*irq_set_wake)(struct irq_data *data, unsigned int on);

        void            (*irq_bus_lock)(struct irq_data *data);
        void            (*irq_bus_sync_unlock)(struct irq_data *data);

        void            (*irq_cpu_online)(struct irq_data *data);
        void            (*irq_cpu_offline)(struct irq_data *data);

        void            (*irq_suspend)(struct irq_data *data);
        void            (*irq_resume)(struct irq_data *data);
        void            (*irq_pm_shutdown)(struct irq_data *data);

        void            (*irq_calc_mask)(struct irq_data *data);

        void            (*irq_print_chip)(struct irq_data *data, struct seq_file *p);
        int             (*irq_request_resources)(struct irq_data *data);
        void            (*irq_release_resources)(struct irq_data *data);

        void            (*irq_compose_msi_msg)(struct irq_data *data, struct msi_msg *msg);
        void            (*irq_write_msi_msg)(struct irq_data *data, struct msi_msg *msg);

        int             (*irq_get_irqchip_state)(struct irq_data *data, enum irqchip_irq_state whichh
, bool *state);
        int             (*irq_set_irqchip_state)(struct irq_data *data, enum irqchip_irq_state whichh
, bool state);

        int             (*irq_set_vcpu_affinity)(struct irq_data *data, void *vcpu_info);

        void            (*ipi_send_single)(struct irq_data *data, unsigned int cpu);
        void            (*ipi_send_mask)(struct irq_data *data, const struct cpumask *dest);

        int             (*irq_nmi_setup)(struct irq_data *data);
        void            (*irq_nmi_teardown)(struct irq_data *data);

        unsigned long   flags;
};
```


As we discussed above, The IRQ chip manages the hardware control for the interrupt controller driver. Each manufacturer's interrupt controllers handle interrupt lines differently. The IRQ chip in the Linux IRQ core layer abstracts and unifies the management of these different controllers on an interrupt pin-by-pin basis. It offers services such as mask, set, or clear, etc.. for each interrupt line/pin. These operations are handled via callback functions linked to several hook pointers in the irq_chip structure.

If the processing, masking, setting, and clearing of interrupt lines vary, the irq_chip can be configured accordingly. In Kernel, the irq_chip which is responsible for managing each interrupt line is set using a function called **irq_set_chip()**. 

##### The figure below illustrates how a single interrupt controller's driver manages all the interrupts through one "struct irq_chip"(refer to above struct irq_chip gic_chip):
<br>

![irq_chip-1](https://github.com/user-attachments/assets/deb0d240-0bca-4c5b-b393-d4f5e34fe83a)

<br>

##### The following figure shows how an interrupt controller's driver manages its controller's interrupt pins by assigning them to different irq_chips, each handling specific hardware control functions:
<br>

![irq_chip-2](https://github.com/user-attachments/assets/9ea82080-8c65-4951-84b2-f7b4545087d9)

###### For reference, the bcm2836 interrupt controller's driver handles each interrupt pin with a different irq_chip because each pin's masking must be managed uniquely, making a generic implementation unsuitable in this case:

```
/drivers/irqchip/irq-bcm2836.c
...
...
static struct irq_chip bcm2836_arm_irqchip_timer = {
        .name           = "bcm2836-timer",
        .irq_mask       = bcm2836_arm_irqchip_mask_timer_irq,
        .irq_unmask     = bcm2836_arm_irqchip_unmask_timer_irq,
};
...
static struct irq_chip bcm2836_arm_irqchip_pmu = {
	.name		= "bcm2836-pmu",
	.irq_mask	= bcm2836_arm_irqchip_mask_pmu_irq,
	.irq_unmask	= bcm2836_arm_irqchip_unmask_pmu_irq,
};
...
static struct irq_chip bcm2836_arm_irqchip_gpu = {
	.name		= "bcm2836-gpu",
	.irq_mask	= bcm2836_arm_irqchip_mask_gpu_irq,
	.irq_unmask	= bcm2836_arm_irqchip_unmask_gpu_irq,
};
...
```

<br>

##### The following figure illustrates a controller driver(A) utilizing two separate irq_chip definitions, while another controller's driver(B) employs a single irq_chip definition due to its design requirements:

<br>

![irq_chip-3](https://github.com/user-attachments/assets/7f7fdc9a-5f71-4ede-8644-717c3f20f861)

<br>


##### The following figure shows how interrupt controllers configured in two or more hierarchies are connected and processed. Interrupts received by child interrupt controller A are cascaded(or chained) to parent interrupt controller B:

<br>

![irq_chip-4](https://github.com/user-attachments/assets/6ed06b37-e863-4197-89b0-e53f54166d3b)

<br>

So far, we have examined how different "struct irq_chip" structures are defined(in a controller driver) to address the specific requirements of interrupt controllers, with examples from GIC and bcm2836 interrupt controller drivers. Furthermore, it is important to understand how an instance(or a pointer) of struct irq_chip is linked to a particular interrupt pin's irq_desc as [depicted here](https://github.com/Sudharshan-07/Linux/blob/Linux-driver-model/Interrupts-Part_1.md#interrupt-controller-abstraction-from-kernel-viewpoint) (note: each interrupt pin in the controllers has a unique irq_desc in the kernel).

##### Assign an irq_chip to an irq_desc:

The process of linking an irq_chip with an irq_desc is implemented either during the initialization of the interrupt controller (using irq_set_chip) or during the hardware-to-software IRQ mapping (using irq_create_mapping). In the latter case, the actual function definition for the mapping resides in the interrupt controller's device driver (at the L2 level).

##### Method-1:
Interrupt controller driver's directly calling irq_set_chip(irq, &irq_chip) during init.

```
/kernel/irq/chip.c

/**
 *	irq_set_chip - set the irq chip for an irq
 *	@irq:	irq number
 *	@chip:	pointer to irq chip description structure
 */
int irq_set_chip(unsigned int irq, struct irq_chip *chip)
{
	unsigned long flags;
	struct irq_desc *desc = irq_get_desc_lock(irq, &flags, 0);

	if (!desc)
		return -EINVAL;

	if (!chip)
		chip = &no_irq_chip;

	desc->irq_data.chip = chip; <<<<<<<< irq_chip is assigned to irq_data which is inside irq_desc.
	irq_put_desc_unlock(desc, flags);
	/*
	 * For !CONFIG_SPARSE_IRQ make the irq show up in
	 * allocated_irqs.
	 */
	irq_mark_irq(irq); <<<<<<<<< explained below.
	return 0;
}
EXPORT_SYMBOL(irq_set_chip);
```

##### Method-2:

irq_set_chip can be called in map function of an irq_domain of an interrupt controller.

```
/kernel/irq/irqdomain.c

irq_create_mapping(irq_domain, hwirq)
+
+++> irq_domain_associate(domain, virq, hwirq)
       +
       +++> domain->ops->map(domain, virq, hwirq);
                 +
                 ++++> calls interrupt controller's definition of map function in struct irq_domain_ops:

			static const struct irq_domain_ops my_irq_domain_ops = {
   				 .map = my_irq_domain_map,
   				 .translate = my_irq_domain_translate,
			};

			static int my_irq_domain_map(struct irq_domain *d, unsigned int irq, irq_hw_number_t hwirq) {
				...
    				irq_set_chip(virq, &irq_chip);
				...
			}
```

So, the process of linking an irq_chip with an irq_desc is implemented either during the initialization of the interrupt controller (using irq_set_chip) or during the hardware-to-software IRQ mapping (while calling irq_create_mapping). In both cases, the actual function definition resides in the interrupt controller's device driver (at the L2 level).

###### Here's a simplified illustration of the irq_set_chip() function linking an irq_chip with an irq_desc for an interrupt:
<br>

![irq_chip-6](https://github.com/user-attachments/assets/b7cb7092-7100-41f0-916b-9955517fbf9f)

<br>

```
Note:
In irq_set_chip(), if sparse IRQ(CONFIG_SPARSE_IRQ) is not used, we mark the corresponding bit in the allocated_irqs bitmap to indicate the irq is in use.
```

![irq_mark_irq-1a](https://github.com/user-attachments/assets/ddad499f-3141-44ac-b625-c1cb9f4d7c57)

<br>

### > L_2.2.3: irq_domain analysis

Why are IRQ Domains (interrupt domains) necessary in the Linux kernel, and what are their significances at the L2, L3, and L4 levels?.

In the past, the Linux kernel used a single large number space to assign unique IRQ numbers directly corresponding to interrupt pins, suitable for systems with one interrupt controller. 

![interrupt_domain_example_1](https://github.com/user-attachments/assets/8152a9f2-898c-4c34-8e11-eaf0945f963b)

<br>

However, this approach becomes challenging in SoCs with multiple interrupt controllers, as the kernel must ensure that each one gets assigned non-overlapping allocations of Linux
IRQ numbers. With the increasing use of multiple interrupt controllers—such as GPIO controllers—the management of IRQ numbers has become more complex. Each controller requires a distinct(unique and non-overlapping) range of IRQ numbers in the kernel. 

In older kernels, software IRQ numbers directly matched hardware IRQ lines if an SoC had only one interrupt controller(as depicted above). For instance, for an interrupt pin 5(of a controller), the software IRQ number would also be 5. In modern kernels, IRQ numbers are abstract identifiers, so IRQ number 5 could represent any interrupt from any controller, not directly tied to a specific interrupt pin.  For this reason, we need a mechanism to separate controller-local interrupt numbers, called hardware IRQs, from Linux IRQs ( or virtual IRQs/Software IRQs).

IRQ domains have the following characteristics:
- Each hwirq is unique within its domain.
- They utilize reverse mapping for implementation. *(i.e. reverse mapping (hwirq -> Linux irq) instead of forward mapping (Linux irq -> hwirq))*.

Assume we have two interrupt controllers in the SoC(interrupt controllers A & B), then each will have its own irq_domain as depicted below. The following figure illustrates the process of finding the irq_desc(interrupt descriptor) using hwirq number when an interrupt occurs and invoking the associated handler function:


![irq_domain-mapping](https://github.com/user-attachments/assets/6055cbac-06d1-4d9c-92ca-ec3747006bd4)


###### Note: This is just a visualization to help understand the underlying concept better. In reality, the creation of interrupt domains follows the hierarchy of interrupt controllers connected in the SoC, which we will analyze further.

<br>





#### How does the kernel do the mapping process?
The IRQ Domain framework has two primary responsibilities:
1. Mapping HW IRQs to SW IRQs(Virqs/Linux IRQs). (using .map)
2. Translate hardware IRQ numbers read from the Device Trees(or ACPI) into software IRQ numbers used by the Linux kernel.(using .xlate)

The functionalities for these two operations should be defined in the **[struct irq_domain_ops](https://elixir.bootlin.com/linux/v4.10/source/include/linux/irqdomain.h#L82)** of the interrupt domain of the interrupt controller driver.





















  
