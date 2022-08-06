---
layout: post
title: VRoom! blog&#58; Rename Optimizations
---

### Introduction

The past few weeks we've been working on two relatively simple changes in the register renamer: renaming
registers known to be 0 to register x0, and optimizing register to register moves.

These are important classes of optimizations on x86 class CPUs, a real hot topic - previously I worked
on an x86 clone
where instructions were cracked into micro-ops - lots of these moves were generated by this process and
optimization is important there. On RISC-V code streams we see relatively fewer (but not 0) register moves.

This particular exercise isn't expected to have a great performance bump, but it's the same spot in the
design that we will also be doing opcode merging so it's a chance to start to crack that part of the
design open.

Our renamer essentially does one thing - it keeps track of where the live value (from the point of view of
instructions coming out of the decoder or the trace cache) of architectural registers are actually stored,
they could either be in the architectural register files, or the commit Q register file - once an instruction
hits the commit Q it can't be executed until all it's operands are either in the commit Q register file (ie
done), or in the architectural register file.

![placeholder](/public/images/trace1.svg "CPU Architecture")

The goal of these optimizations is not so much to remove
move instructions from the instruction stream but to allow subsequent instructions that depend on them
execute earlier (when the source register for the move instruction is ready rather than it's destination register is).

### 0 register operations

This optimization is very simple - on the fly the renamer recognizes a bunch of arithmetic operations as generating
'0' - add/xor/and/or.

For example "li a0, 0" is really "add a0,x0,0", or "xor a0,t0,t0" etc - it's a
simple operation and is very cheap.

(by the time they hit the renamer all the subtract operations look like adds)

### Move instruction operations

These essentially optimize:

	mov	a0, a1
	add	t1, a0, t5

into:

	mov	a0, a1
	add	t1, a1, t5

Note: they don't remove the mov instruction (that's a job for another day) as there's not enough
information in the renamer to tell if a0 will be subsequently needed. The sole goal here is to allow the two
instructions to be able to run in the same clock (or rather for the second instruction to run earlier
if there's an ALU slot available).

Recognizing a move in the renamer is easy, we're looking for add/or/xor with one 0 operand.

Managing the book keeping for this is much more complex than the 0 case - largely because the
register being moved could be in 1 of 4 different places - imagine some code like:

	sub	X, ....
	....
	mov	Y, X
	...
	add	,,Y

The live value for operand 'Y' to the add instruction could now be in:

- the actual architectural register X
- the commitQ entry for the sub instruction
- the commitQ entry for the mov instruction
- the actual architectural register Y

The commitQ logic that keeps track of registers now needs be more subtle and recognize these state changes. 
As does the existing logic in the renamer that handles the interactions between instructions
that are decoded in the same clock and that update registers referenced by each other.

### Conclusion

There's not discernible performance bump for dhrystone here (though we can see the operations happening at
a micro level) - there's only 1 move and a couple of 0 optimizations per dhrystone loop and they're
just reducing pressure on the ALUs - that's not surprising.

We think this is worth keeping in though, as we start running more complex benchmarks, especially
ALU bound ones we're going to see the advantages here. 

### A note on Benchmarking

While checking out performance here we noticed a rounding bug in the code we've been running on the simulator
(but not on the FPGAs). The least significant digits of the "Microseconds for one run through Dhrystone"
number we've been quoting have been slightly bogus - that number is not used for calculation of the rest of the
numbers (including the Dhrystone numbers) so those numbers remain valid.

We recompiled the offending code and reran it for a few past designs ... they gave different (and lower) 
numbers (6.42 vs. 6.49 DMips/MHz) than we had before .... after a lot of investigation we've decided
that this is caused by code alignment in the recompiled code (another argument for a
fully working trace cache).

We could hack on the code to manually align stuff and make it faster (or as fast as it was) but
that would be cheating - we're just compiling it without any tweaking. Instead we'll just note that this
change has happened and start using the new code and compare with the new numbers. What's important
for this optimization process is knowing when the architecture is doing better.

### Next time

We've been putting off finishing the FPU(s) - we currently have working FP adders (2 clocks) and
multipliers (3 clocks) that pass many millions of test vectors, some of the misc instructions still need work, and it all still needs to be integrated 
into a new FP renamer/scoreboard. Then on to a big wide vector unit .....

Next time: FPU stuff