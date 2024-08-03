<div align="center">
  <h1> INTERRUPTS-PART 4  </h1>
</div>

## Objective:
In this section, we will explore the call flow of how an interrupt is serviced in a hierarchical interrupt controller configuration. we can cover this using max732x.c driver as a reference ([driver code](https://elixir.bootlin.com/linux/v6.3.1/source/drivers/gpio/gpio-max732x.c)).This GPIO driver also functions as an interrupt controller, making it an excellent example for illustrating this concept.

### Physical level
To fully grasp the subsequent explanation, let's first examine the mechanics of the MAX732x. Below is a simplified application circuit from the [datasheet](https://www.analog.com/en/products/max7325.html#part-details):

### MAX7325 Typical Application Circuit:
![MAX7325](https://github.com/user-attachments/assets/20677cf3-2b89-4702-8473-d140d2365e2f)

<br>

When the voltage level changes on P0-P7 pins, MAX7325 will generate an interrupt on the INT pin. The driver (running on SoC) can read the status of P0-P7 pins via I2C (SCL/SDA pins) and generate separate interrupts for each P0-P7 pins. This is why this driver acts as an interrupt controller.

Consider the next configuration:

### Interrupt cascading:

![interrupt-cascading](https://github.com/user-attachments/assets/d61465ce-f88d-452e-bc97-d7ce45758e68)

<br>

"Some device" changes the level on the P4 pin, causing the MAX7325 to generate an interrupt. This interrupt from the MAX7325 is connected to the GPIO4 IP-core (inside the SoC), which uses line "gpio4_irq" of the GPIO4 module to notify the CPU about the interrupt. Therefore, the MAX7325 is cascaded to the GPIO4 controller. Additionally, GPIO4 can also act as an interrupt controller and is cascaded to the GIC interrupt controller.

### Device tree:
Let's declare the above configuration in the device tree. We can use bindings from [Documentation/devicetree/bindings/gpio/gpio-max732x.txt](https://kernel.googlesource.com/pub/scm/linux/kernel/git/jikos/livepatching/+/9ec0de0ee0c9f0ffe4f72da9158194121cc22807/Documentation/devicetree/bindings/gpio/gpio-max732x.txt) as reference:

```
expander: max7325@6d {
    compatible = "maxim,max7325";
    reg = <0x6d>;

    gpio-controller;
    #gpio-cells = <2>;

    interrupt-controller;
    #interrupt-cells = <2>;

    interrupt-parent = <&gpio4>;
    interrupts = <29 IRQ_TYPE_EDGE_FALLING>;
};
```
The meaning of properties is as follows:
- **interrupt-controller** property defines that the device generates interrupts; it will be needed further to use this node as interrupt-parent in the "Some device" node.
- **#interrupt-cells:** defines the format of interrupts property; in our case, the  2 cell represents: 1 cell for the interrupt number and 1 cell for the interrupt type.
- **interrupt-parent and interrupts** properties in the above snippet describe interrupt connection.

Let's assume we have a driver for the MAX7325 and a driver for "Some device," both running on the CPU. In the "Some device" driver, we want to request an interrupt for an event when "Some device" changes the level on the P4 pin of the MAX7325.

Let's first declare this in the device tree:

```
some_device: some_device@1c {
    reg = <0x1c>;
    interrupt-parent = <&expander>;
    interrupts = <4 IRQ_TYPE_EDGE_RISING>;
};
```

### Interrupt propagation:

#### Hardware Side Propagation:
Now we can register an interrupt handler for the pin(in the "Some device" driver) using below kernel API:

```
devm_request_threaded_irq(core->dev, core->gpio_irq, NULL,
        some_device_threaded_isr, IRQF_TRIGGER_RISING | IRQF_ONESHOT,
        dev_name(core->dev), core);
```

And **"some_device_threaded_isr()"** will be called each time the level on the P4 pin of the MAX7325 transitions from low to high (rising edge). How does this work? From left to right, if you refer to the picture above:

- "Some device" changes the level on P4 of the MAX7325.
- The MAX7325 changes the level on its INT pin.
- The GPIO4 module is configured to detect such a change and generates an interrupt to the GIC.
- The GIC notifies the CPU.
All these actions occur at the hardware level. 

#### Software Side Propagation:
Now, let's break down what happens at the software level when an interrupt occurs. This process happens in reverse order compared to the hardware actions (from right to left in the picture):

**GIC Interrupt Handler:**
- When the CPU receives an interrupt notification, it enters the interrupt context in the GIC interrupt handler.
- The function [gic_handle_irq()](https://github.com/torvalds/linux/blob/v5.13/drivers/irqchip/irq-gic.c#L334) is called, which then calls handle_domain_irq().
- [handle_domain_irq()](https://github.com/torvalds/linux/blob/v5.13/include/linux/irqdesc.h#L173) calls [generic_handle_irq()](https://github.com/torvalds/linux/blob/v5.13/kernel/irq/irqdesc.c#L640), which routes us to the SoC's GPIO controller interrupt handler. Refer to [Documentation/gpio/driver.txt]  (https://www.kernel.org/doc/Documentation/gpio/driver.txt) for more details.
  
**SoC's GPIO Controller IRQ Handler:**
- In SoC's GPIO driver, [generic_handle_irq()](https://github.com/torvalds/linux/blob/v5.13/kernel/irq/irqdesc.c#L640) is again used to run the handler set for each specific pin.
- For instance, in the [omap_gpio_irq_handler()](https://github.com/torvalds/linux/blob/master/drivers/gpio/gpio-omap.c#L559C20-L559C41), this mechanism is demonstrated. At this stage, we are now in the MAX7325 IRQ handler.
  
**MAX7325 IRQ Handler:**
- The MAX7325 IRQ handler calls [handle_nested_irq()](https://github.com/torvalds/linux/blob/v5.13/drivers/gpio/gpio-max732x.c#L486).
- This function ensures that all IRQ handlers of devices connected to the MAX7325 (such as the IRQ handler for "Some device") are invoked within the 
  [max732x_irq_handler()](https://github.com/torvalds/linux/blob/v5.13/drivers/gpio/gpio-max732x.c#L473) thread.

**"Some Device" IRQ Handler:**
- Finally, the IRQ handler for the "Some device" driver is called.


#### IRQ domain API:

GIC driver, GPIO driver, and MAX7325 driver all use IRQ domain API to represent those drivers as interrupt controllers. Let's look at how it's done in the MAX732x driver. It was added in this commit. It's easy to figure out how it works by reading IRQ domain documentation and looking at this commit. The most interesting part of that commit is this line (in max732x_irq_handler()):

```
handle_nested_irq(irq_find_mapping(chip->gpio_chip.irqdomain, level));
```

irq_find_mapping() will find the Linux IRQ number by hardware IRQ number (using the IRQ domain mapping function). Then **handle_nested_irq()** function will be called, which will run the IRQ handler of the "Some device" driver.


[Refer link 1](https://www.kernel.org/doc/Documentation/gpio/driver.txt), and [link 2](https://stackoverflow.com/questions/34377846/what-is-chained-irq-in-linux-when-are-they-need-to-used) to know and understand when to use chained irq handlers, generic chained irq handlers, nested irq handlers. This is very essential for a driver developer to decide how to register an interrupt handler based on the SoC design and requirements.

Below is a brief about the concept:
There are two approaches on calling interrupt handlers for child interrupt controllers in the IRQ handler of the parent interrupt controller.

**1. Chained interrupts:**
- "chained" means that those interrupts are just chain of function calls (for example, SoC's GPIO module interrupt handler is being called from GIC interrupt 
  handler, just as a function call)
- generic_handle_irq() is used for interrupts chaining
- child IRQ handlers are being called inside of parent HW IRQ handler
- you can't call functions that may sleep in chained (child) interrupt handlers, because they are still in atomic context (HW interrupt)
- this approach is commonly used in drivers for GPIO controllers inside SoC itself.

**2. Nested interrupts:**
- "nested" means that those interrupts can be interrupted by another interrupt; but they are not really HW IRQs, but rather threaded IRQs
  handle_nested_irq() is used for creating nested interrupts
- Child IRQ handlers are being called inside of new thread created by handle_nested_irq() function; we need them to be run in process context, so that we can 
  call sleeping bus functions (like I2C functions that may sleep)
- You can call functions that may sleep inside of nested (child) interrupt handlers.
- This approach is commonly used in drivers for external chips, like GPIO expanders, because they are usually connected to SoC via I2C bus, and I2C functions may 
  sleep

#### Regarding the drivers mentioned earlier:

- The **irq-gic** driver uses the CHAINED GPIO irqchips approach to handle systems with multiple GICs.
- The **gpio-omap** driver employs the GENERIC CHAINED GPIO irqchips approach. Refer to this [commit](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=450fa54cfd66e3dda6eda26256637ee8928af12a) for details. It was converted from the regular CHAINED GPIO irqchips to utilize a threaded IRQ handler in real-time kernels and a hard IRQ handler in non-RT kernels.
- The **gpio-max732x** driver utilizes the NESTED THREADED GPIO irqchips approach. [refer](https://github.com/torvalds/linux/blob/v5.13/drivers/gpio/gpio-max732x.c#L486)

#### what does chained_irq_enter and chained_irq_exit do?

Those functions implement hardware interrupt flow control, i.e. notifying the interrupt controller chip when to mask and unmask the current interrupt.

**For FastEOI interrupt controllers:**
- chained_irq_enter() do nothing.
- chained_irq_exit() calls **irq_eoi()** callback to tell the interrupt controller that interrupt processing is finished.

**For interrupt controllers with mask/unmask/ack capabilities:**
- chained_irq_enter() masks current interrupt, and acknowledges it if ack callback is set as well.
- chained_irq_exit() unmasks interrupt.

<br>

**Reference:** <br>
https://blog.csdn.net/qq_18804879/article/details/132966631 <br>
