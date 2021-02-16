---
layout: post
title: Doing Donuts with Fomu
---

I was Googling something related to Fomu, and hit this tweet from [@enjoy-digital](https://twitter.com/enjoy_digital):

<hr>

[![https://twitter.com/enjoy_digital/status/1313788215409684481](/images/litex-fomu.png)](https://twitter.com/enjoy_digital/status/1313788215409684481)

<hr>
<center>
<a href="https://twitter.com/enjoy_digital/status/1313788215409684481"><img src="/images/litex-fomu.png" width="75%"></a>
</center>
<hr>

I must have missed that when it was first sent.  I'd been trying for a while to figure out
how to get a normal LiteX BIOS prompt on Fomu.  I knew that LiteX was in there, from having poked around
[foboot](https://github.com/im-tomu/foboot), and also from having done the [LiteX portion](https://workshop.fomu.im/en/latest/migen.html) of the Fomu Workshop (spoiler: the example build doesn't include a CPU, and so no BIOS and no prompt).

So here was a recipe, a one-liner, to build a "normal" LiteX on Fomu --- two options in fact,
one with the usual default VexRiscv core, and the other with the tiny serial SERV core.

I made a fresh LiteX checkout in a new virtual environment and tried the recipes.   The build worked...the flash worked....now to connect!    I connected using "`lxterm /dev/ttyACM0`", and saw the start of the LiteX banner...but then a hang...oh wait, it was just a pause to calculate the CRC.   Then the rest of the banner, and the green "`litex>`" prompt.   It worked!

# Donuts

So I went through the rest of Florent's twitter feed to see what else I might have missed.   I found this tasty tweet:

<hr>

[![https://twitter.com/enjoy_digital/status/1341095343816118272](/images/litex-donut.png)](https://twitter.com/enjoy_digital/status/1341095343816118272)

<hr>

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
Ah, that is a warning that we cannot ignore.   The program's linker specification is looking for a memory region `main_ram` that is not defined by the LiteX build.   The Arty LiteX build, for example, maps the `main_ram` region to the Arty's DDR3 memory.   But let's look at what Fomu has:

```
% cat build/fomu_pvt/software/include/generated/regions.ld
MEMORY {
        sram : ORIGIN = 0x00000000, LENGTH = 0x00020000
        spiflash : ORIGIN = 0x80000000, LENGTH = 0x01000000
        rom : ORIGIN = 0x80060000, LENGTH = 0x00008000
        csr : ORIGIN = 0x82000000, LENGTH = 0x00010000
}
```

I will quickly summarize my misguided attempt to serialboot the binary into spiflash.   This seemed natural to me at first, since according to my understanding, "`dfu-util -D`" always wrote to the spiflash.  However, after poking around a bit after having no luck, I realized that spiflash was locked from the perspective of the RISC-V in my LiteX "user bitstream".   For a few seconds, I thought "Oh, I just need to figure out how to unlock it"...until I remembered how easy it is to brick a Fomu if you accidentally corrupt the bootloader that's stored in spiflash.

So, here's how I loaded demo.bin into sram:

* Edit "`demo/linker.ld`" to put the sections in "`sram`" instead of "`main_ram`".
* Problem: the code then wants to be loaded at offset 0x0 in "`sram`"...but BIOS is using that as its working memory during the serialboot.  So the serialboot will be corrupted and/or hang
* Solution: start at an offset 0x3000 by adding a dummy section before the `.text` section:

After these two changes, the lines around the `.text` section looks like this:
```
        .dummy : { . += 0x3000; } > sram

        .text :
        {
                _ftext = .;
                *(.text .stub .text.* .gnu.linkonce.t.*)
                _etext = .;
        } > sram
```
The next three sections also get changed to "`> sram`".

Then, you need to clean and remake **in** the demo directory:
```
% make BUILD_DIR=../build/fomu_pvt clean
% make BUILD_DIR=../build/fomu_pvt
```

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
Ok, looks good!  Let's try to load it now, at the 0x3000 offset that we chose:

```
% lxterm  --kernel demo.bin --kernel-adr 0x00003000 /dev/ttyACM0
```

Hmm, it hung.   After much investigation, I found that earlier versions of `litex/tools/litex_term.py` did work!  But there's a more direct fix: make this change in `litex/tools/litex_term.py`:
```diff
-sfl_payload_length = 255
-sfl_outstanding    = 128
+sfl_payload_length = 64
+sfl_outstanding    = 0

```

See [Litex Issue #73](https://github.com/enjoy-digital/litex/issues/773) for more information.

With this fix, serialboot worked!  
```
--============== Boot ==================--
Booting from serial...
Press Q or ESC to abort boot completely.
sL5DdSMmkekro
[LXTERM] Received firmware download request from the device.
[LXTERM] Uploading ./demo.bin to 0x00003000 (9236 bytes)...
[LXTERM] Upload complete (1.0KB/s).
[LXTERM] Booting the device.
[LXTERM] Done.
Executing booted program at 0x00003000

--============= Liftoff! ===============--

LiteX minimal demo app built Feb 12 2021 19:12:26

Available commands:
help               - Show this command
reboot             - Reboot CPU
led                - Led demo
donut              - Spinning Donut demo
litex-demo-app>
```


Let's watch the donut spin...get to the menu, and type 'donut\<ret\>'.

Hmm, nothing's happening.  Actually, I typed something into chat that I kind of got the demo working, but the donut part wasn't working.   But when I came back to my `litex_term` window, there was a donut!  It was working...just really slowly.   I timed it, and measured one update evry XX seconds.


# Faster Donuts!

Of course, I wanted to see if I could make the donut animation go faster.   Looking at the code, I saw a few multiplies, so I checked if the the VexRiscv build was using the hard multipliers.   Nope, it was performing each multiply iteratively in software.   It should speed things up quite a bit if we can use the DSP blocks on the ICE40.  


To build a new version of VexRiscv, go to `pythondata-cpu-vexriscv/pythondata_cpu_vexriscv/verilog/` and load the submodules.
```
git submodule update --init
```
Also, you'll need to install `sbt` to build a new VexRiscv. (reference?)


The existing recipe in the Makefile for the `VexRiscv_Min` variant is:

```make
VexRiscv_Min.v: $(SRC)
        sbt compile "runMain vexriscv.GenCoreDefault --iCacheSize 0 --dCacheSize 0 --mulDiv false --singleCycleShift false --singleCycleMulDiv false --bypass false --prediction none --outputFile VexRiscv_Min"
```

Let's make a new target, minimal EXCEPT for adding the DSP multipliers (`--mulDiv true` and `--singleCycleMulDiv true`), so we'll call it "MinMult":
```make
VexRiscv_MinMult.v: $(SRC)
        sbt compile "runMain vexriscv.GenCoreDefault --iCacheSize 0 --dCacheSize 0 --mulDiv true --singleCycleShift false --singleCycleMulDiv true --bypass false --prediction none --outputFile VexRiscv_MinMult"
```

and now make it:
```
make VexRiscv_MinMult.v
```

We also need to make changes to LiteX (temporarily) to tell it about the new variant that we just created.  Find the file `litex/litex/soc/cores/cpu/vexriscv/core.py`.   Add the following lines:
```
TODO
```

Rebuild the gateware.

Rebuild the demo with the variant=minmult

In my run, with the single cycle mult, I only hit 44MHz, but the tool is conservative; it worked fine on my Fomu. The design used 97% of the logic cells, so it's a pretty tight fit.



.... ALSO enables single cycle shift. That won't fit unless you do even more hacking.





# Still Faster Donuts!

