# Introduction to embedded systems

- An embedded system is build for a specific application. As such, the hardware
and software compontents are hightly integrated, and development model us the
hardware and software co-design model.

- Embedded systems are generally build using embedded processors
	- an embedded processor if specialized processor, such as DSP. that is
	cheaper to design and produce, can have buld-in integrated devices, is
	limited in functionality etc.

- Real-time systems are characterized by the fact that timing correctness is
just as important as functional or logical correctness (hard-soft RTOS)

- The severity of penalty incurred for not satisfying timing constraints diffs
hard real-time systems from soft real-time systems.

- real-time systems gave a sighnificant amount of application awareness similar
to embedded ststems.

- real-time embedded systems are thise embedded system with real-time behaviors.


# Embedded systems development

## Overview of Linkers and the linking Process

Compilation process with gcc:
1. Preprocessing (gcc -E > preprocessor (including etc. out.i))
2. Compilation	(gcc -S myprogram.c - out.s)
2. Assembing	 (gcc myprogram.c -o out.o )
3. Linking	 (linker.ld - adding static library)

The compile creates a symbol table containing the sybmol name to address mapping
as a part of the object fule it produces. When creating relocatable output, the
compiler  generates  the address that. for each symbol, is relative to the file
being compiled. These addrs are  generated with respect to offset 0. The symbol
The symbol table contains the global symbols defined in the file being compiled
as well as the external symbols referenced in the file that the linker needs to
resolve. The linking process performed by the linker involves symbol resolution
and symbol relocation.

___symbol resolution___ is the process in which the linker goes through each obj
file in which (other) object fle or files the external symbols are defined. The
linker, sometimes, must process the list of object file multiple times trying to
resolve all of the external symbols. When the symbols are defined in a _static
library_, the linker copies the object fules from the library and writes  them
into the final image.

___symbol relocation___ is the process in which the linker maps a symbol refe-
rence to its definition. The linker modifies the machine  code of the  linked
object files so that code references to the symbols reflect the actual address
assigned to these symbols.

### section types of the ELF format

* NULL		- inactive header without a section
* PROGBITS	- code or initialized data
* SYMTAB		- symbol table for static linking
* STRTAB		- string table
* RELA/REL	- relocation entries
* HASH		- run-time symbol hash-table
* DYNAMIC		- information used for dynamic linking
* NOBITS		- unitialized data
* DYNSYM		- symbol table for dynamic linking
* WRITE		- section contains writeable data
* ALLOC		- section contains allocated data
* EXECINSTR	- section contains executable instructions

For PROGBITS header has common system-created sections with pre-defined name:

* .text		- read-only section, contain program code and constant data
* .monitor	- write attribute, contains the monitor conde
* .sdata		- write attribute, initialized data less that 64 KB
* .data		- write attribute, initialized data larger that 64 KB
* .sbss		- small unitialized data
* .bss		- common unitialized data

_example: possible section allocation_

```
MEMORY {
 ROM: origin = 0x00000h, length = 0x000100h
 FLASH: origin = 0x00110h, length = 0x004000h
 RAMB0: origin = 0x05000h, length = 0x020000h
 RAMB1: origin = 0x25000h, length = 0x200000h
}
SECTION {
 .rodata : > ROM
 _loader : > FLASH
 _wflash : > FLASH
 _monitor : > RAMB0
 .sbss (ALIGN 4) : > RAMB0
 .sdata (ALIGN 4) : > RAMB0
 .text : > RAMB1
 .bss (ALIGN 4) : > RAMB1
 .data (ALIGN 4) : > RAMB1
}
```

## conclusion:

1. The linker performs symbol resolution and symbol relocation
2. You need to understand the exact memory layout of the target system (SoC)
3. An executable target image is comprised of multiple program sections
4. Each program section can reside in different types of physical memory

# Embedded system initialization

## Image transfer

An executable image built for a target embedded system can be stransferred from
the host development system onto the target, which is called _loading the image_

* programming the entire image into the EEPROM or flash memory
* downloading the image over either a serial or network connection
	* this proccess requires the presence of data transfer untility programs
	* also it requires the presense of a target loader, an embedded monitor
* downloading the image through either a JTAG or BDM interface

## Embedded loader

The loader is a small memory footprint, so it typically can be programmed into a
ROM chip. A  data transfer utility resides on the host  system side. The  loader
works in conjuction with its host ulitily counterpart to perform the image xfer.

## Embedded monitor

An alternative to the boot image plus loader approach is to use an embedded mon-
itor. A _monitor_ is an embedded software application commonly provided by the
target system manufacturer for its evaluation boards. The monitor enables devs
to examine and debug the target system at runtime. Similar to the boot image,
the monitor is executed on power-up and performs system initialization such as:

* initializing the required peripheral devices
* initializing the memory system for downloading the image
* initializing the interrupt controller and installing default ISR handlers

Also, monitor provide access through the serial interface, defines a set of com:

* download the image
* read from and write to system memory locations
* read and write system registers
* set and clear different types of breakpoints
* single-step instructions
* reset the system

## Target Boot Scenarios

Embedded processors, after the are powered on, fetch and execute code from a pre-
defined and hard-wired address offset. The code contained at this memory location
is called the the ___reset vector___. The reset vector is usually a jump assebly
instruction into another part of the memory space where the read initialization
code is found. The reason for jumping to  another part of memory is to keep the
reset vector small. The setet vector, as well as  the system boot startup  code,
must be in permanent storage. Because  of this, the system startup code,  called
the ___bootstrap code___, resides in the system ROM, the on-board flash-memory,
or other types of non-volatile memory devices.
So, The loader refers to the code that performs system bootstrapping, image down
loading, peripheral and SoC system basic initialization.

### Executing from ROM using RAM or Data

The boot sequence for an image running from ROM is as follows:

1. The CPU`s IP (instruction pointer) is wired to execute the reset vector - ROM
2. The *reset vector* jumps to the first instruction of the .text section of the
boot image. The .text section remains in ROM. The CPU uses IP to execute .text.
The code initializes the memory system, including the RAM.
3. The .data section of the boot image is copied into RAM because it is both R/W
4. Space is reserved in RAM for the .bss section of the boot image beacuse it is
both readable and writable. There is nothing to transfer because of the content
for the .bss section is empty.
5. Stack space is reserved in RAM.
6. The CPU IP register is set to point to the beginning of the newly created stack.
At this point, the boot completes. The CPU continues to execute the code in the
.text section until it is complete or until the system is shut down.


### Executing from RAM after Image Transfer from ROM

The first six steps are identical to the previous boot scenario. After completing
those steps, the process continues as follows:

7. The compressed application image is copied from ROM to RAM
8. Initialization steps that are part of decomplession procedure are completed.
9. The loader transfers control to the image. This is done by jumping to the
beginning address of the initialized image using a processor-specific jump in-
struction. This JMP instruction effectively sets a new value into the IP.
10. The memory area that the loader program occupies is recycled. The stack ptr
is reinitialized to point this area, so it can be used as the stack for the new
program. The decompression work area is also recycled into the available memory.

The loader program is still available for use because it is stored in the ROM.

### Executing from RAM after Image Transfer from Host ( late debugging)

The debug agent downloads the image into a temporary area in RAM first. After the
download is complete and the image integrity verifed, the debug agent initializes
the image according to the inforamtion presented in the program section header
table. The frist six steps of booting process are identical to the initial boot
scenario. After completing those steps, the process continues as follows:

7. The application image is downloaded from the host development system
8. The image integrity is verified.
9. The image is decompressed if nessesary
10. The debug agent loads the image sections into their respective run RAM addrs.
11. The debug agent transfers control to the download image.

## Target system software initialization sequence

The embedded SW components include the following:

* The board support package (BSP) - contains the full spectrum of drivers for
the system hardware components and devices.
* The RTOS, which provides basic services, such as:
	* resource syncronization services
	* I/O services
	* scheduling services needed by the embedded applications
	* additional components, such as filesystem services and network services
* The protocol stacks (i.e TCP/IP)
* Other embedded components and modules
* Application layer of embedded system

The steps required to initialize most target systems, the main stages:

1. Hardware initialization
2. RTOS initialization
3. application initialization

### Hardware Initialization

CPU begins executing instructions from the reset vector. Typically at this stage,
the minimum hardware initialization required to get the boot image to execute is
performed, which includes:

1. starting execution at the reset vector
2. putting the processor into a known state by setting the appropriate registers
	* getting the processor type
	* getting or setting the CPU`s clocks
3. disabling interrupts and caches
4. initializatiing memory controllers, memory chips, and cache units
	* getting the start address for memory
	* getting the size of memory
	* performing preliminary memory tests, if required

After the boot sequence initialized the CPU and memory, the boot sequence copies
and decompresses, if mecessary, the sections of code that need to run. It also
copies and decompresses its data into RAM.
Most of the early initialization code is low-level assembly language that is
specific to the target system`s CPU architecture. Later-stage initialization
code might be written in a higher-level language, such as C.

As the boot code executes, the code calls the appropriate functions to initialize
other hardware components, if present, on the target system, as follows:

* setting up execution handlers
* initializing interrupt handlers
* initializing bus interfaces, such as VME, PCI, USB and etc.
* initializing board peripherals such as serial, LAN, and SCSI

### RTOS Initialization

A Real Time Operation System`s software initialization includes the following:

1. Initializing the RTOS
2. Initializing different RTOS objects and services:
	* task objects
	* semaphore objects
	* message-queue objects
	* timer services
	* interrupt services
	* memory-management services
3. Creating necessary stacks for RTOS
4. Initializing additional RTOS extensions:
	* TCP/IP stack
	* file systems
5. Starting the RTOS and its initial tasks

### Application software initialization

After the RTOS is initialized and running with the required components, control
is transferred to a user-defined application. This transfer takes when the RTOS
code calls a predefined function ( that is RTOS dependent) which is implemented
by the user-defined application. At this point, the RTOS services are available.

## On-Chip Debugging

Many vendors recognize the need for built-in-microprocessor debugging, called -on
chip debugging (OCD). BDM and JTAG are two types of OCD solutions that allow devs
a direct access and control over the MPU and system resources w/o needing software
debug agents on the target.

* JTAG - Joint Test Action Group IEEE1149.1
* BDM - background debug mode, microprocessor debug interface (Motorola chips)

## conclusion

* You can use target-monitor-based, debug-agent-based, hardware-assisted imaging
* the boot ROM can contain boot image, loader image, monitor image, debug agent
or even executable images.
* HW-assisted connections are ideal, both when first initializing a phy target
system as well as later, for programming the final executable image into ROM
* Some ways to boot a target include running an image out of ROM, out from RAM
after copying it fro ROM, and running an image out of RAM after downloading it
from a host.
* A system typically undergoes three distinct initialization stages:
	* hardware initialization
	* OS initialization (RTOS)
	* Application initialization
* After the target system is initialized, application developers can use this
platform to download, test, and debug applications that use an underlying RTOS
