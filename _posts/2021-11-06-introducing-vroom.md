---
layout: post
title: Blog - Introducing Vroom!
---
![placeholder](/talk/assets/chip.png "Branch Target Cache example")

### Executive Summary

* Very high end RISC-V implementation – goal cloud server class
* Out of order, super scalar, speculative
* RV64-IMAFDCHB(V)
* Up to 8 IPC (instructions per clock) peak, goal ~4 average on ALU heavy work
* 2-way simultaneous multithreading capable
* Multi-core
* Early (low) dhrystone numbers: ~3.6 DMips/MHz - still a work in progress. Goal ~4-5
* Currently boots Linux on an AWS-FPGA instance
* GPL3 – dual licensing possible

### Downloads

VRoom! is hosted with GitHub. Head to the <a href="https://github.com/MoonbaseOtago/vroom">GitHub repository</a> for downloads.

### Licensing

VRoom! is currently licensed GPL3. We recognize that for many reasons one cannot practically build a large GPL3d chip 
design - VRoom! is also available to be commercial licensed.


