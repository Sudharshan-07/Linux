<div align="center">
  <h1> INTERRUPTS-PART 3  </h1>
</div>

## Objective:
In [Interrupts-Part_2](https://github.com/Sudharshan-07/Linux/blob/Linux-driver-model/Interrupts-Part_2.md), we explored Layer 1 and Layer 2 of the Linux kernel's interrupt subsystem software layers, discussing the underlying hardware GIC controller driver and architecture-specific interrupt code in the kernel. In this part, we will focus on the generic, hardware-independent interrupt-handling framework of the Linux kernel, covering the L3 and L4 layers.

##### This article will address two questions:
1. How do device drivers in the L4 level register interrupt?
2. When a peripheral triggers an interrupt signal, how is the interrupt handler ultimately called? Though this was partially covered in part 2, we will now delve into the underlying call flows with visual representations.





