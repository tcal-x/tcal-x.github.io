---
layout: post
title: Doing Donuts with Fomu
---

I was Googling something related to Fomu, and hit this tweet from [@enjoy-digital](https://twitter.com/enjoy_digital?lang=en):


[![https://twitter.com/enjoy_digital/status/1313788215409684481](/images/litex-fomu.png)](https://twitter.com/enjoy_digital/status/1313788215409684481)


I must have missed that when it was first sent.  I'd been trying for a while to figure out
how to get a normal LiteX BIOS prompt on Fomu.  I knew that LiteX was in there, from having poked around
[foboot](..), and also from having done the LiteX portion of the Fomu Workshop (spoiler: the workshop
build doesn't include a CPU, and so no BIOS and no prompt).

So here was a recipe, a one-liner, to build a "normal" LiteX on Fomu --- two options in fact,
one with the usual default VexRiscv core, and the other with the tiny serial SERV core.

I made a fresh LiteX checkout in a new virtual environment and tried the recipes.   The build worked...the flash worked....now to connect!    I connected using `lxterm /dev/ttyACM0`, and saw the start of the LiteX banner...but then a hang...oh wait, it was just a pause to calculate the CRC.   Then the rest of the banner, and the green `litex>` prompt.   It worked!

# Donuts

So I went through the rest of Florent's twitter feed to see what else I might have missed.   I found this tasty tweet:


[![https://twitter.com/enjoy_digital/status/1341095343816118272](/images/litex-donut.png)](https://twitter.com/enjoy_digital/status/1341095343816118272)


Hmm, I know how I would run that on an Arty board -- I would use `lxterm --kernel demo.bin /dev/ttyXXXX`.   The LiteX BIOS would attempt a serialboot by sending a magic ASCII string; then lxterm would recognize it and send the binary over the serial connection.

But would that work on Fomu?   I've run RISC-V applications on Fomu, but they were loaded using `dfu-util -D app.dfu`, relying on the magic of the Fomu bootloader.    Since I now have a `litex>` prompt on Fomu, serialboot should work....right?

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

I will quickly summarize my misguided attempt to serialboot the binary into spiflash.   This seemed natural to me at first, since according to my understanding, `dfu-util -D` always wrote to the spiflash.  However, after poking around a bit after having no luck, I realized that spiflash was locked from the perspective of the RISC-V in my LiteX `user bitstream`.   For a few seconds, I thought "Oh, I just need to figure out how to unlock it"...until I remembered how easy it is to brick a Fomu if you accidentally corrupt the bootloader that's stored in spiflash.

So, here's how I loaded demo.bin into sram:

* Edit `demo/linker.ld` to put the sections in `sram` instead of `main_ram`.
* Problem: the code then wants to be loaded at offset 0x0 in `sram`...but BIOS is using that as its working memory during the serialboot.  So the serialboot will be corrupted and/or hang
* Solution: start at an offset 0x3000 by adding a line [`. += 0x3000;`] at the beginning of the first section, `.text`.

After these two changes, the `.text` section looks like this:
```
        .text :
        {
                . += 0x3000;
                _ftext = .;
                *(.text .stub .text.* .gnu.linkonce.t.*)
                _etext = .;
        } > sram
```
The next three sections also get changed to "`> sram`".

Then, you need to clean and remake **in** the demo directory:
```
% make BUILD_DIR=../build/fomu_pvt SOC_DIR=../../../../litex/litex/soc clean
% make BUILD_DIR=../build/fomu_pvt SOC_DIR=../../../../litex/litex/soc
```
You might need to adjust the `SOC_DIR` setting to match your layout.

Ok, to be safe, let's disassemble demo.elf to make sure we're at the intended offset:
```
% riscv64-unknown-elf-objdump -d demo.elf > demo.elf.dis
% head -12 demo.elf.dis

demo.elf:     file format elf32-littleriscv


Disassembly of section .text:

00000000 <_ftext-0x3000>:
        ...

00003000 <_ftext>:
    3000:       0b00006f                j       30b0 <crt_init>
    3004:       00000013                nop
```
Ok, looks good!  Let's try to load it now.





`(litex_term issue)`

(background: foboot, dfu-util -D prog.bin, flash, compare to ram-load)

(donutmark)

# Faster Donuts!




# Still Faster Donuts!

