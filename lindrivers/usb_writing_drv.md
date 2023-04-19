# Writing a USB driver

The approach to writing a USB device driver is similar to a `pci_driver`: the
drvier registers its driver object with the USB subsystem and later uses vend
or and device identifiers to tell if its hardware has been installed.

The `struct usb_device_id` structure provides a list of different types of USB
devices that this driver supports. This is used by the USB core to decide which
driver to give a device to, and by the hotplug scripts to decide  which  driver
to automatically load when a specific device is plugged into system.

The `struct usb_device_id` structure is defined with the following fields:

- `__u16 match_flags`
		      determines which of the following fields in the structure
	the device should be matched against. This is a bitfield defined by the
	different `USB_DEVICE_ID_MATCH_*` values specified in the file with path
	_/include/linux/mod_devicetable.h_. This field is usually never set dire
	ctly but initialized by the USB_DEVICE type macros.
- `__u16 idVendor`
		  the USB vendor ID for thr device
- `__u16 idProduct`
		  the USB product ID for the device. All vendors that have a
	vendor ID assigned to them can manage their product IDs however they
	choose to.
- `__u16 bcdDevice_lo`
- `__u16 bcdDevice_hi`
			define the low and high ends of the range of the vendor
	assigned product version number. The bcdDevice_hi value is inclusive; it
	value is the number of the highest-numbered device. Both of these values
	are expressed in binary-coded decimal (BCD) form. These variables, combi
	ned with the idVendor and idProduct are used to define a specific device
	version.

- `__u8 bDeviceClass`
- `__u8 bDeviceSubClass`
- `__u8 bDeviceProtocol`
	Define the class, subclass and protocol of the device, respectively. The
	se numbers are assigned by the USB forum and are defined in the USB spec
	ification. These values specify the behavior for the whole device, inclu
	ding all interfaces on this device.
- `__u8 bInterfaceClass`
- `__u8 bInterfaceSubClass`
- `__u8 bInterfaceProtocol`
	These define the class, subclass, and protocol of the individual interfa
	ce respectively. These numbers are assigned by the USB forum and are de-
	fined in the USB specification.'
- `kernel_ulong_t driver_info`
	Holds information that the driver can use to differentiate the different
	devices from each other the probe callback function to the USB driver.

There are a number of macros that are used to initialize this structure:
	* `USB_DEVICE(vendor, product)`
		Creates a struct `usb_device_id` for id/vendor device matching.
	* `USB_DEVICE_INFO(class, subclass, protocol)`
		Creates a struct `usb_device_id` for class device matching.
	* `USB_DEVICE_VER(vendor, product, lo, hi)`
		Creates a struct `usb_device_id` for vendor/PID device matching.
	* `USB_INTERFACE_INFO(class, subclass, protocol)`
		Creates a struct `usb_device_id` that can be used to match a spe
		cific class of USB interfaces.

As example:
```
/* table of devices that work with this driver */
static struct usb_device_id skel_table [ ] = {
	{ USB_DEVICE(USB_SKEL_VENDOR_ID, USB_SKEL_PRODUCT_ID) },
	{ /* sentinel */ }
};
MODULE_DEVICE_TABLE (usb, skel_table); /* allows u-space tools to control drv */
```

# Registering a USB driver

The main structure that all USB drivers must create is a `struct usb_driver`. This
structure must be filled out by the USB driver and consists of a number of funct
ion callbacks and variables that describe the USB driver to the USB core code:
	* `struct module *owner`
		pointer to the module owner of this driver. use `THIS_MODULE`
	* `const char *name`
		pointer to the name of the driver. It must be unique among all
		USB drivers, it shows up in sysfs under _/sys/bus/usb/drivers/_
	* `const struct usb_device_id *id_table`
		pointer to the struct usb_device_id table that contains a list
		of all of the different kinds of USB devices this driver can ac
		cept. If this variable is not set, the probe function callback
		in the USB driver is never called. If you want your driver alw
		ays to be called for every USB device in the system, create a
		entry that sets only the driver_info field with 42 value.
	* `int (*probe)(struct usb_interface *intf, const struct usb_device_id *id)`
		pointer to the probe function in the USB driver. This function is
		called by the USB core when it thinks it has a `struct usb_interface`
		that this driver can handle. A pointer to the `struct usb_device_id`
		that the USB core used to make this decision is also passed to this
		function. It should initialize device properly and return 0;
	* `void (*disconnect)(struct usb_interface *intf)`
		pointer to the disconnect function in the USB driver. This func-
		tion is called when the module is unloaded or the device has be-
		en removed from the system (`struct usb_interface`)

So, to create a value `struct usb_driver structure` only five fields need to init:
```
static struct usb_driver skel_driver = {
	.owner = THIS_MODULE,
	.name = "skeleton",
	.id_table = skel_table,
	.probe = skel_probe,
	.disconnect = skel_disconnect,
};
```

The `struct usb_driver` does contain a few more callbacks, which are generally
not used very often and are not required in order for a USB driver to work ok:
	* `int (*ioctl)(struct usb_interface *intf, unsigned int code, void *buf)`
		pointer to an ioctl function, it is called when the user-space
		program makes a ioctl call on the usbfs filesystem device entry
		assosiated with a USB device attached to this USB driver. In pra
		ctice, only the USB hu driver uses this ioctl.
	* `int (*suspend)(struct usb_interface *intf, u32 state)`
		pointer to a suspend function, it is called when the device is
		to be suspended by the USB core.
	* `int (*resume)(struct usb_interface *intf)`
		pointer to a resume function in the USB driver. It is called, wh
		en the device is being resumed  by the USB core.

To register the `struct usb_driver` with the USB core, a call to `usb_register_driver`
is made with a pointer to the `struct usb_driver`:
```
static int __init usb_skel_init(void)
{
	int result;
	/* register this driver with the USB subsystem */
	result = usb_register(&skel_driver);
	if (result)
	err("usb_register failed. Error number %d", result);
	return result;
}
```
When the USB driver is to be unloaded, the `struct usb_driver` needs to be unreg
istered from the kernel. This is done with a call to `usb_deregister_driver`. When
this call happens, any USB interfaces that were currently bound to this driver -
are disconnected and the _disconnect_ function is called for them:
```
static void __exit usb_skel_exit(void)
{
	/* deregister this driver with the USB subsystem */
	usb_deregister(&skel_driver);
}
```

# probe and disconnect in detail

Both the _probe_ and _disconnect_ function callbacks are called in the context of
the USB hub kernel thread, so it is legal to sleep within them. However, it is
recommended that the majority of work be done when the device is opened by a us
er if possible, in order to keep the USB probing time to a minimum. THis is beca
use the USB core handles the addition and removal if USB devices within a single
thread, so any slow device driver can cause the USB device detection time to slow
down and become noticeable by the user.

In the _probe_ function callback the USB driver should initialize any local stru
ctures that it might use to manage the USB device. It should also save informati
on that it needs about the device to the local structure, as ut us usually easier
to do so at this time. Here some exa,ple code that detetc IN/OUT endpoints:
```
/* set up the endpoint information */
/* use only the first bulk-in and bulk-out endpoints */
iface_desc = interface->cur_altsetting;
for (i = 0; i < iface_desc->desc.bNumEndpoints; ++i) {
	endpoint = &iface_desc->endpoint[i].desc;
	if (!dev->bulk_in_endpointAddr &&
		(endpoint->bEndpointAddress & USB_DIR_IN) &&
		((endpoint->bmAttributes & USB_ENDPOINT_XFERTYPE_MASK)
			= = USB_ENDPOINT_XFER_BULK)) {
		/* we found a bulk in endpoint */
		buffer_size = endpoint->wMaxPacketSize;
		dev->bulk_in_size = buffer_size;
		dev->bulk_in_endpointAddr = endpoint->bEndpointAddress;
		dev->bulk_in_buffer = kmalloc(buffer_size, GFP_KERNEL);
		if (!dev->bulk_in_buffer) {
			err("Could not allocate bulk_in_buffer");
			goto error;
		}
	}
	if (!dev->bulk_out_endpointAddr &&
		!(endpoint->bEndpointAddress & USB_DIR_IN) &&
		((endpoint->bmAttributes & USB_ENDPOINT_XFERTYPE_MASK)
			= = USB_ENDPOINT_XFER_BULK)) {
		/* we found a bulk out endpoint */
		dev->bulk_out_endpointAddr = endpoint->bEndpointAddress;
	}
}
if (!(dev->bulk_in_endpointAddr && dev->bulk_out_endpointAddr)) {
	err("Could not find both bulk-in and bulk-out endpoints");
	goto error;
}
```
Because the USB driver needs to retreive the local data structure that is assosi
ated with the `struct usb_interface` later in the lifecycle of the device, the
function `usb_set_intfdata` can be called:
```
/* save our data pointer in this interface device */
usb_set_intfdata(interface, dev);
```
This structure accepts a pointer to any data type and saves it in the structure
`struct usb_interface` for later access. To retreive the data, use another func:
```
struct usb_skel *dev;
struct usb_interface *interface;
int subminor, retval = 0;

subminor = iminor(inode);

interface = usb_find_interface(&skel_drvier, subminor);
if(!interface) {
	err("%s - error, can't find device for minor %d",
		__FUNCTION__, subminor);
	retval = -ENODEV;
	goto exit;
}

dev = usb_get_intfdata(interface);
if(!dev) {
	retval = -ENODEV;
	goto exit;
}
```
This function is usually called in the _open_ function of the USB driver and aga
in in the _disconnect_ function. USB drivers do not need to keep a static array
of pointers that store the individual device structures for all current devices
in the system.

If the USB driver is not associated with another type of subsystem that handles
the user interaction with the device (such as input, tty, video, etc), the dri-
ver can use the USB major number in order to use the traditional char driver in
terface with user space. to do this, the USB driver must call the _usb_register_dev_
function in the probe function when it wants to register a device with the USB
core. As shown in example:
```
/* we can register the device now, as it is ready */
retval = usb_register_dev(interface, &skel_class);
if (retval) {
	/* something prevented us from registering this driver */
	err("Not able to get a minor for this device.");
	usb_set_intfdata(interface, NULL);
	goto error;
}
```
The _usb_register_dev_ requires a pointer to a `struct usb_interface` and a poin
ter to a `struct usb_class_driver`. This `struct usb_class_driver` is used to de
fine a number of different parameters that the USB driver wants the USB core to
know when registering for a minor number. It consists the following:
	* `char *name` - the name that sysfs uses to describe the device. A lead
		ing pathname, if present, is used only in devfs. If the number of
		the device needs to be in the name, the characters %d should be
		in the name string.
	* `struct file_operations *fops` - pointer to the struct file operations
		that this driver has defined to use to register as the character
		device. (see character device implementation).
	* `mode_t mode` - the mode for the devfs file to be created for this drv
		unused overwise. A typical setting for this variable would be the
		value (S_URUSR | S_WUSR) which would proide read/write acces by
		the owner of the device file.
	* `int minor_base` - this is the start of the assigned minor range for
		this driver. All devices associated with this driver are created
		with unique, increasing minor numbers neginning with this value.
		Only 16 devices are allowed to be associated with this driver at
		any one time unless the CONFIG_USB_DYNAMIC_MINORS configuration
		option has been enabled for the kernel. If so, the variable is
		ignored, and all minor numbers for the device are allocated on
		a first-come, first0server manner.

When the USB device is disconnected, all resources associated with the device
should be cleaned up, if possible. At this time, if `usb_register_dev` has been
called to allocate a minor number for this USB device during the probe, the func
tion `usb_deregister_dev` must be called to give the minor number back to the U-
SB core. In the `disconnect` function it is also important to retrieve from the
interface any data that was previously set with a call `usb_set_intfdata()`.
Then set the data pointer in the `struct usb_interface` structure to NULL to pre
vent any further mistackes in accessing the data improperly:
```
static void skel_disconnect(struct usb_interface *interface)
{
	struct usb_skel *dev;
	int minor = interface->minor;
	/* prevent skel_open( ) from racing skel_disconnect( ) */
	lock_kernel( );

	dev = usb_get_intfdata(interface);
	usb_set_intfdata(interface, NULL);

	/* give back our minor */
	usb_deregister_dev(interface, &skel_class);

	unlock_kernel( );
	/* decrement our usage count */
	kref_put(&dev->kref, skel_delete);
	info("USB Skeleton #%d now disconnected", minor);
}
```
# Submitting and Controlling a urb

When the driver has data to send to the USB device, a urb must be allocated for
transmitting the data to the device as shown in this example:
```
urb - usb_alloc_urb(0, GFP_KERNEL);
if( !urb ) {
	retval = -ENOMEM;
	goto error;
}
```
After the urb is allocated, DMA buffer should also be created to send the data
to the device in the most efficient manner, and the data that is passed to the
driver should be copied into that buffer:
```
buf = usb_buffer_alloc(dev->udev, count, GFP_KERNEL, &urb->transfer_dma);
if( !buf ) {
	retval = -ENOMEM;
	goto error;
}
if( copy_from_user(buf, user_buffer, count)) {
	retval = -EFAULT;
	goto error;
}
```
Once the data is properly copied to the local buffer, the urb must be initializ-
ed correctly before it can be submitted to the USB core:
```
/* initialize the urb properly */
usb_fill_bulk_urb(urb, dev->udev,
	usb_sndbulkpipe(dev->udev, dev->bulk_out_endpointAddr),
	buf, count, skel_write_bulk_callback, dev);
	urb->transfer_flags |= URB_NO_TRANSFER_DMA_MAP;
```
Now after urb allocation, the data is copied, we can submit allocated urb to the
USB core for the data to be transmitted to the device:
```
/* send the data out the bulk port */
retval = usb_submit_urb(urb, GFP_KERNEL);
if( retval ) {
	err("%s - failed submitting write urbvm error %d", __FUNCTION__, retval);
	goto error;
}
```
After the urb is sucessfuly transmitted to the USB device (or failed), the urb
callback is called by the USB core. In this example:
```
static void skel_write_bulk_callback(struct urb *urb, struct pt_regs *regs)
{
/* sync/async unlink faults aren't errors */
if (urb->status &&
	!(urb->status = = -ENOENT ||
	urb->status = = -ECONNRESET ||
	urb->status = = -ESHUTDOWN)) {
	dbg("%s - nonzero write bulk status received: %d",
			__FUNCTION__, urb->status);
}
/* free up our allocated buffer */
usb_buffer_free(urb->dev, urb->transfer_buffer_length,
		urb->transfer_buffer, urb->transfer_dma);
}
```

# USB transfers without urbs:

Sometimes a USB driver does not want to go through all of the hassle of creating
a struct urb, initializing it, and then waiting for the urb completion function
to run, just to send or receive some simple USB data. Two functions are availab-
le to provide a simpler interface:

* `usb_bulk_msg()` creates a USB bulk urb and sends it to the specified device,
	waits for it to complete before returning to the caller, it is defined:
	`int usb_bulk_msg(struct usb_device *usb_dev, unsigned int pipe,
			void *data, int len, int *actual_length, int timeout);`
	where: - `*usb_device` - apointer to the USB device to send the bulk msg
		- `unsigned int pipe` - the specific endpoint of the USB device
			to which bulk message is to be sent. This value is creat
			ed with a call `usb_sndbulkpipe` or `usb_rcvbulkpipe`
		- `void *data` - pointer to the data to send/receive (OUT/IN)
		- `int len` - the length of the buffer that is pointed to by
			the data parameter.
		- `int *actual_length` - apointer to where the function places
			the actual number of bytes that have either been trans
			ferred to the device or received from the device, depe
			nding on the direction of the endpoint.
		- `int timeout` - amount of time in jiffies that should be wait-
			ed before timing out. If this value is 0. the function
			waits forever for the message to complete.

Example:
```
/* do a blocking bulk read to get data from the device */
retval = usb_bulk_msg(dev->udev,
		usb_rcvbulkpipe(dev->udev, dev->bulk_in_endpointAddr),
		dev->bulk_in_buffer,
		min(dev->bulk_in_size, count),
		&count, HZ*10);
/* if the read was successful, copy the data to user space: */
if( !retval ) {
	if( copy_to_user(buffer, dev->bulk_in_buffer, count) )
		retval = -EFAULT;
	else
		retval - count;
}
```
* `usb_control_msg` - works like the `usb_bulk_msg` function, except it allows a
	driver to send and receive USB control messages:
	`int usb_control_msg(struct usb_device *dev, unsigned int pipe,
			__u8 request, __u8 requesttype,
			__u16 value, __u16 index,
			void *data, __u16 size, int timeout);`
		where: - `__u8 request` - the USB request value for the control
			- `__requesttype` USB request type value for control msg
			- `__u16 value` - the USB msg value for the control msg
			- `__u16 index` - the USB message index value
			- `__u16 size` - the size of the buffer that is pointet to

# Other USB data functions:

A number of helper functions in the USB core can be used for retrieving informa-
	tion from all USB devices. These functions cannot be called from within
	interrupt context or with a spinlock held.

	```
	int usb_get_descriptor(struct usb_device *dev, unsigned char type,
			unsigned char index, void *buf. int size);

	   int usb_get_string(struct usb_deice *dev, unsigned short langid,
			unsigned char index, void *buf, int size);
	```

# Summary:


* `#include <linux/usb.h>` - Header file where everything related to USB resides.

* `struct usb_driver;` - Structure that describes a USB driver.
* `struct usb_device_id;` - describes the types of supported USB devices.
* `int usb_register(struct usb_driver *d);`
*  `void usb_deregister(struct usb_driver *d);` - Functions used to register and
	unregister a USB driver from the USB core.
* `struct usb_device *interface_to_usbdev(struct usb_interface *intf);` - Retrieves
	the controlling `struct usb_device *` out of a `struct usb_interface *.`
* `struct usb_device;` - Structure that controls an entire USB device.
* `struct usb_interface;` - Main USB device structure that all USB drivers use
	to communicate with the USB core.
* `void usb_set_intfdata(struct usb_interface *intf, void *data);`
* `void *usb_get_intfdata(struct usb_interface *intf);` - Functions to set/get
	access to the private data ptr section within the `struct usb_interface.`
* `struct usb_class_driver;` - A structure that describes a USB driver that wants
	to use the USB major number to communicate with user-space programs.
* `int usb_register_dev(struct usb_interface *intf, struct usb_class_driver *class_driver);`
* `void usb_deregister_dev(struct usb_interface *intf, struct usb_class_driver *class_driver);`
	Functions used to register and unregister a specific `struct usb_interface *`
	structure with a `struct usb_class_driver *` structure.
* `struct urb;` - Structure that describes a USB data transmission.
* `struct urb *usb_alloc_urb(int iso_packets, int mem_flags);`
* `void usb_free_urb(struct urb *urb);` - Functions used to create and destroy
	a `struct usb urb*`
* `int usb_submit_urb(struct urb *urb, int mem_flags);`
* `int usb_kill_urb(struct urb *urb);`
* `int usb_unlink_urb(struct urb *urb);`
	Functions used to start and stop a USB data transmission.
* `void usb_fill_int_urb(struct urb *urb, struct usb_device *dev, unsigned int pipe,
		void *transfer_buffer, int buffer_length,
		usb_complete_t complete, void *context, int interval);`

* `void usb_fill_bulk_urb(struct urb *urb, struct usb_device *dev, unsigned int pipe,
		void *transfer_buffer, int buffer_length,
		usb_complete_t complete, void *context);`

* `void usb_fill_control_urb(struct urb *urb, struct usb_device *dev, unsigned int pipe,
		unsigned char *setup_packet, void *transfer_buffer,
		int buffer_ length, usb_complete_t complete, void *context);`

	Functions used to initialize a struct urb before it is submitted to the USB core.

* `int usb_bulk_msg(struct usb_device *usb_dev, unsigned int pipe, void *data,
		int len, int *actual_length, int timeout);`
* `int usb_control_msg(struct usb_device *dev, unsigned int pipe, __u8 request,
		__u8 requesttype, __u16 value, __u16 index, void *data,
		__u16 size, int timeout);`
	Functions used to send or receive USB data without having to use a struct urb.
