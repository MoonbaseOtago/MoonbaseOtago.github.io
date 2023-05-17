---
layout: post
title: VRoom! blog - Adding Branch Prediction to the Trace Cache
---

### Introduction

We've been working on the Trace Cache - if you read back a couple of blog entries you
will remember that it was performing better than not having a trace cache - but suffered
from mispredicted branches (while equivalent non trace-cache cases didn't).

Now we've upgraded the trace cache to include some local prediction history - it gives us another 10%
performance bringing our Dhrystone number up to 11.3 DMips/MHz

## Trace Cache

Our original trace cache had N (N=64) lines of I (I=8) instructions. Each cache line has 1-8 instructions
and a 'next instruction' entry telling the PC controller where to go next. We terminate filling a line
when we find a subroutine call or return - because we want to keep the branch predictor's call/return
predictor up to date and use its results on returns.

In essence each line already has some branch prediction capability, it predicts all the branches 
in the line (a line could be up to 8 branches), and predicts where the last instruction jumps to (or 
what instruction it is followed by).

So the problem we're trying to solve here is: for each trace line where sometimes it runs to completion and
sometimes it takes a branch from the middle (or even the last instruction) of the trace line what happens?. What we need to predict are:

* Which of the first J or the I instructions in the line will be executed
* What will be the next instruction to execute after the Jth instruction is executed

Our normal (non
trace) branch prediction uses a dual predictor that predicts either from a global branch history
or a simple bimodal predictor - we essentially already have the simple predictor in place in our trace line
that predicts a
next instruction address, and that all the valid instructions in the line will be executed.

What we've done is added a local predictor history to each cache line, it's M (initially M=5) bits updated
on every fetch and every branch misprediction, 1 if the prediction should be taken and 0 if it wasn't.

For every line we also have A (initially A=2) sets of &lt;next instruction, num of instructions to issue&gt; tuples, each set also has B (initially B=4) sets of histories and  2-bit counters. Essentially if the line hits in the cache then if its local history mask matches one of the masks and its associated counter isn't 0 then we
predict a non-default branch and that it's destination is from the associated &lt;next instruction&gt; and only the associated
&lt;num of instructions to issue&gt; are issued - otherwise the entire line and the default next instruction
is used.

History masks, counters and &lt;next instruction, num of instructions to issue&gt; tuples are allocated on
mispredicted branches - counters are incremented on correct predictions, and decremented on
mispredictions, and slowly decay if not used over time (as do trace cache entries themselves). If a &lt;next instruction, num of instructions to issue&gt; set runs out of history masks to allocate then another &lt;next instruction, num of instructions to issue&gt; tuple with the same results can be allocated.

Having multiple &lt;next instruction, num of instructions to issue&gt; tuples per line is intended to handle the case
of a line with multiple mispredicted branched in it - something like a decision tree in a case statement
or the like - all this stuff is parameterized - it may turn out that after a lot of simulation we find out
that that doesn't happen much and maybe we just need A=1 (one next address/etc) but a larger B (more masks)
and/or larger/smaller masks.

### Results

On our dhrystone benchmark we now get 10% more performance - we see an issue rate of 11.31 DMips/Mhz - another 10% more
performance above our previous results. We're seeing an issue rate for 6.12 instructions per clock but
are executing 4.15 instructions per clock (IPC). This is with 3 ALUs, if we instantiate 4 ALUs performance
increases by a tiny fraction of a percent, by itself not really worth doing. Theoretically if we can get
the execution rate to match the issue rate we'd see a DMips/Mhz ~16-17 - another 50% more performance. 

Why aren't wee seeing this? largely it seems to be because of stalls in the memory subsystem, mostly
loads and stores waiting for stores that haven't resolved their addresses yet - there are various ways
to potentially resolve this issue, mostly speculative loads that might cause cache activity and as a result
leave us open to Meltdown like security issues - we've been pretty conservative in this area and probably
wont return to this issue for a while.

Despite this &gt;4 IPC is pretty amazing,  4 IPC was our initial goal for carefully crafted media/etc
kernels, exceeding it with branchy, compiled C code is great!.
it with 

Dhrystone has also been done to death here, it's been useful because it's helped to balance our
fetch/decode and execution stages, next time we do serious benchmarking we'll probably build
a much wider range of tests (we need to tweak the FP performance for example).

### What's Next?

It's time to work on a vector ALU(s) - we have all the components in place and a core that 
can feed them efficiently.

Next time: Vectors!
