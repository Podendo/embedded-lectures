# Semaphores

## Semaphores - synchronization privitives

Multiple concurrent threads of execution within an  application must be able  to
synchronize their execution and coordinate  mutually  exclusive access to shared
resources. To address these requirements, RTOS kernels provide  semaphore object
and assosiated semaphore management services.

## Defining Semaphores

_A semaphore (sometimes called a semaphore token)_ is  kernel object that one or
more threads of execution can acquire or release for the purposes of synchroniza
tion or mutual exclusion.
When a semaphore is first created, the kernel assigns to it an associated semaph
ore control block (SCB), a unique ID, a value (binary or count), and a task-wait
ing list. A semaphore is like a key that allows a task to carry out some operati
on or to access a resource.

The kernel tracks the number of times a semaphore has been acquired or  released
by maintaining a token count, which is initialized to a value when the semaphore
is created. As a task acquires the semaphore, the token count is decremented; as
a task releases the semaphore, the count is incremented.

If the token count reaches the '0' value, the semaphore has no token left, thus,
a requested task cannot acquire the semaphore,  and this task blocks if it chose
to wait for the semaphore  to become  available. The task-waiting list in either
FIFO (First in - First out) order or highest priority first order.

When an unavailable semaphore becomes available, the kernel allow the first task
in the task-waiting  list to acquire  it. The kernel moves  this unblocked  task
either to the running state, if it is the highest priority task, or to the ready
state, until it becomes the highest priority task and is able to run.

A kernel can support many different types of semaphore, including binary, R or W,
counting and mutual-exclusion (mutex) semaphores.

### Binary Semaphores

_A binary semaphore_ can have a value of either 0 or 1. When a binary semaphore's
value is 0, the semaphore is considered _unavailable_ (or _empty_); when the val
ue is 1, the binary semaphore is considered _available_ (or _full_). Note that when
a binary semaphore is first created, it can be initialized with 0 or 1 value.
Binary semaphore are treated as _global resources_, which means they are  shared
among all tasks that need  them. Making the semaphore a global  resource  allows
any task to release it, even if the task did not initially acquire it.


### Counting Semaphores

A _counting semaphore_ uses a count to allow it to be acquired or released by pair
or more threads of execution multiple times. When creating a counting semaphore,
assing the semaphore a count that denotes the number of semaphore tokens it  has
initially. If the initial count is 0, the counting  semaphore is created in  the
unavailable state. If the count is greater that 0, the  semaphore is created  in
the available state, and the number of tokens it has equals its count.

As with binary semaphores, counting semaphores are global resources that can  be
shared by  all tasks that  need them. This feature  allows any task to release a
cointing semaphore token.  Each release operation  increments the count  by one,
even if the task making this call did not acquire a token in the first place.

Some implementations of counting semaphores might allow the count to be bound. A
___bounded count___ is a ount in which the initial count set for the counting sema
phore was first created, acts as the maximum count for the semaphore. An _unbound
ed count_ allows the counting semaphore to count beyond the initial count to the
maximum value that can be held by the count's data type.

### Mutual Exclusion (Mutex) Semaphores

_A mutual exclusion (mutex) semaphore_ is a special binary semaphore that supports
ownership, recursive acccess, task  deletion safety,  and one or more  protocols
for avoiding problems inherent to mutual exclusion. (states locked/unlocked).

As opposed to the available and unavailable states in binary and counting semaph
ores, the states of a mutex are unlocked and locked (0 and 1 respectively). A mu
tex is initially created in the unlocked state, in which it can be acquired by a
task. After being acquired, the mutex moves to the locked state. Conversely, when
the task releases the mutex, the mutex returns- to the unlocked state. Note that
some kernels might use the terms lock and unlock for a mutex instead of  acquire
and release.

Depending on implementation, a mutex can support  additional features not  found
in binary or counting semaphores. These  key differentiating  featuers including
ownership, recursive locking, task deletion safety and priority inversion avoid-
ance protocols.

* _Ownership_ of a mutex is gained when a task first locks the mutex by acquiring
it. Conversely, a task loses ownership of the mutex when it unlocks it by releas
ing it. When a task owns the mutex it is not possible for any other task to lock
or unlock that mutex. Contrask this concept with the binary semaphore, which can
be released by any task, even a task that did not originally acquire the semaph.

* _Recursive locking_ allows the task that owns the mutex to acquire it multiple
times in the locked state. Depending on the  implementation, recursion with  the
mutex can be automatically built in the mutex. This type of mutex is called _a re
cursive mutex_. It is most useful when a task requiring exclusive access to a sha
red resource calls one or more routines that also require acess to the same reso
urce. A recursive mutex allows nested attempts to lock the mutex to succeed,  or
it can cause a _deadlock_.

* _task deletion safety_ - premature task deletion is avoided by using _task dele
tion locks_ when a task locks and unlocks a mutex. Enabling this capability with
in a mutex ensures that while a task owns the mutex, the task cannot be deleted.
Typically protection from premature deletion is enabled by setting the appropria
te initialization options when creating the mutex.

* _priority inversion avoidance_ - priority inversion occurs when a higher priori
ty task is blocked and is waiting for a resource being used by a lower  priority
task, which has itself been  preemted by an  unrelated medium-priority task.  In
this situation the higher priority task's  priority level has  effectively  been
inverted to the lower priority task's level. Enablong certain protocols that are
typically built into mutexes can help avoid priority inversion. Two common proto
cols used for avoiding priority inversion include:

	* priority ingeritance protocol: ensures that the priority level  of the
	lower priority task that has acquired the mutex is raised to that of the
	higher priority task that has requested the mutex when inversion happens

	* ceiling priority protocol ensures that the priority level of the  task
	that acquires the mutex is automatically set to the highest priority  of
	all possible tasks that might request that mutex when it is first acquir
	ed unlit it is released. When the mutex is release, the priority of  the
	task is lowered to its original value.

##Typical Semaphore Operations

Typical operations that developers might want to perform with the semaphores:

* Creating and deleting semaphores
* Acquiring and releasing semaphores
* Clearing a semaphore task-waiting list
* Getting semaphore information

###Creating and Deleting Semaphores

If  kernel supports different types of semaphores, different calls might be used
for creating binary, counting, and mutex semaphores, as follows:

* binary specify the initial semaphore state and the task-waiting order
* counting specify the initial semaphore count and the task-waiting order
* mutex specify the task-waiting order and enable task deletion safety, recursion
and priority inversion avoidance protocols, if supported

Semaphores can be deleted from within any task by specifying their IDs and making
semaphore-deletion calls. Deleting a semaphore  is no the same as  releasing  it.
When a semaphore is deleted, blocked tasks in its task-waiting list are unblocked
and moved either  to the ready state  or to the  running  state (if the unblocked
task has the highest priority). Any tasks, however, that try to acquire the delet
ed semaphore return with an error because the semaphore no longer exists. Additio
nally, do not delete a semaphore while it is in use (e.g. acquired). This  action
may result in data corruption or other serious problems if the semaphore is prote
cting a shared resource or a critical section of code.

###Acquiring and Releasing Semaphores

The ops for acquiring and releasing semaphore might have different names, depend
ing on the kernel: for example `take` and `give`, `pend` or post`, `lock` or `un
lock`. Tasks typically make a requests to acquire a semaphore in one of the ways:

* Wait forever task remains blocked until it is able to acquir a semaphore
* wait with a timeout task remains blocked until it is able to acquire a semapho
re or until a set interval of time, called the _timeout interval_ passes. At this
point, the task is removed from the semaphore's task-waiting list and put in eith
er the ready-state or the running state.
* Do not wait task makes a request to acquire a semaphore token, bit if one isn't
available, the task does not block.

The ISRs can also release bibary and counting semaphores. Note that most kernels
do not support ISRs locking and unlocking mutexes, as it is not meaningful to do
so from an ISR. It is also bot meaningful  to acquire either binary or  counting
semaphores inside an ISR.

###Clearing Semaphore Task-Waiting Lists

To clear all task waiting on a semaphore task-waiting list, some kernels support
a _flush_ operation - unblocking all tasks waiting on a semaphore. The fush oper
ation is useful for broadcast signaling to a group of tasks. For `For example, a
developer might design multiple tasks to complete certain  activities first  and
then block while trying to acquire a common semaphore that is made  unavailable.
After the last task finishes doing its job, the task can execute a semaphore flu
sh operation on the common semaphore. This operation frees all takss waiting  in
the semaphore's task waiting list. The synchronization scenario  just  described
is also called ___thread rendezvous___, when multiple tasks executions needs to me
et at some point in time to synchronize execution control.

###Getting Semaphore information

At some point in the application desigh, developer need to obtain semaphore info
rmation to perform monitoring or debugging, in  these cases, use the  operations
_show info_ - general info about semaphore, and _show blocked tasks_ - get a list
of IDs of tasks that are blocked on a semaphore.

## Typical Semaphore Use

Semaphores are useful either for  synchronizing execution of  multiple tasks  or
for coordinates access to a shared resource. The following examples and  general
discussions illustrate using different types of semaphores to address common syn
chronization desigh requirements:

* ___wait-and-signal synchronization___
tWaitTask has higher priority and runs first. binary semaphore is initialized as
zeroed-locked, so acquiring binary semaphore blocks the waiting process and  the
tSignalTask goes on - it releases the binary semaphore and unblocks tWaitTask.

Because tWaitTask's priority is higher that tSignalTask's  priority, as soon  as
the semaphore is released, tWaitTask preemts tSignalTask and starts to execute.
```
tWaitTask ( )
{
	:
	Acquire binary semaphore token
	:
}
tSignalTask ( ){
	:
	Release binary semaphore token
	:
}
```
* ___multiple-task-wait-and-signal synchronization___
The binary semaphore is initially unavailable (value is 0). The higher  priority
tWaitTasks 1, 2, and 3 all do some processing; when they are done,  they try  to
acquire the unavailable semaphore and as a result, block. This action gives  the
tSignalTask a chance to complete its processing and execute  a flush command  on
the semaphore, effectively unblocking the three tWaitTasks
```
tWaitTask ()	/* higher priority */
{
	:
	Do some processing specific to task Acquire binary semaphore token
	:
}
tSignalTask ()	/* lower priority */
{
	:
	Do some processing Flush binary semaphore's task-waiting list
	:
}
```
* ___credit-tracking synchronization___
The counting  semaphore provides  a mechanism to count each signalong occurence.
With a counting semaphore, the signaling task can continue to execute and incre-
ment a count at uts own pace, while the wait task, when  unblocked, executes  at
its onw pace. Initial value of semaphore is 0.

```
tWaitTask ()	/* lower priority */
{
	:
	Acquire counting semaphore token
	:
}
tSignalTask ()	/* higher priority */
{
	:
	Release counting semaphore token
	:
}
```
The lower priority tWaitTask tries to acquire this semaphore but blocks until  a
tSignalTask makes the  semaphore available  by performing a release on it.  Even
then, tWaitTask will waits in the ready state until the higher priority tSignal-
Task eventually relinquishes the CPU my making a blocking call / delaying itself.

* ___single-shared-resource-access synchronization___

```
tAccessTask ()
{
	:
	Acquire binary semaphore token
	Read or write to shared resource
	Release binary semaphore token
	:
}
```
* ___recursive shared-resource-access synchronization___

```
tAccessTask (){
:
	Acquire mutex
	Access shared resource
	Call Routine A
	Release mutex
	:
}
Routine A ()
{
	:
	Acquire mutex
	Access shared resource
	Call Routine B
	Release mutex
	:
}
Routine B ()
{
	:
	Acquire mutex
	Access shared resource
	Release mutex
	:
}
```
* ___multiple shared-resource-access synchronization___

```
tAccessTask ()
{
	:
	Acquire a counting semaphore token
	Read or Write to shared resource
	Release a counting semaphore token
	:
}
```

```
tAccessTask ()
{
	:
	Acquire first mutex in non-blocking way
	If not successful then acquire 2nd mutex in a blocking way
	Read or Write to shared resource
	Release the acquired mutex
	:
}
```


#Conclusion:

TDB

