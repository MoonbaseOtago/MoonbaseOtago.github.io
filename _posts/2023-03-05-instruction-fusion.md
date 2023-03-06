---
layout: post
title: VRoom! blog - Experiments in Macro Op Fusion
---

### Introduction

Spent some time this week experimenting with macro op fusion. Many of the
arguments around the RISCV ISA have been addressed with "you can solve that
with macro op fusion" which is the idea that you can take adjacent instructions,
next to each other, in an instruction stream and merge them into a single instruction
that by itself can't be expressed directly in the instruction set.

We've always planned that VRoom! would support fusion that made sense in the
renamer stage (post decode) - today's work is a trial experiment fusing some
common sequences just to get a feel for how hard they are to implement.

## Limits

First some limits:

- we're going to avoid creating new instructions that would require
adding more read/write register ports to ALUs
-  we're also going to avoid (this time around)
trying to deal with instructions where the second of a pair might fault and not be executed.
For example if we fuse 'lui t0, hi(xx);ld t0, lo(xx)(t0)' the load might take a page fault
and get restarted expecting the side effect of the lui had actually happened - it's not that
this can't necessarily be done, just that this is hard and we're performing a simple experiment

## Choices

We decided to choose 3 simple instruction pairs, these occur reasonably often in instruction streams
and because they involve fused branches and ALU operations they particularly work well with
our change a while back to merge our ALU and branch units, the downside is that previously
it was possible to build ALUs which shared the adder used by normal addition operations
and the magnitude comparator used for branches, now we will need to be able to do both at once.

The first choice is to merge an add immediate, or an add to x0 instruction, and an indirect branch that doesn't
write an output register, this catches a whole bunch of common code that happens at the end
of a subroutine:

	add	sp, sp, 40
	ret

	li	a0, 0
	ret
	
	mv	a0, s0
	ret

The one limitation is that the destination of the add must not be the indirect register used by the jump/ret.

The second and third choices marry a load immediate to a register (add immediate to x0) instruction and a conditional
branch comparing that register with another into one instruction

	li	a0, #5
	beq	a0, a1, target

	li	a0, #5
	beq	a1, a0, target


becomes:

	li      a0, #5;beq	#5, a1, target

	li      a0, #5;beq	a1, #5, target

This mostly means the branch instruction runs one clock earlier, which makes no difference in 
performance if it
is correctly predicted (a predicted branch just checks that it did the right thing and
quietly continues on), but reduces mispredicted branch times by effectively 1 clock.

All 3 optimizations free up an ALU slot for another instruction.

### Implementation

We perform this optimization in our renamer - remember we have 8 renamers renaming up to 8 decoded
instructions from the previous clock. We add logic to each renamer unit to recognize the types of
addition operations we want to recognize for the first instruction, and pass that information to the
next renamer (the first renamer gets told "no instructions match").

Each renamer also recognizes the second instruction of the pair and gates that with the match from the
previous renamer in the chain. Care is taken to make logic paths that do not go through the full 
layer of renamers.

If a renamer matches a branch (and a previous add) then it modifies the 'control' field (the portion
that describes when sort operation is being done, it also tags itself as now creating an output register
(and tags the addition's output register), it also copies the immediate portion of the addition
and the source register (in the ret operation case which it fetches as rs2).

If a renamer discovers that it's feeding an add to a fused pair then it marks itself as not creating
an output (doesn't write a register), this effectively makes it a no-op, the commit stage
will mark this instruction as done and it will never be scheduled to an ALU. The renamer logic
already contains a layer that packs down decoded instructions, if we used that to remove this instruction
it would create circular logic that wouldn't be synthesisable, If we removed this in the next stage
(added a 'pack' layer to the logic that feeds the commitQ) it would be very expensive and would
break the renaming (which renames to future commitQ locations) - so instead we create a no-op
entry (but see below).

The actual implementation in the ALU is pretty simple, we have to allow for a r2+immed addition
and for immediates to be fed into the branch comparator.

### Further optimization

You likely noticed that we can't optimize add/branch pairs that cross instruction bundle boundaries (the boundaries in the instruction stream that our 4-8 instruction decoder decodes from). Once we implement
a trace cache instruction streams will be presented to the renamer again, each time they are replayed,
if such a split instruction pair is presented again here they can be re-optimized. 

If we're running a trace cache we can also purge no-op instructions from the cached instruction stream
as it's filled.

### Performance

Here's a simple example of an add/ret optimization, before the return (in black)

![placeholder](/public/images/before1.png "pipeline before change")

and after:

![placeholder](/public/images/after1.png "pipeline after change")

Notice that the add (at 0x36d4) is nullified, all that green in the instructions around there
means that the ALUs are busy so removing one helps. Those blue dotted lines coming down out of the
add instruction (dependencies on the new SP) now come out of the branch (return).

So we know it's working - how much faster is it on our favorite dhrystone benchmark? Like last time it makes no
difference at all - at the moment on that benchmark our performance there is completely limited by the issue
rate from the instruction fetch/decoders.

We actually don't expect these optimizations to help dhrystone much anyway as all it's branches are 
predicted, and the big advantage of executing branches early is that it speeds up mispredictions. However
it will also speed up instruction streams that are ALU bound as it frees up a slot.

Any design is a balance between the decode rate and the
rate that the decoded instructions are executed, that balance is going to change with different
sets of code.
Essentially our current bunch of changes, and last week's, are building pressure on the execution side
of that trade off, we can't easily speed up the decoder but we will return to our trace cache design
which should be able it issue 8 instructions/clock to support that we'll need more execution capability.
	



Next time: More misc extensions
