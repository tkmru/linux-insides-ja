Kernel booting process. Part 1.
================================================================================

bootloaderからカーネルまで
--------------------------------------------------------------------------------


もし私のブログの[記事](http://0xax.blogspot.com/search/label/asm)を読まれた方はご存じかと思いますが、ちょっと前からレイヤーが低レベルのプログラミングを行っています。
Linux用x86_64アセンブリによるプログラミングについて記事を書いていて、Linuxのソースコードにも触れるようになりました。
ローレイヤーがどのように機能しているのか、コンピュータでプログラムがどのように実行されるのか、どのようにメモリに配置されるのか、カーネルがどのようにプロセスとメモリを扱うのか、ローレイヤーでネットワークスタックがどのように動くのか等、多くのことを理解しようととても興味が湧いています。
それで、**x86_64** のLinux カーネルについてのシリーズを書こうと決心しました。

私はプロのカーネルプログラマではないことと、仕事でもカーネルのコードを書いていないことをご了承ください。
ただの趣味です。私はローレイヤーが単に好きで、どのようにして動いているのかとても興味があります。もし何か困惑した点や、ご質問やご意見がありましたら、twitter [0xAX](https://twitter.com/0xAX) や [email](anotherworldofworld@gmail.com) でお知らせいただくか、[issue](https://github.com/0xAX/linux-insides/issues/new)を作成してください。
そうしてくれると助かります。全ての記事は [linux-insides](https://github.com/0xAX/linux-insides) からアクセスでき、私の英文が間違っていたり内容に問題があったりした場合は、気軽にプルリクエストを送ってください。

*これは正式なドキュメントではありません。あくまでも学習のためや知識共有のためのものですのでご注意ください。*

**必要な知識**

* Cコードの理解
* アセンブリ(AT&T記法)の理解

ツールについて学び始めている人のために、この記事とつづく記事の中で説明を入れようと思います。さて、簡単な導入はここで終わりにして、今からカーネルとローレイヤーにダイブしましょう。

全てのコードはカーネル 3.18のものです。変更があった場合は、私はそれに応じて更新します。

魔法の電源ボタンの次はなにが起こるのか？
--------------------------------------------------------------------------------

本連載はLinux カーネルついてのシリーズですが、カーネル コードからは始めません。 - 少なくともこの段落では。ラップトップやデスクトップコンピューターは魔法の電源ボタンを押すと起動します。
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
全てのメモリは65535 Byteまたは64KBの固定長の小さなセグメントに分けられます。16-bit レジスタでは、64KB以上のメモリ位置にアクセスできないので、別の方法でアクセスします。
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

つまり、最初の1MBよりも65519Byteオーバーしていることになります。リアルモードでアクセスできるのは最大で1MBのため、[A20ライン](https://en.wikipedia.org/wiki/A20_line)が無効になっていると`0x10ffef`は`0x00ffef`になります。

リアルモードとmemory addressingが分かったところで、リセット後のレジスタの値について説明しましょう。

`CS`レジスタは、見えるセグメントセレクタと隠れたベースアドレスの2つの部分で構成されています。 ベースアドレスは、セグメントセレクタの値に16を乗算することによって形成されるが、ハードウェアがリセットされる間、CSレジスタ内のセグメントセレクタには`0xf000`が代入され、ベースアドレスに`0xffff0000`が代入されます。 プロセッサは、`CS`が変更されるまで、この特殊なベースアドレスを使用します。


開始するアドレスはベースアドレスをEIPレジスタの値に足すことで生成されます。:

```python
>>> 0xffff0000 + 0xfff0
'0xfffffff0'
```

その結果、`0xfffffff0`ができ、この値は4GBより16byte小さいです。
このポイントを[Reset vector](http://en.wikipedia.org/wiki/Reset_vector)と呼びます。
このメモリ配置には、リセット後にCPUが最初に実行するプログラムが置かれています。
これには、[`JMP`](http://en.wikipedia.org/wiki/JMP_%28x86_instruction%29) 命令が含まれ、BIOSのエントリポイントを指しています。
例えば、[coreboot](http://www.coreboot.org/)のソースコードを見ると、次のように書かれています。:

```assembly
    .section ".reset"
    .code16
.globl  reset_vector
reset_vector:
    .byte  0xe9
    .int   _start - ( . + 2 )
    ...
```

JMP命令の[オペコード](http://ref.x86asm.net/coder32.html#xE9)である`0xe9`と、そのデスティネーションアドレスである`_start - ( . + 2)`があります。また、`reset`セクションが16 Byteで`0xfffffff0`から始まることが分かります。:

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

ここでBIOSが実行されます。ハードウェアの初期化とチェックを行い、BIOSはブートできるデバイスを探す必要があります。
ブート順位はBIOSの設定に保存されており、カーネルがどのデバイスを使用して起動するのかを操作します。
ハードドライブから起動しようとする場合、BIOSはブートセクタを探そうとします。
ハードディスクにMBRのパーティションがある場合、ブートセクタは最初のセクター（512 Byte）の最初の446 Byteに置かれています。最初のセクターの最後2バイトは`0x55`と`0xaa`で、BIOSにこのデバイスがブート可能であることを知らせます。例:

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

ビルドして実行します:

```
nasm -f bin boot.nasm && qemu-system-x86_64 boot
```

上のコードが[QEMU](http://qemu.org)にディスクイメージとしてビルドした`boot`バイナリを使用するよう命令します。
上のアセンブリコードによって生成されるバイナリはブートセクタの要件（開始位置は`0x7c00`に設定され、マジックシーケンスで終点を指定）を満たしているので、QEMUはそのバイナリをディスクイメージのMBR(master boot record)として扱います。

このようになります:

![Simple bootloader which prints only `!`](http://oi60.tinypic.com/2qbwup0.jpg)

この例では、16-bit リアルモードでコードが実行され、メモリの`0x7c00`から始まります。
実行されると、[0x10](http://www.ctyme.com/intr/rb-0106.htm) 割り込みが呼び出され、`!`シンボルが出力されます。残りの510 Byteを0で埋め、2つのマジックバイト`0xaa`と`0x55`で終わります。

`objdump`でダンプした結果は以下のコマンドで見れます:

```
nasm -f bin boot.nasm
objdump -D -b binary -mi386 -Maddr16,data16,intel boot
```

実際のブートセクタの場合、この続きは多くの0たちや感嘆符ではなく、起動処理とパーティションテーブルになります。これ以降はBIOSからブートローダに動作が移ります。

**注**: 上でも書いたようにCPUはリアルモードで動作します。リアルモードでは、メモリ内の物理アドレスを次のように計算します。:

```
PhysicalAddress = Segment Selector * 16 + Offset
```

前述したように、16bit の汎用レジスタしかなく、16-bit レジスタの最大値は`0xffff`のため、最大値を取ると次のようになります。:

```python
>>> hex((0xffff * 16) + 0xffff)
'0x10ffef'
```

0x10ffefは、`1MB + 64KB - 16B`と同じになります。
しかし、[8086](https://en.wikipedia.org/wiki/Intel_8086)プロセッサは、リアルモードが搭載された初めてのプロセッサであり、A20アドレスラインを持っています。
また、2^20 = 1048576は1MBなので、実際に使用可能なメモリは1MBとなっています。

一般的なリアルモードでのメモリマップは次のとおりです。:

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

本稿の最初の部分でも書きましたが、CPUが実行する最初の処理は `0xFFFFFFF0` アドレスに配置されています。
これは、`0xFFFFF` (1MB)よりはるかに大きい領域です。CPUはどのようにしてこのリアルモードでアクセスするのでしょうか。
これは[coreboot](http://www.coreboot.org/Developer_Manual/Memory_map)のドキュメントに記載されています。:

```
0xFFFE_0000 - 0xFFFF_FFFF: 128 kilobyte ROM mapped into address space
```

実行時、BIOSはRAMではなくROMに置かれています。

ブートローダー
--------------------------------------------------------------------------------

[GRUB2](https://www.gnu.org/software/grub/) や [syslinux](http://www.syslinux.org/wiki/index.php/The_Syslinux_Project) のような、Linuxを起動させることができるブートローダは数多くあります。
Linuxカーネルは、Linuxサポートを実行するためのブートローダに必要な条件を指定する[Boot protocol](https://github.com/torvalds/linux/blob/master/Documentation/x86/boot.txt)を持っています。
ここでは例として GRUB2 について述べます。

BIOSはブートデバイスを選んで、ブートセクタコードに対する制御を伝達し、[boot.img](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/boot/i386/pc/boot.S;hb=HEAD)から実行を開始します。
このコードは、利用可能な空間が限られているため非常にシンプルで、GRUB2 のコアイメージの位置へジャンプするためのポインタを含んでいます。
コアイメージは [diskboot.img](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/boot/i386/pc/diskboot.S;hb=HEAD) で始まりますが、最初のパーティションの前の未使用のスペースにある最初のセクタの直後に格納されます。
上記のコードは残りのコアイメージをメモリにロードしますが、それには GRUB2 のカーネルとファイルシステムを取り扱うためのドライバを含んでいます。
残りのコアイメージをロードした後に、[grub_main](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/kern/main.c)を実行します。

`grub_main` は、コンソールの初期化、モジュールのためのベースアドレスの取得、ルートデバイスの設定、GRUB設定ファイルの ロード/パース、モジュールのロードなどを行います。実行の最後には、`grub_main` がGRUBを通常モードへ移動させます。
`grub_normal_execute`（grub-core/normal/main.cより）が最後の準備を完了させ、オペレーティングシステムを選択するためのメニューを表示します。
GRUBメニューを選択する際に、`grub_menu_execute_entry` が起動し、grub`boot`コマンドを実行して、選択したオペレーティングシステムをブートします。

カーネルのブートプロトコルを見て分かるように、ブートローダはカーネルのセットアップヘッダを読み込み、いくつかのフィールドを満たさなければいけません。
そしてそれは、カーネルの設定コードのオフセット `0x01f1` から始まります。
[リンカスクリプト](https://github.com/torvalds/linux/blob/master/arch/x86/boot/setup.ld#L16)を見ることで、このオフセットは確認できます。
カーネルヘッダ([arch/x86/boot/header.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S)) は次のようにスタートします。:

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

ブートローダは、これと、（[この例](https://github.com/torvalds/linux/blob/master/Documentation/x86/boot.txt#L354)のようなLinuxブートプロトコルのwriteでマークされている）残りのヘッダを、コマンドラインまたは計算し求めた値で埋める必要があります。
(カーネルのセットアップヘッダの全てのフィールドの記述や説明についてはここでは触れませんが、後でカーネルがこれらを使用する時に説明します。)
[boot protocol](https://github.com/torvalds/linux/blob/master/Documentation/x86/boot.txt#L156)で全てのフィールドの記述を見つけることができます。

カーネルのブートプロトコルを見て分かるように、メモリマップはカーネルをロードした後、次のようになるでしょう。:

```shell
         | Protected-mode カーネル  |
100000   +------------------------+
         | I/O memory hole        |
0A0000   +------------------------+
         | Reserved for BIOS      | Leave as much as possible unused
         ~                        ~
         | Command line           | (Can also be below the X+10000 mark)
X+10000  +------------------------+
         | Stack/heap             | For use by the カーネル real-mode code.
X+08000  +------------------------+
         | Kernel setup           | The カーネル real-mode code.
         | Kernel boot sector     | The カーネル legacy boot sector.
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

ブートローダがカーネルに制御を移したとき、以下のアドレスで開始されます。:

```
X + sizeof(KernelBootSector) + 1
```

`X` がカーネルのブートセクタがロードされている位置を示します。この場合は、`X` が `0x10000` で、メモリダンプに見て取れます。:

![kernel first address](http://oi57.tinypic.com/16bkco2.jpg)

ブートローダーはLinuxカーネルをメモリへロードし、ヘッダのフィールドを埋め、該当のメモリアドレスへジャンプします。
今、われわれはカーネルのセットアップコードへ直接移動することができます。

Kernelの設定を始める
--------------------------------------------------------------------------------

Finally, we are in the カーネル! Technically, the カーネル hasn't run yet; first, we need to set up the カーネル, memory manager, process manager, etc. Kernel setup execution starts from [arch/x86/boot/header.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S) at [_start](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S#L293). It is a little strange at first sight, as there are several instructions before it.

A long time ago, the Linux カーネル used to have its own bootloader. Now, however, if you run, for example,

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

The actual カーネル setup entry point is:

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

The カーネル setup entry point is:

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

This is the first code that actually runs (aside from the previous jump instructions, of course). After the カーネル setup received control from the bootloader, the first `jmp` instruction is located at the `0x200` offset from the start of the カーネル real mode, i.e., after the first 512 bytes. This we can both read in the Linux カーネル boot protocol and see in the grub2 source code:

```C
segment = grub_linux_real_target >> 4;
state.gs = state.fs = state.es = state.ds = state.ss = segment;
state.cs = segment + 0x20;
```

This means that segment registers will have the following values after カーネル setup starts:

```
gs = fs = es = ds = ss = 0x1000
cs = 0x1020
```

In my case, the カーネル is loaded at `0x10000`.

After the jump to `start_of_setup`, the カーネル needs to do the following:

* Make sure that all segment register values are equal
* Set up a correct stack, if needed
* Set up [bss](https://en.wikipedia.org/wiki/.bss)
* Jump to the C code in [main.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/main.c)

Let's look at the implementation.


セグメントレジスタのアライメント
--------------------------------------------------------------------------------

First of all, the カーネル ensures that `ds` and `es` segment registers point to the same address. Next, it clears the direction flag using the `cld` instruction:

```assembly
    movw    %ds, %ax
    movw    %ax, %es
    cld
```

As I wrote earlier, grub2 loads カーネル setup code at address `0x10000` and `cs` at `0x1020` because execution doesn't start from the start of file, but from

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

これで、Linux カーネル insidesの最初のパートは終わりです。
もし質問や提案があれば twitter [0xAX](https://twitter.com/0xAX) や [email](anotherworldofworld@gmail.com) で連絡していただくか、Issueを作成してください。
次のパートでは、Linux カーネルの設定で実行する最初の`Cコード`、`memset`、`memcpy`、`earlyprintk`の実装といったメモリルーチンの実装、初期のコンソールの実装と初期化などを見ていく予定です。

リンク
--------------------------------------------------------------------------------

  * [Intel 80386 programmer's reference manual 1986](http://css.csail.mit.edu/6.858/2014/readings/i386.pdf)
  * [Minimal Boot Loader for Intel® Architecture](https://www.cs.cmu.edu/~410/doc/minimal_boot.pdf)
  * [8086](http://en.wikipedia.org/wiki/Intel_8086)
  * [80386](http://en.wikipedia.org/wiki/Intel_80386)
  * [Reset vector](http://en.wikipedia.org/wiki/Reset_vector)
  * [Real mode](http://en.wikipedia.org/wiki/Real_mode)
  * [Linux カーネル boot protocol](https://www.カーネル.org/doc/Documentation/x86/boot.txt)
  * [CoreBoot developer manual](http://www.coreboot.org/Developer_Manual)
  * [Ralf Brown's Interrupt List](http://www.ctyme.com/intr/int.htm)
  * [Power supply](http://en.wikipedia.org/wiki/Power_supply)
  * [Power good signal](http://en.wikipedia.org/wiki/Power_good_signal)
