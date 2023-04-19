# Introduction

A good real-time embedded operating system avoids implementing the kernel as a
large, monolithic program. The kernel is developed  instead as a micro-kernel.
The micro-kernel provides core services, including task-related services,  the
scheduler service, and synchronization primitives.

These other common building blocks make up the additional kernel services that
are part of various embedded applications. The over building blocks include:

* TCP/IP protocol stack
* file system component
* remote procesdure call component
* command shell
* target debug agent
* other (miscellaneous)

# TCP/IP Protocol Stack

The TCP/IP protocol stack provides transport services to both higher and lower
layer protocols, including Simple Network Management Protocol (SNMP);  Network
File System (NFS) and Telnet, or user-defined protocols. TCP/IP protocol stack
can operate over various types of physical connections and networks, including
Ethernet, Frame Relay, ATM, and ISDN networks using different frame encapsulat
ion protocols, including point-to-point protocol.

`APPS -> Socket Interface -> TCP/IP proto stack -> Network Device -> HW I-FACE`

# File System Component

Provides efficient access to both local and network mass storage devices. These
storage devices include to CD-ROM, tapes, floppy-disks, hard disks, flashes etc

# Remote Procedure Call Component

The remote procedure call (RPC) component allows for distributed computing. The
RPC server offers services to external systems as remptely callable procedures.
To use a service provided by an RPC server, a client application calls routines
known as _stubs_ provided by the RPC client residing on the local machine.

# Command Shell

The _command shell_ also called command interpreter, is an interactive component
that provides an interface between the user and the real-time operating system.
The yser can invoke commands such as ping, ls, loader, and route through the sh
ell. THe shell interprets these commands and makes corresponding call into real
time operating systems routines. These routines can be in the form of  loadable
program images, dynamically created programs (dynamic tasks), or direct  system
function calls if supported by the RTOS. The programmer can experiment with dif
ferent global system calls if the command shell supports this feature.

# Target Debug Agent

Every good RTOS provides a target debug agent. The debug agent offers the progr
ammer to set up  both execution  and data access break points. In addition, the
programmer can use the debug agent to examine and modify system memory,  system
registers, and system objects, such as tasks, semaphores, and message queues.

# Component Configuration

The selection and consequently configuration of service components are accompli
shed through a set of system configuration files, which are RTOS-dependent.
The first level of configuration us done in a component inclusion  header file.
For example, call it sys_comp.h:
```
#define INCLUDE_TCPIP		1
#define INCLUDE_FILE_SYS	0
#define INCLUDE_SHELL		1
#define INCLUDE_DBG_AGENT	1
```
The second level of configuration is done in a component-specific configuration
file, sometimes called the component description file, for example, the TCP/IP
component configuration file contains user-configurable, component-specific op
erating parameters.
For example, set these parameters default values in net_conf.h config file:
```
#define NUM_PKT_BUFS		100
#define NUM_SOCKETS		20
#define NUM_ROUTES		35
#define NUM_NICS		40
```
Component-specific parameters must be passed to the component during the initia
lization phase. The component parameters are set into a data structure called
the component configuration table. The configuration table is passed into a com
ponent initialization routine. This level is the third configuration level. As
example, see the file named net_conf.c, whic continues the network component:
```
#include "sys_comp.h"
#include "net_conf.h"
#if (INCLUDE_TCPIP)
struct net_conf_parms params;
params.num_pkt_bufs = NUM_PKT_BUFS;
params.num_sockets = NUM_SOCKETS;
params.num_routes = NUM_ROUTES;
params.num_NICS = NUM_NICS;

tcpip_init(&params);

#endif
```
Note that the components are pre-built and archived. THe function tcpip)init()
is part of the component, If (INCLUDE_TCIP) is defined as 1 at the time the ap
plication is built, the call to this function triggers the linker to link the
cmponent into the final executable image. At this point, the TCP/IP protocol
stack is included and fully configured.

# Conclusion

* Micro-kernel design promotes a framework in which additional service comp-s
	can be developed to extend the kernel's functionalities easily.
* Debug agents allow programmers to debug every piece of code running on target
* Devs should choose a host debugger that understands different OS debug-agents
* Components can be included and configured through a set of system config files
* Devs should only include the necessary components to safeguard memory usage
