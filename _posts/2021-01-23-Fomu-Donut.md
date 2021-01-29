---
layout: post
title: Doing Donuts on Fomu
---

I was Googling something related to Fomu, and hit this tweet from [@enjoy-digital](https://twitter.com/enjoy_digital?lang=en):

Fomu tweet: https://twitter.com/enjoy_digital/status/1313788215409684481

I must have missed that when it was first sent.  I'd been trying for a while to figure out
how to get a normal LiteX BIOS prompt on Fomu.  I knew LiteX was in there, from having poked around
[foboot](..), and also from having done the LiteX portion of the Fomu Workshop (spoiler: the workshop
build doesn't include a CPU, and so no BIOS and no prompt).

So here was the recipe, a one-liner, to build a "normal" LiteX on Fomu --- two options in fact,
one with the usual default VexRiscv core, and the other with the tiny serial SERV core.

I made a fresh LiteX checkout in a new virtual environment and tried the recipes.   The build worked...the flash worked....now to connect!    I connected using lxterm, and saw the start of the LiteX banner...but then a hang...oh wait, it was just a pause to calculate the CRC.   Then the rest of the banner, and the green `litex>` prompt.   It worked!

# Donuts

So I went through the rest of Florent's twitter feed to see what else I might have missed.   I found this tasty tweet:

Donut demo: https://twitter.com/enjoy_digital/status/1341095343816118272
(screencap?)

Hmm, I would know how to run that on an Arty board -- I'd use `lxterm --kernel demo.bin`.   The LiteX BIOS would attempt a serialboot by sending a magic ASCII string; then lxterm would recognize it and send the binary over the serial connection.

But would that work on Fomu?   I'd run RISC-V applications on Fomu, but they were loaded using `dfu-util -D app.dfu`, relying on the magic of the Fomu bootloader.    But since I now have a `litex>` prompt on Fomu, serialboot should work....right?

Well, let's build the demo and find out:
```
% litex_bare_metal_demo --build-path=build/fomu_pvt
 CC       isr.o
 CC       donut.o
 CC       main.o
 LD       demo.elf
riscv64-unknown-elf-ld:linker.ld:17: warning: memory region `main_ram' not declared
chmod -x demo.elf
 OBJCOPY  demo.bin
chmod -x demo.bin
```
Ah, that warning is not one we can ignore.   The program's linker specification is looking for a memory region `main_ram` that is not defined by the LiteX build.   The Arty LiteX build, for example, maps the `main_ram` region to the Arty's DDR3 memory.   But let's look at what Fomu has:

```
% cat build/fomu_pvt/software/include/generated/regions.ld
MEMORY {
        sram : ORIGIN = 0x00000000, LENGTH = 0x00020000
        spiflash : ORIGIN = 0x80000000, LENGTH = 0x01000000
        rom : ORIGIN = 0x80060000, LENGTH = 0x00008000
        csr : ORIGIN = 0x82000000, LENGTH = 0x00010000
}
```



`(litex_term issue)`

(background: foboot, dfu-util -D prog.bin, flash, compare to ram-load)

(donutmark)

# Faster Donuts!




# Still Faster Donuts!

