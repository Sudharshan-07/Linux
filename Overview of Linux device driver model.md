# Overview of Linux device driver model(Part 1):

### Abstract:
In this discussion, we will try to understand:
1. what is the need for sysfs filesystem?
2. Why Linux device driver model is brought into the Linux kernel?
3. Analysis and source code examples based on Linux Kernel v5.2.x

   
## 1. Need for sysfs filesystem:
1.1 sysfs is a feature of the Linux kernel 2.6 that allows kernel code to export information to user processes via an in-memory filesystem. The organization of the filesystem directory is
strict and based the internal organization of kernel data structures.

1.2 sysfs is a mechanism of representing kernel objects(kobjects), their attributes, and their relationships with each other. It provides two components: a kernel programming interface for
   exporting these items via sysfs and a user interface to view and manipulate these items that map back to kernel objects that they represent.
   
1.3 The table below shows the mapping between their internal(kernel) constructs and external(userspace) sysfs mappings:
<p align="center">
<img width="400" alt="Screenshot 2024-03-19 at 6 03 24 AM" src="https://github.com/Sudharshan-07/Linux/assets/52316856/74ef2d5b-bd08-4490-b29a-0900676fbdd5">
</p>

### History of sysfs:
sysfs is an in-memory filesystem that was originally based on ramfs. sysfs was originally called *"ddfs"(device driver filesystem)* and was written to debug the new driver model as it was 
being written. That debug code had used procfs to export a device hierarchy, but under strict urging from Linus Torvalds, it was converted to use a new filesystem based on ramfs. 

By the time the new driver model was merged into the kernel around 2.5.1, it had changed name to *"driverfs"* to be little more descriptive. During the next year of 2.5 development, the 
infrastructural capabilities of the driver model and driverfs began to prove useful to other subsystems. kobjects were developed to provide central object management mechanism and driverfs 
was converted to sysfs to represent its subsystem agnosticism.

### Mounting sysfs:
sysfs can be mounted from userspace just like any other memory-based filesystem. The command for doing that is:

```
mount -t sysfs sysfs /sys
```
     
Typically,  **/sys** is mounted from *"initrd"* (ex: "/init") script file.

##### Typical view of sysfs hierarchy:
```
# ls /sys -l
total 0
drwxr-xr-x   2 root root 0 Mar 15 04:11 block
drwxr-xr-x  33 root root 0 Mar 15 04:11 bus
drwxr-xr-x  74 root root 0 Mar 15 04:11 class
drwxr-xr-x   4 root root 0 Mar 15 04:11 dev
drwxr-xr-x  50 root root 0 Mar 15 04:11 devices
drwxr-xr-x   7 root root 0 Mar 15 04:11 firmware
drwxr-xr-x  11 root root 0 Mar 15 04:11 fs
drwxr-xr-x  15 root root 0 Mar 15 04:11 kernel
drwxr-xr-x 224 root root 0 Mar 15 04:11 module
drwxr-xr-x   3 root root 0 Mar 18 17:45 power
```

## 2. Why Linux driver model/Unified device model?
  
One of the stated goals for the 2.5 development cycle was the creation of a unified device model for the kernel. **Previous kernels had no single data structure to which they could turn to obtain information about how the system is put together**. Despite this lack of information, things worked well for some time. The demands of newer systems, with their more complicated topologies and need to support features such as **power management,** made it clear, however, that a general abstraction describing the structure of the system was needed.

The Linux kernel 2.6 device model provides that abstraction. It is now used within the kernel to support a wide variety of tasks, including:

**2.1 Power management and system shutdown:** <br>
When a device operation is in progress and the system receives a shutdown command, the kernel will use the driver model(kobjects/ksets) to traverse along the device lists and call the shutdown/release callbacks appropriately which helps in gracefully shutting down the whole system, where the shutdown/release callbacks(of the respective device drivers) will properly get the device to a stable state appropriately.

For example, a USB host adaptor cannot be shut down before dealing with all of the devices connected to that adaptor. The device model enables a traversal of the system's hardware in the right order. 

**2.2 Communications with user space:** <br>
The implementation of the sysfs virtual filesystem is tightly tied into the device model and exposes the structure represented by it. The provision of information about the system to user space and knobs for changing operating parameters is increasingly done through sysfs and, therefore, through the device model.

**2.3 Hot-pluggable devices:** <br>
Computer hardware is increasingly dynamic; peripherals can come and go at the whim of the user. The hotplug mechanism used within the kernel to handle and (especially) communicate with user space about the plugging and unplugging of devices is managed through the device model.

**2.4 Device classes:** <br>
Many parts of the system have little interest in how devices are connected, but they need to know what kinds of devices are available. The device model includes a mechanism for assigning devices to classes, which describe those devices at a higher, functional level and allow them to be discovered from user space.

**2.5 Object lifecycles:** <br>
Many of the functions described above, including hotplug support and sysfs, complicate the creation and manipulation of objects created within the kernel. The implementation of the device model required the creation of a set of mechanisms for dealing with object lifecycles, their relationships to each other, and their representation in user space.
> #### 2.5.1 Correspondence between kernel structures and sysfs:
&nbsp;&nbsp;&nbsp;&nbsp; From the perspective of sysfs, Kobject and Kset create corresponding folders. Thus establishing the device driver model. <be>

&nbsp;&nbsp;&nbsp;&nbsp; **Kobject  :**  Represents a directory under sysfs. <br>
&nbsp;&nbsp;&nbsp;&nbsp; **Kset     :**  Is a collection or a set containing multiple object. If you need to include multiple subdirectories in your directory, you need to define it as one kset. <br>
&nbsp;&nbsp;&nbsp;&nbsp; **Kobj_type:**  Represents file properties. <br>

## 3. Kobject/Kset/ktype:

Before describing the data structure, it is necessary to explain the three concepts: Kobject, Kset and Ktype.

##### First of all, what is kobject?
It is just a structure in the kernel. Below is the structure details:
```
/* Kobject: include/linux/kobject.h */

struct kobject {
const char *name;                            // It is a name of the kobject, which is also the directory name in sysfs.
struct list_head entry;                      // Used to add kobject to kset->list (or) list_head in kset structure.
struct kobject *parent;                      // Pointing to the parent kobject to form a hierarchical structure (shown as a directory structure in sysfs).
struct kset *kset;                           // The Kset to which this kobject belongs. It can be NULL. If it exists and no parent is specified, Kset will be used as the parent (don't forget that Kset is a special Kobject).
struct kobj_type *ktype;                     // The kobj_type to which this Kobject belongs. Each Kobject must have a ktype, or Kernel will prompt an error.
struct kernfs_node *sd;                      // The representation of this Kobject in sysfs.
struct kref kref;                            // Reference count object, Supporting the reference counting function for the kobject.
unsigned int state_initialized:1;            // Record whether the kobject is initialized or not. After calling kobject_init(), it will be set.
unsigned int state_in_sysfs:1;               // Record if kobj is registered in sysfs and set it in kobject_add_internal().
unsigned int state_add_uevent_sent:1;        // Set when sending a KOBJ_ADD message. Prompt that an ADD message has been sent to user space.
unsigned int state_remove_uevent_sent:1;     // Set when a KOBJ_REMOVE message is sent. Prompt that a REMOVE message has been sent to user space.
unsigned int uevent_suppress:1;              // If this field is 1, it means that all reported uevent events are ignored.
};
```
So what does this structure represent in the kernel? <br>
- Each *"struct kobject"* corresponds to a directory under /sys sysfs filesystem as mentioned below block.

```
root@dev#: ls /sys -l
total 0
drwxr-xr-x   2 root root 0 Mar 15 04:11 block/ <<<<<<< corresponds to struct kobject.
drwxr-xr-x  33 root root 0 Mar 15 04:11 bus/ 
drwxr-xr-x  74 root root 0 Mar 15 04:11 class/
drwxr-xr-x   4 root root 0 Mar 15 04:11 dev/
drwxr-xr-x  50 root root 0 Mar 15 04:11 devices/
drwxr-xr-x   7 root root 0 Mar 15 04:11 firmware/
drwxr-xr-x  11 root root 0 Mar 15 04:11 fs/
drwxr-xr-x  15 root root 0 Mar 15 04:11 kernel/
drwxr-xr-x 224 root root 0 Mar 15 04:11 module/
drwxr-xr-x   3 root root 0 Mar 18 17:45 power/
```

#### What is Kset?
It is also a structure in the kernel which embeds a kobject and user event operations for the kset. 
Kset contains a collection of kobjects.
Below is the structure details:
```
 /* Kset: include/linux/kobject.h */

 struct kset {
     struct list_head list;                    // It corresponds to kobj->entry(in struct kobject), and is used to organize kobj objects managed by this kset.
     spinlock_t list_lock;                     // To hold lock for the linked list.
     struct kobject kobj;                      // The kset's own kobject(kset has a kobject that will also be reflected in the form of a directory in sysfs).
     const struct kset_uevent_ops *uevent_ops; // The uevent operation function set of this kset. When any Kobject needs to report a uevent, it must call the
                                                  uevent_ops of the kset it belongs to, add environment variables, or filter events (kset can decide which events
                                                  can be reported). Therefore, if a kobject does not belong to any kset, uevent is not allowed to be sent.
 };
```

#### What is Ktype(or kobj_type)?
struct kobj_ktype represents a type of kobject, used to manage the life cycle of kobject and control the behavior of object.

```
/* Ktype: include/linux/kobject.h */

struct kobj_type {
    void (*release)(struct kobject *kobj);          // Release function to be called to release resources when kobject is no longer referenced.
    const struct sysfs_ops *sysfs_ops;              // sysfs operation structure's pointer.
    const struct attribute_group **default_groups;  // an array pointing to the default attribute group, used to register files with sysfs. Each attribute corresponds to a file in sysfs.
    const struct kobj_ns_type_operations *(*child_ns_type)(struct kobject *kobj); // is related to the namespace of the file system (sysfs)
    const void *(*namespace)(struct kobject *kobj); // is related to the namespace of the file system (sysfs)
    void (*get_ownership)(struct kobject *kobj, kuid_t *uid, kgid_t *gid);
};
```

#### attribute_group:
The attribute group interface is a simplified interface for easily adding and removing a set of
attributes with a single call.

```
 struct attribute_group {
         const char              *name;
         umode_t                 (*is_visible)(struct kobject *,
                                               struct attribute *, int);
         umode_t                 (*is_bin_visible)(struct kobject *,
                                                   struct bin_attribute *, int);
         struct attribute        **attrs;
         struct bin_attribute    **bin_attrs;
 };

```

#### structure for representing a file in sysfs:
The attribute object represents an attribute file in the file system. By modifying the file, the kernel data can be modified. 
A user interface similar to debugfs can read and write the parameters of the specified kernel space. 
The attribute structure contains two members: name and mode, which are used to specify the name and read and write permissions of the created file respectively.

```
struct attribute {
	const char		*name;
	umode_t			mode;
#ifdef CONFIG_DEBUG_LOCK_ALLOC
	bool			ignore_lockdep:1;
	struct lock_class_key	*key;
	struct lock_class_key	skey;
#endif
};

```

### Relationship between kobject and kset:
Below is the kobject and kset relationship in Linux kernel: <br>

#### UML Class view:
![kobject:kset_UML](https://github.com/Sudharshan-07/Linux/assets/52316856/c42e91af-da9a-475f-ac7d-718c4273f497)

#### Block diagram view:
![Kobject:Kset_relationship_diagram](https://github.com/Sudharshan-07/Linux/assets/52316856/09dfed84-ca09-4ae8-a88d-19b27ddc3c16)

### Hierarchy of kobject and kset for /sys/bus/platform/devices and /sys/bus/platform/drivers:
For below sysfs hierarchy:

```
/sys/
├── block
...
├── bus
│   ├── platform
│   │   ├── devices
│   │   │   ├── dev_1 -> ../../../devices/platform/dev_1
│   │   │   ├── dev_2 -> ../../../devices/platform/dev_2
...
│   │   │   ├── dev_3 -> ../../../devices/pci0000:00/0000:00:01.0/XYZ0001:00
│   │   ├── drivers
│   │   │   ├── drv_1
│   │   │   │   ├── bind
│   │   │   │   ├── uevent
│   │   │   │   └── unbind
│   │   │   ├── drv_2
│   │   │   │   ├── bind
│   │   │   │   ├── uevent
│   │   │   │   └── unbind
...
│   │   │   ├── drv_n
...
...

```
Below is the pictorial representation of how Linux kernel constructs the hierarchy using kobject and kset:

<img src="https://github.com/Sudharshan-07/Linux/assets/52316856/3f7029b6-ed59-40fe-af69-adbcd4202da6" width="2200">

## summary:
The core of the Linux device model is to use the four core data structures such as **struct class, device, and device_driver** to combine a large number of hardware devices with different functions (also the drivers that drive the hardware devices) and abstract in the form of a tree structure to facilitate the unified device model.

As the number and types of hardware devices are very large, these data structures must have some common functions and need to be abstracted and implemented uniformly, otherwise, redundant code will inevitably be generated. This is the background of the birth of Kobject.

The core function of kobject is to maintain a reference count. When the count is reduced to 0, the memory space occupied by Kobject is automatically released (responsible for the kobject APIs). This determines that Kobject must be dynamically allocated (only in this way can it be dynamically released).

Most usage scenarios of Kobject are embedded in large data structures (such as struct kset, device_driver, device, etc.), so these large data structures must also be dynamically allocated and released. The release of Kobject is automatically completed by the Kobject APIs(when the reference count goes to 0), so how to release the large data structure containing itself at the same time?.

This is where Ktype(kobj_type) comes in handy. We know that the release callback function in Ktype is responsible for releasing the memory space of Kobject (even the data structure containing Kobject). so who implements Ktype and its internal functions? It is the module wherein the upper data structure(such as class, device, device_driver) kobject is located. Because only it knows which data structure the Kobject is embedded in, and through the Kobject pointer and its own data structure type, it can find the pointer of the upper data structure that needs to be released, and then release it.

At this point, it becomes much clearer. Therefore, every data structure with embedded Kobject, such as kset, device, device_driver, etc., must implement a Ktype and define the callback function. In the same way, sysfs related operations must go through ktype transfer, because sysfs sees Kobject, and the main body of the real file operation is the upper data structure of the embedded Kobject!

So far, Kobject mainly provides the following functions:
- Through the parent pointer, all kobjects can be combined in a hierarchical structure.
- Use a reference count to record the number of times Kobject is referenced, and release it when the number of references becomes 0 (this is the only function of Kobject when it was born).
- Cooperating with the sysfs virtual file system, each Kobject and its characteristics are opened to the user space in the form of files.
- Sends uevents to userspace.

*Note 1: In Linux, Kobject rarely exists alone. Its main function is to be embedded in a large data structure and provide some underlying functional implementations for this data structure.*

*Note 2: Linux driver developers rarely use Kobject and the interfaces it provides **directly**. Instead, they use the device model interface built on Kobject(i.e, which means kobject acts as an abstract or base class of core data structure in the kernel).*

*Note 3: Kobject is the ultimate embodiment of object-oriented thinking in the Linux kernel, but the advantages of the C language don't support OOPs, so the Linux kernel needs to use more clever (and verbose) means to implement it.*
