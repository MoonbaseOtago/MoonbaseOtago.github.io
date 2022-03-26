---
layout: post
title: Vroom! blog&#58; Core VRoom! Architecture
---


### Introduction

This week we're going to try and explain as simply as possible how our core architecture works. Here's an
overview of the system:

![placeholder](/talk/assets/overview.svg "System Architecture")

The core structure is a first-in-first out queue called the 'commitQ'. Instruction-bundles
are inserted in-order at one end, and removed in-order at the other
end once they have been committed.

While in the commitQ instructions can be executed in any order (with some limits like memory
aliasing for load/store instructions). At any time, if a branch instruction is discovered to have
been wrongly predicted, the instructions following it in the queue will be discarded.

A note on terminology - previously I've referred to 'decode-bundle's which are a bunch of bytes fetched from the i-cache 
and fed to the decoders. Here we talk about 'instruction-bundle's which are a bunch of data being passed around 
through the CPU representing an instruction and what it can do - data in an instruction-bundle includes its PC,
its input/output registers, immediate constants, which ALU type it needs to be fed to, what operation will be
performed on it there, etc.

You can think of the decoders as taking a decode-bundle and producing 1-8 instruction-bundles 
per clock to feed to the renamer.


### Registers and Register Renaming

We have a split register file:

![placeholder](/talk/assets/registers.svg "Register Files")

On the right we have the 31 architectural integer register and the 32 floating point registers. If the commitQ is empty
(after a trap for example) then they will contain the exact state of the machine.

The commit registers are each staticly bound to one commitQ entry.

When an instruction-bundle leaves the decode stage it contains the architectural register numbers of its source
and destination registers, the renamer pipe stage assigns a commitQ entry (and therefore a commit register) to
each instruction and tags its output register to be that commit register.

The renamer also keeps a table of the latest 
commit register that each architectural register is bound to (if any) - it uses these tables to point each source
register to either a particular commit register or an architectural register.

### Execution


Once an instruction-bundle is in the commitQ it can't be assigned an ALU until each of its source registers
is either an architectural register or it's a commit register that has reached the 'completed' state (ie it's
either in a commit register, or is being written to one and can be bypassed). Once a commitQ entry is in this state
it goes into contention to be assigned an ALU and will execute ASAP.

Once a commitQ entry has been executed its result goes into its associated commit register, and the commitQ
entry waits until it can be committed 

Each commitQ entry is continually watching the state of it's source registers, if one of them moves from the commitQ
to the architectural registers it updates the source register it will use when executing.

### Completion

In every clock the CPU looks at the first 8 entries in the commitQ, it takes the first N contiguous entries 
that are completed, marks them 'committed' and removes them from the queue.

Committing an entry causes 
its associated commit register to be written back into the architectural registers (if multiple commit entries
write the same architectural register only the last one succeeds). It also releases any store instructions
in the load-store unit's write buffer to be actually written to caches or IO space.

### Speculative Misses

As mentioned above when a branch instruction hits a branch ALU and is discovered to have been mispredicted - either a 
conditional branch where the condition was mispredicted, or an indirect branch where the destination was mispredicted - then
the instructions in the commitQ after the mispredicted instruction are discarded and the instruction fetcher
starts filling again from that new address.

While this is happening the renamer must update its state, to reflect the new truncated commitQ.

Branch instructions effectively act as barriers in the commitQ, until they are resolved into the completed
state (ie their speculative state has been resolved) instructions after them cannot be committed - which means that
their commit registers can't be written into the architectural registers, and data waiting to be stored into cache or main memory
can't be written.

However instructions after an unresolved branch can still be executed (out of order) using the results of
other speculative operations from their commit registers. Speculative store instructions can 
write data into the load-store unit's write 
queue, and speculative loads can also snoop that data from the write queue and from the cache - I'll
talk more about this in another blog post.

Also branches can be executed out of order resulting in smaller/later portions of the commitQ being discarded.

### Traps and Interrupts

Finally - the CSR ALU (containing the CSR registers and the trap and interrupt logic) is special - only
an instruction at the very beginning of the commitQ can enter the CSR - that effectively means only one CSR/trap instruction
can be committed in that clock (actually the next clock) this acts as a synchonising mechanism.

Load/store traps (MMU/PMAP/etc) are generated by converting a load/store tagged instruction in the commitQ into a trap
instruction.

Fetch traps (again MMU/PMAP but also invalid instructions) are injected into the commitQ by the instruction fetcher
and decoder which then stalls.

Interrupts are treated much the same as fetch traps, they're injected into the instruction stream which then stops
fetching.

When any of these hit the CSR unit all instructions prior to them have been executed and committed. They then trigger
a full commitQ flush, throwing away all subsequent instructions in the queue.
They then tell the fetcher to start fetching from a new address.

Some other CSR ALU operations also generate commitQ flushes: some of the MMU flush operations, loading page tables, fence.i etc. We also optimise interrupts that occur when you write a register to unmask interrupts and make them synchronous (so that we don't end up executing a few instructions after changing the state).

### To Recap
The important ideas here:

* we have two register files one for speculative results, one for the architectural registers
* our commitQ is an in-order list of instructions to execute
* each commitQ entry has an associated output register for its speculative result
* commitQ instructions are executed in any order
* however instructions are committed/retired in chunks, in order
* at that point their results are copied from the speculative commit registers to the architectural registers
* badly speculated branches cause the parts of the commit Q after them to be discarded

Next time: Memory Layout
