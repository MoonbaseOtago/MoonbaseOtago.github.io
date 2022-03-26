---
layout: post
title: VRoom! blog&#58; Memory Parallelism
---

### Introduction

I've not posted in 2 months, mostly because I've been spending time redesigning the load/store unit
this posting is about these changes. This is a long post, but then I've been doing a lot of work.

### The problem

Back when I was bringing up Linux on the system I spent a lot of time looking at low level
assembler trace, watching instructions flow through the CPU it became obvious that one of
the bottlenecks is when large numbers of store instructions occur together they form an
effective block to earlier scheduling of loads, in particular this happens at the beginning
of subroutine calls when registers are being saved. 

Ideally a subroutine should start loading
data from memory as soon as possibly after entry - in practice code can't load a value into say 
s0 until after it has been saved on the stack - however we have an out-of-order CPU with register
renaming, it can happily load s0 BEFORE it is saved - provided there's no memory aliasing between the
place where the save is being done to and where the load is being done from. Ideally we'd like to be
able to push as many load instructions before all those register save instructions as possible, we want to 
get any cache miss from main memory started as soon as possible, then we can save the
registers into the storeQ and then into cache while we wait.

### The old design

The old memory design issued N loads and M stores in every clock (variously 2:1, 2:2, and 3:2) - loads and
stores executed in one clock (either into the store queue or retired if there was a cache/storeQ snoop hit.

We used an old trick, the L1 TLB is fully associative, the L1 data cache is set associative - we can look up
the TLB at the same time that we do the SRAM index portion (with address bits that are not translated by the
VM system) and then compare the output of the TLB (the
physical address) with the fetched data cache tags to choose a set - this gives us a lot of useful
very low level parallelism in the load/store unit - currently we run it in 1 clock (maybe not at 5GHz :-).

These days there are some downsides to
this design - essentially it's a general computer architecture problem: page sizes have not been
increasing while cache line sizes and cache sizes have -
the data cache index is generated from bits of the page index (the 12 LSBs of a virtual address) that
are the same for a virtual and physical address - page sizes really haven't changed since the early 80s,
and RISC-V uses the same 4k/12-bits that have been used since then - but cache lines have gotten
longer and caches have got larger- we
use a 512-bit cache line (64 bytes) - that's 6-bits of addressing - leaving 6-bits for indexing the
data cache.
This means that a direct mapped cache (ie 1 set) can at most have 64x64 bytes - ie 4K - to get a 64k
L1 cache we need 16 sets, 128k needs 32 sets. One can't have a large cache without large numbers
of sets.

There are some advantages to having large numbers
of sets mostly around Meltdown/Spectre/etc using random replacement and large numbers of sets muddies
the water for those sorts of exploits - the downsides are essentially a 16:1 or 32:1 mux (and a larger fanout between the TLBs and the comparators) in a critical path.

![placeholder](/public/images/ls.svg "Load Store Unit")

Getting back to the reason for this redesign - the big problem though is that in order to safely reorder the execution of load and store instructions 
safely we need to be able to compare their physical addresses when we're doing the scheduling (not their
virtual addresses as their might be aliasing) - and here we have a design where the TLB lookup
and physically tagged data cache lookup are deeply intertwined. So it has to go ....

There's another issue - the storeQ - this implicitly orders stores (and loads waiting for stores, or
for cache fills) -
it's a queue, embedded in it's design is an ordering, transactions must be retired in order, when a store
reaches the head of the queue AND it's associated instruction has reach the commit state (ie there's no
chance of a branch misprediction or a trap will cause it not be executed)`it attempts to update the
L1 cache, if it gets a miss it triggers a cache line fetch and a subsequent update. This means that we can't reorder stores to disjoint addresses post commit - and limits the number of cache line fetches we can
have in parallel. The queue is nice in that it inherently embeds ordering of operations, but in
reality most operations don't require this. Again this also needs to go ....

### The new design

So here's the new design, it got a lot bigger ....:

![placeholder](/public/images/ls-new.svg "Load Store Unit")

First thing to note it's a 2 clock latency design - first clock is triggered when the address register in
a load or store is available and just performs the TLB lookup portion of an operation.
The second clock is triggered when all the addresses conflicts have been 'resolved' (that means that there
are no instructions touching the same bytes that must be done in instruction order that haven't been pushed into the storeQ) and, if the instruction
is a store, the data to be stored is available from the pipe.

In the second clock load transactions either
get resolved because they hit in the dcache or are snooped (from as yet uncommitted stores) from the storeQ. 
All store and fence transactions, IO transactions, loads that miss in dcache/snoop, loads that suffer
hazards (for example a 4 byte load that discovers a single byte in that range waiting to be stored in the
storeQ), all of these cases get put into the storeQ.

### Pending transactions

The minimum transaction time is now 2 clocks, but it can be many more - we're processing A address
transactions (TLB lookups) per clock - we're simulating with 6, but will likely have to drop back to 4
on the FPGA system. Some
transactions can't be completed right away, some store transactions might still be waiting for the data to
store to become available, some might be hung up because there's an instruction ahead of them that needs
to be executed (in this context 'executed' means a load that hits in the icache or snoops in the storeQ,
or any other instruction that has be pushed into the storeQ), or because there's a preceding instruction that
hasn't been through the address unit (and therefore we can't tell if there might be a load/store dependency.

The Pending Transactions buffer is a list of transactions (one entry for every instruction entry in
the commitQ) that have been through the address unit - each entry contains a physical address, a mask of
the bytes it will read or write (8 bits on a 64-bit machine), fence/amo dependency information and a hazard
bitmask. The hazard bitmask is similar to the 'hazard' structure that we used in the old storeQ (and in the
new storeQ described below) essentially each entry contains a bitmask (1 bit for every other pending
transaction), each bit is set if that other pending transaction entry blocks this one.

For example: a load transaction might be blocked by one or more stores to one or more of the bytes the
load loads, it might also be blocked by a fence or amoXXX instruction - it wont leave the pending
transactions store until all of these blocking instructions have also left - it will likely find these
hazards again (and maybe more, new ones) as it's entered into the storeQ via the snoop operation.

### Load/Store Units

The load store units have a scheduler that looks at how many free storeQ entries are ready (have 0 hazards)
and chooses
N load/store operations to perform, it prioritizes loads because in general we want to move loads
before stores wherever possible and because we also use the load unit to retire completed loads (and
things like AMOXXX, LC and SC) - each load unit has a dedicated write port into the general register file.

The load/store scheduler bypasses the pending transaction buffer in a way that it picks up entries from
the address unit that are about to be written, this is how we can do 2 clock operations.

The load unit starts a dcache access and a snoop operation into the storeQ and then ....
- I/O accesses go straight into the storeQ
- if the snoop into the storeQ hits then it returns the newest data (effectively if there are multiple hits it's the entry that has hazard bits that none of the others have) and the instruction completes
- if the snoop hits but it returns a hazard (something we need to wait on to complete before we can proceed,
like a fence, or a partially written location) we put the load into the storeQ
- if there are no hazards and the dcache hits we return the dcache data
- otherwise we put the entry into the storeQ

Stores and fences are similar, they always go into the storeQ, we perform a simpler snoop
just to discover hazard information.

### The new store queue

As mentioned above the storeQ is no longer a simple queue of entries where stuff gets put in at one end and
interesting stuff happens to entries that reach the other end (and are ready to be committed).

Now it's more of a heap, with multiple internal ad-hoc queues being created dynamically on the fly. This is
done as part of the snoop we used to look for speculatively satisfying load transactions from as yet
uncommitted stores, and a search for hazards (as described above) - we still do that but now we search for
a wider class of 'hazards' that in also represent store-store ordering (making sure that stores to the
same location
occur in the same order), load-store dependencies (loads from the same location as stores occur after the 
ones that are before them in instruction order) and store-load dependences (stores don't occur until after
the loads that read the current data in a location have completed), we also use hazards to represent
fence and amo-style blockages.

A 'hazard' is actually represented by a bit mask with one bit for each storeQ entry, a storeQ entry is
created with the hazards detected when the storeQ is snooped at the load/store unit stage. Logic keeps
track of which hazard bits are valid, clearing bits as transactions ahead in the ad-hoc queues are retired
a storeQ entry is not allowed to proceed until all its hazards have been cleared. 

Store entries also keep track of activity in the cache lines they would access - the load/store unit has
an implicit understanding with the underlying cache coherency fabric that it will never make multiple
concurrent transaction requests for the same cache line. Now if one storeQ entry makes a request for a
cache line the others must wait (but when it arrives they all get woken up). With this new storeQ we 
don't have to wait for an entry to reach the head of the queue and we can now have many
outstanding cache transactions (before it was just one store and a few loads).

### Summary

We're currently at the point where this new design passes the sanity tests, I'm sure it's still buggy
and I need to spend some times writing some architectural white-box tests trying to poke in the
obviously hard to trigger corners.

It's big! lots of NxM stuff - but then this whole design was always based on asking the question "what if
we throw gates at it?" we have to do lots of comparisons to discover where the hazards are if we want
to get real memory parallelism.

I'm currently simulating a 6:4:4:6 system - 6 address units, 4 load and 4 store units and 6 write ports to
the storeQ (so no more than 6 of the load/store units can be scheduled per clock, this area needs some work).
4 load units means 4 read ports on the dcache, there's also still only 1 dcache write port there which
limits write bandwidth, and how fast the storeQ can drain, this also needs some work, and will change
in the future.

## Performance

I've done a small amount of micro-benchmarking - close examination of code sequences show that the area
I was particularly targeting (write bursts at the beginning of subroutines due to register saves) perform
well, the address unit fills to capacity (they're all offsets from the same register SP, and they're ready
to schedule almost right away), the store unit also fills, and the load units fill as soon as the commitQ
starts presenting loads, at that point stores start sitting as pending transactions and loads pass them 
to be handled - which is also one of our goals.

The main downside is that our one clock load has become a 2 clock load, places where we load an address
from memory and then immediately load something from that address suffer.

I had thought I was done with dhrystone, but it still proves to be useful as a small easy to run 
test that exposes microarchitectural issues so here are the results:

Dhrystone on the old design sat at 4.27 dhrystone/MHz - we recently switched to a clang compile because we
expected it to expose more parallelism to the instruction stream - all numbers (before and after)  here
are running that same code image and now is at 5.88 which meets our original architectural
target (a hand wavy number  "5+" pulled out of thin air ...) - not a bad increase.

Dhrystone is a very branchy test, we can issue up to 8 instructions per clock but
running dhrystone we only get 3.61 - that limits how full our pipe can get and how much parallelism can
likely happen - on this new run we see an average of 2.33 instructions executed per clock (it can't
be larger than that 3.61 issue rate). This is still a useful benchmark as it's worth spending time
figuring out where those IPC are being lost, I'm pretty sure the 2-clock load is now getting some 
of it, but on the whole it's been a great trade for an almost 40% performance increase.
Note that that 3.61 issue rate/2.33 IPC implies that the largest dhrystone/MHz we can
reach with this code base is effectively with tweakage is ~9.1.

Switching to clang seems to have exposed a BTC issue, which needs to be fixed which might push us to
~6.2ish (more hand waving here), but after that probably the next big dhrystone boost will come
from installing an L0 instruction trace cache
that should bust the 3.61 issue rate up to close to 8 and expose more parallelism to this new load/store
architecture - that's for another day.

### Future Work

This has been a big change, it needs more testing, I need to move it back onto the AWS/FPGA platform
for more shaking out, that in itself will be a big job, I'll probably build a 4:2:2:4 system and even
then it may not fit in a VU9P (anyone want to contribute a VU13/VU19 board to the cause?). That's going
to take a long time.

The new storeQ and the pending transactions block have evolved to be quite similar - I can't help but feel
there's fat there that can be removed. The one main sticking point is data life times, the pending
transactions are tied 1-1 with commitQ entries (just located elsewhere because otherwise routing might be a
nightmare) while storeQ entries can long outlive the commitQ entries that birth them (a storeQ
entry can't issue a dcache change, or the memory request to fill the dcache entry it wants to change
until after its associated commitQ entry has been committed, and is about to be recycled).

I also want to start some more serious work on timing, not to make a real chip but to give me more
feedback on my microarchitecture decisions (I'm sure some stuff will blow up in my face :-) to that end I've
started investigating getting a build into OpenLane/Skyworks - again not with the intent of
taping something out (it' probably too big for any of the open source runs) but more as a reality check.


Next time: Verilog changes
