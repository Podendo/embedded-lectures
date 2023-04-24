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






















