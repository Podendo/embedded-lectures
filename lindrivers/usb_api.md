# Intro

Topologically, a USB subsystem is not laid out as a bus. It is rather a tree
built out of several point-to-point links. The links are four-wire cables as
follows: ground, power, two signal wires d+ d-.

A USB device can never start sending data without first being asked to by the
host controller. This config allows for a very easy plug and play type of sys
The bus is a single-master implemented. Features: ability for a device to re-
quest a fixed bandwidth for its data transfers in order to reliably support a
vildeo and audio I/O. Another USB feature is it acts merely as a communication
channel between the device and the host, without requiring specifing  meaning
or struct to the data it delivers.

The linux kernel supports two main types of USB drivers: drivers on a host sys
and drivers on a device. USB gadget is a driver that control a USB device that
connects to the computer (host system).

# Endpoints

A USB endpoint can carry data in only one direction, either from the host compu
ter to the device (called OUT endpoint) or from the device to the host computer
(called an IN endpoint). Endpoints can be though of as unidirectional pipes. A
USB endpoint can be one of four different types according to data transmission:

	* CONTROL - control endpoints are used to allow access to a different pa
		rts of the USB device. They are commonly used for device config,
		retrieving info about the device, sending commands to the device
		or retrieving status reports  about the device.  These endpoints
		are usually small in size. "endpoint 0" used for the device con-
		figuration during insertion time.

	* INTERRUPT - transfer small amounts of data at a fixed rate every time
		the USB host asks the device for data. These endpoints are pri-
		mary transport method for USB keyboards and mice. They are gua-
		ranteed by the USB protocol to always have enough reserved band
		width to make it through.

	BULK - bulk endpoints transfer large amounts of data. Used for large da-
		ta transfers through with no data loss. They are not guaranteed
		to the USB protocol to always make it through in a specific amo-
		unt of time. this endpoint can split one transaction if there is
		no space on the bus. Common for network, storage, printer device

	ISOCHRONOUS - similar to BULK, but the data is not always guaranteed to
		make it through.  These endpoints are used in devices  that can
		handle loss of data, and rely more on keeping a constant stream
		of data flowing. Real-time data collections use these ones. For
		example, audio devices, video devices.

# The USB endpoint description:
```
#include <linux/usb.h>

struct usb_host_endpoint {
	struct usb_endpoint_descriptor		desc;
	struct usb_ss_ep_comp_descriptor	ss_ep_comp;
	struct usb_ssp_isoc_ep_comp_descriptor	ssp_isoc_ep_comp;
	struct list_head			urb_list;
	void					*hcpriv;
	struct ep_device			*ep-devl /* for sysfs info */

	unsigned char *extra; /* extra descriptors */
	int extralen;
	int enabled;
	int streams;
};

struct usb_endpoint_descriptor {
	__u8  bLength;
	__u8  bDescriptorType;

	__u8  bEndpointAddress; /* address and direction, USB_DIR_{OUT | IN} */
	__u8  bmAttributes;	/* type of endpoints USB_ENDPOINT_XFERTYPE_MASK */
	__le16 wMaxPacketSize;	/* max bytes that this endpoint can handle at once */
	__u8  bInterval;	/* this value is the time between interrupt requests ms */

	/* NOTE:  these two are _only_ in audio endpoints. */
	/* use USB_DT_ENDPOINT*_SIZE in bLength, not sizeof. */
	__u8  bRefresh;
	__u8  bSynchAddress;
} __attribute__ ((packed));

```
# Interfaces

For one USB device you should use one interface. If the device is complex, you
need to write different drivers for this (ex: speaker: usb buttons, usb audio).
Interfaces in kernel are described with the *struct usb_interface*:
```
/**
 * struct usb_interface - what usb device drivers talk to
 * @altsetting: array of interface structures, one for each alternate
 *	setting that may be selected.  Each one includes a set of
 *	endpoint configurations.  They will be in no particular order.
 * @cur_altsetting: the current altsetting.
 * @num_altsetting: number of altsettings defined.
 * @intf_assoc: interface association descriptor
 * @minor: the minor number assigned to this interface, if this
 *	interface is bound to a driver that uses the USB major number.
 *	If this interface does not use the USB major, this field should
 *	be unused.  The driver should set this value in the probe()
 *	function of the driver, after it has been assigned a minor
 *	number from the USB core by calling usb_register_dev().
 * @condition: binding state of the interface: not bound, binding
 *	(in probe()), bound to a driver, or unbinding (in disconnect())
 * @sysfs_files_created: sysfs attributes exist
 * @ep_devs_created: endpoint child pseudo-devices exist
 * @unregistering: flag set when the interface is being unregistered
 * @needs_remote_wakeup: flag set when the driver requires remote-wakeup
 *	capability during autosuspend.
 * @needs_altsetting0: flag set when a set-interface request for altsetting 0
 *	has been deferred.
 * @needs_binding: flag set when the driver should be re-probed or unbound
 *	following a reset or suspend operation it doesn't support.
 * @authorized: This allows to (de)authorize individual interfaces instead
 *	a whole device in contrast to the device authorization.
 * @dev: driver model's view of this device
 * @usb_dev: if an interface is bound to the USB major, this will point
 *	to the sysfs representation for that device.
 * @reset_ws: Used for scheduling resets from atomic context.
 * @resetting_device: USB core reset the device, so use alt setting 0 as
 *	current; needs bandwidth alloc after reset.
 *
 * USB device drivers attach to interfaces on a physical device.  Each
 * interface encapsulates a single high level function, such as feeding
 * an audio stream to a speaker or reporting a change in a volume control.
 * Many USB devices only have one interface.  The protocol used to talk to
 * an interface's endpoints can be defined in a usb "class" specification,
 * or by a product's vendor.  The (default) control endpoint is part of
 * every interface, but is never listed among the interface's descriptors.
 *
 * The driver that is bound to the interface can use standard driver model
 * calls such as dev_get_drvdata() on the dev member of this structure.
 *
 * Each interface may have alternate settings.  The initial configuration
 * of a device sets altsetting 0, but the device driver can change
 * that setting using usb_set_interface().  Alternate settings are often
 * used to control the use of periodic endpoints, such as by having
 * different endpoints use different amounts of reserved USB bandwidth.
 * All standards-conformant USB devices that use isochronous endpoints
 * will use them in non-default settings.
 *
 * The USB specification says that alternate setting numbers must run from
 * 0 to one less than the total number of alternate settings.  But some
 * devices manage to mess this up, and the structures aren't necessarily
 * stored in numerical order anyhow.  Use usb_altnum_to_altsetting() to
 * look up an alternate setting in the altsetting array based on its number.
 */
struct usb_interface {
	/* array of alternate settings for this interface,
	 * stored in no particular order */
	struct usb_host_interface *altsetting;

	struct usb_host_interface *cur_altsetting;	/* the currently
							 * active alternate setting */
	unsigned num_altsetting;			/* number of alternate settings */

	/* If there is an interface association descriptor then it will list
	 * the associated interfaces */
	struct usb_interface_assoc_descriptor *intf_assoc;

	int minor;			/* minor number this interface is
					 * bound to */
	enum usb_interface_condition condition;		/* state of binding */
	unsigned sysfs_files_created:1;	/* the sysfs attributes exist */
	unsigned ep_devs_created:1;	/* endpoint "devices" exist */
	unsigned unregistering:1;	/* unregistration is in progress */
	unsigned needs_remote_wakeup:1;	/* driver requires remote wakeup */
	unsigned needs_altsetting0:1;	/* switch to altsetting 0 is pending */
	unsigned needs_binding:1;	/* needs delayed unbind/rebind */
	unsigned resetting_device:1;	/* true: bandwidth alloc after reset */
	unsigned authorized:1;		/* used for interface authorization */

	struct device dev;		/* interface specific device info */
	struct device *usb_dev;
	struct work_struct reset_ws;	/* for resets in atomic context */
};
```

# Configurations

To summarize, USB devices are quite complex and are made up of lots of different
logical units. The relationships among these units can be simply described as:

* Devices usually have one or more configuration (struct usb_host_config)
* Configurations often have one or more interfaces
* Interfaces usually have one or more settings
* Interfaces have zero or more endpoints

# USB and SysFS

Both the physical USB device ( struct usb_device) and the individual USB interfa
ces ( as represented by a struct usb_interface) are shown in sysfs as individual
```
/sys/devices/pci0000:00/0000:00:09.0/usb2/2-1
|-- 2-1:1.0
|
|-- bAlternateSetting
|
|-- bInterfaceClass
|
|-- bInterfaceNumber
|
|-- bInterfaceProtocol
|
|-- bInterfaceSubClass
|
|-- bNumEndpoints
|
|-- detach_state
|
|-- iInterface
|
\-- power
|
\-- state
|-- bConfigurationValue
|-- bDeviceClass
|-- bDeviceProtocol
|-- bDeviceSubClass
|-- bMaxPower
|-- bNumConfigurations
|-- bNumInterfaces
|-- bcdDevice
|-- bmAttributes
|-- detach_state
|-- devnum
|-- idProduct
|-- idVendor
|-- maxchild
|-- power
|
\-- state
|-- speed
\-- version
```
So to summarize, the USB sysfs device naming scheme is:
`root_hub-hub_port:config.interface`
`root_hub-hub_port-hub_port:config.interface`

Another specific USB interface configuration can be found int he _usbfs_ filesys-
tem, which is mounter in the _/proc/bus/usb/_ directory on the system. The _devices_
file show all of the same information exposed in sysfs as well as the alternate
configuration and endpoint information for all USB devices that are present in
the system. usbfg allows user-space programs to directly talk to USB devices, wh
ich has enabled a lot of kernel drivers to be moved out to user-space, where its
easier to maintain and debug.

# USB Urbs

All USB devices communicate in the LK using USN request block - URB (struct urb).
A URB is used to send or receive data to or from a specific USB endpoint on a sp
ecific USB device in an asynchronous manner.
The typical lifecycle of an urb is as follows:

* Created by a USB device driver
* Assigned to a specific endpoint to a specific USB device
* Submitted to the USB core, by the USB device driver
* Submitted to the specific USB host controller driver for the specified device
	by the USB core.
* Processed by the USB host controller driver that makes a USB transfer to device
* When the URB is completed, the USB host controller driver notifies the USB dev
	ice driver.

URBs can also be cancelled any time by the drier that submitted the urb, or by
the USB core if the device is rempved from the system. They are dynamically crea
ted and contain an internal reference count that enables them to be automatically
freed when the last user of the urb releases it.

# struct urb

The fileds of the struct urb structure that matter to a USB device driver are:

* `struct usb_device *dev` - pointer to the device to which this urb is sent, it
	must be initialized by the USB driver before the urb can be sent to core

* `unsigned int pipe` - endpoint information for the specific usbdevice that this
	urb is to be sent to. Must be initialized.

* `unsigned int transfer_flags` - can be set to a number of different bit values
	depending on what the USB driver wants to happen to the urb, flags are:
	* `URB_SHORT_NOT_OK` - any short read on an IN endpoint that might occur
		should be treated as an error by the USB core. This value is use
		ful only for urbs that are to be _read_ from the USB device.
	* `URB_ISO_ASAP` - if the urb is isochronous, this bit can be set if the
		driver wants the urb to be scheduled, as soon as the bandwidth
		utilization allows it to be, and to set the start_frame variable
		in the urb at that point
	* `URB_NO_TRANSFER_DMA_MAP` - should be set when the urb contains a DMA
		buffer to be transferred. The usb core uses the buffer pointed
		to by the transfer_dma variable and not the buffer pointed to by
		the transfer_buffer variable.
	* `URB_NO_SETUP_DMA_MAP` - this bit is used for control urbs that have a
		DMA buffer already set-up. If it is set, the USB core uses the b
		uffer pointed to by the setup_dma variable instead of the setup_
		packet variable.
	* `URB_ASYNC_UNLINK` - the function call to usb_unlink_urb for this urb
		returnes almost immediately, and the urb is unlinked in the back
		ground. Otherwise, the function waits until the urb is completly
		unlinked and finished before returning.
	* `URB_NO_FSBR` - used only the UHCI USB Host controller drier and tells
		it to not try to do Front Side Bus Reclamation logic. This bit
		should generally not be set, because machines with a UHCI host
		controller create a lot of CPU overhead, and the PCI bus is sat
		urated waiting on a urb that sets this bit.
	* `URB_ZERO_PACKET` - if set, bulk out urb finishes by sending a short
		packet containing no data when the data is aligned to an endpoint
		packet boundary. (used for broken USB devices)
	* `URB_NO_INTERRUPT` - if set, the HW may not generate an interrupt when
		the urb is finished. This bit should be used with care and only
		when queuing multiple urbs to the same endpoint. The USB Core
		functions use this in orer to do DMA buffer transfers.

* `void *transfer_buffer` - pointer to the buffer to be used when sending data
	to the device (for an OUT urb) or when receiving data from the device
	(for an IN urb). In order for the host controller to properly access
	this buffer, it must be created with a call to kmalloc, not on the stack
	or statically. For control endpoints, this buffer is for the data stage
	of the transfer.

* `dma_addr_t tranfer_dma` - buffer to be used to transfer data to the USB via DMA

* `int transfer_buffer_length` - the length of the buffer pointed to by the tra-
	nsfer buffer or dma buffer. If this is 0, neither xfer buffer are used
	by the USB core.

* `unsigned char *setup_packet` - pointer to the setup packet for a control urb.
	It is transfered before the data in the transfer buffer. This variable
	is valid only for control urbs.

* `dma_addr_t setup_dma` - DMA buffer for the setup packet for a control urb.

* `usb_complete_t complete` - pointer to the completion handler function that is
	called by the USB core when the urb is completely transferred or when an
	error occurs to the urb.

* `void *context` - pointer to a data blob that can be set by the USB driver. It
	can be used in the completion handler when the urb is returned.

* `int actual length` - when the urb is finished, this var is set to the actual
	ength of the data either sent or received by the urb (OUT/IN urbs). For
	IN urbs this ine should be used instead of the transfer_buffer_length.

* `int status` - when the urb is being processed b usb core, this var is set to
	the current status of the urb. The only time a USB driver can safely ac-
	cess this variable is in the urb completion handler function. returns:
	* -ENOENT - the urb was stopped by a call to usb_kill_urb
	* -ECONNRESET - the urb was unlinked by a call to usb_unlink_urb
	* -EINPROGRESS - the urb is still being progressed by the USB host ctrlr
		(if the driver ever sees this value, the byg in your driver)
	* -EPROTO - bitstuff error or no-response packet received by hw error
	* -EILSEQ - CRC mismatch in the urb transfer
	* -EPIPE - the endpoint is now stalled. usb_clear_halt to clear this err
	* -ECOMM - data was received faster during the xfer plan, IN urb error
	* -ENOSR - data could not be retrieved from the system memory during the
		transfer fast enough to keep up with the requested USB data rate
	* -EOVERFLOW - a "babble" error, endpoint receives more data than the max
	* -EREMOTEIO - occurs if the URB_SHORT_NOT_OK (1) - full amount of data
		requested by the urn was not received.
	* -ENODEV - The USB device is now gone from the system.
	* -EXDEV - isochronous urb - xfer was only partially completed. The dri-
		ver must look at the individual frame status.
	* -EINVAL - something very bad happened with the urb, ISO madness
	* -ESHUTDIWN - there was a severe error with the USB host controller drv
		it has now been disabled, or the device was disconnected form a
		system, and the urb was submitted after the deice was removed.
		It can also occur if teh configuration was changed for the device,
		while the urb was submitted to the device.

* `int start_frame` - sets or returns the initial frame number for isochronous xfer
* `int interval` - the interval at which the urb is polled.
* `int number_of_packets` - valid only for isochronous urbs and specifies the num
	ber of transfer buffers to be handled by this urb. THis value must be set
	by the USB driver for ISO URBs before the URB is sent to the USB core.

* `struct usb_iso_packet_descriptor iso_frame_desc[0]` - ISO URBs, arrat of the
	struct usb_iso_packet_descriptor structures that make up this urb.
	Allows to a ingle urb to define a number of xfers at once. struct fields:
		* offset
		* length
		* actual_length
		* status

# creating and destroying urbs

The struct urb structure must never be created statically in a driver or within
another structure, because that would break the reference counting scheme used by
the USB core for urbs. It must be created with a call to the _usb_alloc_urb_ func

	`struct urb *usb_alloc)urb(int iso_packets, int mem_flags)`

iso packets is the number of isochronous oackets this urb should contain. If you
want to create a non-isochronous urb, this value should be set to 0. The second
parameter, mem_flags, is the same type of flag that is passed to the kmalloc fun
ction call to allocate memort from the kernel.

In order to tell the USB core that the driver is finished with the urb, the driv
er must call the usb_free_urb() function:

		`void usb_free_urb(struct urb *urb)`

The argument is a pointer to the struct urb you want to release. After this fun-
ction is called, the urb structure is gone, and the drvier cannot access it again

Initialization procedure depends on the type of the used urb:

* interrupt urbs
	`void usb_fill_int_urb(struct urb *urb, struct usb_device *dev,
				unsigned int pipe, void *transfer_buffer,
				int buffer_length, usb_complete_t complete,
				void *context, int interval);`

	where: - *urb is a pointer to the created urb
		- *dev is the USB device to which this urb is to be sent
		- pipe is the specific endpoint of the USB device to which this
			urb is to be sent. This value is created whith the pre
			viously mentioned usb_sndintpipe() or usb_rcvintpipe()
		- *transfer_buffer - a pointer for xfer data, must be created with
			a call to kmalloc();
		- buffer_length - the len of the buffer pointed to
		- complete - pointer to the blob that is added to the urb structure
			for later retrieval by the handler.
		- *context - pointer to the blob that is added to the urb struct
			ure for later retrieval by the completion handler func.
		- interval - at which that this urb should be scheduled.

* bulk urbs
	`void usb_fill_bulk_urb(struct urb *urb, struct usb_device *dev,
				unsigned int pipe, void *transfer_buffer,
				int buffer_length, usb_complete_t complete,
				void *context);`

* control urbs
	`void usb_fill_control_urb(struct urb *urb, struct usb_device *dev,
				unsigned int pipe, unsigned char *setup_packet,
				void *transfer_buffer, int buffer_length,
				usb_complete_t complete, void *context);`

* isochronous urbs
```
urb->dev = dev;
urb->context = uvd;
urb->pipe = usb_rcvisocpipe(dev, uvd->video_endp-1);
urb->interval = 1;
urb->transfer_flags = URB_ISO_ASAP;
urb->transfer_buffer = cam->sts_buf[i];
urb->complete = konicawc_isoc_irq;
urb->number_of_packets = FRAMES_PER_DESC;
urb->transfer_buffer_length = FRAMES_PER_DESC;
for (j=0; j < FRAMES_PER_DESC; j++) {
	urb->iso_frame_desc[j].offset = j;
	urb->iso_frame_desc[j].length = 1;
}
```
# Submitting urbs

Once the urb has been properly created and initialized by the USB driver, it is
ready to be submitted to the USB core to be sent out to the USB device with func
		`int usb_submit_urb(struct urb *urb, int mem_flags);`
	You can use flags as follows:
		* GFP_ATOMIC - call is within a urb completion handler, interrupt
			a bottom half, a tasklet, or a timer callback. Or when the
			caller is holding spinlock or rwlock. Or when the current->state
			is not TASK_RUNNIG. The stat is always TASK_RUNNING, unless the
			driver has changed the current state itself.
		* GFP_NOIO - use this flag when the driver is in the block I/O patch. It
		should also be used in the error handling path of all storage type device
		* GFP_KERNEL - this should be used for all other situations that do not
			fall into one of the previously mentioned categories.

# Completing urbs:

If the call to usb_submit_urb() was successfulm xferring control of the urb to a
USB core, the function returns 0; otherwise, a negative erorr.In success, the co
mpletion handler (complete function pointer in initialization) is called exactly
once when the urb is completed. There are 3 ways with completion function call:
	* The urb is successfully sent to the device status variable is 0
	* some ind of error happened from the USB core. This happens either when
		the driver tells the USB core to cancel a submitted urb with a
		call to usb_unlink_urb or usb_kill_urb, or whena device is removed
	* Some kind of error happened when sending or receiving data from the device
		This is noted by the error value in the status valiable.

# Cancelling urbs

To stop irb that has been submitted to the USB core, should be called these func:
		`int usb_kill_urb(struct urb *urb);`
		`int usb_unlink_urb(struct urb *urb);`

