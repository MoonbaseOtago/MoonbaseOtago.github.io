---
layout: post
title: VRoom! blog&#58; Memory Layout
---

### Introduction

A short post this week about physical memory layout and a little bit about booting. I'll talk more
about virtual memory another time

### Physical memory layout

We currently use a 56-bit physical address, this is the address used with the MMU disabled or after a
virtual address has been translated. 

Addresses with bit 55 (the most significant bit) set to 0 are treated as cacheable memory space - instruction
and data accesses go through separate L1 caches but the cache coherency protocol makes sure that they see the
same data. Cache lines are 512 bits.

Everything in this space is cache coherent, once the system is booted it's the only place that code can
be executed from. Because it's always coherent, the fence.i instruction is a simple thing for us, all it has to do is to
wait for the writes to drain from the store queue and then flush the commitQ (fences go in the commitQ so all
that happens when it reaches the end of the queue), subsequent instruction fetches
pull live modified data from the data cache into the instruction cache.

At the moment we have either a fake memory emulator on the simulator, or connect to DRAM on the AWS FPGA 
implementation - both implementations back onto the coherent MOESI bus

Within this space main memory (DRAM) starts at address 0x00000000. What happens if you access memory
outside of the area where there's real DRAM is undefined, with the proviso that it wont lock up
(writes probably get lost or wrap around, reads may return undefined data - it's up to the real
memory controller to choose how it behaves - on a big system the memory controller may be on another die). 

Addresses with bit 55 set are handled differently for data accesses and instruction accesses:
* instruction accesses got to a boot ROM at a known fixed address, the MMU never generates these addresses
* data accesses go to a shared 64-bit IO bus (can be accessed through the MMU)

The current data IO bus contains:
* a ROM containing a device tree image
* timers
* a uart
* gpio
* multiple interrupt controllers PLIC/CLNT/CLIC
* a fake disk controller

### Booting

Currently reset causes the CPU to jump to the start of code compiled into hardware in the special instruction space.
Currently that code:
* checks a GPIO pin (reads from IO space)
* if it's set it jumps to address 0 (this is for the simulator where main memory can be side loaded)
* if it's clear we read a boot image from the fake disk controller (connected to test fixture code on
the simulator, and to linux user space on the AWS FPGA world) into main memory, then jump to it
(currently we load uboot/OpenSBI)

Longer term we plan to put the two L1 caches into a mode at reset where the memory controller is disabled
and the data cache allocates lines on write, the instruction cache will use the existing cache coherency protocol
to pull shared cache lines from the data cache. The on-chip bootstrap will copy data into L1 from an SD/eMMC, validate it
(a crypto signature check), and jump into it. This code will initialize the DRAM controller, run it through
its startup conditioning, initialize the rest of the devices, take the cache controller out of
its reset mode and then finally load OpenSBi/UEFI/uboot/etc into DRAM.

Next time: Building on AWS
