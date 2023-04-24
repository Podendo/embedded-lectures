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
-------------------------------------------------------------------------------
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
-------------------------------------------------------------------------------


























