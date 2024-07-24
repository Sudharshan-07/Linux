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

This article will first cover the principles and drivers related to hardware, and then focus directly on the core topics.

### What problem are we examining here?
  Let's clearly outline the objectives of our analysis:
- When a peripheral generates an interrupt signal, how is the interrupt handler eventually invoked when multiple interrupt controllers are involved? As depicted in the block diagram below, although there are multiple interrupt sources, the CPU receives only one INTR signal. How does the CPU/Kernel determine the source of the interrupt in this scenario?

![flow3](https://github.com/user-attachments/assets/38f830d5-a18e-47da-94ea-d37114b069d6)

<br>

#### Now, What is the underlying process in the software layers above for identifying the source of an interrupt? Let’s examine the importance of each layer.

### Hardware layer (L1):

In addition to the [Hardware view](https://github.com/Sudharshan-07/Linux/edit/Linux-driver-model/Interrupt-part-2.md#hardware-view), lets take an example of ARM based interrupt controller's design:

<img width="596" alt="ARM-Intrctrlr-design" src="https://github.com/user-attachments/assets/2c129e3f-2242-45e8-9f9d-d374b6640d0b">












