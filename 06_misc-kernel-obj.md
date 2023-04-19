# Other Kernel Objects

In addition to the key kernel objects, such as tasks,  semaphores, mutexes,  and
message queues, kernels provide many other important objects as well. Common add
itional kernel objects to embedded systems are:

* pipes, event registers, signals, condition variables

## Pipes

_Pipes_ are kernel objects that provide  unstructured data exchange and  facili-
tate synchronization among tasks. A pipe is a unidirectional data exchange faci-
lity. A pipe provides a simple data flow facility so that the reader becomes blo
cked when the pipe is empty, and the writer becomes blocked when the pipe's full

Note that a pipe is conceptually similar to a message queue but with significant
differences. For example, unlike a message queue, a pipe does not store multiple
messages. The data flow is strictly FIFO - first-in-first-out.

### Pipe Control Blocks

Pipes can be dynamically created and destroyed. The kernel creates and maintains
pipe-specific information in a _pipe control block PCB_. In its general  form, a
PipeControlBlock contains a kernel-allocated  data buffer for  the pipe's  input
and output operation. The size of the buffer is maintained in  the control block
and is fixed when the pipe is created; it cannot be altered at runtime. The cur-
rent data byte count, along with the current input and output position indicator
s, are part of the pipe control block. PCB  also contains the current  data-byte
count - indicator of the readable data in the pipe.

Also, two task-waiting lists are assosiated with each pipe: one waiting list ke-
eps track of tasks that are waiting to write into the pipe while it is full; the
other keeps track of tasks that're waiting to read from the pipe while its empty

### Pipe states

A pipe has a limited number of states assosiated with ot from the time of its cr
eation to its termination. Each state corresponds to the data transfer state bet
ween the reader and the writer of the pipe:

* __Empty (E)__ - pre-created, no data written
* __Not empty (NE)__ - Data was Written (E-NE) or Data was Read (NE-E)
* __Full (F)__ - Data was Read (F-NE), Data was Written to full(NE-F)

### Named and Unnamed Pipes

A kernel typically supports two kinds of pipe objects: named and  unnamed pipes.
A _named pipe_ also know as FIFO has a name similar to a filename and appears in
the filesystem as if it were a file or a device. Any task for ISR that needs  to
use the named pipe can reference it by name. The  _unnamed pipe_ does not have a
name and does not appear in the filesystem. It must be referenced by the descrip
tors that the kernel returns when the pipe is created.

### Typical Pipe Operations

The following operatins can be performed on a pipe:

* Create and Destroy a pipe
	* pipe - creates a pipe, open - opens a pipe, close - closes or deletes

The pipe operation creates an unnamed pipe. This operation returns two descripto
rs to the calling task, and subsequent calls reference these descriptors. One is
used for writing, other is used for reading.

Creating a named pipe has a specific call, similimar to creating a file. commons
are mknod and mkfifo. After creating, the named pipe  can be opened by using the
open operation. The calling task must specify whether it is opening the pipe for
the read operation or for the write operation; it cannot be both.

The close operation is the counterpart of the open  operation. Similar  to open,
the close operation can only be performed on a named pipe. Some  implementations
will delete the named pipe permanently once the close operaion completes.

* Read from or Write to a pipe
	* read - reads from the pipe, write - writes to a pipe

The read() returns  data from the  pipe to the calling task.  The task specifies
how much data to read. The task may choose to block waiting for the remaining da
ta to arrive if the size specified exceeds what is available in the pipe.  Data,
readed from a pipe makes it unavailable for other readers.
Data in pipe is not structured and prioritized, like it is in message queue.

* Issue control commands on the pipe
	* fcntl - provides control over the pipe descriptor
	* flush - removes all data from the pipe

The fnctl operation provides generic control over a pipe's descriptor using vari
ous commands which control the behaviour of the pipe  operation. For  example, a
commonly implemented command  is the non-blocking  command. the command controls
whether the calling task is blocked of a read operation is performed on an empty
pipe or when a write operation is performed on a full pipe.

Flush command removes all data the pipe and clears  all other the conditions  in
the pipe to the same state as when the pipe was  created. Sometimes the task can
be preempted for a too long time,  and when it finally  gets to read data from a
pipe, the data might no longer be useful. Therefore, the task can flush the data
from the pipe and reset its state.

* Select on a pipe
	* select - waits for condition to occur on a pipe

The select op allows a task to block and wait for a specified condition to occur
on one or more pipes. The wait condition can be waiting for data to become avail
able in the pipe, or waiting for data to be emptied form the pipe.

In contrast to pipes, message queues do not support the select  operation. Thus,
while a task can have access to multiple message queues, it cannot blockwait for
data  to arrive on any one of a group of empty messag queues. The same restrict-
ion applies to a writer. In this case, a task can write to multiple queues,  but
a task cannot block-wait on a group of full message queues, while waiting for sp
ace to become available on any one of them

### Typical Uses of Pipes

Because a pipe is a simple data channel, it is mainly u sed for task-to-task  or
ISR-to-task data transfer, another usef of pipes if for inter-task synchronizing

## Event Registers

Some kernels provide a special  register as part of  each task's control  block.
This register, called an _event register_, is an object belonging to a task  and
consists of a group of binary eventflags used to track the occurence of specific
events. It can be 8- 16- 32-bits wide, maybe more. Each bit is treated as a bina
ry flag, called an event flag. Can be either set or cleared.

Trough the event register, a task can check for the presence of particular event
that can control its execution. An external source, such as ISR or task, can set
bits in the event register to inform the task that a particular event's occured.

### Event Register Control blocks

Typically, when the underlying kernel supports the event register mechanism, the
kernel created an event register control block as part of the task control block
when creating a task
* Event register control block
	* Wanted events
	* Received events
	* Timeout value
	* Notification conditions
The task specifies the set of events it wishes to receive. This set of events is
maintained in the wanted events register. Arrived events are kept in the received
event register. The task indicates a timeout to specify how long it wishes to wa
it for the arrival of certain events. The kernel wakes uo the task when this tim
eout has elapsed if no specified events have arrived at the task.

Using the notification conditions, the task directs the kernel as to when it wis
hes to be notified (awakened) upon event arrivals. For example, the task can spe
cify the notification conditions as send notification when the both event type 1
and event type 3 arrive or when event 2 arrives. This option provides flexibili-
ty in defining complex notification patterns.

### Typical Event Register Operations

Two main operations are assosiated with an event register, the  sending and  the
receiving operations. The receive operation allows the  calling task to  receive
events from external sources. The task can specify if it wishes to wait, as well
as the length of time to wait for the arrival of desired events before giving up

The event set is constructed using the bitwise AND / OR operations: with the AND
operation, the task resumes execution inly after every event bit from the set is
on. A task can also block-wait for the arrival of a single event  from an  event
set, which is constructed using the bit-wise OR operation. In this case the task
resumes execution when any one event bit from the set is on.

The send operation allows an external source, either task or ISRm to send events
to another task. The sender can send multiple events to the designated task thro
ugh a sindle send operation. Events that have been sent and are  pending on  the
event bits but have not been chosen for  reception by the task remain pending in
the received events register of the event register control block.

Events in the event register are not queued. An event register cannot count  the
occurrences of the same event while its pending; therefore, subsequent occurren-
ces of the same event are lost. For example, if an OSR sends an event to a task,
and the event is left pending; and later another task sends the same event again
to the same task while it is still pending, first occurence of the event is lost

### Typical Uses of Event Registers

Event regusters are typically used ofr unidirectional activity  synchronization.
For multiple event-sources you should use a subse t of event registers. Then the
task can associate each subset with a known source, in this way, the task can su
rely identify the source of an event if each relative bit position of each  sub-
set is assigned to the same event type.

## Signals

A _signal_ is a software interrupt that is generated when an event has  occured.
it diverts the signal receiver  from its normal  execution path and triggers the
assosiated asynchronous processing. Signals notify tasks of events that  occured
during the execution of other tasks or ISRs. The difference between a normal int
errupt and a signal is that signals are so-called software interupts, which  are
generated by via the execution of some software within the system, when the norm
al interrupts are usually generated by the arrival of interrupt signal on one of
the CPU's external pins. They are not generated by  software within the  system,
but by external devices.

When a signal arrives, the task is diverted from its normal execution path,  and
the correspinding signal routine in invoked. Each signal is identified by an int
eger value, which is the _signal number_ or _vector number_.
	* _signal routine_
	* _signal handler_
	* _asynchronous event handler_
	* _asyncronous signal routine_

### Signal Control Blocks

If the underlying kernel provides a signal facility, it creates the signal contr
ol block as part of the task control block (TCB). The signal control block maint
ains a set of signals the wanted signals which the task is prepared to handle.

When a task is prepared to handle a signal (ready to catch), and this signal occ
urs in the process, it interrupts a task (raised to the task); the task can prov
ide a signal handler for each signal to be processed or it can execute a default
handler that the kernel provides. It is possible to have a single handler for mu
ltiple types of signals. Signals also can be ignored, made pending, processed or
blocked. The signals to be ignored by the task are maintained in the ignored sig
nals set. Any signal in this set does not interrupt the task.

The pending signals set is a subset of the wanted signals set: either the  task-
supplied or kernel default signal handler can be used for signal processing a pa
rticular signal. It is also possible for the task  to process the signal  first,
and then pass it on for additional processing by the default handler.

Blocking a signal is similar to the concept of  entering a critical section: the
task can istruct the kernel to block a certain signals by setting the blocked si
gnas set, the kernel does not deliver any signal from this set unlit that signal
is cleared from the set.

### Typical Signal Operations

* Catch		- Installs a signal handler

A task can catch a signal after the task has specified a handler (ASR) for  this
signal. THe tatch operation installs a handler for a particular signal. The task
can install the kernel-supplied default handler for default actions. Handling si
gnals is similar to handling hardware interrupts, and the nature of the ASR s si
milar to that of the interrupt service routine (ISR).

* Release	- Removes a previously installed handler

The release operation de-installs a signal handler. It is good practice for a ta
sk to restore the previously installed signal handler after calling release.

* Send		- Sends a signal to another task

The send operation allows one task to send a signal to another task. Signals are
usually associated with hardware events that occur  during execution  of a task,
such as generation of an unaligned memory address or a floating-point exception.
Such signals are generated automatically when their corresponding events  occur.
THe send operation, by contrast, enables a task to explicitly generate a signal.

* Ignore	- Prevents a signal from being delivered

The ignore operation allows a task to instruct the kernel that a particular  set
of signals should never be delivered to that task. Some signals, however, cannot
be ignored; when these signals are generated, the kernel calls a default handler

* Block		- Blocks a set of signal from being delivered

The block operation does not cause signals to be ignored but temporarily prevent
them from being delivered to a task. The block operation protects critical secti
ons of code from interruption. Another reason for blocking signal is to  prevent
conflict when the signal handler is already executing and is in the midst of pro
cessing the same signal. A signal remains pending while it is blocked.

* Unblock	- Unblocks the signals so they can be delivered

Unblock operation allows a previously blocked signal to apss. The signal is deli
vered immediately of ot os already pending.

### Typical Uses of Signals

* Using signals can be expensive due to the complexity of the signal facility wh
en used for inter-task-synchronization. A signal alters the  execution state  of
its destination  task. Because signals occur  asynchronously, the receiving task
becomes nondeterministic, which can be undesirable in a real-time systems.

* Many implementations do not support queuing or  counting of signals.  In these
implementations, multiple occurences  of the same signals  overwrite each other.
For example, a signal delivered to a task multiple  times before its handler  is
invoked has the same effect as a single delivery. The task has no way to determi
ne if a signal has arrived multiple times.

* Many implementations do not support signal delivery that carries  information,
so data cannot be attached to a signal during its generation.

* Many implementations do not support delivery that carries information, so data
cannot be attached to a signal during its generation.

* Many implementations do not support a signal delivery order,  and signals  are
treated as having equal priority, which is not ideal. For example, a signal trig
gered by a page fault is obviously more important that a signal generated by the
task indicating it is about to exit. On an equal-priority system, the page fault
migh not be handled first.

* Many implementations do not guarantee when an unblocked pending signal will be
delivered to the destination task.

But, some kernels do implement real-time extencions for traditional signal handl
ing, which allows to use a signals for:

* for the prioritized delivery of a signal based on the signal number
* each signal to carry additional information
* multiple occurences of the same signal to be queued

## Condition Variables

A _condition variable_ is a kernel object that is associated with a shared resou
rce, which allows one task to wait for other task(s) to create a desired conditi
on in the shared resource. A condition variable can be assosiated with  multiple
conditions. A condition variable implements a predicate.

The predicate is a set of logical expressions concerning the conditions  of  the
shared resource. The predicate evaluates to either true or false. A task evaluat
es the predicate. If the evaluation is true, the task assumes that the condition
are satisfied, and it continues execution. Otherwise, the task must wait for oth
er tasks to create the desired conditions.

Note that when the task examines a condition variable, the task must have exclus
ive access to that condition variable. Without it, another task could alter  the
condition variable's conditions at the same time. A mutex is always used in conj
unction with a condition variable. The mutex ensures that one task has exclusive
access to the condition variable until that task is finished with it.

A task must first acquire the mutex before evaluating the  predicate. This  task
must subsequently release the mutex and then, if the predicate evaluates to fal-
se, wait for the creation of the desired conditions. Using the condition variab-
les, the kernel quarantees that the task can release the mutex and then block-wa
it for the condition in one atomic operation, which is the essence of the condit
ion variable. An _atomic operation_ is an operation that cannot be interrupted.

Condition variables are not mechanisms for synchronizing access to a shared reso
urce, you should use them to allow tasks waiting on a shared recource to reach a
desired value or state.

### Condition Variable Control Blocks

The kernel maintains a set  of information with the condition variable when  the
variable is first created. As exmplained, tasks must block and wait when a condi
tion variable's predicate evaluates to false. These waiting tasks are maintained
in the task-waiting list.

The criteria for selecting which task to awaken can be priority-based or FIFO-ba
sed, but it is kernel-defined. The essence of the condition variable is the atom
icity of the unlock-and-wait and resume-and-lock operations provided by the kern

### Typical Condition Variable Operations

* Create - creates and initializes a condition variable

* Wait - waits on a condition variable

* Signal - signals the condition variable on the presence of a condition

* Broadcast - signals to all waiting tasks the presence of a condition

### Typical use of Cindition Variables


```
Task 1
Lock mutex
	Examine shared resource
	While (shared resource is Busy)
		WAIT (condition variable)
	Mark shared resource as Busy
Unlock mutex

Task 2
Lock mutex
	Mark shared resource as Free
	SIGNAL (condition variable)
Unlock mutex
```
Task 1 on the left locks the quarding mutex as its first step. It then  examines
the state of the shared resource and finds that the resource is busy. It  issues
the wait operation to wait for the resource to become available, or free.

The free condition must be created by Task 2 on the right after it is done using
the resource. To create the free condition, task 2 first  locks the mutex,  then
creates the condition by marking the resource as free, and finally, invokes  the
signal operation, which informs Task 1 that the free condition is now present.

A signal on the condition variable is lost when nothing is waiting on it. There-
fore, a task should always check for the presence of the desired condition befo-
re waiting on it.

# Conclusion

TBD


