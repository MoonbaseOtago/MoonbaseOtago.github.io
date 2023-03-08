---
layout: post
title: VRoom! blog - More Experiments in Macro Op Fusion
---

### Introduction

This week we've done some more experiments in macro op fusion, and found an unrelated
surprising performance boost - ~12% to 9.76 DMips/Mhz - almost at 10!


## Limits

A reminder from last time, we've set some limits on our macro op fusion experiments:

- we're going to avoid creating new instructions that would require
adding more read/write register ports to ALUs
-  we're also going to avoid (this time around)
trying to deal with instructions where the second of a pair might fault and not be executed.
For example if we fuse 'lui t0, hi(xx);ld t0, lo(xx)(t0)' the load might take a page fault
and get restarted expecting the side effect of the lui had actually happened - it's not that
this can't necessarily be done, just that this is hard and we're performing a simple experiment

## Choices

Last time we merged return instructions and preceding add instructions, and constant loads and compares.

This time around we've add 4 more optimizations: merged lui/auipc instructions and add instructions, and lui instructions and load/store instructions. These are the 'traditional' candidates for fusion.

Planning ahead for this we'd previously plumbed 32-bit constants everywhere through our design, up until now
they've either been high 20 bit lui/auipc values or low 12-bit signed constants. Now we've added logic to
combine these values - if we limit merging to constants that the decoder actually makes then it doesn't
require a full 32-bit adder per renamer (7 of them for 8 renamers 
to merge the data between two adjacent units) instead the lower 12 bit just get passed and the upper 20 bits
have an optional decrementer. We share this logic for the 4 newly merged instructions.

The new 4 instructions pair we're supporting are:

1)

	lui	a, #hi(X)
	add	a, a, #lo(X)

becomes:

	nop
	lui	a, #X

2)

	auipc	a, #hi(X)
	add	a, a, #lo(X)

becomes:

	nop
	auipc	a, #X 

3)

	lui	a, #hi(X)
	ld.x	y, #lo(X)(a)

becomes:

	lui	a, #hi(X)
	ld.x	y, #X(x0)

4)

	lui	a, #hi(X)
	st.x	y, #lo(X)(a)

becomes:

	lui	a, #hi(X)
	st.x	y, #X(x0)

Some notes:
- as per last time nops get swallowed in the commit stage
- the new auipc created in 2) gets the pc of the original pc
- load/store instructions still perform the lui (so that exceptions can be restarted, it doesn't
break the second limitation above), but can run a clock
earlier because they don't have to wait for the lui - we may remove this limitation in the future



### Performance

Like previous optimizations this doesn't affect Dhrystone numbers at all, particularly because
Dhrystone doesn't use any of these in its core. Nothing surprising here .... however ....

### More Performance

However this week we also did some 'maintenance'. We'd been using an old library for our benchmarks,
but a few weeks back we switched to supporting the B and K RISCV architectural extensions, the B
extension in particular contains instructions designed to help with string processing and strcmp
is a core part of Dhrystone, this week we upgraded the library to use the new libraries, we also optimized
them for VRoom! (mostly this involved aligning code to 16-byte boundaries).

The results have been better than we expected - a 12% increase in Dhrystone numbers, 9.76 DMips/MHz - a 
better number than we'd previously assumed possible without a trace cache - we're
now seeing an IPC of 3.58 (max 8) and an issue rate of 3.66 (out of 8). In the future with a functioning
trace cache (and a n issue rate close to 8)  we can predict that the max performance could be
in the ~20 DMips/MHz range (assuming that
our execution side improvements keep up, at this point they're already ahead a bit).

So excellent news, not from recent changes, but essentially due some of the changes due to updates a month or so ago that had not fully been counted yet.

Next time: More misc extensions
