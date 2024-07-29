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

Fig:1

![IRQ-DS-flow](https://github.com/user-attachments/assets/ea6ec013-11f6-434c-b1d8-b3928dc85b49)

<br>

The Linux kernel's interrupt processing revolves around the interrupt descriptor structure **struct irq_desc**. The kernel offers two ways to organize these interrupt descriptors:

Fig:2

![sparse-irq-1](https://github.com/user-attachments/assets/d646814f-dda9-481e-8074-7ab438b790e5)

<br>

#### Sparse IRQ:
- If the kernel config option **CONFIG_SPARSE_IRQ** is enabled, dynamically allocate struct irq_desc structures for required IRQ numbers and manage them using Radix Tree.
- arm64 based systems enables the CONFIG_SPARSE_IRQ kernel config by default.
- Functions used to create/destroy an irq descriptor:
  - irq_alloc_desc*()
  - irq_free_desc*()

###### Allocation flow during early kernel init:

![early_irq_init-1](https://github.com/user-attachments/assets/589b7728-1782-4b48-b1d0-b08ab91b2744)

<br>

#### Flat IRQ:
- If you do not use kernel options, an array of irq_dest structures equal to the max IRQ number is statically allocated and used at compile time.
Regardless of the above ways, the corresponding interrupt descriptor in Linux can ultimately be found using the Linux IRQ number.

###### Allocation flow during early kernel init:

![early_irq_init-2](https://github.com/user-attachments/assets/e24c7fc7-7cbd-427d-a667-a839c321ceca)

<br>



Therefore, from Fig 1, The grey section on the left represents the L2 software layer, where the interrupt controller driver initializes struct irq_chip and irq_domain. The grey area at the top of the figure depicts the creation of the interrupt descriptor, accomplished during the process of acquiring device interrupt information. The remaining parts of the figure are configured during L4 software layer device driver initialization, including setting up "struct irqaction" to link to the specific interrupt handler function.














