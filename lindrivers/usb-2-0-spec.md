# USB 2.0, Introduction to specification

## USB Endpoints

A device endpoint is a uniqely addressable portion of a USB device that is the
source or sink of information in a communication flow between the host and dev.

Host always initiates a communication with a USB-device. After plug-in, the USB
Enumeration and Configuration procedure occurs, before other descriptor informa
tion, such as the endpoints descriptors are read by the host (later in the enum
eration process). During this enumeration sequence, a special set of endpoints
are used: _Control Endpoints_ or _Endpoint 0_. (defined as EP0 IN and EP0 OUT).

By nature, endpoints are bidirectional, but the are configured that they take on
a single direction (becoming unidirectional). Endpoints use CRCs to detect error
in transactions. The recepient of a transaction checks the CRC against the data,
if the two match, then the receiver issues an ACK. If not - no handshake is sent

The four types of endpoints and characteristics are:

* Control Endpoint - support control transfers, all devices must support. They
	send and receive information about device across the bus.

* Interrupt Endpoint - these endpoints support interrupt transfers. These xfers
	are used in devices that must use a high reliability method to communic
	ate a small amount of data. This is commonly used in HID designes. It is
	not a truly interrupt, but uses a polling rate. You get a guarantee that
	the host checks for data at a predictable interval. Its maximum packet
	size if a function of device speed. High-Speed supports 1024 bytes. Full
	speed support a maximum packet size of 64 bytes, Low-Speed - 8 bytes.

* Bulk Endpoints - Support bulk transfers, which are commonly used on devices
	that move relatively large amount of data at highly variable times where
	the transfers can use any available bandwidth space. They are the most
	common transfer type of USB devices. Delivery time with a bulk transfer
	is variable cause there is no set aside bandwidth of the transfer.

* Isochronous Endpoints - these ep support continious, real-time transfers that
	have a pre-negotiated bandwidth. Isochronous transfers must support str
	eams of error tolerant data, because they do not have an error recovery
	mechanism or handshaking. Errors are detected through the CRC field, but
	not corrected.

## Communication Protocol

### Frames and packets

Looking at the USB communication from a time perspective, it contains a series
of frames. each frame consists of a Start of Frame (SOF) followed by one or mo
re transactions. A packet is preceded with a sync pattern and ends with an End
of Packet (EOP) pattern. At a minimum, a transaction has a tocken packet.

As example:

time ---->
(FRAME)---(FRAME)---(FRAME)---(FRAME)---(FRAME)

(FRAME):
(SOF)--(TRANSACTION)--(TRANSACTION)--(TRANSACTION)

(TRANSACTION):
(SYNC)(TOCKET-PACKET)(EOP)-(SYNC)(DATA-PACKET)(EOP)-(SYNC)(HANDSHAKE-PACKET)(EOP)

Transactions are an exchange of packets and are comprised of three different pa-
ckets: a tocken packet, optional data packet, and a handshake packet. Transacti-
ons are placed within the frames and are never split across frames (with the ex-
ceptions for a hight-speed isochronous transfers) or interrupt the middle of ano
ther transaction.

Each packet can contain different pieces of information. What information is inc
luded depends on he packet type. A packet template is showed below:

* Packet id (PID) - (8 bits: 4 type bits and 4 error check bits), declare a
	transaction as an IN/OUT/SETUP/SOF

* Optional Device Address - 7 bits: Max of 127 devices.

* Optional Endpoint Address - 4 bits: Max of 16 endpoints. The USB specification
	supports up to 32 endpoints. While 4 bits gives a maximum value of 16,
	we achieve 32 endpoints with IN PID and an endpoint address between 1
	and 16 and an OUT PID with an endpoint address betwen 1 and 16, giving
	a total of 32. This is the endpoint address, not a number!

* Optional payload data - 0 ti 1023 bytes

* Optional CRC 5 or 16 bits

### Packet Types

* Token packets - used to direct traffic on the bus (IN/OUT/SETUP/SOF)
	* initiate transaction
	* Identify device involved in transaction
	* Always sourced by the host

* Data packets - follow IN/OUT/SETUP tokens, consist data toggle, DATA, CRC16
	* Delivers Payload Data
	* Sourced by host or device
The data toogle is updated at the host and the device for each successful data
packet transfer. One advantage to the data toggle is that it acts as an additi
onal error detection method.
(IN-ADDR-EP-CRC5-DATA0-DATA-CRC16-ACK)---(IN-ADDR-EP-CRC5-DATA1-DATA-CRC16-ACK)
(OUT-ADDR-EP-CRC5-DATA0-DATA-CRC16-ACK)-(OUT-ADDR-EP-CRC5-DATA1-DATA-CRC16-ACK)

* Handshake packets - conclude each transaction. includes 8-bit PID
	* Acknoledge error-free data receipt
	* Sourced by receiver of data
This handshake packet can be:
	- ACK - acknoledge succefull completion (LS/FS/HS)
	- NAK - negative acknoledgement (LS/FS/HS)
	- STALL - error indication sent by a device (LS/FS/HS)
	- NYET - indicates the device is not ready to receive another packet (HS)

* Special packets
	* Facilitates speed differentials
	* Sourced by host-to-hub devices
The USB specification defines four special packets:
	- PRE - issued to hubs by the host to indicate that the next is LowSpeed
	- SPLIT - preceds a tocken packet to indicate a split transaction (HS)
	- ERR - returned by a hub to report an error in a split transaction (HS)
	- PING - checks the status of the Bulk OUT or COntrol Write after recei-
		ving a NYET handshake (HighSpeed Only).

### Transaction Types

1. IN/Read/Upstream Transactions
	Transaction is sent from the device to the host. They are initiated by
		the host by sending an IN token packet. The targeted device re
		sponds by sending one or more data packets, and the host respo
		nds- with a handshake packet. (H TOKEN)(D DATA)(H HANDSHAKE).

	The device responds with NAKs to show that it is not ready to send data
		when the host makes request. The host continues to retry and the
		device responds with a data packet when it is ready. The host th
		en acknoledges the receipt of the data with an ACK handshake:

	(TOKEN(NAK))-(TOKEN(NAK))-(TOKEN(DATA-IN)(ACK))
	(TOKEN) is (IN-ADDR-EP-CRC5)
	(DATA-IN) is (DATA0-PAYLOAD DATA-CRC16)

2. OUT/Write/Downstream Transactions
	Transaction that occurs from the host to the device. In this type of tra
		nsaction, the host sends the appropriate token packet, SETUP or
		OUT, and follows with the one or more data packets. The device,
		after receiving, ends the transaction by sending the appropriate
		handshake packet: (HOST TOKEN)(HOST DATA)(DEVICE HANDSHAKE)

	(TOKEN)(DATA OUT)(NAK)---(TOKEN)(DATA OUT)(ACK)
	(TOKEN) is (OUT-ADDR-EP-CRC5)
	(DATA OUT) is (DATA0-PAYLOAD DATA-CRC16)

3. Control Transactions
	Transfers identify, configure, and control devices. The enable the host
		to read information about a device, set the device address, est
		ablish configuration, and issue certain commands. A control tra
		snfer is always directed to the control endpoint of a device.

	Control xfers have three stages: the setup, (optional) data, status. The
		setup stage(packet) is only used in a control transaction. This
		packet sends USB requests from the host to the device and requi
		res the data packet to contain an 8-byte USB request. The device
		must always acknoledge the setup stage, you cannot NAK a setup.
		Ex: (SETUP-ADDR-EP-CRC5)(DATA0-USB REQUEST DATA-CRC16)(ACK)

	The data stage is optional, this stage consists of multiple data transac
		tions and is only required when a data payload is to be transfer
		ed between the host & device. Frequently, relevant data for the
		control transfer can be transferred in the setup stage.
		Ex: (OUT-ADDR-EP-CRC5)(DATA0-PAYLOAD DATA-CRC16)(ACK)

	The status stage includes a single IN or OUT transaction that reports on
		the success of failure of the previous stages. The data packet
		is always DATA1 (unlike normal IN/OUT xfers toggling) and contai
		ns za zero length data packet. The status stage ends with a hand
		shake transaction that is sent by the rcvr of the preceding pack
		Ex: (TOKEN)(DATA IN)(ACK) (TOKEN)(DATA OUT)(ACK)

## USB Descriptors

When a device is connected to a USB host, the device gives information to the ho
st about its capabilities and power requirements. The device typically gives this
information to the host about its capabilities and power requirements. The devi-
ce typically gives this information through the a descriptor table that is part
of its firmware. A descriptor table is a structured sequence of values that desc
ribe the device; these values are defined by the developer. If a design conforms
to the requirement of a particular USB device class, additional descriptor infor
mation that the class must have is included the device desctiptor structure.

In USB descriptors, many parameters are 2 bytes long, so you need to be sure, th
at the low byte is first followed by the hight byte. Example of the desc tree:
-------------------------------------------------------------------------------
				Device Descriptor
					||
				-----------------
				|		|
			Configuration Desc    Configuration Desc
				|			|
			---------		-----------------
			|			|		|
		Interface Desc		Interface Desc	    Interface Desc
			|			|		   |
	Endpoint Desc----			--Endpoint Desc	   --Endpoint Desc
			|					   |
	Endpoint Desc----					   --Endpoint Desc
--------------------------------------------------------------------------------
### Device Descriptor

Device desc give the host information suck as the USB specification to which the
device conforms, the number of device configurations, and protocols supported by
the device, Vendor Identification (VID), Product Identification (PrID), a serial
number if the device has one. The device descriptor is where some of the most cr
utial information about the USB device is contained.

```
Offset	Field		Size(bytes)		Description
0	bLength			1	Length of this descriptor = 18 bytes
1	bDescriptorType		1	Descriptor type = DEVICE (01h)
2	bcdUSB			2	USB specification version (BCD)
4	bDeviceClass		1	Device Class
5	bDeviceSubClass		1	Device subclass
6	bDeviceProtocol		1	Device protocol
7	bMaxPacketSize0		1	Max Packet Size for EndPoint 0
8	idVendor		2	Vendor ID (or VID, assigned by USB-IF)
10	idProduct		2	Product ID (or PrID, assigned by manuf)
12	bcdDevice		2	Device release number
14	iManufacturer		1	index of manufacturer string
15	iProduct		1	Index of product string
16	iSerialNumber		1	Index of serial number string
17	bNumConfigurations	1	Number of configs supported
```

- bcdUSB reports the USB revision that the device supports, which should be the
	latest supported revision. This is a BCD value, uses a format 0xAABC, wh
	ere A is the major version, B is the minor version, and C is sub-minor.
	For example, USB 2.0 device would have a value of 0x0200, when USB 1.1
	would have a 0x0110 value. This is normally used by the host in determi-
	ning which driver to load.

- bDeviceClass, bDeviceSubClass, bDeviceProtocol used by the operation system to
	identify a driver for a USB device during the enumeration process. Fill-
	ing this filed in the device descriptor prevents different interfaces fr
	om functioning independently, suck as a composite device. Most USB devi-
	ces define their classes in the interface descriptor and leave those fi-
	elds as 0x00.

- bMaxPacketSize reports the max number of packets supported by endpoint 0. depe
	nding on the device, the possible sizes are 8 bytes, 32 bytes, 64 bytes.

- iManufacturer, iProduct, iSerialNumber are indexes to string descriptors. Stri
	ng descriptors give details about the followed, if no string exists, the
	respective field should be assigned a value of zero.

- bNumConfiguration - defines the total number of configurations the device can
	support. Multiple configurations allow the device to be configured diffe
	rently depending on certain conditions suck as being bus powered or self
	powered, as example.

### Configuration Descriptor

```
Offset	Field			Size(Bytes)		Description
0	bLength				1	Length of this descriptor 9bytes
1	bDescriptorType			1	Descriptor type = CONFIG (02h)
2	wTotalLength			2	Total length (i-face, endpoint desc)
4	bNumInterfaces			1	Number of interfaces in this cfg
5	bConfigurationValue		1	`SET_CONFIGURATION` - cfg value
6	iConfiguration			1	Index of string that describes cfg
7	bmAttributes			1	bit 7: resrv,1, 6: selfpwr, 5: remote wkup
8	bMaxPower			1	maximum power required for this cfg, x 2mA
```

### Interface Assosiation Descriptor (IAD)

This descriptor describes 2 or more ifaces that are assosiated with a single dev
ice function. The interface association descriptor (IAD) informs the host that
the interfaces are linked together. For example, a USB UART has two interfaces
assosiated with it: a control interface and a data interface. The IAD tells the
host that these two interfaces are part of the same function, which is a USBUART
and falls under the communication device class (CDC). This descriptor is not req
ired in all cases.

```
Offset	Field		Size(bytes)		Description
0	bLength			1	Descriptor size in bytes
1	bDescriptionType	1	Descriptor type - INTERFACE ASSOCIATION(0Bh)
2	bFirstInterface		1	Number identifying the 1st assosiate iface
3	bInterfaceCount		1	The number of contiguous asossiated ifaces
4	bFunctionClass		1	Class code
5	bFunctionSubClass	1	Subclass code
6	bFunctionProtocol	1	Protocol code
7	iFunction		1	Index of string description for the func
```

### Interface Descriptor

Describes a specific interface within a configuration. The number of endpoints
for an interface is identified in this descriptor. The interface descriptor is
also where the USB Class of the device is declared. A USB device class identi-
fies the device functionality and aids in the loading of a proper driver for
that specific functionality.

```
Offset	Field		Size(Bytes)		Description
0	bLength			1	Length of this descriptor = 9 bytes
1	bDescriptorType		1	Descriptor type = INTERFACE (04h)
2	bInterfaceNumber	1	Zero based index of this interface
3	bAlternateSetting	1	Alternate setting value
4	bNumEndpoints		1	Number of endpoints used by this iface
5	bInterfaceClass		1	Interface Class
6	bInterfaceSubClass	1	Interface Subclass
7	bInterfaceProtocol	1	Interface Protocol
8	Interface		1	Index to string describing this iface
```

### Endpoint Descriptor

Each endpoint used in a device has its own descriptor. This descriptor gives the
endpoint information that the host must have. This info includes direction of EP
transfer type and maximum packet size:

```
offset	Field		Size(Bytes)		Description
0	bLength			1	Length of this descriptor = 7 bytes
1	bDescriptorType		1	Descriptor type = ENDPOINT (05h)
2	bEndpointAddress	1	Bit 3..0 the endpoint number
					bit 6..4 Reserved, reset to zero
					bit 7 direction, ignored for control
						0 - OUT endpoint
						1 - IN endpoint
3	bmAttributes		1	bits 1..0 Transfer type
						00 Control
						01 Isochronous
						10 Bulk
						11 Interrupt
					If not asynchronous epm bits 5..2 are re
					zerved and must be set to zero, if iso:
					bits 3..2 - synchronization type
						00 - no sync
						01 - async
						10 - adaptive
						11 - sync
					bits 5..4 usage type
						00 - data endpoint
						01 - feedback endpoint
						10 - implicit feedback data ep
						11 - reserved
4	wMaxPacketSize		2	maximum packet size for this endpoint
6	bInterval		1	Polling interval in milliseconds for int
					errupt endpoints (1 for isochronousm ign
					ored for control or bulk endpoints)
```
### String Descriptor

Optional descriptor, gives user-readable information about the device. Possible
information contained in the descriptor is the name of the device, manufacturer,
the serial number, or names for the various interfaces or configurations. If str
ings are not used in a device, any string index field of the descriptors mention
ed earlier must be set to 00h.

```
offset	Field			Size(bytes)		Description
0	bLength				1	Length of this descriptor 7 bytes
1	bDescriptorType			1	Descriptor type = STRING (03h)
2..n	bString-or-wLangID		var	Unicode encoded text, LANGID code
```

### Other Misc Descriptor Types:
	- Report Descriptors (file with extended device attributes description)
	- MS OS Descriptors (used for vendor-specific devices for MS Windows)
	- Device-Qualifier Descriptor (more info about High-Speed capable device)
	- BOS Descriptor (Binary-Device-Object-Store (BOS) for linkPowerManagement)

### Using multiple USB descriptors

USB devices have only one device descriptor. However, a device can have multiple
configurations, interfaces, endpoints, and string descriptors. Also, a device
can have multiple interfaces thus multiple interface descriptors. A USB device
with multiple interfaces that perform different functions is called a composite
device. An example is a USB audio head set(USB device with two ifaces: audio and
volume adjustment). Multiple ifaces can be active at the same time.
```
USB DEVICE ---> CONFIGURATION --> (Interface 0 (Audio Class (ISOCHR EP-1)))
			      --> (Interface 1 (HID Class (INTERRUPT EP-2)))
```
Or, each interface can have multiple configurations. These multiple configurati-
ons are called alternate settings. You can allow the ability to alter endpoint
configuration in a device to reserve different amounts of bandwidth. As example,
in one alternate setting, a device can have its endpoints configured to bulk, in
another alternate setting, it can have its endpoints configured to isochronous:
```
USB DEVICE ---> CONFIGURATION 1 -> Interface 0, bAtlernateSetting0 (EP1,EP2 Bulk)
				-> Interface 0, bAlternateSetting1 (EP1, EP2 ISO)
```

### USB Class Devices

* Human Interface Device HID
* Mass Storage Device (MSD)
* Communication Device Class (CDC)
* Vendor (Vendor Specific)

```
Class	 Usage		Description			Examples
00h	Device		Uspecified		No device class, info in device dsrc
01h	Interface	Audio			Speaker, Microphone, sound card, MIDI
02h	Both		comm, CDC control	Modem, ethernet adapter, Wi-Fi adapter
03h	Interface	human iface device HID	Keyboard, mouse, joystick
05h	Interface	phy iface device PID	Fource feedback joystick
06h	Interface	image			Camera, scanner
07h	Interface	printer			Printers, CNC machines
08h	Interface	mass storage		External hard drivers, memory cards
09h	Device		usb hub			USB hubs
0Ah	Interface	cdc-data		Used in conjuction with 02h class
0Bh	Interface	smartcard		USB smart card reader
0Dh	Interface	content security	Fingerprint reader
0Eh	Interface	video			Webcam
0Fh	Interface	personal healthcare	Heart-Rate monitor, glucose meter
DCh	Both		diagnostic device	USB compliance testing device
E0h	Interface	wireless controller	BlueTooth adapter
EFh	Both		miscellaneous		ActiveSync Device
FEh	Interface	application specific	IrDA Bridge, Test & Measure, DFU
FFh	Both		vendor specific		Indicates a device needs vendor drivers

```
## USB Enumeration and Configuration

* Dynamic Detection
	* The device is connected to a USB port and detected, powered state
	* The hub detects the device by monitoring voltages on the port

* Enumeration
	* The host learnes of the newly attached device by using an interrupt
		endpoint to get a report about the hub's status. This include
		changes in port status. After the hub tells the host about the
		device detection, the host issues a request to the hub to learn
		more details about the status change that occured using the req
		uest GET_PORT_STATUS.

	* After the host gathers this information, it detect the speed of the
		device (USB Speeds). This info's reported by next GET_PORT_STATUS.

	* The host issues a SET_PORT_FEATURE request to the hub asking it to reset
		the newly attached device. This reset state is held for 10 ms.

	* During the reset, a serues if J-state and K-state occurs to determine
		if the device supports high-speed. This step is skipped on LS FS

	* The host checks to see if the device is still in a reset state by issu
		ing the GET_PORT_STATUS request, after device leaves the reset s
		tate, it is in the default state, ready for responding to host in
		form of control transfers to its default address of 00h.

	* The host begins the process of learning more information about the dev
		ice. It starts by learning the maximum packet size of the default
		pipe (endpoint 0). The host starts issuing a GET_DESCRIPTOR requ
		est to the device.

	* The host applies an address to the device with the SET_ADDRESS request
		The device completes the status stage of this request using the
		default address 00h, befire using the newly assigned address.
		All communication beyond this point will use the new address.
		The address may change if the device is detached from a port, the
		port is reset or the PC reboots. THe device is in the ADDR state.

* Configuration
	* After the device returns from its reset, the host issues a command nam
		ed GET_DESCRIPTOR, using the newly assigned address, to read the
		descriptors from the device. However, this time all the descript
		ors are read. Next GET_DESCRIPTOR asking for the configuration
		descriptor. (plus interface and endpoints descriptors).

	* For the host PC to successfully use the device, the most must load a
		device driver. The host searches for a driver to manage communi
		cation between itself and the device.

	* After all descriptors are received, the hst sets a specific device con
		figuration using the SET_CONFIGURATION request. Most devices ha-
		ve only one configuration. Devices that support multiple configu
		rations can allow the user or the driver to select the proper co
		nfiguration.

	* The device is now in the configured state. It tooks on its properties
		that were defined with the descriptors. The defined maximum pow
		er can be drawn from Vbus and the device is now ready for use in
		an Application.
