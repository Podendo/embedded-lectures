# Message Queues

## Introduction

In many cases, task activity synchronization alone does not yield a sufficiently
responsive application. Tasks must also be able to exchange messages. To facilit
ate inter-task data communication, kernels provide a message queue  object and a
messagequeue services.

## Defining Message Queues

A message queue is a buffer-like object through which tasks and ISRs send and re
ceive messages to communicate and synchronize with data. A message queue is like
a pipeline. It temporarily holds messages from a sender- until the intended rece
iver is ready to read them. This temporary buffering decouples a sending and rec
eiving task; that is, it frees the tasks from having to send and receive message
simultaneously.

When a message queue is first created,, it is assigned an assosiated queue contr
ol block (QCB), a message queue name, a unique ID, memory buffers,  queue length
a maxumum message length, and one or more task-waiting lists.

The message queue itself consists of a number of oleents, each of which can hold
a single message. The elements holding  the first and last  messages are  called
_head_ and _tail_ respectively. Some elements of the queue may be empty - not contai
ning a message). The total number of elements, (empty or not) in the queue is  a
_total length of the queue_. The dev specified the queue length at the creation.

## Message Queue States

As with other kernel objects, message queues follow  the logic of a simple  FSM:
when a message queue if first created, the FSM is in the empty state.  If a task
attempts to receive messages from this message queue while the  queue  is in the
empty state, the task block and, if it chooses to, is held on the message queues
task-waiting list, in either a FIFO or priority-based order.

## Message Queue Content

message queues can be used to send and receive a variety of data. Some examples:

* a temperature value from a sensor
* a bitmap to dtaw on a display
* a text message to print to an LCD
* a keyboard event
* a data packet to send over the network

To evercome the limit on message length is to send a pointer to the data, rather
than the data itself. Even if a long message  might fit into  this queue. it  is
sometimes beter to send a pointer instead in order to  improve both  performance
and memory utilization.

## Message Queue Storage

Different kernels store message queues in different locations in memory. For exa
mple, one kernel might use a system pool, in which the messages of all queues ar
e stored in one large shared area of memory. Another  kernel might use  separate
memory areas called private buffers, for each message queue.

### System Pools

using a system pool can be advantageous if it is certain that all message queues
will never be filled to capacity at the same time. The  advantage occurs because
system pools typically save on memory use. The downside is that a  message queue
with large messages can easily use most of the pooled memory, not leaving enough
memory for other message queues. Indications that this problem occursing include
a message queue that is not full that starts rejecting messages sent to it or  a
full message queue that continues to accept more messages.

### Private Buffers

using priate buffers, on the other hand, requires enogh reserved memory area for
the full capacity of every message queue that will be created. This approach cle
arly uses up more memory, but, it also insures that message do not get overwrit-
ten and that room is available for all messages, resulting in better reliability
than the pool approach.

## Typical message Queue Operations

Typical message queue operations include the following:

* ___creating and deleting message queues___

There is two signal call for this operation: create() and delete(). When created,
message- queues are treated as global objects and are not owned by any particular
task. Typically, the queue to be used by each group of tasks or ISRs is assigned
in the design.

When creating a message queue, a developer needs to make some initialization pro
cesses, such as determine the length of the  message queue, the  maximus size of
the messages it can handle, and the waiting order for tasks when they block on a
message queue.

Deleting a message queue automatically unblocks waiting tasks. The blocking call
in each of these tasks returns with an error. Messages that were queued are lost
when the queue is deleted.

* ___sending and receiving messages___

Typical operations are: send() the message to the msg-queue, receive() a message
from a message queue, and broadcast() messages.

When sending messages, a kernel typically fills a message  queue in FIFO  order,
from head to tail. Many message queue  implementations allow urgent messages  to
go straignt to the head of the queue. If all arriving messages are urgent,  they
all go to the head of the queue, and the queuing order effectively becomes LIFO.
many message-queue implementations also allow ISRs to send messages to a message
queue. In any case, messages are sent to a message queue in the _following way_:
	* _not block_ (ISRs and tasks)
	* _block with a timeout_ (tasks only)
	* _block forever_ (tasks only)
At times, messages must be sent without blocking the sender. If a message  queue
if already full, the send call returns with an error, and the task or ISR making
the call continues  executing. This type of approach to sending messages is  the
only way to send messages from ISRs, beacuse ISR cannot block.

As with sending messages, tasks can receive messages with different blocking pol
icies the same way as they send them with a policy of not blocking, timeout blo-
cking, blocking forever. For the message queue to become full, either the receiv
ing task list must be empty or the rate at which messages are posted in the mes-
sage queue must be greater than the rate at which messages are  removed. For the
task-waiting list for receiving tasks to start  to full, the message queue  must
be empty. Messages can be read from the head of a message queue in two different
ways:
	* destructive read
	* non-destructive read
In a _destructive_ read, when a task successfully receives a message from a queue,
the task permanently removes the messages from the message queue's storage  buf-
fer. In a _non-destructive_ read, a receiving task peeks at the message at the  he
ad of the queue without removing it.

Both ways of reading a message can be useful, but not all kernel implementations
support the non-destructive read. Some kernel support additional ways of sending
and receiving messages. one way is the example of peeking at a message. Other ke
rnels allow broadcast messaging.

* ___obtaining message queue information___

Obtaining message queue information can be done from an application by using the
operations like show-queue-info() or show-queue's-task-waiting-list(). Different
kernel allow the developers to obtain different types of information about a mes
sage queue, including the message queue ID, the  queuing order used for  blocked
tasks (FIFO or prioriy-based), and the number of message queued. Some call might
even allow to get a full list of messages that have been queued up.

As with other calls, that get information about a particular kernel object, note
that this information is dynamic and might have changed by the time it is viewed
These types of calls should only be used for debugging purposes.

## Typical message queue usage

There are typical ways to use message queues within an application:

* _non-interlocked, one-way data communication_

Sending task (also called the message source), also called non-interlocked  one-
way data communication. The activities of tSourceTask and tSinkTask are not sync
hronized, Source simply sends a messages without acknoledgement from tSinkTask.
```
tSourceTask ()
{
	:
	Send message to message queue
	:
}
tSinkTask ()
{
	:
	Receive message from message queue
	:
}
```
If tSinkTask is set to a higher priority, it runs first until it blocks on an em
pty message queue. As soon as tSourceTask sends the message to the queue, tSink-
Task receives the message and starts to execute again.
if tSinkTask is set to a lower priority - tSourceTask fills the message queue wi
th messages. Eventually, tSourceTask can be made to block when sending a message
to a full message queue, this action makes tSinkTask wakeup and start taking mes
sages out of the message queue.

Practical usage: ISR - when it's triggered by the HW, the ISR  puts one or  more
messages into the msg-queue, after ISR completion, tSinkTask gets an opportunity
to run (if it is the highest-priority task) and takes the messages out of the me
ssage queue. When ISRs send messages to the message queue,  the must do it in  a
non-blocking way. If the msg-queue  becomes full, any additional  messages, that
the ISR sends to the message queue are lost.

* _interlocked, one-way data communication_

A sending task might require a acknoledgement, also called a handshake, that the
receiving task has been successful in receiving the message. This process is cal
led interlocked communication, in which the sending task sends a message and wa-
its to see if the message is received. (usage - task synchronization).
```
tSourceTask ()
{
	:
	Send message to message queue
	Acquire binary semaphore
	:
}
tSinkTask ()
{
	:
	Receive message from message queue
	Give binary semaphore
	:
}
```
In this case tasks use a binary semaphore, initially set to 0 and a msg-queue wi
th a length of 1 (also called mailbox). tSourceTask sends the message to the mes
sage queue and blocks on the binary semaphore. The semaphore that has just  been
available wakes up tSourceTask, then, this task executes and  posts another mes-
sage tointo the message queue, blocking again afterward on the binary semaphore.

* _interlocked, two-way data communication_

Sometimes data must flow bidirectionally between tasks,  which is called  inter-
locked, two-way data communication (also called full-dumplex or tightly  coupled
communication). This form of communcation can be useful when designing a client/
server-based system.
```
tClientTask ()
{
	:
	Send a message to the requests queue
	Wait for message from the server queue
	:
}
tServerTask ()
{
	:
	Receive a message from the requests queue
	Send a message to the client queue
	:
}
```
Two separate message queues are required for full-duplex  communication. If  any
data needs to be exchanged, message queues are required; otherwise, a simple sem
aphore can be used to synchronize acknowledgement.
tServerTask is typically set to a higher priority, allowing it to quickly  full-
fill client requests. If multiple clients need to be set up, all clients can use
the client message queue to post requests. If multiple clients need to be setup,
all clients can use the client message queue to post requests, while tServerTask
uses a separate message queue to fullfill the different clients requests.

* _broadcast communication_

Some message-queue implementations allow developers to broadcast  a copy of  the
same message to multiple tasks. Message broadcasting is a one-to-many-task relat
ionship. tBroadcastTask sends the message on which multiple tSinkTask're waiting
```
tBroadcastTask ()
{
	:
	Send broadcast message to queue
	:
}
		/* Note: similar code for tSignalTasks 1, 2, and 3. */
tSignalTask ()
{
	:
	Receive message on queue
	:
}
```
In this scenario, tSinkTask1, 2, and 3 have all made calls to block on the broad
cast message queue, waiting for a message. When tBroadCast executes, it sends on
e message to the message queue, resulting in all three waiting tasks exiting the
blocked state.

# Conclusion:

* Message queue are buffer-like kernel objects used for data  communication  and
synchronization between two tasks or netween an ISR and a task.

* Message queue have an assosiated message queue control block (QCB), a name, an
unique ID, memory buffers, a message queue length, a maximum message length, and
one ore more task-waiting lists.

* The beginning and end of message queues are called the head and tail, each buf
fer can hold one message is called a message-queue element.

* Message quques are empty when created, full when all msg-queu elements contain
messages, and not empty when some elements are still available for holding new M

* Sending messages to full message queues  can casue the sending task to  block,
and receiving messages from an empty message queue can cause a receiving taks to
block

* Tasks can send/receive from msg-queues wo blocking, via blocking, with timeout
or vua blocking forever. An ISR can only send messages without blocking.

* The task waiting list is assosiated with a message-queue can release tasks, or
unblock them in FIFO or priority-based order. When messages are sent from one to
another task, the message is typically copied twice: from  sending task's memory
area to the message queue's and a second time from the message queue's memory ar
ea to the task's memory area.

* The data itself can either be sent as the message or as a pointer to the  data
as the message. The first case is  better suited  for smaller messages, and  the
latter case is better suited for large messages.

* Common message-queue operations include creating and deleting message  queues,
sending to and receiving from message queues, and obtaining message queue info.

* Urgent messages are inserted at the head of the queue if urgent message are su
pported by the message-queue kernel implementation

* Some common ways to use messages queues  for data based communication  include
non-interlocked and interlocked queues providing one-way or two-way data com-ion
