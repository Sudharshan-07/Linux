<div align="center">
  <h1> INTERRUPTS !!! </h1>
</div>

## Objective:

I presume the reader has a fundamental understanding of how interrupts function in an embedded system. Based on that we are going to analyze:
1. How the Interrupt subsystem works in the Linux kernel for x86 and ARM-based interrupt controllers.
2. Hardware and equivalent Linux kernel software representations of IRQ controllers and their peripheral devices(interrupt sources).

## INTERRUPT CONTROLLERS FROM HARDWARE VIEWPOINT:
Generally, each architecture has a different interrupt controller configuration, so let's briefly understand that it has different processing methods as shown in the picture below:
1. **x86 with PIC:**
- The early x86 system, which used only one CPU, was configured using one or more 8259 PIC (Programmable Interrupt Controller).
2. **x86 with APIC:**
- When configuring SMP in x86, the APIC (Advanced Programmable Interrupt Controller) was divided into two chips, one used for the core processor chip where the CPU is located, and the APIC was configured for the system chipset in charge of I/O.
3. **ARM SoC:**
- ARM has a very diverse configuration. Each company that produces SoC uses its own interrupt controller logic by configuring it as an IP block of the SoC. Still, there is a tendency to gradually use GIC (Generic Interrupt Controller) designed by ARM.

![interrupt-1a](https://github.com/Sudharshan-07/Linux/assets/52316856/8f0f08df-09e2-492b-b3f8-9146801add37)

### Interrupt Controller Configurations:
Typically, Interrupt controller can be configured in two ways:

1. Single Interrupt controller.
2. Cascaded (or) Hierarchical Interrupt controller.

#### The below image gives an overview of connections and it's equivalent device tree representation:

![interrupt-configurations](https://github.com/Sudharshan-07/Linux/assets/52316856/c4a26def-3f1d-4204-bdf5-f67a6f20d045)

### Interrupt transfer types:

1. Signal-based interrupt.
    - Transmitted as a signal(Edge/Level trigger): Interrupt line <-> Interrupt controllers <-> CPU.
3. Message-based interrupt.(for ex: MSI/MSI-X in PCIe).
   - Message based interrupts are transmitted to interrupt controllers as messages(memory write).

#### For ex: In ARM based SoC:
![image](https://github.com/Sudharshan-07/Linux/assets/52316856/15829d36-4a21-4b8e-b613-9ea8baf26a7c)

![image](https://github.com/Sudharshan-07/Linux/assets/52316856/ad307a32-d834-4a37-ba9a-55e5f8217aa0)

#### Interrupts from Legacy PCI and PCIe EndPoints:
![image](https://github.com/Sudharshan-07/Linux/assets/52316856/f71c5a21-014f-4c89-9881-29bdf047aa61)

### Types of Signal-based Interrupt Triggers:
Generally, Interrupt triggers are of 2 types:

1. Edge Triggered Interrupts(Positive/Raising Edge or Negative/Falling Edge).
2. Level Triggered Interrupts(High or Low Level).
   - In level-sensitive trigger mode, the pending state continues while an interrupt occurs. In a level-sensitive interrupt system, the CPU does not sample the interrupt line during the execution of the ISR because it assumes that the interrupt condition is still true due to the level-sensitive nature of the interrupt(refer to state machine diagram). The CPU disables interrupts to maintain a controlled interrupt-handling environment and prevents nested interrupts from occurring. The software/IRQ handler should clear the interrupt flag in the kernel.

- ### Kernel Interrupt State Machine for each trigger type:

  1. **Inactive**
      - State in which no interrupt occurs.
  2. **Pending**
      - An interrupt occurred in hardware or software but was not delivered to the target process. In other words, **Pending** is the state when the CPU has detected the interrupt but has not yet cleared it.
      
  3. **Active**
      - An interrupt occurred and was delivered to the process, and the process Acked and processed the interrupt, but it was not completed. In other words, **Active** is the state when the interrupt is continuously active and the CPU is executing the ISR.
  4. **Active & Pending**
      - An interrupt occurred and was delivered to the process, and the same interrupt occurred and is pending. In other words, **Active and Pending** is the state where another interrupt pulse is detected while the CPU is executing the ISR.

![Interrupt-trigger-types](https://github.com/Sudharshan-07/Linux/assets/52316856/90cfb530-783b-4c62-b560-d1007d4fa45c)

<br>

## INTERRUPT CONTROLLER ABSTRACTION FROM KERNEL VIEWPOINT:

Typically, six data structures are involved in the kernel IRQ subsystem:
#### 1. struct irq_domain
#### 2. struct irq_desc
#### 3. struct irq_domain_ops
#### 4. struct irq_chip
#### 5. struct irqactions
#### 6. struct irq_data 

#### Below are the data structures that represent an interrupt controller/interrupt line/interrupt action handler relationship:
![f1f9135ff0cb4be0baeb379329bdcac4](https://github.com/Sudharshan-07/Linux/assets/52316856/ed4ed341-53f9-439d-a1a7-644d819469d9)



