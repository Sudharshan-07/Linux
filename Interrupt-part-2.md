<div align="center">
  <h1> INTERRUPTS-PART 2  </h1>
</div>

## Objective:
  In [Interrupts part 1](https://github.com/Sudharshan-07/Linux/blob/Linux-driver-model/Interrupts-Part_1.md), we covered the basic hardware overview and list of structures associated in the Linux kernel. In this part, we will delve deeper into both the hardware aspects and the software implementation involved in processing an interrupt.
  
  This part will explain the interrupt flow from a peripheral to the CPU and describe the corresponding software actions and responses from the CPU back to the peripheral. It is recommended to review Part 1 for a better understanding before proceeding with Part 2.

## Approach to understand the irq subsytem:
 As we explore the fundamentals of the interrupt subsystem, we'll divide the concepts into two categories:
 1. Hardware view.
 2. Software view.

## Hardware view:

Each architecture has a different interrupt controller configuration, so let's briefly understand that they have different processing methods as shown in the figure below:

**x86 with PIC**
-  In the early x86 systems that used only one CPU, one or more 8259 Programmable Interrupt Controllers (PICs) were used.
  
**x86 with APIC**
-  When configuring SMP in x86, the APIC (Advanced Programmable Interrupt Controller) is divided into two chips, one is used for the core processor chip with the CPU, and the APIC is configured in the south chipset that is in charge of I/O.

**ARM SoC**
-  ARM has a very diverse configuration. Each company that manufactures SoC uses its own interrupt controller logic as an IP block of the SoC, but there is a tendency to gradually use the GIC (Generic Interrupt Controller) designed by ARM.

<br>

![interrupt-1a](https://github.com/user-attachments/assets/f41971c3-b870-433e-8537-7d88d8104ed7)

Reviewing the above depictions, it is natural to be unclear(from software point of view) about how the CPU determines the source of an interrupt from different peripherals. This will be addressed in the upcoming sections, where we will explore the data structures used to identify the interrupt source, as well as the roles of an architecture-dependent and hardware-independent interrupt framework implementation in the Linux kernel.











