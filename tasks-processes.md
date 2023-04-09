# Introduction to tasks

_Concurrent desigh_ reqires developers to  decompose an application into  small,
schedulable, and sequential program units. When it's done correctly,  concurrent
design allows system multitasking to meet performance and timing requirement for
a real-time system. Most RTOS kernel provide task objects and task  management's
services to facilitale designing concurrency within an application.

## Defining a Task

A __task__ is  an independent thread of  execution that  can complete with other
concurrent tasks for _processor execution time_. A task is _schedulable_ - it is
able to co,plete for execution time on a system, based on a predefined  schedule
algorithm. A task is defined by its distinct set of parameters and supporting da
ta structures. Specifically, upon creation, each task has an assosiated name, an
unique ID, a priority, a task control block (TCB), a stack and a task routine -
executable code - our program.

When the kernel  first  starts, it  creates its own set of  __system tasks__ and
allocated the appropriate priority for each from a set of _reserved priority lvl_
The reserved priority levels refer to the priorities used internally by the RTOS
for its system tasks.
` Examples of system tasks include:
* initialization or startup task - inits the system and creates and starts tasks
* idle task - uses up processor idle cycles when no other activity is present
* logging task - logs system messages
* excepion-handling task - hadles exceptions
* debug agent task - allows debugging with a host debugger`

As the developer creates new tasks, the developer must assing each a task name,
priority, stack size, and a task routine. The kernel dies the rest by assgning
each task a unique ID and creating an assosiated TCB and stack space in memory.

## Task states and Scheduling

Althought kernel can define task-state grouping differently, generally there is
a three main states, used in most typical preemptive-scheduling kernels:
* __ready-state__ - the task is ready to run but cannot because the higher prio-
rity yask is executing at this moment;
* __blocked-state__ - the task has requested a resource that is not available or
has requested to wait until some event occurs, or has delayed itself.
* __running-state__ - the task is the highest priority yask and is running
* for Linux-based there is another type of task - __zombie__ - the child process
has been died, but has not yet been reaped. kill command has  no effect on  this
type of process.

### Ready State

When a task is first created and made ready-to-run, the kernel puts in into  the
ready-state. In this state, task actively completes with all  other  ready-state
tasks for the processor's execution time.Tasks in the ready state cannot move to
the blocked state directly. A task first needs to run so it can make a _blocking
call_, which  is a call to a function that cannot immediately run to completion,
thus, putting the task in the blocked state. Ready tasks, therefore, can move to
the running state only.
Most kernel that supports only one task per priority  level, allowing many  more
tasks in an application. In this case, the scheduling alorithm is more complicat
ed and invloves maintaing a _task-ready list_. Some  kernel maintain a  separate
task-ready list for each priority level. Others have one combined list.

` As example, we have RTOS with a single task-ready list for all task-priorities`

```
In this example, tasks 1, 2, 3, 4, and 5 are ready to run, and the kernel queues
them by priority in a task-ready list. Task-1 is the highest priority task (70);
tasks 2,3, and 4 are at the next-highest priority level (80); and task 5 is  the
lowest priority (90). THe following steps  explains how  a kernel might use  the
task-ready list to move tasks to and from the ready-state:

1. Taks 1,2,3,4 and 5 are ready to run and are waiting in the task-ready-list.
2. Beacuse task-1 is the highest priority (70), it`s the first task ready-to-run
3. During execution, task-1 makes a blocking call. As a result, the kernel moves
task-1 ti the blocked state; takes task-2 which is first in the list of the next
highest priority tasks (80), off the ready list, and moves task-2 to the running
4. Next task-2 makes a blocking call. The kernel moves task-2 to the blocked sta
te, takes task-3, which is next in line of priority 90 tasks, off the ready list
and moves task-3 ti the running state.
5. As task 3 runs, frees the resource that task 2 requested. The kernel returns
task-2 to the ready state and inserts in the end of the list of tasks ready  to
run at priority level 80. Task 3 continues as the currently running task.
```
### Running State

On a single prcessor-system, ony one task can run at a time. In this case,  when
a task is moved to the running state, the processor loads its registers with this
task's context, saved in TaskControlBlock. THe CPU can  then execute the  task's
instructions and manipulate the associated stack.

Unlike a _ready task_, a running task can move to the _blocked_ state in ways:
* by making a call that requests an unavailable resource
* by making a call that requests to wait for an event to occur
* by making a call to delay the task for some duration

In each of these cases, the task is moved from the running state to the *blocked*

### Blocked State

Without blocked states, lower priority tasks could not  run. If higher  priority
tasks are not designed to block, CPU starvation can result.
__CPU starvation__ occurs when higher priority tasks use all of the CPU execution
time and lower priority tasks do not get to run.
A task can only move to the blocked state by making a blocking call,  requesting
that some blocking condition be met. Example of blocking conditions are met incl
ude the following:

* a semaphore token for which a task is waiting is released,
* a message, on which the task is waiting, arrives in a message queue
* a time delay imposed on the task expires.

## Typical Task Operations

### Task creating and deletion

In addition to providing a task object, kernels also provide  ___task-management
services___. Thask0management services include the actions that a kernel perform
behind the scenes to support tasks, for example,  creating and  maintaining  the
TaskControlBlock and task stacks. A kernel also provides and API that allows dev
eloperms to manipulate tasks. Some of the more common operations  with the  task
objects from within the application include:

* creating and deleting tasks
* controlling task scheduling
* obtaining task information

The kernels also provide  _user-configurable hooks_, which  are mechanisms  that
execute programmer-supplied functions, at the time of specific kernel events.
The programmer _registers_ the function with the  kernel by passing a  function-
pointer to a kernel-provided API. The kernel executes this function when the eve
nt of interest occurs. Such events can include:

* event when a task is first created
* when a task is suspended for any reason and a context switch occurs
* when a task is deleted

During the deletion process, a kernel  terminates the task and  frees memory  by
deleting the task's TaskControlBlock (TCB) and tasks's stack. Howerer, when  the
tasks execute, they can acquire memory or access resurces using other kernel obj
ects. If the task is deleted incorrectly, the task might not get to release the-
se resources. For example, assume that a task acquires a semaphore token to  get
exclusive access to a shared data structure. While the task is operating on this
data structure, the task gets deleted. In not handled appropriately, thus abrupt
deletion of the operating task can result in:

* a corrupt data structure, due ti an incomplete write operation
* an unreleased semaphorem which will not be available for other tasks
* an inaccessible data structure, due to the unrelease semaphore.

A _memory leak_ occurs when memory is acquired but not released, which causes the
system to run out of memory eventually. A _resource leak_ occurs when a resource
is acquired but never released, which results in a memory leak because each reso
urce takes up space in memory. Many kernel provide _task-deletion locks_, a pair
of calls that protect a task from being  prematurely  deleted during a  critical
section of code.

### Task Scheduling

Although much of task's states changing is automatic, many kernels provide a set
of API calls that allow developers to control when a  task moves to a  different
state, this capability is called ___manual scheduling___.

* _suspend_ - suspend a task
* _resume_ - resumes a task
* _delay_ - delays a task
* _restart_ - restarts a task
* _get priority_ - gets the current task's priority
* _set priority_ - dynamically sets a task's priority
* _preemption lock_ - locks out higher priority tasks from preempting this task
* _preeption unlock_ - unlocks a preemption lock

` Delaying a task causes it to relinquish the CPU and allow another task to exec
ute, After the delay expires, the task is returned to the task-ready  list after
all other ready tasks at its priority level. A delayed task waiting for an exter
nal condition can wake up after  a set time to check whether a specified conditi
on or event has occured, which is called ___polling___.

Also, the kernel might support _preeption locks_, a pair of calls used for enabl
ing and disabling preemption in applications. This  feature can be  useful if  a
task is executing in a _critical section of code_: one in which the task must not
be preempted by other tasks.

### Obtaining Task Information

* Get ID - get the current task's ID
* Get TCB - get the current task's TCB (Task-Control-Block)

Obraining a TCB only takes a snapshot of the task context: if a task is not dor-
mant (e.g. suspended), its context might be dynamic, and the snapshot information
might change by the time it is used - so info would be deprecated.

## Typical Task Structure

When writing code for tasks, tasks are structured in one of the two ways:
* run to completion - useful for initialization and startup
* endless loop - do the majority of the work in the apps (I/O handling)

___Run-to-completion Taks example (pseudocode)___
```
RunToCompletionTask ()
{
	Initialize application
	Create endless loop tasks
	Create kernel objects
	Delete or suspend this task
}
```
___Endless-Loop Tasks___
```
EndlessLoopTask ()
{
	Initialization code
	Loop Forever
	{
		Body of loop
		Make one or more blocking calls
	}
}
```
## Synchronization, Communication, and Concurrency

Tasks synchronize and communicate amongst themselves by using _intertask primiti-
ves_, which are kernel objects that facilitate synchonization and  communication
between two or more threads of execution. Examples of such objects include pipes,
message queues, signals, semaphores, mutexes, atomic_ops on shared resource, etc.

# Conclusion

* Most RTOS kernels provide task objects and task-management services
* Applications can contain sysem tasks or user-created tasks, each of which has
a name, a unique ID, a priority, a task control block TCB, a stack, task routine
* A realtime apps is composed of multiple concurrent tasks that are  independent
threads of execution, competing on their own for processor execution time.
* Tasks can be in one of three primary states during the lifecycle:
	* Ready
	* Running
	* Blocked
* Priority-based, preempitve scheduling kernels that allow multiple tasks  to be
assigned to the same priority use task-ready lists to help  scheduled tasks run.
* Tasks can run to completion or can run in an endless loop. For tasks that  run
in endless loops, structre the code so that the task blocks, which  allows lower
priority tasks to run.
* Typical task operations that kernel provide for application development inclu-
de task creation and deletion, manual task scheduling and dynamic acquisition of
task information.
