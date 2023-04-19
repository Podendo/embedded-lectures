# Exceptions and Interrupts

An axception is any event that disrupts the normal execution of the processor
and forces the processor into execution of special instructions in a privileged
state. Exceptions can be classified into two categories:

* synchronous exceptions (internal events)
	* allignment exception (read/write that begin at an odd memory addr)
	* an arithmetic operation that results in a division by zero

* asynchronous exceptions, raised by external events, which are events that do
not relate to the execution of processor instructions. In general, these exter
nal events are assosiated with hardware signals
	* Pushing the reset button on the board triggers an async exception
	* the communication processor module that raise async exception when
		it receives data packets.

_An Interrupt_ or an _external interrupt_, is an asynchronous exception trigger
ed by an event that an external hardware device generates. Interrupts are one
class exception. The event source for an async exception is an external hw dev.

# Applications of Exceptions and Interrupts

In general, exceptions and interrupts come in handy in three areas:

* **Internal errors and special conditions management**
	Exceptions are either error conditions or special conditions that the
processor detects while executing instructions. Many processor architectures ha-
ve two modes of execution: normal and privileged. Some instructions, called pri-
vileged instructions, are allowed to execut only when the processor is in the pr
ivileged execution mode. An exception is raised when a privileged instruction is
issued while the processor is in normal execution mode.

* **Hardware concurrency**
	Many external hardware devices can perform device-specific operations in
parallel to the core processor. These device require minimum intervention from
the core processor.

* **Service requests management**
	Another use of external interrupts is to provide a communication mecha-
nusm to signal or alert an embedded processor that an external hardware device
is requesting service. For example, an initialized programmable interval timer
chip communicates with the embedded processor through an interrupt when a pre-
programmed time interval has expired. Or, the network interface device uses an
interrupt to indicate the arrival of packets after the received packets have be
en stored into memory.

# A Closer Look at Exceptions and Interrupts

## Programmable Interrupt Controllers and External Interrupts

The PIC is implementation-dependent. It can appear in a variety of forms and it
sometimes given different names, however, all serve the same purpose and provide
* Prioritizing multiple interrupt sources so that at any time the highest prio-
rity interrupt is presented to the core CPU for processing.
* Offloading the core CPU with the processing required to determine an interrupt
exact source.

The PIC has a set of interrupt request lines. An external source generates inter
rupts by asserting a physical signal on the interrupt request line. Each interr-
upt request line has a priority assigned to it.

The interrupt table lists all available interrupts in the embedded system. In ad
dition, several other properties help define the dynamic characteristics of the
interrupt source.

This priorities and tables help to understand the concept of _nested interrupts_.
The term refers to the ability of a higher priority interrupt source to preempt
the processing of a lower priority interrupt. The vector address column speci-
fies where in memory the ISR must be installed. The processor automatically fet-
ches the instruction from one of these known addresses based on the interrupt nu
mber, which is specified in the IRQ column. This instruction begins the interru-
pr-specific service routine (ISR).

In some designs, a column of indexes is applied  to a formula used to  calculate
an actucal vector address. In other designs, the processor  uses a more  complex
formulation zto obtain a vector address before fetching the instructions. In gen
eral, the vector table also covers the service routines for synchronous excepti-
ons. The service routines are also called _vectors_ in short.

## Classification of General Exceptions

Most of the more recent processors have these types of exceptions:

* asynchronous-non-maskable
	Asynchronous exceptions that cannot be blocked by software, they are al-
	ways aknowledged by the processor and processed immediately. many proce-
	ssors have a dedicated non-maskable interrupt (NMI) request line.
	External interrupts, with the exception of NMI, are the only async exce-
	ptions that can be disabled by software.

* asynchronous-maskable
	Asynchronous exceptions that can be blocked or enabled by software

* synchronous-precise
	The processor's program counter (PC) points to the exact instruction th
	at caused the exception, which is the offending instruction, and the pr
	ocessor knows where to resume execution upon return from the exception.

* synchronous-imprecise
	If an embedded processor implements heavy pipelining or pre-fetch algor-
	ithms, it can often be impossible to determine the ex<F6>act instruction
	and assosiated data that caused an exception. This issue indicates an im
	precise exception. Consequently, when some exceptions do occur, the rep-
	orted program counters does not point to the offending instruction, whi-
	ch makes the program counter meaningless to the exception handler.

# Processing General Exceptions

The overall exception handling mechanism is similar to the mechannism for inter-
rupt handling. Simplified, the CPU takes the following steps when E/I is raised:

* Save the current processor information

* Load the exception or interrupt handlig function into the program counter (PC)

* Transfer control to the handler function and begin execution

* Restore the processor state information after the handler function completes

* Return form the exception or interrupt and resume previous execution.

A typical handler function does the following:

* Switch to an exception frame or an inteerupt stack

* Save additional processor state information

* Mask the current interrupt level but allow higher priority interrupts to occur

* Perform a minimum amount of work, so task can complete the main processing

## Installing Exception Handlers

Exception service routines (ESR) and interrupt service routines (ISR) must be in
stalled into the system beore exceptions and interrupts can be handled. The inst
allation of ESR and ISR requires knowledge of the exception and interrupt table,
called the ___general exception table___.

The embedded system startup code typically installs the ESRs at the time of the
system initialization. Hardware device drivers typically install the appropriate
ISRs at the time of driver initialization.

If either an exception or an inteerupt occurs when no assosiated handler functi-
on is installed, the system suffers a system fault and may halt, to prevent this
it is common to install default handler functions into the vector table for eve-
ry possible exception and interrupt in the system.

## Saving processor states

The processor typically saves a minimum amount of its state information, includ
ing the status register (SR) that contains the current processor execution stat
us bits and the program counter (PC) that contains the returning address, which
is the instruction to resume execution after the exception.

Stacks are used for the storage requirement of saving processor state informati
on. A _stack_ is a statically reserved block of memory and an active dynamic po
inter called a stack pointer. It could be 2 stacks: USP - user stack, and SSP -
the supervisor stack. USP used for non-privileged mode, SSP used in priveleged.
(in linux kernel and another RTOSes STACK grows down, the head grows Up).

The active stack pointer (SP) is reinitialized to that of the active task each
time a task context switch occurs. The underlying real-time kernel performs th
is work. As mentioned earlier, the processor uses whichever stack the SP points
to for storing its minimum state information before invoking the exception hand
ler. The general idea of sizeing and reserving exception stack space: when gene
ral exceptions occur and a task is running, the task's stack is used to handle
the exception or interrupt.

If a lower priority EST/ISR us running at the time of exception or interrupt, wh
ich ever stack the ESR/ISR is using is also the stack used to handle the new exc
eption or interrupt.This default approach on stack usage can be problematic with
nested exceptions/interrupts (ex stack overflow).

## Loading and Invoking Exception Handlers

An interrupt can be disabled, active, or pending. A disabled interrupt is also
called a masked interrupt. The PIC ingores a disabled interrupt. A pending int
errupt is an unacknoledged interrupt, which occurs when the processor is curr-
ently processing a higher priority interrut. The pending interrupt is aknowled
ged and processed after all higher priority interrupts that were pending have
been processed.

An active interrupt is the one that the processor is acknowledging and process-
ing. Being aware of the existence of a pending interrupt and raising this inte-
rrupt to the processor at the appropriate time us accomplished through hardware.

For synchronous exceptions, the processor first determines thich exception has
occured and then calculates the correct index into the vector table to retrieve
the ESR. This calculation is dependent on implementation. So an extra step is in
olved. The PIC must determine if the interrupt has been disabled (or masked). If
so, the PIC ignores the interrupt and the processor execution state in not affec
ted. If the interrupt is not masked, the PIC raised the interrupt to the proces-
sor and the processor calculates the interrupt vector address and then loads the
exception vector for execution.

Some silicon vendors implement the table lookup in hardware, while others rely
on software approaches. The mechanisms are the same. When an exception occurs,
a value or index is calculated for the table. The context of the table at this
index or offset refrects the address of a service routine.

## Nested Exceptions and Stack Overflow

Nested exceptions refre to the ability for higher priority exceptions to preempt
the processing of lower priority exceptions. Much like a context switch for task
When interrupts can nest, the application stack must be large enough to accommo-
date the maximum requirements for ther application's own nested function invoca-
tion, as well as the maximum exception or interrupt nesting possible.

When data is copied onto the stack past the statically defined limits, TCB beco-
mes corrupted, which is a _stack overflow_.

Two solutions to the problem are available: increasing  the application's  stack
size to accomodate all possibilities and the deepest levels of exception and int
errupt nesting, or having the ESR or ISR switch to its own exception stack, call
ed an _exception frame_.

The maximum exception stack size is a direct function of the number of exceptions,
the number of external devices connected to each distinct IRQ line, and the priori
ty levels supported by the PIC. The simple solution is having the application to
allocate a large enough stack space to acommodate the worst case, which is if the
lowest priority exception handler exectues and is preemted by all higher priority
exceptions or interrupts. A better approach, however, is using an independent exc
eption frame inside the ESR or the ISR. It requires far less total memory.

## Exception Handlers

After control is transerred to the exception handler, the ESR/ISR performs the
actual work of exception processing. Usually the exception handler has two parts
The first part executes in the exception or interrupt context, second half execu
tes in a task context.

* Exception frame also called interrupt stack (in async exceptions)
	The common approach to the exception frame is for the ESR/ISR to alloca-
	te a block of memory, either statically or dynamically, before installi-
	ng itself into the system. The exception handler then saves the current
	stack pointer into temporary memory storage, reinitializes the SP to the
	private satck, and begins processing. It also can store addition proces-
	sor state information onto this stack.

* Differences between ESR and ISR
One difference between an ESR and ISR is in the additional processor state info
saved. The tree ways of masking interrupts are:

	* Disable the device so that it cannot assert additional interrupts. In-
	terrupts at all levels can still occur (masking interrupts)

	* Mask the interrupts of equal or lower prio-lvl, while allowing higher
	prio interruots to occur. The device can continue to generate interrupts
	but the processor ignores them (masking interrupts)

	* Disable the global system-wide interrupt request line to the processor
	Interrupts of any prio-lbl do not reach the processor.

An ISR would typically deploy one of these 3 methods to disable interrupts for
one or all of these reasons:

	* The ISR tries to reduse the total number of interrupts raised by device
	* the ISR is non-reentrant
	* the ISR needs to perform some atomic operations

One other related difference is hat an exception handler in many cases cannot pr
event other exceptions from occuring, while an ISR can prevent interrupts of the
same or lower priority from occuring.

* Exception timing

The interrupt latency, TB, refers to the interval between the time when the inte
rrupt is raised and the time when the ISR begins to execute. Interrupt latency is:
	* The amount of time it takes the processor to acknoledge the interrupt
	and perform the initial housekeeping work
	* A higher priority interrupt is active at the time
	* The interrupt is disabled and then later re-enabled by software

``` The interrupt responce tume is TD = TB + TC```
Where TB is interrupt latency, TC is the processing time.

# The Nature of Spurious Interrupts

A spurious interrupt is a signal of very short duration on one of the interrupt
input lines, and it is likely caused by a signal glitch. An external device uses
a triggering mechanism to raise interrupts to the core processor. Two types of
triggered mechanism are _level triggering_ and _edge triggering_.

# Conclusion

* Exceptions are classified into synchronous and synchronous exceptions
* Exceptions are prioritized, also can be nested
* External interrupts belongs to the category of asunchronous exceptions
* External interrupts are the only exceptions that can be disabled by software
* Using a dedicated exception frame is one solution to solving the stack over-
	flow problem that nested exception cause
* Exception processing should consider the overall timing requirements
* Spurious interrupts can occur and should be handled as any other interrupts
