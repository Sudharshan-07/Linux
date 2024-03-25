## Abstract:
In  [Part 1 of Overview of LDD model](https://github.com/Sudharshan-07/Linux/blob/Linux-driver-model/Overview%20of%20Linux%20device%20driver%20model.md) we understood the need for sysfs and how sysfs is orchestrated in Linux kernel using kobject/kset/ktype. If you have observed it keenly, the kobjects are not used alone in any drivers in the Linux kernel, instead, it is used as embedded objects in other core data structures(like struct class, device, device_driver) in the kernel. so in this part, we can see how it is represented as an object or abstract class(like C++), with the help of layered representation.

## Data structure and implementation of kobject:
Below are the structural links in the kernel for kset/kobject/ktype. these objects are not used alone in any driver, instead, it is embedded or derived by other structures in the kernel. The kobject should be viewed as an abstract class representation in the kernel.
![Kobject_Struct](https://github.com/Sudharshan-07/Linux/assets/52316856/b8371576-a97f-4741-a979-4180835e2dcd)

## LDD - Layered Representation:
This section briefs about how the Linux device driver model is implemented and the layered approach shows how inheritance is achieved in the kernel structures. so here you can observe that, kobject is a base class inherited by layer 2 and layer 3 inherits layer 2. This is the approach of the Linux device model. As a driver author, we tend to use only Layer 3 and should not bother or modify the structure or value of Layer 1 or 2 directly in the author's device driver. It's the underlying core driver's job to handle those core structures.

![LDD layer copy](https://github.com/Sudharshan-07/Linux/assets/52316856/f990eba6-c9c1-44ff-aa9f-55ff708e591a)

## BUS-DEVICE-DEVICE_DRIVER KERNEL STRUCTURE RELATIONSHIP DIAGRAM:
![bus-device-driver-relationship (1)](https://github.com/Sudharshan-07/Linux/assets/52316856/f3c57720-7353-4398-bd56-e40e86591866)


## The connection relationship between the three data structures in actual use:
![BusList](https://github.com/Sudharshan-07/Linux/assets/52316856/07e28700-ce8f-4cb1-9f50-99e276c16664)


## EXAMPLE OF STRUCTURE RELATIONSHIP OF PLATFORM BUS/DEVICE/DEVICE_DRIVER:
![20181218095210607](https://github.com/Sudharshan-07/Linux/assets/52316856/2184217f-4602-4af4-907f-52d355023dbb)


#### References:
1. https://www.slideserve.com/susane/chapter-14-the-linux-device-model <br>
2. https://carlyleliu.github.io/2020/Linux%E9%A9%B1%E5%8A%A8%E4%B9%8B%E8%AE%BE%E5%A4%87%E9%A9%B1%E5%8A%A8%E6%A8%A1%E5%9E%8B/ <br>
3. https://gist.github.com/carloscn/3f0179ecfa599969556e86eb80555266#user-content-fnref-2-443c0747c05ab3e7bb6acd3ca6f58809 <br>
