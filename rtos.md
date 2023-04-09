# Introduction to Real-Time Operating Systems

## Introduction

RTOS used for embedded systems with moderate-to-large software applications, wh-
ich require some form of scheduling. Over the years, many versions of  operating
system evolved. These ranged from general-purpose operating systems (GPOS)  such
as UNIX and Microsoft Windows, to smaller  and more  compact real-time operating
systems, such as VxWorks, FreeRTOS, etc.

core functional similarities between a RTOS na GPOS:

* some level of multitasking
* software and hardware resource management
* provision of underlying OS services to applications, HW-to-SW abstraction

key functional differences that set RTOS aprat from GPOS:

* better reliability in embedded application contexts
* the ability to scale up or down to meet application needs
* faster performance
* reduced memory requirements
* scheduling policies tailored for real-time embedded systems
* support of diskless embedded systems (booting and running from ROM or RAM)
* better portability to different hardware platforms

## Defining an RTOS

A real-time operating system RTOS is a program that schedules in a timely manner
and manages system resources, provides a consistent foundation for developing an
application code. For example, in some embedded applications, an RTOS  comprises
only a kernel, which is the core  supervisory software  that provides min logic,
scheduling, resource-management algorithms.

**High-level view of an RTOS**:

1. Application
2. RTOS - Kernel and in addition:
	* Networking protocol
	* File system
	* POSIX support
	* Device I/O support
	* Device drivers
	* Debugging Facilities
	* C/C++ Support Libraries
	* Other componets
3. BSB ( Board support Package)
4. Target Hardware

**Most RTOS kernels contain the following components**
 * _scheduler_ contained within each kernel and follows a set of algorothms that
 determines which task executes when. Scheduling algorithms include round-robin,
 preemtive scheduling.
 * _Objects_ are special kernel construct that help devs create applications for
 real-time embedded systems. Common kernel obkects include tasks, semaphores and
 message queues.
 * _Services_ are operations that the kernel performs on an object or, generally
 operations such as timing, Interrupt handling, memory/device management service

## The Scheduler

### Schedulable entities

A schedulable entity is a kernel object that cam complete for execution on a sys
based on a predefined scheduling algorithm. Tasks and processes are all examples
of schedulable entities found in most kernels.

__A task__ is an independent thread of execution, contains a sequence of instruc
tions, which are independently schedulable.
__Processes__ like tasks can independently complete for CPU execution time. They
differ from tasks in that they provide better memory protection features, at the
expense of performance and memory overheading.
__message queues and semaphores__ are not schedulable entities. These items are
inter-task or inter-process-communication (IPC) objects used for synchronization
and communication between processes.

### Multitasking

Multitasking is the ability  of the OS to  handle multiple activities within set
deadlines. The kernel is actually interleaving executions sequentially. based on
a preset scheduling  algorithm. The scheduler  must ensure that the  appropriate
task runs at the right time.

An important point to note here is that the tasks follow the kernel a scheduling
algorithm, while **interrupt service routines (ISR)** are triggered to run cause
of hardware interrupts and  their established priorities. As the number of tasks
to schedule increases, so do CPU performance  requirements. This fact is due  to
increased switching between the contexts of the different threads of execution.

### The context Switch

Each task has its own context, which is the state of the CPU registers  required
each time  it is  scheduled to  run. A context  switch occurs when the scheduler
swithces from one task to another.

Every time a new task is created< the kernel also creates and maintains an TCB -
task control block. TCBs are system data structures that the kernel uses to main
tain task-specific information. Task control blocks contain everything a  kernel
needs to know about a particular task. When task is running, its context is high
dynamic. This dynamic  context is maintained in  the TCB. When  the task is  not
running, the context is frozen within the TCB, to be restored the next time  the
task runs.
A typical context switch scenario: when the kernels scheduler determines that it
needs to stop running task 1 and start running task 2, it takes the steps:

1. The kernel saves task-1`s context information to its TaskControlBlock (TCB)
2. It loads task-2`s context information from its TCB, which becomes the current
thread of execution.
3. The context of task-1 is frozen while task-2 executes, but if the k-scheduler
needs to run task-1 again, task-1 continues from where it left  off just  before
the context switch.

Every time an application makes a system call, the scheduler has an  opportunity
to determine if it needs to switch contexts. When the scheduler determines a con
text switch is necessary, it relies on  an assosiated module, which iscalled the
dispatcher, to make that switch happen.

### The dispatcher

The dispatcher is the part of the scheduler that performs context switching  and
changes the flow of execution. At any time an RTOS is running, the flow of execu
tion, also known as __flow control__ is passing through one of three areas:
* through an application task -> system call
* trough an ISR -> system call
* through the kernel -> system call

Depending on how the kernel is first entered, dispatching can happen differently.
When a task makes system calls, the dispatcher is used to exit the kernel  after
every system call completes. In this case, the dispatcher is used on a __call-by
-call__ basis so that it can coordinate task-state  transitions that any of  the
system calls might have caused(one or more tasks may have become ready to run).

On the other hand, if an ISR makes system calls. the dispatcher is bypassed until
the ISR fully completes its execution. THis process is true even if some resource
have been freed that would normally trigger a context switch between tasks. These
context switches do not take place because  the ISR must  complete without  being
interrupted by tasks. After the ISR completes execution, the kernel exits fhrough
the dispatcher so that is can then dispatch the correct tasks.

### Scheduling algorithms

The scheduler determines which task runs by folloing a scheduling algorithm (also
known as scheduling policy). Most of kernels today support two common  scheduling
algorithms, such as:
* preemptive priority-based scheduling
* round-robin scheduling

The Real-Time-Operating-Systems typically  predefines these algorithms;  however,
in some cases, developers can create and define their one scheduling alogrithms.

#### _Preemptive Priority-Based Scheduling_

The most real-time kernels use preemptive priority-based scheduling by default.
With this type of scheduling, the task that gets to run at any point is the task
with thee highest priority among all other tasks ready to run in the system.
	Real-time kernels generally support 256 priority levels, in which '0' is
the highest and '255' is the lowest. Some kernels appoint the priorities in  the
reversed order. With a  _preemptive-priority-based_ scheduler, each  task has  a
priority, and the highests priority task runs first. If a  task with  a priority
higher that the current task becomes ready to run, the kernel immediately  saves
the current task's context in its Task-Control-Block and switches to the higher-
priority task.
	Although task are asigned a priority when they are created, a task's pri
ority can be changed dynamically using kernel-provided syscalls. The ability  to
change task priority dynamically allows an embedded applications the flexibility
to adjust to external events as they occur, creating a 'true' real-time system.

#### _Round-Robin Scheduling_

Round-Robin scheduling  provides each task an equal share of the CPU's execution
time. Pure round-robin scheduling cannot satisfy real-time systems  requirements
because in real-time os`es, tasks perform work of varying degrees of importance.
Instead, preemptive, priority-based scheduling can be augmented with round-robin
scheduling which uses time slicing  to achieve equal  allocation of the CPU  for
tasks if the save priority. With time-slicing, each task executes for a  defined

With time slicing, each task executes for a defined interval -  time-slice, in an
ongoing cycle, which is the round-robin. A run-time counter tracks the time slice
for each task, incrementing on every clock tick. When one task's time-slice compl
etes, the counter is cleared, and the task is placed at the end of the cycle.
Newly added tasks of the same priority are placed at the end  of the cycle,  with
their run-time counters initialized to '0' value.
If a task in a round-robin cycle is preempted by a higher-priority task, its run-
time count is saved as context in TaskControlBlock and then restored when the int
errupted task is again eligible for execution.

## Objects

`Kernel objects are special constructs that are the building blocks for the apps
development for real-time embedded systems. The most common RTOS kernel objects`

* _Tasks_ are concurrent and independent thrreads of execution that can complete
for CPU execution time
* _Semaphores_ are token-like objects that can be incremented or decremented  by
tasks for synchronization or mutual exclusions.
* _Message queues_ are buffer-like data structures that can be sued for synchoni
zation, mutual exclusion, and data exchange by  passing messages  between tasks.

## Services

`Along with objcets, most kernel provide services that help developers create an
applications for realtime embedded systems. These services comprise sets of  API
calls that can be used to perform operations on kernel objects or can be used in
general to facilitate timer-management, interrupt handling, device I/O, and memo
ry management. Again, other services might be provided; these services are those
most commonly found un TROS kernels.`

## Key Characteristics of an RTOS

An application requirements define the requirements of the underlying RTOS. Some
of the more common attributes are:

* reliability - combination of RTOS, BSP, and apps architecture must be reliable
* predictability - time requirements for completing tasks (syscalls) is the key
* performance - RTOS embedded systems must be fast-enough not to meet a deadline
* compactness - static and dynamic memory consumption of the RTOS / APPS
* scalability - those OS'es must be able to scale up or down for app's require


# Conclusion:

* RTOSes are best suited for realtime, apps-specific embedded systems, while the
GPOSES are typically used for general-purpose systems.
* RTOSes are programs that schedule execution in a timely manner, manage systems
resources, and provide a consistent foundation for developing application code.
* Kernels are the code module of every RTOS and typically contain kernel objects,
services, and scheduler.
* Kernels can deploy different alogithms for task scheduling. The most common two
alogrithms of scheduling are preemptive priority-based and round-robin scheduling
* PRTOSes for real-time embedded systems should be reliable, predictable, high -
performance, compact, and scalable.
