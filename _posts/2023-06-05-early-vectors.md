---
layout: post
title: VRoom! blog - Early Vector Processor Architecture

---

### Introduction

We've started work on our vector support, trying to figure out how to fit it into our existing
design, this blog entry is about a couple of possible ways to implement RISC-V's vector instruction set
on VRoom! and the tradeoffs between them.

### Background

First a reminder of how our basic architecture works:

![placeholder](/public/images/trace1.svg "trace cache")

Instructions are decoded, renamed and the inserted into the commitQ. ALUs are assigned to instructions 
from the commitQ when they're ready (when the registers they use are available), they're executed
out of order, but instructions near the end of the queue get priority. Instructions are taken from
the end of the commitQ when they are ready to be committed (all instructions prior them will also committed)
then their results are written back to the architectural register file.

A vector implementation will likely have multiple vector ALUs, possibly of different types depending on the throughput we want to target.

RISC-V's vector instruction set is different from the one in most mainline CPUs, some have compared
it more with Cray's vector instructions. The main difference is that each instruction may operate on a 
variable sized range of vector registers (or memory locations) - a simple add might be a SIMD add of registers V1-V3 to V4-V6 leaving the result
in V7-V9 - there are architectural registers that describe how long a vector is, and support to automatically
split vectors up into architecturally meaningful chunks. With this ISA we can run out
of architectural vector registers pretty fast (there are 32) quite fast, the commitQ's large number of associated 
renamed registers is a big advantage here, it means we can be working on far more registers at once.

There are also new architectural registers that describe the length (and shape) of vectors for subsequent instructions,
to build an out-of-order system they need to be renamed and/or predicted in some way.

### Design 1 - multi-cycle instructions

The first design we looked at issues vector instructions to the vector ALUs and expects them to execute
multiple beats (clocks) worth of execution depending on the vector length. Instructions would be issued
to ALUs when the first register comes ready and would stall if subsequent registers are not available. Multiple vector ALUs could dynamicly form pipelines.

Advantages:

* relatively simple
* vector length handling is easily renameable

Disadvantages:

* renamer needs to be able to represent ranges of registers
* beats must be done in order
* stalls badly on a cache miss, no other instruction can be scheduled to that ALU until it's done
* throughput is low (load/store unit can load/store 4 registers per clock but a vector unit like this would retire 1 register per clock)
* retiring instructions becomes harder (one instruction might use more write ports than the transfer pipe has available)
* commitQ entries need to be able to hold multiple uncommitted instruction results

### Design 2 - renamer breaks instructions into beats

In the second design we have the renamer break each vector instruction into multiple beats - issuing cloned
copies of the instruction into the commitQ. This has the
advantage that they can then be done out of order and spread among multiple vector ALUs and be executed in parallel. 

Advantages:

* ALU use is per register, much more like how everything else works today
* more parallelism available, we can do multiple parts of an instruction in parallel
* everything can be aggressively out of order
* load store unit can read/write multiple registers per clock 

Disadvantages:

* renaming the length information is much more difficult - we will need to know it before the renamer can progress - sometimes this will be easy, sometimes the renamer will have to stall until the value has been calculated (when vset gets executed)
* renamer becomes messy - probably becomes a 2-clock process at higher clock speeds (we've long though this was likely anyway)

### What's Next?

Initially we're going to implement the second option, probably with some vector length prediction hardware
(a bit like a BTC mixed in with some knowledge of how "strip mining" normally works).

So probably decode, a simple adder ALU and the renamer work to feed it.

Next time: More Vectors!
