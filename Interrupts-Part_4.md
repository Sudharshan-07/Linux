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

"Some device" changes the level on the P4 pin, causing the MAX7325 to generate an interrupt. This interrupt from the MAX7325 is connected to the GPIO4 IP-core (inside the SoC), which uses line #29 of the GPIO4 module to notify the CPU about the interrupt. Therefore, the MAX7325 is cascaded to the GPIO4 controller. Additionally, GPIO4 can also act as an interrupt controller and is cascaded to the GIC interrupt controller.

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
- **#interrupt-cells:** defines the format of interrupts property; in our case, it's 2: 1 cell for the line number and 1 cell for the interrupt type
interrupt-parent and interrupts properties describe interrupt line connection.

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
        some_device_isr, IRQF_TRIGGER_RISING | IRQF_ONESHOT,
        dev_name(core->dev), core);
```

And **"some_device_isr()"** will be called each time the level on the P4 pin of the MAX7325 transitions from low to high (rising edge). How does this work? From left to right, if you refer to the picture above:

- "Some device" changes the level on P4 of the MAX7325.
- The MAX7325 changes the level on its INT pin.
- The GPIO4 module is configured to detect such a change and generates an interrupt to the GIC.
- The GIC notifies the CPU.
All these actions occur at the hardware level. 

#### Software Side Propagation:
Now, let's see what happens at the software level. It proceeds in reverse order(from right to left in the picture):

CPU now is in interrupt context in GIC interrupt handler. From gic_handle_irq() it calls handle_domain_irq(), which in turn calls generic_handle_irq(). See Documentation/gpio/driver.txt for details. Now we are in SoC's GPIO controller IRQ handler.
SoC's GPIO driver also calls generic_handle_irq() to run handler, which is set for each particular pin. See for example how it's done in omap_gpio_irq_handler(). Now we are in MAX7325 IRQ handler.
MAX7325 IRQ handler (here) calls handle_nested_irq(), so that all IRQ handlers of devices connected to MAX7325 ("Some device" IRQ handler, in our case) will be called in max732x_irq_handler() thread
finally, IRQ handler of "Some device" driver is called
IRQ domain API
GIC driver, GPIO driver and MAX7325 driver -- they all are using IRQ domain API to represent those drivers as interrupt controllers. Let's take a look how it's done in MAX732x driver. It was added in this commit. It's easy to figure out how it works just by reading IRQ domain documentation and looking to this commit. The most interesting part of that commit is this line (in max732x_irq_handler()):

handle_nested_irq(irq_find_mapping(chip->gpio_chip.irqdomain, level));
irq_find_mapping() will find linux IRQ number by hardware IRQ number (using IRQ domain mapping function). Then handle_nested_irq() function will be called, which will run IRQ handler of "Some device" driver.

GPIOLIB_IRQCHIP
Since many GPIO drivers are using IRQ domain in the same way, it was decided to extract that code to GPIOLIB framework, more specifically to GPIOLIB_IRQCHIP. From Documentation/gpio/driver.txt:

To help out in handling the set-up and management of GPIO irqchips and the associated irqdomain and resource allocation callbacks, the gpiolib has some helpers that can be enabled by selecting the GPIOLIB_IRQCHIP Kconfig symbol:

gpiochip_irqchip_add(): adds an irqchip to a gpiochip. It will pass the struct gpio_chip* for the chip to all IRQ callbacks, so the callbacks need to embed the gpio_chip in its state container and obtain a pointer to the container using container_of(). (See Documentation/driver-model/design-patterns.txt)
gpiochip_set_chained_irqchip(): sets up a chained irq handler for a gpio_chip from a parent IRQ and passes the struct gpio_chip* as handler data. (Notice handler data, since the irqchip data is likely used by the parent irqchip!) This is for the chained type of chip. This is also used to set up a nested irqchip if NULL is passed as handler.
This commit converts IRQ domain API to GPIOLIB_IRQCHIP API in MAX732x driver.
