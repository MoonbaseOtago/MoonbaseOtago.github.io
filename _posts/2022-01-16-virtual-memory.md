---
layout: post
title: VRoom! blog&#58; Virtual Memory
---

### Introduction

Booting Linux isn't going to work without some form of virtual memory, RISC-V has
a well defined spec for VM, implementation isn't hard - page tables are well defined,
there's nothing particularly unusual or surprising there

### L1 TLBs

We have separate Instruction and Data level one TLBs, they're fully associative which means that
we're not practically limited to power of two sizes (though currently they have 32 entries each),
each entry contains a mapping between an ASID and upper bits of a virtual address and a physical
address - we support the various sizes of pages (both in tags and data).

A fully associative TLB with random replacement makes it harder for Meltdown/Spectre sorts of
attacks to use TLB replacement to attack systems. On VROOM memory accesses that miss in the TLB never result in
data cache changes.

Since the ALUs, and in particular the memory and fetch units, are shared between HARTs (CPUs) in the same
simultaneous multi-threaded core then so are the TLBs shared.  We need a way to distinguish mappings between HARTs - this implementation
is simple, we reserve a portion of the ASID which is forced to a unique value for each core - each HART thinks it has N bits of ASID, in reality there are N+1. There's also
a system flag that we can optionally set that lets SMT HARTs share the full sized ASID (provided the software understands
that ASIDs are system rather than CPU global).

Currently we use the old trick of doing L1 TLB lookup with the upper bits of a virtual address while using
the lower bits in parallel to do the indexing of the L1 data/instruction caches - large modern cache line sizes 
mean that you have to go many ways/sets to get large data and instruction caches - this also helps
with Meltdown/Spectre/etc mitigation.

I'm currently redesigning the whole memory datapath unit to split TLB lookup and data cache access
into separate cycles - mostly to expose more parallelism during scheduling - more about that in
a later posting once it's all working.

### L2 TLBs

TLB misses result in stalled instructions in the commitQ - there's a small queue of pending TLB 
lookups in the memory unit and 2 pending lookups in the fetch unit - they feed the table walker
state machine which starts by dipping into the L2 TLB cache - currently it's a 4-way 256 entry (so
1k entries total) set associative cache shared between the instruction and data TLBs - TLB data found here is fed directly to the L1 TLBs (a satisfied L1 miss takes 4
clocks).

### Table walking

If a request also misses in the L2 TLB cache the table walker state machine starts walking page table trees.

Full
cache lines of TLB data are fetched into a local read-only cache which contains a small number of entries,
essentially enough for 1 line for each level in the page hierarchy, and 2 for the lowest level, repeated 
for instruction TLB and the data TLB (then repeated again for multiple HARTs).

After initial filling most table walks hit in this cache. This cache is slightly integrated into the data L1 I-cache, they share a 
read-only access port into the cache coherency fabric, and both can be invalidated by data cache snoops shootdowns.

### TLB Invalidation

TLB invalidation is triggered by executing a TLB flush instruction - these instructions let the instructions
before them in the commitQ execute before they themselves are executed. 

At this point they trigger a
commitQ flush (tossing any speculative instructions executed with the old VM mappings). At the same time
they trigger L1 and L2 TLB flushes. Note: there is no need to invalidate the TLB walker's small data cache
as it will have been invalidated (shot down) by the cache coherency protocols if any page tables were
changed in the process.

Next time: (Once I get it working) Data memory accesses
