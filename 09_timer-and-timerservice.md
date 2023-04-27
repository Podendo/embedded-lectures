# Introduction

System tasks and user tasks ofter schedule and perform activities after some
time has elapsed. For exa,ple, a RTOS scheduler must perform a context switch
Also, a software-based memory refresh mechanism must refresh the dynamic memo
ry every so often pr data loss will occur.

Timers are an integral part of many real-time embedded systems. A timer is the
scheduling of an event according to a predefined time value in the future, sim
ilar to setting an alarm clock. Most embedded systems use two different forms
of timers to drive time-sesitive activities: _hard timers_ and _soft timers_.

A soft-timer facility allows for efficiently scheduling of non-high-precision
software events. A practical design for the soft-timer handling should have:

* efficient timer maintenance, i. e. counting down a timer
* efficient timer installation, i. e. starting a timer
* efficient timer removal, i. e. stopping a timer

# Real-Time Clocks and System Clocks

In some references. the real-time-clock is interchangeable with the term system
clock. Actually, they are different on various architectures. Commonly, RTC are
integrated with battery-powered DRAM. see RTC ICs and RTC API for linux kernel.

# Programmable Interval Timers

The programmable interval timer (PIT) also known as the timer-chip, is a device
designed mainly to function as an event counter, elapsed time indicator, rate-
controllable periodic event generator, as well as other applications for solving
system-timing control problems.

The functionality of the PIT is commonly incorporated into the embedded process-
or, whre it is called an _on-chip-timer_. The timer interrupt rate is the number
of timer interrupts generated per second. The timer interrupt rate is calculated
as a function of the input clock frequency and is set into a timer control regs.

Customized embedded systems come with schematics detailing the interconnection
of the system components. From these schematics, a developer can determine which
external component are dependent on the timer chip as the input clock source.

Timer-chip initialization is performed as part of the system startup. Generally,
initialization of the timer chip involves the following steps:

* Resetting and bringing the timer chip into a known hardware state
* Calculating the proper value to obtain the desired timer interrupt frequency
	and programming this value into the appropriate timer control register.
* Programming other timer control registers that are related to the earlier int
	errupt frequency with correct values. This steps is dependent on the ti
	mer chip and is specified in detail by the timer chip hardware RM.
* Programming the timer chip with the proper mode of operation
* Installing the timer interrupt service routine into the system
* Enabling the timer interrupt

The behavior of the timer chip output is programmable through the control regist
ers, the most important of whicj os the timer-interrupt-rate-register (TINTR):
	`TINTR = F(x)` where x = frequency of the input crystal.

# Timer Interrupt Service Routines

Part of the timer chip initialization involves installing an interrupt service
routine (ISR) that is called when the timer interrupt occurs. Typically, the ISR
performs these kind of duties:

* Updating the system clock - both the absolute time and elapsed time is updated
	Absolute tume us tume kep in calendar dates, hours, mins, secs, Elapsed
	time is usually kept in ticks and indicates how long the system has been
	running since power-up.

* Calling a registered kfunc to notify the passage of a preprogrammed period -
	as example, the registered kernel function is `called announce_time_tick`

* Acknoledging the interrupt, reinitializing the neccesart time control register
	and returning from interrupt.

The `announce_time_tick` function is invoked in the context of the IRQ, therefo-
re, all of the restrictions placed on an ISR are applicable to this kernel funct
ion `announce_time_tick`. It is called to notify the kernel scheduler about the
occurence of a timer tick.

# A Model for Implementing the Soft-Timer Handling Facility

The functions performed by the soft-timer facility, called the `timer facility`:

* allowing applications to start a timer

* allowing applications to stop or cancel a previously installed timer

* internally maintaining the application timer

The soft timer facility is comprised of two components: one component lives with
in the timet tick ISR and the other component lives in the context of a task.

The timer tick handler must be short and must be conducting the least amount of
work possible. Processing of expired soft timers is delayed into a dedicated pro
cessing task because applications using soft timers can tolerate a bounded timer
inaccuracy. The _bounded timer inaccuracy_ refers to the impression the timer may
take on any value. This value is guaranteed to be within a specific range.

Therefore, a workable model for implementing a soft-timer handling facility is to
create a dedicated processing task and call it a worker task, in conjuction with
its counter part that is aprt of the system timer ISR. The ISR counterpart is gi
ven a fictious name of `ISR_timeout_fn` for this discussion.

The system timer chip is programmed with a particular interrupt rate, which must
accomodate various aspects of the system operation. The assosiated timer tick gr
anularity reqired by the application level soft timers. The `ISR_timeout_fn` fun
ction must work with this value and notify the worker task appropriately.

These application-installed timers are called soft-timers because processing is
not synchronized with the hardware timer tick. It is a good idea to explore this
concept further by examining possible delays that can occur along the delivery
path of the timer tick.

## Possible Processing Delays

The first delay is the event-driven, task-scheduling delay. As shown in the prev
ious example, the maintenance of soft timers is part of `ISR_timeoutfn` and invo
lves decrementing the expiration time value by one. When the expiration time rea
ches zero, the timer expores and the assosiated function is invoked. Because the
`ISR_timeout_fn` is part of the ISR, it must perform the smallest amount of work
possible and postpone makor work to a later stage outside the context of the ISR

The second delay is the priority-based, task-scheduling delay. In a typical RTOS
tasks can execute at different levels of execution priorities.For example, a wor
ker task that performs timer expiration related functions might not have the hig
hest execution priority. In a priority-based, kernel scheduling scheme, a worker
task must wait until all other hightly priority tasks complete execution before
being allowed to continue. With a round-robin scheduler, the worker task must wa
it for its scheduling cycle in order to execute.

Another delay is introduced when an application installs many soft timers.

## Inplementation Considerations

A soft-timer facility shoud allow for efficient timer insertion, timer deletion
and cancellation, and timer update. These requirements, however, can conflict
with each other in practice. For example, imagine the linked list-timer imple
mentation: the fastest way to start a timer is to insert it either at the head
of the tier list or at the tail of the timer list.
So, with linked list, timer installation can be performed in constant time, tim-
er cancellation and timer update migh require O(N) in the worst case. Sorting ex
piration times in ascending order results in efficient timer bootkeeping, so whe
n timer bookkeeping is performed in constant time, timer installation requires
search and insertion. The cos is O(log(N)), where N is the number of entries in
the timer list. The cost of timer cancellation is also O(log(N))

# Timing Wheels

The timing wheel is a construct with a fixed size array in which each slot repre
sents a unit of time with respect to the precision of the soft-timer facility.
The timing wheel approach has the advantage of the sorted timer list for updating
the timer efficiently, and it also provides efficient operations for timer insta
llation and cancellation.

In each timeslot a doubly linked list of timeout event handlers (also named call
back functions or callbacks) is stored, and is invoked upon timer expiration. Th
is list of timer represents events with the same expiration time.

The clock dial increments to the next time slot on each tick and wraps to the be
ginning of the time-slot array when it increments past the final array entry. Th
e idea of the timing wheel is derived from this property.

When installing a new timer event, the current location of the clock dial is used
as the reference point to determine the time slot in which the new event handler
will be stored.

## Issues

Overflow condition, because the number of slots in the timing wheel has a limit,
and each timing slot has a fixed amount of time, we can not take a timestamp lar
ger than a timewheel length. One approach is to deny installation of timers outs
ide the fixed range. A better solution is to acculmulate the events causing the
overflow condition in a temporary event buffer until the clock dial has turned
enough so that these events become schedulable.

Another issue in timing wheels is the precision of the installed timeouts. When
a 150ms timer event is being scheduled while the clock is ticking but before the
tick announcement reaches the timing wheel, Should the timer event be added to
the +150ms slot or placed in the +200ms slot? So on average, the error is approx
imately half the size of the tick. (In this example the error is 25ms).

One othe important issue relates to the invocation time of the callbacks install
ed at each time slot. In theory, the callbacks should all be invoked at the same
time at expiration, but in reality, this is impossible. The work performed by ea
ch callback is unknown; therefore, the execution length of each callback is also
unknown. Consequently, no guarantee or predictable measures exist concerning when
a callback in a later position of the list can be called.

As example, first event handler is invoked at t1 when the timeout has just expir
ed, but event handler n is invoked at tn, when the previous event (n-1) event ha
ndlers have finished execution. The interval starting frim t1 to tn is non-deter
ministic because the length of execution of each handler is unknown. These inter
vals are also unbounded. Ideally, the timer facility could guarantee an upper bo
und; for example, regardless of the number of timers already installed in the sy
stem, event handler n is invoked no later that 200ms from the actual expiration
time. This problem is difficult and the solutin is application-specific.

## Hierarchical Timing Wheels

This timer overflow problem, mentioned earlier, can be solved using the hierarch
ical timing wheel approach. The soft-timer facility needs to accomodate spanning
a range of values. This range can be very large. For example, accommodate timer
events spanning a range of values. This range can be very large. (for example ac
comodating timers ranging from 100ms to 5 minutes requires a timing wheel with
3,000 (5'60'10) entries. Because the timer facility needs to have a granularity
of at least 100ms and there is a single array representing the timing wheel:
```
10 ' 100ms = 1 sec
10 entries/sec

60 sec = 1 minute
60 ' 10 entries/min

therefore:
5 ' 60 ' 10 = total number of entries
		needed for the timing wheel with a 100ms granularity
```

A hiearchical timing wheel is similar to a digital clock. Intstead of having a
single timing wheel, multiple timing wheels are organized in a hierarchical ord-
er. Each timing wheel in the hierarchy set has a different granularity. A clock
dial is assosiated with each timing wheel. The clock dial turns by one unit when
the clock dial at the lower level of the hierarchy wraps around. Using a hierarc
hical timing wheel reqires only 75 (10 + 60 + 5) entries to allow for timeouts
with 100ms resolution and duration of up to 5 minutes.

With a hierarchical timing wheels, there are multiple arrays, therefore
```
The first array (timing wheel 1, unit 100 ms):
100 ' 100ms = 1 sec
10 entries / sec
The second array (timing wheel 2, unit 1 sec):
60 sec = 1 min
60 entries / min
The third attay (timing wheel 3, unit 1 min)
5 entries for 5 minutes
```
The reductiom in space allows for the construction of higher precision timer fac
ilities with a large range of timeout values. For example, it is possiple to ins
tall timeouts of 2 minutes, 4 secinds, and 300 milliseconds. The timeout handler
is installed at the 2-minute slot first. The timeout handler determines that the
re are still 4.3 seconds to go when the 2 minutes is up; the handler installs it
self at the 4-second timeout slot. Again, after 4 seconds elapsing, the same han
dler determines that 300 milliseconds are left before expiring the timer. Final-
ly, the handler is reinstalled at the 300-millisecond timeout slot. The real req
uired work is performed by the handler when the last 300ms expire.


# Soft Timers and Timer Related Operations

Many RTOS provide a set of timer-related operations for external software compon
ents and applications through API sets. These common operations can be cataloged
into these groups:

* __group 1__ provides low-level hardware related operations
Developed and provided by the BSP developers. The group is considered low-level
system operations. Actual function names are implementation-dependent:

 - `sys_timer_enable`, ENA the system timer chip interrupts. Only after this ope
	ration is complete can kernel task scheduling take place.

- `sys_timer_disable` - disables the system timer chip interrupts. After this op
	eration is complete, the kernel scheduler is no longer in effect. Other
	system-offered services based on time ticks are disabled by this operati
	on as well.

- `sys_timer_connect` - installs the system timer ISR into the system exception
	vector table. The new timer ISR is invoked automatically on the next tim
	er interrupt. The installed function is either part of the BSP or the ke
	rnel code and represents the `timer ISR`

- `sys_timer_getrate` - returns the system clock rate as the number of ticks per
	second that the timer chip is programmed to generate.

- `sys_timer_set_rate` - sets the system clock rate as the number of ticks per
	second the timer chip generates. Internally, this operation reprograms
	the PIT to obtain the desired frequency.

- `sys_timer_getticks` - returns the elapsed timer ticks since system power up.
	This figure is the total number of elapsed timer ticks since the system
	was first powered on.


* __group 2__ provides soft-timer-related services, includes the core timer ope-
rations that are heavily used by both the system modules and applications. Eith-
er an independent timer-handling facility or a built-in-one that is part of the
kernel offers these operations. These functions are also implementation-dep-t:

- `timer_create` - creates a timer, allocates a soft-timer structure. Any SW-mod
	ule intending to install a soft timer must first create a timer structu-
	re. The timer structure contains control information that allows the tim
	er handlng facility to update and expire soft timers. A timer created by
	this operation refres to an entry in the soft timers array.
	_Input_: Expiration time, user function to be called at the time exprire
	_Output_: An ID identifying the newly created timer structure
	_Note_: The returned timer ID is also implementation-dependent

- `timer_delete` - deletes a timer. This operation deletes a previously created
	soft timer, freeing the memory occupied by the timer structure.
	_input_: an ID identifying a preiously created timer structure
	_Note_: This timer ID is implementation dependent

- `timer_start` - starts a timer. This operation installs a previously created
	soft timer into the timer-handling facility. The timer beins running at
	the completion of this operation.

- `timer_cancel` - cancels a currently running timer. This operation cancels the
	timer by removing the currently running timer from the timer-handling fa
	cility. As input give to func an ID identifying a created timer struct.

* __group 3__ provides access either to the storage of the realtime/system clock
This is mainly used by user-level applications. The operations in this group int
eract either with the system clock or with the real-time clock. A system utility
library offers these operations. Each operation in the group is given a fictious
name for this discussion. Actual function names are implementation dependent.

- `clock_get_time` - gets the current clock time, which is the current running
	value either from the system clock or from the real-time clock.

- `clock_set_time` - sets the clock to a specified time. The new time is set ei-
	ther into the system clock or into the real-time clock. As input parame-
	ter give to this function a time structure containing secs, mins, hours.

# Conclusion

* Hardware timers (hard timers) are handled within the context of the ISR.
	the timer handler must comform to general restrictions placed on the ISR

* The kernel scheduler depends on the announcement of time passing per tick

* Soft TIMs are built on hard TIMs and're less accurate cause of various delays

* A soft-timer handling facility should allow for efficient timer installation,
	cancellation, and timer bookkeeping

* A soft-tomer facility built using the timing wheel approach provides efficient
	operations for installation, cancellation, and timer bookkeeping.
