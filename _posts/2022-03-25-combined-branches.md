---
layout: post
title: VRoom! blog&#58; Combining ALUs and Branch Units
---

### Introduction

Wow, this was a fun week <a href="https://news.ycombinator.com/item?id=30755716">VRoom! made it to the front page of HackerNews</a> - for those new here this is an occasional blog post on architectural issues as
they are investigated - VRoom! is very much a work in progress.

This particular blog entry is about a recent exploration around the way that we handle branches
and ALUs.

### Branch units vs ALU units

The current design has 1 branch unit per hart (a 'hart' is essentially a RISCV CPU
context - an N-way simultaneous multi-threaded machine has N harts even if they share
memory interfaces, caches and ALUs). It also has M ALUs - currently two.

After a lot of thinking about branch units and ALU units it seemed that they have a lot in
common - both have a 64-bit adder/subtracter in their core, and use 2 register read ports and
a write port. Branch units only use the write port for call instructions - some branches, unconditional
relative branches don't actually hit the commitQ at all, conditional branches simply check that the
predicted destination is correct, if so they become an effective no-op, otherwise they
trigger a commitQ flush and a BTC miss.

So we've made some changes to the design to be able to optionally make a combined ALU/branch unit,
and to be able to build the system with those instead of the existing independent branch and ALUs units
(it's a Makefile option so relatively easy to change). Running a bunch of tests on our still
useful Dhrystone we get:


| Branch | ALU | Combined | DMips/MHz | Area | Register Ports |
|:------ |:--- |:-------- |:--------- |:---- |:-------------- |
| 1      | 2   | 0        | 6.33      | 3x   | 6/3            |2022-03-25-combined-branches.md
| 0      | 0   | 2        | 6.14      | 2x   | 4/2            |
| 0      | 0   | 3        | 6.49      | 3x   | 6/3            |
| 0      | 0   | 4        | 6.49      | 4x   | 8/4            |

Which is interesting - 3 combined Branch/ALU units outperform the existing 1 Branch/2 ALU with
roughly the same area/register file ports. So we'll keep that.

It's also interesting that 4 combined ALUs performs exactly the same as the 3 ALU system (to the clock) even
though the 4th ALU gets scheduled about 1/12 of the time - essentially this is because because we're scheduling ALUs
out of order (and speculatively) the 3rd ALU happily takes on that extra work without changing how fast the
final instructions get retired. 

One other interesting thing here, and the likely reason for much of this performance improvement is
that we can now retire multiple branches per clock - we need to be able to do something sensible if
multiple branches fail branch prediction in the same clock - the correct solution is to give priority
to the misprediction closest to the end of the pipe (since the earlier instruction should cause the later one
to be flushed from the pipe).

What's also interesting is: what would happen if we build a 2-hart SMT machine? previously such a system
would have had 2 branch units and 2 ALUs - looking at current simulations a CPU is keeping 1 ALU busy
close to 90%, the second to ~50%, the 3rd ~20% - while we don't have any good simulation data yet we
can guess that 4 combined ALUs (so about the same area) would likely satisfy a dual SMT system - mostly
because the 2 threads
would share I/Dcaches and as a result run a little more slowly (add that to the list of future
experiments).

### VRoom! go Boom!

Scheduling N units is difficult - essentially we need to look at all the entries in the commitQ and
choose the one closest to the end of the pipe ready to perform an ALU operation on ALU 0. That's easy
the core of it looks something a bit like this (for an 8 entry Q): 

	always @(*)
	casez (req) 
	8'b???????1: begin alu0_enable = 1; alu0_rd = 0; end
	8'b??????10: begin alu0_enable = 1; alu0_rd = 1; end
	8'b?????100: begin alu0_enable = 1; alu0_rd = 2; end
	.....
	8'b10000000: begin alu0_enable = 1; alu0_rd = 7; end
	8'b00000000: alu0_enable = 0;
	endcase

for ALU 1 it looks like (answering the question: what is the 2nd bit if 2 or more bits are set):

	always @(*)
	casez (req) 
	8'b??????11: begin alu1_enable = 1; alu1_rd = 1; end
	8'b?????101,
	8'b?????110: begin alu1_enable = 1; alu1_rd = 2; end
	8'b????1001,
	8'b????1010,
	8'b????1100: begin alu1_enable = 1; alu1_rd = 3; end
	.....
	8'b00000000: alu1_enable = 0;
	endcase

For more ALUs it gets more complex for a 3264 entry commitQ it's also much bigger, for dual
SMT systems there are 2 commitQs so 64/128 entries (we interleave the request bits from the
two commitQs to give them fair access to the resources).

Simply listing all the bit combinations with 3 bits set out of 128 bits in a case statement
just listing all of them gets unruly - but really we do want to express this in a manner
where we can provide a maximum amount of parallelism to the synthesis tools we're using, hopefully
they'll find optimizations that are not obvious - so long ago we had reformulated it
to something like this:

	always @(*)
	casez (req) 
	8'b???????1: begin alu0_enable = 1; alu0_rd = 0;
			casez (req[7:1]) 
			7'b??????1: begin alu1_enable = 1; alu1_rd = 1; end
			7'b?????10: begin alu1_enable = 1; alu1_rd = 2; end
			7'b????100: begin alu1_enable = 1; alu1_rd = 3; end
			.....
			7'b1000000: begin alu1_enable = 1; alu1_rd = 7; end
			7'b0000000: alu1_enable = 0;
			endcase
		     end
	8'b??????10: begin alu0_enable = 1; alu0_rd = 1; 
			casez (req[7:2]) 
			6'b?????1: begin alu1_enable = 1; alu1_rd = 2; end
			6'b????10: begin alu1_enable = 1; alu1_rd = 3; end
			6'b???100: begin alu1_enable = 1; alu1_rd = 4; end
			.....
			6'b100000: begin alu1_enable = 1; alu1_rd = 7; end
			6'b000000: alu1_enable = 0;
			endcase
		     end
	8'b?????100: begin alu0_enable = 1; alu0_rd = 2; 
	.....
	8'b10000000: begin alu0_enable = 1; alu0_rd = 7; alu1_enable = 0; end
	8'b00000000: begin alu0_enable = 0; alu1_enable = 0; end
	endcase

etc etc we have C code that will spit this out for an arbitrary number of ALUs (arbitrary depths)
the 2 ALU scheduler for a 32 entry commitQ happily compiles under verilator/iverilog and on
Vivado (yosys currently goes bang! we suspect upset by this combinatorial explosion). When we switched
to 3 ALUs (we tried this a while ago) it happily compiled on verilator (takes a while) and runs. When
we compiled up the 4 ALU scheduler on verilator it went Bang! the kernel OOM killer got it (on a
96Gb laptop) - looking at the machine generated code it was 200K lines of verilog .... oops ... 3 ALUs
was compiling 50k, 2 ALUs ~10k .... serious combinatorial explosion!

Luckily we'd already found another way to solve this problem elsewhere (we have 6! address units)
so dropping some other code in to generate this scheduler wasn't hard (900 lines rather than 200k) -
we're going to need to
spend some time looking at how well this new code performs, it had always been expected to be
one of the more difficult areas for timing - we might need to find some 95% heuristics here that are
not perfect but allow us higher core clock speeds - time will tell. Sadly Yosys still goes bang, must be something else.

### Conclusion

Combined branch ALUs seem to provide a performance improvement with little increase in area - we'll
keep them.

Next time: Probably something about trace caches, might take a while
