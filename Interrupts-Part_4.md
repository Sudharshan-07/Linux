<div align="center">
  <h1> INTERRUPTS-PART 4  </h1>
</div>

## Objective:
In this part, we will cover the call flow of how an interrupt is serviced in a hierarchical interrupt controller configuration.

### Physical level
To completely understand further explanation, let's first look into MAX732x mechanics. Application circuit from datasheet (simplified for our example):

### MAX7325 Typical Application Circuit:


When the voltage level changes on P0-P7 pins, MAX7325 will generate an interrupt on the INT pin. The driver (running on SoC) can read the status of P0-P7 pins via I2C (SCL/SDA pins) and generate separate interrupts for each P0-P7 pins. This is why this driver acts as an interrupt controller.

### Consider the next configuration:

### Interrupt cascading:

"Some device" changes the level on the P4 pin, tempting MAX7325 to generate an interrupt. Interrupt from MAX7325 is connected to GPIO4 IP-core (inside of SoC), and it uses line #29 of that GPIO4 module to notify the CPU about the interrupt. So we can say that MAX7325 is cascaded to the GPIO4 controller. GPIO4 also acts as an interrupt controller and it is cascaded to the GIC interrupt controller.

### Device tree:
Let's declare the above configuration in the device tree. We can use bindings from Documentation/devicetree/bindings/gpio/gpio-max732x.txt as reference:

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

interrupt-controller property defines that device generates interrupts; it will be needed further to use this node as interrupt-parent in "Some device" node.
#interrupt-cells defines format of interrupts property; in our case it's 2: 1 cell for line number and 1 cell for interrupt type
interrupt-parent and interrupts properties are describing interrupt line connection
Let's say we have driver for MAX7325 and driver for "Some device". Both are running in CPU, of course. In "Some device" driver we want to request interrupt for event when "Some device" changes level on P4 pin of MAX7325. Let's first declare this in device tree:

```
some_device: some_device@1c {
    reg = <0x1c>;
    interrupt-parent = <&expander>;
    interrupts = <4 IRQ_TYPE_EDGE_RISING>;
};
```

### Interrupt propagation:
Now we can do something like this (in "Some device" driver):

```
devm_request_threaded_irq(core->dev, core->gpio_irq, NULL,
        some_device_isr, IRQF_TRIGGER_RISING | IRQF_ONESHOT,
        dev_name(core->dev), core);
```

And some_device_isr() will be called each time when level on P4 pin of MAX7325 goes from low to high (rising edge). How it works? From left to the right, if you look to the picture above:

"Some device" changes level on P4 of MAX7325
MAX7325 changes level on its INT pin
GPIO4 module is configured to catch such a change, so it's generates interrupt to GIC
GIC notifies CPU
All those actions are happening on hardware level. Let's see what's happening on software level. It actually goes backwards (from right to the left on the picture):

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
