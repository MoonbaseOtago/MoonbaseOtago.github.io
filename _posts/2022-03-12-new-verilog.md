---
layout: post
title: Verilog changes, new performance numbers
---

### Introduction

Not really an architectural posting, more about tooling, skip if it's not really your thing, also new
some performance numbers.

### New Performance Numbers

I found the bug in the BTC - mispredicted 4-byte branches that crossed a 16 byte boundary (our
instruction bundle size) this is now fixed and we have better dhrystone numbers: 6.33 DMIPS/MHz
at an IPC of 2.51 - better than I expected! We cans till do better.

### More System Verilog, Interfaces for the (mostly) win

Last week I posted a description about my recent major changes to the way that the load/store
unit works, what I didn't touch on is some pretty major changes to the ways the actual Verilog code 
is written.

I'm an old-time verilog hack, I've even written a compiler (actually 2 of them, you can
thank me for the \* in the "always @(\*)" construct). Back in the
90s, I tried to build a cloud biz (before 'cloud' was a word) selling time with it for that last
couple of months when you need zillions of licenses and machines to run them on. Sadly we were
probably ahead of our time, and we got killed by Enron the "smartest guys in the room" who made 
any Californian business that depended on the price of electricity impossible to budget for. I
opened sourced the compiler and it died a quiet death.

I started this project using Icarus Verilog, using a relatively simple System Verilog subset,
making sure I could also build on Verilator once I started running larger sims.

I'm a big fan of Xs, verilog's 4-value 'undefined' values - in particular using them as clues to
synthesis about when we don't care and using them during simulation to propagate errors when
assumptions are not met (and to check that we really don't care)

For example - a 1-hot Mux:

	always @(*)
	casez (control) // synthesis full_case parallel_case
	4'b1000: out = in0;
	4'b0100: out = in1;
	4'b0010: out = in2;
	4'b0001: out = in3;
	default: out = 'bx;
	endcase

One could just use:

	always @(*) 
	casez (control) // synthesis full_case parallel_case
	4'b1???: out = in0;
	4'b?1??: out = in1;
	4'b??1?: out = in2;
	4'b???1: out = in3;
	endcase

Chances are synthesis will make the same gates - but in the first case simulation is more likely to blow up
(and here I mean blow up 'constructively' - signalling an error) if more than 1 bit of
control is set. "full_case parallel_case" is a potentially dangerous thing, it gives the synthesis tool
permission to do something that might not be the same as what the verilog compiler does during simulation
(if the assumption here that control is one-hot isn't met) the "out = 'bx" gets us a warning as the Xs 
propagate if our design is broken.

Xs are also good for figuring out what really needs to be reset, propagating timing failures etc etc

Sadly Verilator doesn't support Xs, while Icarus Verilog does. Icarus Verilog also compiles
a lot faster making for far faster turn around for small tests - I had got to a point where I would
use Verilator for long tests and Icarus for actual debugging once I could reproduce a problem "in the
small".

One of the problems I've had with this project is that in simple verilog there really wasn't a
way to pass an arbitrary
number of an interface to an object - for example the dcache needs L dcache read ports, where L is the
number of load ports - you can pass bit vectors of 1-bit structures, but not an array of n-bit structures
like addresses, or an array of data results.

System Verilog provides a mechanism to do this via "interfaces" - and this time around I have embraced them
with a passion - here's NLOAD async dcache lookups:

	interface DCACHE_LOAD #(int  RV=64, NPHYS=56, NLOAD=2);     
		typedef struct packed {
			bit [NPHYS-1:$clog2(RV/8)]addr; // CPU read port
		} DCACHE_REQ;
		typedef struct packed {
			bit hit;
			bit hit_need_o;                                     
			bit [RV-1:0]data;
		} DCACHE_ACK;
		DCACHE_REQ req[0:NLOAD-1];
		DCACHE_ACK ack[0:NLOAD-1];
	endinterface

Sadly Icarus Verilog (currently) doesn't support interfaces and I've had to remove them from our
supported tools - there's still stuff in the Makefile for its support (I live in hope :-). Even Verilator's
support is relatively minimal, it doesn't support arrays within arrays, nor does it support
unpacked structures (so debug in gtkwave is primitive).

I'm really going to miss 4-value simulation, I hope IV supports interfaces sometime - I wish
I had the time to just do it and offer it up, but realisticly one can really only do one Big Thing at a time.

What this does mean though is that longer term a lot of those places where I'm currently making
on-the-fly verilog from small C programs to get around the inability to pass arbitrary numbers
of a thing to a module (the register file comes to mind), or using ifdef's to enable
particular interfaces will likely go away in the future to be replaced by interfaces. They won't completely
go away, I still need them to make things like arbitrary width 1-hot muxes - and more generally
for large repetitive things that could be ruined by fat finger typos.

You can find the new load/store interfaces in "lstypes.si".


Next time: Not quite sure yet - watch this space
