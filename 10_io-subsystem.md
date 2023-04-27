# I/O subsystem introduction

All embedded systems include some form of input and output (I/O) operations. The
se I/O operations are performed over different types of I/O devices. I/O operati
ons are interpreted differently depending on the viewpoint taken and place diffe
rent requirements on the level of understanding of the hardware details.

From the perspective of a system software developer, I/O operations imply commun
icating with the device, programming the device to initiate an I/O request, perf
orming actual data transfer between the device and the system, and notifying the
requestor when the operation completes.

From the perspective of the RTOS, I/O operations imply locating the right device
for the I/O request, locating the right device driver for the device, and issuing
the request to the device driver. Sometimes the RTOS is required to ensure synch
ronized access to the device. The RTOS must facilictate an abstraction that hide
s both the device characteristics and specifics from the application developers.

# Basic I/O Concepts

The combination of I/O devices, assosiated device drivers, and the I/O subsystem
comprises the overall I/O system in an embedded environment. The puprose of the
I/O subsystem is to hide the device-specific information from the kernel as well
as from the application developer and to provide a uniform access method to the
peripheral I/O devices of the system.

In the layered operating system model:

1. Application Software (Generic)
2. I/O Subsystem (Less Generic)
3. Device drivers & Interrupt Handlers (Specific details)
4. I/O Device Hardware (More Specific Details)

## Port-Mapped vs Memory-Mapped I/O and DMA

The bottom layer contains the I/O device hardware. The I/O device hardware can
range from low-bit rate serial lines to hard drives and gigabit network interfa-
ce adaptors. All I/O devices must be initialized through device control register
which are usually external to the CPU. They are located on the CPU board or in
the devices themselves. During operation, the device registers are accessed aga-
in and are programmed to process data transfer requests, which is called the dev
ice control. To access these devices, it is necessary for the developer to deter
mine of the device is port mapped or memory mapped. This information deterimes
which of two methods, port0,apped I/O or memory-mapped I/O is deployed to access
an I/O device.

For _port-mapped I/O_ the I/O device address is reffered to as the port number
when specified for these special instructions. So In the system address space
we have a part for I/O address space, where we have a range of addressed used
for different peripherals (LCD, Serial Line, etc).

Each device is on a different I/O port. They are accesed through special proces-
sor instructions, and actual physical access is accomplished through special har
dware circuitry. This I/O method is also called _isolated I/O_ because the memo-
ry space is isolated from the I/O space, thus the entire memory address space is
available for application use.

For _memory-mapped I/O_ the device address is part of the system memory address
space, any machine instruction that is encoded to transfer data between a memory
location and the processor or between two memory locations can potentially be us
ed to access the I/O device. Because tje I/O address space occupies a range in
the system memory address space, this region of the memory address space is not
available for an application to use.

So we have a system address space, part of it is used for I/O address space, whe
re we have ranges for our peripherals (in example, LCD, Serial Line, etc). The
memory-mapped I/O space does not necessarily begin at offset 0 in the system add
ress space: it can be mapped anywhere inside the address space. It depends on
implementation.

The CPU has to do some work in botj of these I/O methods. Data transfer between
the device and the system involves transferring data between the device and the
processor register and then from the processor register to memory.

The transfer speed might not meet the needs of high-speed I/O devices because of
the additional data copy involved. Direct memory access (DMA) chips or controll-
ers solve this problem by allowing the device to access the memory directly with
out involving the processor. The processor is used to set up the DMA controller
before a data transfer operation begins, but the processor is bypassed during da
ta transfer, regardless of whether it is a read or write operation. The transfer
speed depends on the transfer speed of the I/O device, the speed of memory devi-
ce and te speed of the DMA controller.

In essence, the DMA controller provides an alternative data path between the I/O
device and the main memory. The processor sets up the transfer operation by spec
ifying the source address, the destination memory address, and the length of the
transfer to the DMA controller.

## Character Mode vs Block Mode Devices

I/O devices are classified as either character-mode devices or block-mode device
The classification refrens to how the device handles data transfers with the sys
tem. Character mode devices allow for unstructured data transfers. The data tran
sfers typically take place in serial fashion, one byte at time. Character-mode
devices are usuallu simple devices, such as the serial interface or the keypad.
The driver buffers the data in cases where the transfer rate from system to the
device is faster that what the device can handle.

Block mode devices transfer data one block at time, for example, 1024 bytes per
one data transfer. The underlying hardware imposes the block size. Some structu-
re must be imposed on the data or some transfer protocol enforced. Otherwise an
error is likely to occur. Therefore, sometimes it is necessary for the block-mode
device driver to perform additional work for each read or write operation. Thus,
the block device driver must first divide the input data into multiple blocks,
each with a device-specific block size. In practice, the partition often is smal
ler that the normal device block size.

Each block is transferred to the device in separate write requests. The block de
vice driver must handle the last block differently from the first three because
the last block has a different size.

# The I/O Subsystem

Each I/O device driver can provide a driver-specific set of I/O application prog
ramming interfaces to the applications. This arrangement requires each applicati
on to be aware of the nature of the underlying I/O device, including the restric
tions imposed by the device. The API set is driver and implementaion specific,
whicj makes the applications using the API set difficult to port. To reduce this
implementation-dependence, embedded systems often include an _I/O subsystem_

The _I/O subsystem_ defines a standart set of functions for I/O operations in or
der to hide device peculiarities from applications. All I/O device drivers confo
rm to and support this function set because the goal us to provide uniform I/O to
applications accross a wide spectrum of I/O devices of varying types.

1. The I/O subsystem defines the API set

2. The device driver implements each function in the set

3. The device driver exports the set of functions to the I/O subsystem

4. The device driver does the work necessary to prepare the device for use. In
	addition, the driver sets up an assosiation between the I/O subsystem API
	set and the corresponding device-specific I/O calls.

5. The device driver loads the device and makes this driver and device associati
	on known to the I/O subsystem. This action enables the I/O subsystem to
	present the illusion of an abstract or virtual instance of the device to
	applications.

## Standart I/O Functions

- `Create()` - creates a virtual instance of an I/O device
	preparations must might include mapping the device into the system memo-
	ry space, allocating an available interrupt request line (IRQ) for device
	installing an ISR for the IRQ, and initializing the device into a known
	state. The driver allocates memory to store instance-specific info for
	subsequent operations. A reference to created device instance is a return

- `Destroy()` - deletes a virtual instance of an I/O device
	No more operations are allowed on the device after this function completes
	This function gives the driver an opportunity to perform a cleanup. The
	driver frees the memory that was used to store instance-specific info.

- `Open()` - Prepares an I/O device for use
	Therefore, one of the operations that the open function might perform is
	enabling the device. Typically, Open() function can also specify mode of
	use, allocationg temporary buffers, enabling some functions, etc.

- `Close()` - Communicates to the device that its services are no longer required
	which typically initiates device-specific cleanup operations. Commonly,
	the I/O subsystem supplies only one of the two functions, destroy and cl
	ose, which implements most of the functionalities of both, in the case
	where one function implements both the create and open operations.

- `Read()` - reads data from an I/O device
	Retrieves data from a previously opened I/O device, the caller is comple
	tely isolated from the device and the location in memory where the data
	is to be stored. The caller imposed by the device.

- `Write()` - writes data into an I/O device
	transfers data from the application to a previously opened I/O device.
	The caller specifies the amount of data to xfer, and the location of
	memory holding the data to be transfered.

`Ioctl()` - Issues control commands to the I/O device (I/O control)
	it is used to manipulate the device and driver operationg parameters at
	runtime. Also it can be used to do a device-specific functions (as event)

"Virtual Instance" means that these functions do not act directly on the I/O dev
ice, but rather on the driver, which passes the operations to the I/O device.
When the open, read, write, close operations are described, these operations
should be understood as acting indirectly on an I/O device through the agency
of a virtual instance.

## Mapping generic Functions to Driver Functions

The individual device drivers provide the actual implementation of each function
in the uniform I/O API set, as shown in example below:

(Application)->(I/O subsystem){create, open, read, write, cose, ioctl, destroy}
		->(Device Driver){d_open, d_read, d_write, d_close}->(Device)

The I/O subsystem-defined API set needs to be mapped into a function set that is
specific to the device driver for any driver that supports uniform I/O. The func
tions that begin with the _d_ prefix refer for implementations that are specific
to a device driver. The uniform I/O API set can be represented as a structure of
function pointers in the C programming language syntax:
```
typedef struct
{
	int (*Create)( );
	int (*Open) ( );
	int (*Read)( );
	int (*Write) ( );
	int (*Close) ( );
	int (*Ioctl) ( );
	int (*Destroy) ( );
} UNIFORM_IO_DRV;

```
The mapping process involves initilizing each function pointer with the address
of an assosiated internal driver function, as shown below:
```
UNIFORM_IO_DRV ttyIOdrv;
ttyIOdrv.Create = tty_Create;
ttyIOdrv.Open = tty_Open;
ttyIOdrv.Read = tty_Read;
ttyIOdrv.Write = tty_Write;
ttyIOdrv.Close = tty_Close;
ttyIOdrv.Ioctl = tty_Ioctl;
ttyIOdrv.Destroy = tty_Destroy;
```
An I/O subsystem usually maintains a _uniform I/O driver table_. Any driver can
be installed into or removed from this driver table by using the utility functi-
ons that the I/O subsystem provides. Each row in the table represents a unique
I/O driver that supports the defined APi set. Then the columns of the table are
a generic names used to assosiate the uniform I/O driver with a particular type
of device. Thus, every [row][column] is the matched driver-specific I/O function

These pointers are written t the table when a driver is installed in the I/O sub
system, typically by calling a utility function for driver installation. When th
is utility function is called, a reference to the newly created driver table ent
ry us returned to the caller.

## Assosiating Devices with Device Drivers

The create function is used to create a virtual instance of a device, as mention
ed earlier. The I/O subsystem tracks these virtual instances, using the _device
table_. A newly created virtual instance is given a unique name and is inserted
into the device table.

Each entry in the device table holds generic information, as well as instance-sp
ecific information. The generic part of the device entry can include the unique
name of the device instance and a reference to the device driver. the driver-dep
endent part of the device entry is a block of memory allocated by the driver for
each instance to hold instance(device)-specific data. The driver initializes and
maintains it. The content of this information is dependent on the driver impleme
ntation. The driver is the only entity that accesses and interprets this data. A
reference to the newly created device entry is returned to the caller of the cre
ate function. Sibsequent calls to the open and destroy function use this referen
ce.

# Conclusion

* Interfaces between a device and the main processor occur in two ways: port map
	ped and memory mapped.

* DMA controllers allows data transfer bypassing the main processor.

* I/O subsystem must be flexible enough to handle a wide range of I/O devices

* Uniform I/O hides device peculiarities from applications

* The I/O subsystem maintains a driver table that assosiates uniform I/O calls
	with a driver-specific I/O routines.

* The I/O subsystem maintains a device table forms an association between this
	table and the driver table.
