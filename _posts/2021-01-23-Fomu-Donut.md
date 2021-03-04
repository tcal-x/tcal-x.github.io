---
layout: post
title: Doing Donuts with Fomu
---

I was Googling something related to Fomu, and hit this tweet from [@enjoy-digital](https://twitter.com/enjoy_digital):

<hr>

[![https://twitter.com/enjoy_digital/status/1313788215409684481](/images/litex-fomu.png)](https://twitter.com/enjoy_digital/status/1313788215409684481)

<hr>

I must have missed that when it was first sent.  I'd been trying for a while to figure out
how to get a normal LiteX BIOS prompt on Fomu.  I knew that LiteX was in there, from having poked around
[foboot](https://github.com/im-tomu/foboot), and also from having done the [LiteX portion](https://workshop.fomu.im/en/latest/migen.html) of the Fomu Workshop (the example built there doesn't include a CPU, and so no BIOS and no prompt).

So here was a recipe, a one-liner, to build a "normal" LiteX on Fomu --- two options in fact,
one with the usual default VexRiscv core, and the other with the tiny serial SERV core.

I made a fresh LiteX checkout in a new virtual environment, changed directory to `litex-boards/litex_boards/targets`, and tried the recipes.   The build worked...the flash worked....now to connect!    I connected using "`lxterm /dev/ttyACM0`", and saw the start of the LiteX banner...but then a hang...oh wait, it was just a pause to calculate the CRC.   Then the rest of the banner, and the green "`litex>`" prompt.   It worked!  


# Donuts

So I went through the rest of Florent's twitter feed to see what else I might have missed.   I found this tweet:

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

We have plenty of room in "`sram`" for the demo, so let's load it there.   Here's how to do it:

* Edit "`demo/linker.ld`" to put the "`.text`", "`.rodata`", and "`.data`" sections in "`sram`" instead of "`main_ram`".
* Problem: the code then wants to be loaded at offset 0x0 in "`sram`"...but BIOS is using that as its working memory during the serialboot.  Writing there while the serialboot code is using it
will cause the serialboot will to hang, or corrupt the executable being loaded.
* Solution: start at an offset 0x3000 by adding a dummy section before the `.text` section:

This is the diff of the changes to demo/linker.ld:
```diff
--- linker.ld-orig
+++ linker.ld
@@ -7,12 +7,17 @@

 SECTIONS
 {
+        .dummy :
+        {
+                . += 0x3000;
+        } > sram
+
        .text :
        {
                _ftext = .;
                *(.text .stub .text.* .gnu.linkonce.t.*)
                _etext = .;
-       } > main_ram
+       } > sram

        .rodata :
        {
@@ -21,7 +26,7 @@
                *(.rodata .rodata.* .gnu.linkonce.r.*)
                *(.rodata1)
                _erodata = .;
-       } > main_ram
+       } > sram

        .data :
        {
@@ -32,7 +37,7 @@
                _gp = ALIGN(16);
                *(.sdata .sdata.* .gnu.linkonce.s.*)
                _edata = .;
-       } > main_ram
+       } > sram

        .bss :
        {
```

Then, you need to clean and remake **in** the demo directory:
```
% cd demo
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
--- a/litex/tools/litex_term.py
+++ b/litex/tools/litex_term.py
@@ -202,8 +202,8 @@ sfl_prompt_ack = b"\x06"
 sfl_magic_req = b"sL5DdSMmkekro\n"
 sfl_magic_ack = b"z6IHG7cYDID6o\n"

-sfl_payload_length = 255
-sfl_outstanding    = 128
+sfl_payload_length = 64
+sfl_outstanding    = 0

 # General commands
 sfl_cmd_abort       = b"\x00"
```

See [Litex Issue #73](https://github.com/enjoy-digital/litex/issues/773) for more information and status.

With this fix, serialboot worked!   _NOTE: if nothing happens after you connect, hit return a couple times.  If you get the `lite>` prompt, just type `serialboot` then return to force a serialboot attempt._

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

At first I thought it wasn't working, and started looking at something else.  But when I looked back, there was a donut!  It was working...just **really slowly**.   I timed it, and measured one update every ~20 seconds.   So we're at 3 donuts/minute.  I think we can do better...and in the next post, we will.


>
> Almost everything in this post also applies to iCEBreaker, with the following differences:
> * Of course, use icebreaker.py instead of fomu.py.
> * You won't need to modify `litex_term.py` -- it work as-is with the iCEBreaker board.
> * You can use `--cpu-variant=lite`, and you should see _many_ more donuts/minute.  (Note: the 'lite' variant doesn't quite fit on the Fomu, since on the Fomu, you need to use some of the fabric to implement USB gateware.)
>



