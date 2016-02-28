---
layout: post
title: Overview of Zephyr
cover: cover.jpg
date:   2016-02-27 20:16:26
categories: unsorted
---

##  System fundamentals

Each image contains the app codes and Zephyr kernel code, they are compiled as a single, fully-linked binary. Both app and kernel codes execute as privileged code with a singe share address space ( they should support privilege and thread mode as in cortex-m3, m4)

A Zephyr application consists app specific code, a collection of kernel configuration settings, and at least one Make file. (code + config + Makefile). App’s kernel configuration setting which enable the system to build a kernel tailor-made to best use of the system’s resource.

Target system in Zephyr is known as boards, each board has its own set of hardware devices and capabilities. It's possible to develop a single app to be used by a set of related target systems, or even based on different CPU architecture.
> comment: Seems this feasible feature is not necessary for mcu based applications as they are seldom portable, not as Linux; which may even bing more expense.

##  Organization
The central of Zephyr include two kernel: microkernel, nanokernel. Kernel also includes some auxiliary systems, such as device drivers and networking software. Apps can be developed by micro and nano, or only nano.

Nanokernel is a high-performance, multiple-thread execution environment with a basic set of kernel features. It’s ideal for sparse memory system (2KB for kernel only) or only simple multiple-threading requirements(a set of interrupt handler and a single background task). Examples: sensor hubs, environment sensors, store inventory tags. If an app only use nanokernel, then it can consist unlimited fibers but only one job.

Microkernel supplements the capabilities of the nanokernel to provide a richer set of kernel features. It’s suitable for system with heftier memory(50K to 900K), multiple communication devices(like wifi, BLE), and multiple data processing tasks. Examples: fitness wearables, smart watches, IoT wireless gateways. No limit to the number of fibers or tasks.

## multi-threading
Support multi-threading for three types of execution contexts:
*  **task context**: preemptible thread, for lengthy or complex work. Priority-based, preemptible; also support round-robin time slicing capability.
* **fiber context**: lightweight and non-preemptible thread, typically used for device drivers and performance critical work. Fiber scheduling is priority-based, none-preemptible; so only if a fiber block itself, other lower fibers would not be executed. Also, fiber task if precedence over the task executions.
* **interrupt context**: a special kernel context used to execute ISRs.

## interrupt(ISRs)
Support interrupt nesting. nanokernel supplies ISRs for a few interrupt sources(IRQs), such as hardware timer and uart device. ISRs for other IRQs are supplied by either device driver or app code.
Each ISR can be registered with kernel at compiling time, also can be registerred dynamically. ISRs can be written entirely by C, also permits the assembly.
When an ISR cannot complete the process of an interrupt, it can leverage the synchronization and data passing mechanism to hand off the remaining work to a fiber of task. The microkernel provide a IRQ object type that streamline the handoff to a task in a manner that does not require the device driver or application code to supply an ISR at all. (*can we say that the IRQ object type is a specific task which aims for ‘bottom half’?*)

## clock & Timer
System tick is a 64-bit timer. Nanokernel allows a fiber or thread to perform time-based processing based on system clock. That’s either by a nano API supports a timeout argument, or a timer object can be set to expire after a specified number of ticks.
Microkernel support time-based processing by using timeout(similar as fiber) or timer( a higher layer than system clock?).

## synchronization
Provide 4 types of objects that allow different contexts to synchronize their execution. And they are mainly for tasks, with limited ability by fiber and ISRs.
* **semaphore**: counting semaphore.
* **event**: binary semaphore
* **mutex**: reentrant mutex with priority inversion protection. Similar to event(binary semaphore) by more logic:
 * ensure that only the owner of the associated resource can be release it.
 * support priority-invention
* **nanokernel semaphore**: only for fibers, limited ability to be used by tasks and ISRs.

## data passing
Zephyr kernel provides six type of objects that allow different contexts to exchange data.
These object types are only aiming for tasks(no fiber or ISR)
* **microkernel FIFO**: a queuing mechanism that allow tasks to exchange fixed size data items in an asynchronous First In, First Out manner.
* **mailbox**: a queuing mechanism to exchange variable-size data items in a synchronous, first in, first out manner. It also support asynchronous exchanges. (*mailbox in normal sense*)
* **pipe**: a queuing mechanism allow a task to send a stream of bytes to another task.  Support synchronous and asynchronous exchanges.
> comment: More smooth support for data exchanging between task than existing RTOS.

These object types are designed to be used by fibers, and only limited used by tasks and ISRs.
* **nanokernel FIFO**: similar as microkernel;
* **nanokernel LIFO**: similar as FIFO, but it’s first-in, last-out manner (as stack)
* **nanokernel stack**: exchange 32-bit items in an asynchonous first-in, first-out manner. ( must wrong, should be first in, last out)
> comment: description here from official site must be wrong; should be first in, last out manner. The difference between stack and LIFO here I think should be the data block size: 32 v.s. user defined fixed size.

## dynamic memory allocation
Zephyr kernel requires all system resources at compile-time, we can say it’s a static system. So it only provides limit support for dynamical allocation.
Zephyr provides two types of objects that allow tasks to dynamically allocate memory blocks:
* **memory map**: single fixed size memory block. An app can has multiple memory maps;
* **memory pool**: multiple fixed size. Allow more efficient used by application which can require different size of blocks.
nanokernel does not provide support for dynamical memory allocation.

## public and Private Microkernel Objects
Public objects, such as semaphore, mailbox, tasks.
* public objects: available for general use by all parts of app; include in zephyr.h.
* private objects: only intended for use by a specific parts of application, such as a single device driver or subsystem.
> comment: object here I think we can consider them as the structure and their operations, e.g. object in OOP meaning.

## microkernel Server
The microkernel perform most operations involving microkernel objects using a special micro kernel server fiber, called `_k_server()`. When a task invokes an API associated with a microkernel object type, such as `task_fifo_put()`; the operation is not carried out directly, Instead, the following sequence of steps typically occurs:
1. task creates a command packet, which contains the input parameters needed carry out the desired operation.
2. task queues the command packet on the microkernel server’s command stack. Then kernel preempts the task and schedules the microkernel stack and perform desired operation, all params are in the packets. ( _that means the sever fiber is a scheduler for tasks, that is all tasks are in fact execute in a fiber, interesting! and the datas are passed by the sync and data passing mechanisms._)
3. microkernel dequeues the command packet from its command stack, then perform desired operation. All output params for the operation, such as the return code, are saved in the command packet.
4. when all the operation is complete, the microkernel server attempts to fetch a command packet from its now empty cmd stack, so the `_k_server()` is blocked. The kernel then schedules the requesting task.
5. task processes the command packet’s output parameters to determine the results of the operation.
> Interesting. Concentrated on processing microkernel objects. Simplify the design and logic, more elegant.

## benefits with this approach
* All operations performed by the server can **avoid the race condition** inherently. Operations are processed in sequence and serially by a single fiber that could not be preempt by any fiber or task. This means the microkernel sever can manipulate any of the microkernel objects in the system without during any operation without have to take the additional steps to prevent the interference by other context
* Microkernel operation has **minimal impact on interrupt latency**; interrupt are never locked for a significant period to prevent race conditions. (minimal these kind of operations, all send to the server)
 > comparing with other RTOS, there would be less works to do for the synchronization work. Simple is better.
* micro kernel can easily **decompose the complex operations** into two or more simpler operations by creating additional command packets and queueing them on the command stack.
 > Server can works as a VM, messages are the instructions:)
* overall **footprint of the system is reduced**; a task using microkernel objects only needs to provides stack space for the first step of the above sequence, rather than for all steps required to perform the operation.
> comment: would the communication efficiency be effected with this design? As one more layer is introduced?


## standard C Library
Zephyr kernel currently provides only the minimal subset of the standard C lib required by kernel’s need; primarily in the areas of string manipulation and display.

Apps that require more extensive C lib can either submit contribution to kernel to enhance the existing lib; or substitute a replacement lib.

## device Driver Lib
Device drivers varies according to the hardwares, the drivers which present on all supported boards:
* interrupt controller: used by the nanokernel’s interrupt management subsystem.
* Timer: This device driver is used by the kernel’s system clock and hardware clock subsystem.
* Serial communication: by kernel’s system console subsystem
* random number generator: provide a source of random numbers.


> Reference:
  1. [Zephyr kernel Fundamentals](https://www.zephyrproject.org/doc/kernel/overview/kernel_fundamentals.html)
