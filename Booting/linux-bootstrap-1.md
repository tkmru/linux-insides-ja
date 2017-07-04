Kernel booting process. Part 1.
================================================================================

bootloaderからkernelまで
--------------------------------------------------------------------------------


もし私のブログの[記事](http://0xax.blogspot.com/search/label/asm)を読まれた方はご存じかと思いますが、ちょっと前からレイヤーが低レベルのプログラミングを行っています。
Linux用x86_64アセンブリによるプログラミングについて記事を書いていて、Linuxのソースコードにも触れるようになりました。
ローレイヤーがどのように機能しているのか、コンピュータでプログラムがどのように実行されるのか、どのようにメモリに配置されるのか、kernelがどのようにプロセスとメモリを扱うのか、ローレイヤーでネットワークスタックがどのように動くのか等、多くのことを理解しようととても興味が湧いています。
それで、**x86_64** のLinux kernelについてのシリーズを書こうと決心しました。

私はプロのカーネルプログラマではないことと、仕事でもカーネルのコードを書いていないことをご了承ください。
ただの趣味です。私はローレイヤーが単に好きで、どのようにして動いているのかとても興味があります。もし何か困惑した点や、ご質問やご意見がありましたら、twitter [0xAX](https://twitter.com/0xAX) や [email](anotherworldofworld@gmail.com) でお知らせいただくか、[issue](https://github.com/0xAX/linux-insides/issues/new)を作成してください。
そうしてくれると助かります。全ての記事は [linux-insides](https://github.com/0xAX/linux-insides) からアクセスでき、私の英文が間違っていたり内容に問題があったりした場合は、気軽にプルリクエストを送ってください。

*これは正式なドキュメントではありません。あくまでも学習のためや知識共有のためのものですのでご注意ください。*

**必要な知識**

* Cコードの理解
* アセンブリ(AT&T記法)の理解

ツールについて学び始めている人のために、この記事とつづく記事の中で説明を入れようと思います。さて、簡単な導入はここで終わりにして、今からkernelとローレイヤーにダイブしましょう。

全てのコードはkernel 3.18のものです。変更があった場合は、私はそれに応じて更新します。

魔法の電源ボタンの次はなにが起こるのか？
--------------------------------------------------------------------------------

本連載はLinux kernelついてのシリーズですが、kernel コードからは始めません。 - 少なくともこの段落では。ラップトップやデスクトップコンピューターは魔法の電源ボタンを押すと起動します。
マザーボードは電源回路([power supply](https://en.wikipedia.org/wiki/Power_supply))に信号を送ります。
信号を受信した後、電源はコンピュータに適切な量の電力を供給します。
マザーボードは、[power good signal](https://en.wikipedia.org/wiki/Power_good_signal)を受信すると、CPUを起動しようとします。
CPUはレジスタに残されたデータをリセットし、事前に定義された値をレジスタに設定します。


[80386](https://en.wikipedia.org/wiki/Intel_80386) や後継のCPUでは、コンピュータがリセットされると次の事前に定義された値がCPUレジスタに書き込まれます。:

```
IP          0xfff0
CS selector 0xf000
CS base     0xffff0000
```

プロセッサは[リアルモード](https://en.wikipedia.org/wiki/Real_mode)で動き始めます。少し戻って、このモードの memory segmentation を理解しましょう。リアルモードは、[8086](https://en.wikipedia.org/wiki/Intel_8086)から、最新のIntel 64-bit CPUまでのすべてのx86互換のプロセッサに導入されています。
8086プロセッサには20-bit アドレスバスがあります。つまり、0-0xFFFFF(1MB)のアドレス空間を利用できます。
しかし、16ビットのレジスタしかなく、16ビットのレジスタが使用できるアドレスは最大で `2^16-1`、 または `0xffff`(64KB)までです。[Memory segmentation](http://en.wikipedia.org/wiki/Memory_segmentation)は、r利用可能なアドレス空間すべてを利用するために用いられる方法です。
全てのメモリは65535 Bytesまたは64KBの固定長の小さなセグメントに分けられます。16-bit レジスタでは、64KB以上のメモリ位置にアクセスできないので、別の方法でアクセスします。
アドレスは2つのパートで構成されます: ベースアドレスを持つセグメントセレクタとそのバースアドレスからのオフセットである。
リアルモードでは、セグメントセレクタのベースアドレスは`Segment Selector * 16`となります。
そのため、物理アドレスを得るには、セグメントアドレスに16をかけたものに、オフセットアドレスを足す必要があります。:

```
PhysicalAddress = Segment Selector * 16 + Offset
```

例えば、`CS:IP`が`0x2000:0x0010`の場合、物理アドレスは次のようになります。:

```python
>>> hex((0x2000 << 4) + 0x0010)
'0x20010'
```

しかし、セグメント部分とオフセット部分を両方最大にした場合、つまり`0xffff:0xffff`の場合は次のようになります。

```python
>>> hex((0xffff << 4) + 0xffff)
'0x10ffef'
```

which is 65520 bytes past the first megabyte. Since only one megabyte is accessible in real mode, `0x10ffef` becomes `0x00ffef` with disabled [A20](https://en.wikipedia.org/wiki/A20_line).

Ok, now we know about real mode and memory addressing. Let's get back to discussing register values after reset:

The `CS` register consists of two parts: the visible segment selector, and the hidden base address. While the base address is normally formed by multiplying the segment selector value by 16, during a hardware reset the segment selector in the CS register is loaded with `0xf000` and the base address is loaded with `0xffff0000`; the processor uses this special base address until `CS` is changed.

The starting address is formed by adding the base address to the value in the EIP register:

```python
>>> 0xffff0000 + 0xfff0
'0xfffffff0'
```

We get `0xfffffff0`, which is 16 bytes below 4GB. This point is called the [Reset vector](http://en.wikipedia.org/wiki/Reset_vector). This is the memory location at which the CPU expects to find the first instruction to execute after reset. It contains a [jump](http://en.wikipedia.org/wiki/JMP_%28x86_instruction%29) (`jmp`) instruction that usually points to the BIOS entry point. For example, if we look in the [coreboot](http://www.coreboot.org/) source code, we see:

```assembly
    .section ".reset"
    .code16
.globl  reset_vector
reset_vector:
    .byte  0xe9
    .int   _start - ( . + 2 )
    ...
```

Here we can see the `jmp` instruction [opcode](http://ref.x86asm.net/coder32.html#xE9), which is `0xe9`, and its destination address at `_start - ( . + 2)`. We can also see that the `reset` section is 16 bytes and that it starts at `0xfffffff0`:

```
SECTIONS {
    _ROMTOP = 0xfffffff0;
    . = _ROMTOP;
    .reset . : {
        *(.reset)
        . = 15 ;
        BYTE(0x00);
    }
}
```

Now the BIOS starts; after initializing and checking the hardware, the BIOS needs to find a bootable device. A boot order is stored in the BIOS configuration, controlling which devices the BIOS attempts to boot from. When attempting to boot from a hard drive, the BIOS tries to find a boot sector. On hard drives partitioned with an MBR partition layout, the boot sector is stored in the first 446 bytes of the first sector, where each sector is 512 bytes. The final two bytes of the first sector are `0x55` and `0xaa`, which designates to the BIOS that this device is bootable. For example:

```assembly
;
; Note: this example is written in Intel Assembly syntax
;
[BITS 16]
[ORG  0x7c00]

boot:
    mov al, '!'
    mov ah, 0x0e
    mov bh, 0x00
    mov bl, 0x07

    int 0x10
    jmp $

times 510-($-$$) db 0

db 0x55
db 0xaa
```

Build and run this with:

```
nasm -f bin boot.nasm && qemu-system-x86_64 boot
```

This will instruct [QEMU](http://qemu.org) to use the `boot` binary that we just built as a disk image. Since the binary generated by the assembly code above fulfills the requirements of the boot sector (the origin is set to `0x7c00` and we end with the magic sequence), QEMU will treat the binary as the master boot record (MBR) of a disk image.

You will see:

![Simple bootloader which prints only `!`](http://oi60.tinypic.com/2qbwup0.jpg)

In this example, we can see that the code will be executed in 16-bit real mode and will start at `0x7c00` in memory. After starting, it calls the [0x10](http://www.ctyme.com/intr/rb-0106.htm) interrupt, which just prints the `!` symbol; it fills the remaining 510 bytes with zeros and finishes with the two magic bytes `0xaa` and `0x55`.

You can see a binary dump of this using the `objdump` utility:

```
nasm -f bin boot.nasm
objdump -D -b binary -mi386 -Maddr16,data16,intel boot
```

A real-world boot sector has code for continuing the boot process and a partition table instead of a bunch of 0's and an exclamation mark :) From this point onwards, the BIOS hands over control to the bootloader.

**NOTE**: As explained above, the CPU is in real mode; in real mode, calculating the physical address in memory is done as follows:

```
PhysicalAddress = Segment Selector * 16 + Offset
```

just as explained before. We have only 16-bit general purpose registers; the maximum value of a 16-bit register is `0xffff`, so if we take the largest values, the result will be:

```python
>>> hex((0xffff * 16) + 0xffff)
'0x10ffef'
```

where `0x10ffef` is equal to `1MB + 64KB - 16b`. An [8086](https://en.wikipedia.org/wiki/Intel_8086) processor (which was the first processor with real mode), in contrast, has a 20-bit address line. Since `2^20 = 1048576` is 1MB, this means that the actual available memory is 1MB.

General real mode's memory map is as follows:

```
0x00000000 - 0x000003FF - Real Mode Interrupt Vector Table
0x00000400 - 0x000004FF - BIOS Data Area
0x00000500 - 0x00007BFF - Unused
0x00007C00 - 0x00007DFF - Our Bootloader
0x00007E00 - 0x0009FFFF - Unused
0x000A0000 - 0x000BFFFF - Video RAM (VRAM) Memory
0x000B0000 - 0x000B7777 - Monochrome Video Memory
0x000B8000 - 0x000BFFFF - Color Video Memory
0x000C0000 - 0x000C7FFF - Video ROM BIOS
0x000C8000 - 0x000EFFFF - BIOS Shadow Area
0x000F0000 - 0x000FFFFF - System BIOS
```

In the beginning of this post, I wrote that the first instruction executed by the CPU is located at address `0xFFFFFFF0`, which is much larger than `0xFFFFF` (1MB). How can the CPU access this address in real mode? The answer is in the [coreboot](http://www.coreboot.org/Developer_Manual/Memory_map) documentation:

```
0xFFFE_0000 - 0xFFFF_FFFF: 128 kilobyte ROM mapped into address space
```

At the start of execution, the BIOS is not in RAM, but in ROM.

Bootloader
--------------------------------------------------------------------------------

There are a number of bootloaders that can boot Linux, such as [GRUB 2](https://www.gnu.org/software/grub/) and [syslinux](http://www.syslinux.org/wiki/index.php/The_Syslinux_Project). The Linux kernel has a [Boot protocol](https://github.com/torvalds/linux/blob/master/Documentation/x86/boot.txt) which specifies the requirements for a bootloader to implement Linux support. This example will describe GRUB 2.

Continuing from before, now that the BIOS has chosen a boot device and transferred control to the boot sector code, execution starts from [boot.img](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/boot/i386/pc/boot.S;hb=HEAD). This code is very simple, due to the limited amount of space available, and contains a pointer which is used to jump to the location of GRUB 2's core image. The core image begins with [diskboot.img](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/boot/i386/pc/diskboot.S;hb=HEAD), which is usually stored immediately after the first sector in the unused space before the first partition. The above code loads the rest of the core image, which contains GRUB 2's kernel and drivers for handling filesystems, into memory. After loading the rest of the core image, it executes [grub_main](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/kern/main.c).

`grub_main` initializes the console, gets the base address for modules, sets the root device, loads/parses the grub configuration file, loads modules, etc. At the end of execution, `grub_main` moves grub to normal mode. `grub_normal_execute` (from `grub-core/normal/main.c`) completes the final preparations and shows a menu to select an operating system. When we select one of the grub menu entries, `grub_menu_execute_entry` runs, executing the grub `boot` command and booting the selected operating system.

As we can read in the kernel boot protocol, the bootloader must read and fill some fields of the kernel setup header, which starts at the `0x01f1` offset from the kernel setup code. You may look at the boot [linker script](https://github.com/torvalds/linux/blob/master/arch/x86/boot/setup.ld#L16) to make sure in this offset. The kernel header [arch/x86/boot/header.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S) starts from:

```assembly
    .globl hdr
hdr:
    setup_sects: .byte 0
    root_flags:  .word ROOT_RDONLY
    syssize:     .long 0
    ram_size:    .word 0
    vid_mode:    .word SVGA_MODE
    root_dev:    .word 0
    boot_flag:   .word 0xAA55
```

The bootloader must fill this and the rest of the headers (which are only marked as being type `write` in the Linux boot protocol, such as in [this example](https://github.com/torvalds/linux/blob/master/Documentation/x86/boot.txt#L354)) with values which it has either received from the command line or calculated. (We will not go over full descriptions and explanations for all fields of the kernel setup header now but instead when the discuss how kernel uses them; you can find a description of all fields in the [boot protocol](https://github.com/torvalds/linux/blob/master/Documentation/x86/boot.txt#L156).)

As we can see in the kernel boot protocol, the memory map will be the following after loading the kernel:

```shell
         | Protected-mode kernel  |
100000   +------------------------+
         | I/O memory hole        |
0A0000   +------------------------+
         | Reserved for BIOS      | Leave as much as possible unused
         ~                        ~
         | Command line           | (Can also be below the X+10000 mark)
X+10000  +------------------------+
         | Stack/heap             | For use by the kernel real-mode code.
X+08000  +------------------------+
         | Kernel setup           | The kernel real-mode code.
         | Kernel boot sector     | The kernel legacy boot sector.
       X +------------------------+
         | Boot loader            | <- Boot sector entry point 0x7C00
001000   +------------------------+
         | Reserved for MBR/BIOS  |
000800   +------------------------+
         | Typically used by MBR  |
000600   +------------------------+
         | BIOS use only          |
000000   +------------------------+

```

So, when the bootloader transfers control to the kernel, it starts at:

```
X + sizeof(KernelBootSector) + 1
```

where `X` is the address of the kernel boot sector being loaded. In my case, `X` is `0x10000`, as we can see in a memory dump:

![kernel first address](http://oi57.tinypic.com/16bkco2.jpg)

The bootloader has now loaded the Linux kernel into memory, filled the header fields, and then jumped to the corresponding memory address. We can now move directly to the kernel setup code.

Kernelの設定を始める
--------------------------------------------------------------------------------

Finally, we are in the kernel! Technically, the kernel hasn't run yet; first, we need to set up the kernel, memory manager, process manager, etc. Kernel setup execution starts from [arch/x86/boot/header.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S) at [_start](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S#L293). It is a little strange at first sight, as there are several instructions before it.

A long time ago, the Linux kernel used to have its own bootloader. Now, however, if you run, for example,

```
qemu-system-x86_64 vmlinuz-3.18-generic
```

then you will see:

![Try vmlinuz in qemu](http://oi60.tinypic.com/r02xkz.jpg)

Actually, `header.S` starts from [MZ](https://en.wikipedia.org/wiki/DOS_MZ_executable) (see image above), the error message printing and following the [PE](https://en.wikipedia.org/wiki/Portable_Executable) header:

```assembly
#ifdef CONFIG_EFI_STUB
# "MZ", MS-DOS header
.byte 0x4d
.byte 0x5a
#endif
...
...
...
pe_header:
    .ascii "PE"
    .word 0
```

It needs this to load an operating system with [UEFI](https://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface). We won't be looking into its inner workings right now and will cover it in upcoming chapters.

The actual kernel setup entry point is:

```assembly
// header.S line 292
.globl _start
_start:
```

The bootloader (grub2 and others) knows about this point (`0x200` offset from `MZ`) and makes a jump directly to it, despite the fact that `header.S` starts from the `.bstext` section, which prints an error message:

```
//
// arch/x86/boot/setup.ld
//
. = 0;                    // current position
.bstext : { *(.bstext) }  // put .bstext section to position 0
.bsdata : { *(.bsdata) }
```

The kernel setup entry point is:

```assembly
    .globl _start
_start:
    .byte  0xeb
    .byte  start_of_setup-1f
1:
    //
    // rest of the header
    //
```

Here we can see a `jmp` instruction opcode (`0xeb`) that jumps to the `start_of_setup-1f` point. In `Nf` notation, `2f` refers to the following local `2:` label; in our case, it is label `1` that is present right after the jump, and it contains the rest of the setup [header](https://github.com/torvalds/linux/blob/master/Documentation/x86/boot.txt#L156). Right after the setup header, we see the `.entrytext` section, which starts at the `start_of_setup` label.

This is the first code that actually runs (aside from the previous jump instructions, of course). After the kernel setup received control from the bootloader, the first `jmp` instruction is located at the `0x200` offset from the start of the kernel real mode, i.e., after the first 512 bytes. This we can both read in the Linux kernel boot protocol and see in the grub2 source code:

```C
segment = grub_linux_real_target >> 4;
state.gs = state.fs = state.es = state.ds = state.ss = segment;
state.cs = segment + 0x20;
```

This means that segment registers will have the following values after kernel setup starts:

```
gs = fs = es = ds = ss = 0x1000
cs = 0x1020
```

In my case, the kernel is loaded at `0x10000`.

After the jump to `start_of_setup`, the kernel needs to do the following:

* Make sure that all segment register values are equal
* Set up a correct stack, if needed
* Set up [bss](https://en.wikipedia.org/wiki/.bss)
* Jump to the C code in [main.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/main.c)

Let's look at the implementation.


セグメントレジスタのアライメント
--------------------------------------------------------------------------------

First of all, the kernel ensures that `ds` and `es` segment registers point to the same address. Next, it clears the direction flag using the `cld` instruction:

```assembly
    movw    %ds, %ax
    movw    %ax, %es
    cld
```

As I wrote earlier, grub2 loads kernel setup code at address `0x10000` and `cs` at `0x1020` because execution doesn't start from the start of file, but from

```assembly
_start:
    .byte 0xeb
    .byte start_of_setup-1f
```

`jump`, which is at a 512 byte offset from [4d 5a](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S#L47). It also needs to align `cs` from `0x10200` to `0x10000`, as well as all other segment registers. After that, we set up the stack:

```assembly
    pushw   %ds
    pushw   $6f
    lretw
```

which pushes the value of `ds` to the stack with the address of the [6](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S#L494) label and executes the `lretw` instruction. When the `lretw` instruction is called, it loads the address of label `6` into the [instruction pointer](https://en.wikipedia.org/wiki/Program_counter) register and loads `cs` with the value of `ds`. Afterwards, `ds` and `cs` will have the same values.

スタックの設定
--------------------------------------------------------------------------------

Almost all of the setup code is in preparation for the C language environment in real mode. The next [step](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S#L467) is checking the `ss` register value and making a correct stack if `ss` is wrong:

```assembly
    movw    %ss, %dx
    cmpw    %ax, %dx
    movw    %sp, %dx
    je      2f
```

This can lead to 3 different scenarios:

* `ss` has valid value `0x10000` (as do all other segment registers beside `cs`)
* `ss` is invalid and `CAN_USE_HEAP` flag is set     (see below)
* `ss` is invalid and `CAN_USE_HEAP` flag is not set (see below)

Let's look at all three of these scenarios in turn:

* `ss` has a correct address (`0x10000`). In this case, we go to label [2](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S#L481):

```assembly
2:  andw    $~3, %dx
    jnz     3f
    movw    $0xfffc, %dx
3:  movw    %ax, %ss
    movzwl  %dx, %esp
    sti
```

Here we can see the alignment of `dx` (contains `sp` given by bootloader) to 4 bytes and a check for whether or not it is zero. If it is zero, we put `0xfffc` (4 byte aligned address before the maximum segment size of 64 KB) in `dx`. If it is not zero, we continue to use `sp`, given by the bootloader (0xf7f4 in my case). After this, we put the `ax` value into `ss`, which stores the correct segment address of `0x10000` and sets up a correct `sp`. We now have a correct stack:

![stack](http://oi58.tinypic.com/16iwcis.jpg)

* In the second scenario, (`ss` != `ds`). First, we put the value of [_end](https://github.com/torvalds/linux/blob/master/arch/x86/boot/setup.ld#L52) (the address of the end of the setup code) into `dx` and check the `loadflags` header field using the `testb` instruction to see whether we can use the heap. [loadflags](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S#L321) is a bitmask header which is defined as:

```C
#define LOADED_HIGH     (1<<0)
#define QUIET_FLAG      (1<<5)
#define KEEP_SEGMENTS   (1<<6)
#define CAN_USE_HEAP    (1<<7)
```

and, as we can read in the boot protocol,

```
Field name: loadflags

  This field is a bitmask.

  Bit 7 (write): CAN_USE_HEAP
    Set this bit to 1 to indicate that the value entered in the
    heap_end_ptr is valid.  If this field is clear, some setup code
    functionality will be disabled.
```

If the `CAN_USE_HEAP` bit is set, we put `heap_end_ptr` into `dx` (which points to `_end`) and add `STACK_SIZE` (minimum stack size, 512 bytes) to it. After this, if `dx` is not carried (it will not be carried, dx = _end + 512), jump to label `2` (as in the previous case) and make a correct stack.

![stack](http://oi62.tinypic.com/dr7b5w.jpg)

* When `CAN_USE_HEAP` is not set, we just use a minimal stack from `_end` to `_end + STACK_SIZE`:

![minimal stack](http://oi60.tinypic.com/28w051y.jpg)

BSSの設定
--------------------------------------------------------------------------------

main関数のCコードにジャンプする前に実行する必要がある最後の2つのステップは、[BSS](https://en.wikipedia.org/wiki/.bss)領域を設定し、"magic" シグネイチャを確認することです。
最初に、シグネイチャを確認します:

```assembly
    cmpl    $0x5a5aaa55, setup_sig
    jne     setup_bad
```

これはシンプルに、[setup_sig](https://github.com/torvalds/linux/blob/master/arch/x86/boot/setup.ld#L39)とマジックナンバー `0x5a5aaa55`を比較し、等しくなければ fatal error を出します。

マジックナンバーが等しければ、すでにセグメントレジスタとスタックのセットをわれわれは持っているので、残すはCコードにジャンプする前にBSS領域の設定をするだけです。

BSSセクションは静的にアロケートされた、初期化されていないデータを保存するために使われます。Linuxでは以下のコードを使い、このメモリ領域が最初は0になることを保証します。

```assembly
    movw    $__bss_start, %di
    movw    $_end+3, %cx
    xorl    %eax, %eax
    subw    %di, %cx
    shrw    $2, %cx
    rep; stosl
```

最初に
[__bss_start](https://github.com/torvalds/linux/blob/master/arch/x86/boot/setup.ld#L47)のアドレスが `di` に代入され、次に `_end + 3`（+3は4バイトにアラインされている）のアドレスが `cx` に代入されます。
`eax` レジスタは0クリアされ（`xor`命令を使います）、BSSセクションのサイズ（`cx`-`di`）が `cx` の中に置かれます。
そして `cx` は2ビット右シフトすることで、4（word長）で除算され、`stosl` 命令を繰り返し`di`が指すアドレスに `eax` の値（0）を格納して、`di` は自動的に4ずつ増加し、`cx` が0になるまで繰り返されます。
このコードの実際の効果は、`__bss_start` から `_end`まで、メモリ内にある全てのワードを通して、0が書きこまれることです。:

![bss](http://oi59.tinypic.com/29m2eyr.jpg)

main関数へのジャンプ
--------------------------------------------------------------------------------

これでスタックとBSSの準備ができたので、われわれは `main()` に飛ぶことが出来ます:

```assembly
    calll main
```

`main()` は [arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/main.c)にあります。 これについてはパート2で扱います。

まとめ
--------------------------------------------------------------------------------

これで、Linux kernek insidesの最初のパートは終わりです。
もし質問や提案があれば twitter [0xAX](https://twitter.com/0xAX) や [email](anotherworldofworld@gmail.com) で連絡していただくか、Issueを作成してください。
次のパートでは、Linux kernelの設定で実行する最初の`Cコード`、`memset`、`memcpy`、`earlyprintk`の実装といったメモリルーチンの実装、初期のコンソールの実装と初期化などを見ていく予定です。

リンク
--------------------------------------------------------------------------------

  * [Intel 80386 programmer's reference manual 1986](http://css.csail.mit.edu/6.858/2014/readings/i386.pdf)
  * [Minimal Boot Loader for Intel® Architecture](https://www.cs.cmu.edu/~410/doc/minimal_boot.pdf)
  * [8086](http://en.wikipedia.org/wiki/Intel_8086)
  * [80386](http://en.wikipedia.org/wiki/Intel_80386)
  * [Reset vector](http://en.wikipedia.org/wiki/Reset_vector)
  * [Real mode](http://en.wikipedia.org/wiki/Real_mode)
  * [Linux kernel boot protocol](https://www.kernel.org/doc/Documentation/x86/boot.txt)
  * [CoreBoot developer manual](http://www.coreboot.org/Developer_Manual)
  * [Ralf Brown's Interrupt List](http://www.ctyme.com/intr/int.htm)
  * [Power supply](http://en.wikipedia.org/wiki/Power_supply)
  * [Power good signal](http://en.wikipedia.org/wiki/Power_good_signal)
