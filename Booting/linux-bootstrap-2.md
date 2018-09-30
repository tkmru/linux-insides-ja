Kernel booting process. Part 2.
================================================================================

カーネルセットアップの最初のステップ
--------------------------------------------------------------------------------

前回の[パート](linux-bootstrap-1.md)では、Linuxカーネルの中へ潜り始め、カーネルをセットアップするコードの初期の部分を見ていきました。
私たちは、[arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/main.c)内の `main`関数（C言語で書かれた最初の関数）を初めて呼び出すところで止まっていました。

このパートでは、引き続きカーネルのセットアップコードについての調査と以下の内容をやります。

* `プロテクトモード` が何なのか、
* `プロテクトモード` に入るための準備、
* ヒープとコンソールの初期化、
* メモリの検出、cpu validation、キーボードの初期化
* その他いろいろ

では、やっていきましょう。

プロテクトモード
--------------------------------------------------------------------------------

ネイティブの Intel64 [ロングモード](http://en.wikipedia.org/wiki/Long_mode) に切り替える前に、 カーネルはCPUをプロテクトモードに切り替える必要があります。

[プロテクトモード](https://en.wikipedia.org/wiki/Protected_mode)とは何でしょう?
プロテクトモードが最初に x86アーキテクチャ に追加されたのは1982年で、
このモードは[80286](http://en.wikipedia.org/wiki/Intel_80286)プロセッサが出てから、Intel 64とロングモードが登場するまでは、主要のモードでした。

[リアルモード](http://wiki.osdev.org/Real_Mode)から移行した主な理由は、RAMへのアクセスが非常に制限されていたからです。
前回のパートの内容を覚えているかもしれませんが、リアルモードで利用可能なRAMはせいぜい2<sup>20</sup>byteか1MBで、中には640KBしかないものもあります。

プロテクトモードになって、多くの点が変わりましたが、メモリ管理で最も大きな変更がありました。
20bitアドレスバスが32bitアドレスバスに置き換えられたことで、リアルモードでは1MBのメモリにしかアクセスできなかったのが、4GBのメモリにアクセスが可能になりました。
さらにプロテクトモードは、[ページング](http://en.wikipedia.org/wiki/Paging)にもまた対応しています。これについては、次のセクションで紹介します。

プロテクトモードにおけるメモリ管理は、ほぼ独立した次の2つの方式に分かれます。:

* セグメンテーション
* ページング

セグメンテーションにのみここでは見ていきます。ページングに関しては次のセクションで見ていきましょう。

あなたは前のパートでリアルモードのアドレスは２つのパートで構成されると学びました:

* セグメントのベースアドレス
* セグメントベースからのオフセット

そして、これらの2つの情報が分かれば、物理アドレスを取得することができます:

```
PhysicalAddress = Segment Selector * 16 + Offset
```

プロテクトモードになって、メモリセグメンテーションが一新され、64KBの固定サイズのセグメントがなくなりました。
その代わりに、各セグメントのサイズと位置は、セグメントディスクリプタと呼ばれる一連のデータ構造体で表現されます。
このセグメントディスクリプタが格納されているのが、`Global Descriptor Table`（GDT）というデータ構造体です。

GDTはメモリ内にある構造体です。GDTの場所はメモリ内で固定されているわけではなく、専用の `GDTR`レジスタにアドレスが格納されています。
LinuxカーネルのコードでGDTを読み込む方法については、後ほど見ていきましょう。以下のようにGDTは、メモリに読み込まれます:

```assembly
lgdt gdt
```

この`lgdt`命令で、`GDTR`レジスタ にベースアドレスとグローバルディスクリプタテーブルの制限（サイズ）を読み込みます。
`GDTR` は48bitのレジスタで、以下の2つの部分から構成されています:

 * グローバルディスクリプタテーブルのサイズ（16bit）
 * グローバルディスクリプタテーブルのアドレス（32bit）

先ほど説明したように、GDTにはメモリセグメントを表す `segment descriptors` が含まれています。
各ディスクリプタのサイズは64bitで、ディスクリプタの一般的な配置は次のようになっています:

```
31          24        19      16              7            0
------------------------------------------------------------
|             | |B| |A|       | |   | |0|E|W|A|            |
| BASE 31:24  |G|/|L|V| LIMIT |P|DPL|S|  TYPE | BASE 23:16 | 4
|             | |D| |L| 19:16 | |   | |1|C|R|A|            |
------------------------------------------------------------
|                             |                            |
|        BASE 15:0            |       LIMIT 15:0           | 0
|                             |                            |
------------------------------------------------------------
```

リアルモードの後でこれを見ると、少し怖いかもしれませんが、これは簡単です。
例えば、LIMIT 15:0というのは、ディスクリプタのbit 0–15に制限の値が含まれていることを意味します。
そして残りはLIMIT 19:16の中にあります。よってリミットのサイズは0–19なので、20bitです。詳しく見てみましょう。:

1. Limit[20bit]は0–15と16–19のbitにあります。これは`length_of_segment – 1`を定義し、`G`（粒度）ビットに依存します。

  * `G`（bit 55）とセグメントリミットが0の場合、セグメントのサイズは1byteです。
  * `G` が1でセグメントリミットが0の場合、セグメントのサイズは4096byteです。
  * `G` が0でセグメントリミットが0xfffffの場合、セグメントのサイズは1MBです。
  * `G` が1でセグメントリミットが0xfffffの場合、セグメントのサイズは4GBです。

  つまり以下のようになります。

  * `G` が0なら、Limitは1バイト単位と見なされ、セグメントの最大サイズは1MBになります。
  * `G` が1なら、Limitは4096バイト ＝ 4KB ＝ 1ページ単位と見なされ、セグメントの最大サイズは4GBになります。実際、`G`が1なら、LIMITの値は1bit分左にずれます。つまり20bit ＋ 12bitで32bit、すなわち2<sup>32</sup> ＝ 4GBになります。

2. Base[32bit]は（0–15、32–39、56–63bit）にあり、これはセグメントの開始位置の物理アドレスを定義します。

3. タイプ/属性（40–47bit）はセグメントのタイプとセグメントに対する種々のアクセスについて定義します
  * bit 44の `S`フラグはディスクリプタのタイプを指定します。`S`が0ならこのセグメントはシステムセグメントで、`S`が1ならコードまたはデータのセグメントになります（スタックセグメントはデータセグメントで、これは読み書き可能なセグメントである必要があります）。

このセグメントがコードとデータ、どちらのセグメントなのかを判別するには、以下の図で0と表記されたEx(bit 43)属性を確認します。
これが0ならセグメントはデータセグメントで、1ならコードセグメントになります。

セグメントは以下のいずれかのタイプになります:

```
|           Type Field        | Descriptor Type | Description
|-----------------------------|-----------------|------------------
| Decimal                     |                 |
|             0    E    W   A |                 |
| 0           0    0    0   0 | Data            | Read-Only
| 1           0    0    0   1 | Data            | Read-Only, accessed
| 2           0    0    1   0 | Data            | Read/Write
| 3           0    0    1   1 | Data            | Read/Write, accessed
| 4           0    1    0   0 | Data            | Read-Only, expand-down
| 5           0    1    0   1 | Data            | Read-Only, expand-down, accessed
| 6           0    1    1   0 | Data            | Read/Write, expand-down
| 7           0    1    1   1 | Data            | Read/Write, expand-down, accessed
|                  C    R   A |                 |
| 8           1    0    0   0 | Code            | Execute-Only
| 9           1    0    0   1 | Code            | Execute-Only, accessed
| 10          1    0    1   0 | Code            | Execute/Read
| 11          1    0    1   1 | Code            | Execute/Read, accessed
| 12          1    1    0   0 | Code            | Execute-Only, conforming
| 14          1    1    0   1 | Code            | Execute-Only, conforming, accessed
| 13          1    1    1   0 | Code            | Execute/Read, conforming
| 15          1    1    1   1 | Code            | Execute/Read, conforming, accessed
```

見て取れるように最初のビット（bit 43）は、_data_ セグメントの場合は`0`で、_code_ セグメントの場合は`1`です。
続く3つのbit(40, 41,42)は`EWA`(*E*xpansion *W*ritable *A*ccessible)またはCRA(*C*onforming *R*eadable *A*ccessible)のどちらかになります。

  *E（bit 42）が0なら上に拡張し、1なら下に拡張します。詳細は[こちら](http://www.sudleyplace.com/dpmione/expanddown.html)を参照してください。
  *W（bit 41）（データセグメントの場合）が1なら書き込みアクセスが可能、0なら不可です。データセグメントでは、読み取りアクセスが常に許可されている点に注目してください。
  *A（bit 40）はプロセッサからセグメントへのアクセス可能か否かを示します。
  *C（bit 43）（コードセレクタの場合）はコンフォーミングビットです。Cが1なら、ユーザレベルなどの下位レベルの権限から、セグメントコードを実行することが可能です。Cが0なら、同じ権限レベルからのみ実行可能です。
  *R（bit 41）（コードセグメントの場合）が1なら、セグメントへの読み取りアクセスが可能、0なら不可です。コードセグメントに対して、書き込みアクセスは一切できません。

4. DPL[2-bits]はbits 45 – 46にあります。これはセグメントの特権レベルを定義し、値は0-3で、0が最も権限があります。

5. Pフラグ（bit 47）は、セグメントがメモリ内にあるか否かを示します。Pが0なら、セグメントは無効であることを意味し、プロセッサはこのセグメントの読み取りを拒否します。

6. AVLフラグ（bit 52）は利用可能な予約ビットで、Linuxにおいては無視されます。

7. Lフラグ（ビット53）は、コードセグメントがネイティブ64ビットコードを含んでいるかを示します。1ならコードセグメントは64ビットモードで実行されます。

8. D/Bフラグ（ビット54）は、デフォルト/ビッグフラグで、例えば16/32ビットのようなオペランドのサイズを表します。フラグがセットされていれば32ビット、そうでなければ16ビットです。

セグメントレジスタには、リアルモードのようにセグメントセレクタが含まれていません。
しかし、プロテクトモードでは、セグメントレジスタの扱いは異なります。各セグメントディスクリプタは
関連する16ビットの構造体であるSegment Selectorを持ちます。:

```
15              3  2   1  0
-----------------------------
|      Index     | TI | RPL |
-----------------------------
```

Where,
* **Index** がGDTにおけるディスクリプタのインデックス番号を示します。
* **TI**(Table Indicator) はディスクリプタを探す場所を示します。0ならば、Global Descriptor Table（GDT）内を検索し、そうでない場合は、Local Descriptor Table（LDT）内を調べます。
* **RPL** は、Requester’s Privilege Levelのことです。

すべてのセグメントレジスタは見える部分と隠れた部分を持っています。

* Visible – セグメントセレクタはここに保存されています。
* Hidden – セグメントディスクリプタ（ベース、制限、属性、フラグ）

以下のステップは、プロテクトモードで物理アドレスを取得するのに要する手順です。:

* セグメントセレクタはセグメントレジスタの1つにロードしなければなりません。
* CPUは、GDTアドレスとセレクタのIndexによってセグメントディスクリプタを特定し、ディスクリプタをセグメントレジスタの *隠れた* 部分にロードしようとします。
* ベースアドレス（セグメントディスクリプタから）+オフセットは、物理アドレスであるセグメントのリニアアドレスになります。（ページングが無効の場合）

図で表すとこうなります:

![linear address](http://oi62.tinypic.com/2yo369v.jpg)

リアルモードからプロテクトモードへ移行するためのアルゴリズムは、

* 割り込みを無効にします。
* `lgdt`命令でGDTを記述、ロードします。
* CR0（コントロールレジスタ0）におけるPE（Protection Enable、プロテクト有効化）ビットを設定します。
* プロテクトモードのコードにジャンプします。

次の章でlinuxカーネル内で完璧にプロテクトモードへの移行をします。ただ、プロテクトモードへ移る前にもう少し準備が必要です。

[arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/main.c)を見てみましょう。キーボード初期化、ヒープ初期化などを実行する部分があるのがわかります。よく見てみましょう。

ブートパラメータを ”zeropage” にコピー
--------------------------------------------------------------------------------

We will start from the `main` routine in "main.c". First function which is called in `main` is [`copy_boot_params(void)`](https://github.com/torvalds/linux/blob/master/arch/x86/boot/main.c#L30). It copies the kernel setup header into the field of the `boot_params` structure which is defined in the [arch/x86/include/uapi/asm/bootparam.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/uapi/asm/bootparam.h#L113).

The `boot_params` structure contains the `struct setup_header hdr` field. This structure contains the same fields as defined in [linux boot protocol](https://www.kernel.org/doc/Documentation/x86/boot.txt) and is filled by the boot loader and also at kernel compile/build time. `copy_boot_params` does two things:

1. Copies `hdr` from [header.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S#L281) to the `boot_params` structure in `setup_header` field

2. Updates pointer to the kernel command line if the kernel was loaded with the old command line protocol.

Note that it copies `hdr` with `memcpy` function which is defined in the [copy.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/copy.S) source file. Let's have a look inside:

```assembly
GLOBAL(memcpy)
    pushw   %si
    pushw   %di
    movw    %ax, %di
    movw    %dx, %si
    pushw   %cx
    shrw    $2, %cx
    rep; movsl
    popw    %cx
    andw    $3, %cx
    rep; movsb
    popw    %di
    popw    %si
    retl
ENDPROC(memcpy)
```

Yeah, we just moved to C code and now assembly again :) First of all we can see that `memcpy` and other routines which are defined here, start and end with the two macros: `GLOBAL` and `ENDPROC`. `GLOBAL` is described in [arch/x86/include/asm/linkage.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/linkage.h) which defines `globl` directive and the label for it. `ENDPROC` is described in [include/linux/linkage.h](https://github.com/torvalds/linux/blob/master/include/linux/linkage.h) which marks the `name` symbol as a function name and ends with the size of the `name` symbol.

Implementation of `memcpy` is easy. At first, it pushes values from the `si` and `di` registers to the stack to preserve their values because they will change during the `memcpy`. `memcpy` (and other functions in copy.S) use `fastcall` calling conventions. So it gets its incoming parameters from the `ax`, `dx` and `cx` registers.  Calling `memcpy` looks like this:

```c
memcpy(&boot_params.hdr, &hdr, sizeof hdr);
```

So,
* `ax` will contain the address of the `boot_params.hdr`
* `dx` will contain the address of `hdr`
* `cx` will contain the size of `hdr` in bytes.

`memcpy` puts the address of `boot_params.hdr` into `di` and saves the size on the stack. After this it shifts to the right on 2 size (or divide on 4) and copies from `si` to `di` by 4 bytes. After this we restore the size of `hdr` again, align it by 4 bytes and copy the rest of the bytes from `si` to `di` byte by byte (if there is more). Restore `si` and `di` values from the stack in the end and after this copying is finished.

コンソールの初期化
--------------------------------------------------------------------------------

After `hdr` is copied into `boot_params.hdr`, the next step is console initialization by calling the `console_init` function which is defined in [arch/x86/boot/early_serial_console.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/early_serial_console.c).

It tries to find the `earlyprintk` option in the command line and if the search was successful, it parses the port address and baud rate of the serial port and initializes the serial port. Value of `earlyprintk` command line option can be one of these:

* serial,0x3f8,115200
* serial,ttyS0,115200
* ttyS0,115200

After serial port initialization we can see the first output:

```C
if (cmdline_find_option_bool("debug"))
    puts("early console in setup code\n");
```

The definition of `puts` is in [tty.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/tty.c). As we can see it prints character by character in a loop by calling the `putchar` function. Let's look into the `putchar` implementation:

```C
void __attribute__((section(".inittext"))) putchar(int ch)
{
    if (ch == '\n')
        putchar('\r');

    bios_putchar(ch);

    if (early_serial_base != 0)
        serial_putchar(ch);
}
```

`__attribute__((section(".inittext")))` means that this code will be in the `.inittext` section. We can find it in the linker file [setup.ld](https://github.com/torvalds/linux/blob/master/arch/x86/boot/setup.ld#L19).

First of all, `putchar` checks for the `\n` symbol and if it is found, prints `\r` before. After that it outputs the character on the VGA screen by calling the BIOS with the `0x10` interrupt call:

```C
static void __attribute__((section(".inittext"))) bios_putchar(int ch)
{
    struct biosregs ireg;

    initregs(&ireg);
    ireg.bx = 0x0007;
    ireg.cx = 0x0001;
    ireg.ah = 0x0e;
    ireg.al = ch;
    intcall(0x10, &ireg, NULL);
}
```

Here `initregs` takes the `biosregs` structure and first fills `biosregs` with zeros using the `memset` function and then fills it with register values.

```C
    memset(reg, 0, sizeof *reg);
    reg->eflags |= X86_EFLAGS_CF;
    reg->ds = ds();
    reg->es = ds();
    reg->fs = fs();
    reg->gs = gs();
```

Let's look at the [memset](https://github.com/torvalds/linux/blob/master/arch/x86/boot/copy.S#L36) implementation:

```assembly
GLOBAL(memset)
    pushw   %di
    movw    %ax, %di
    movzbl  %dl, %eax
    imull   $0x01010101,%eax
    pushw   %cx
    shrw    $2, %cx
    rep; stosl
    popw    %cx
    andw    $3, %cx
    rep; stosb
    popw    %di
    retl
ENDPROC(memset)
```

As you can read above, it uses the `fastcall` calling conventions like the `memcpy` function, which means that the function gets parameters from `ax`, `dx` and `cx` registers.

Generally `memset` is like a memcpy implementation. It saves the value of the `di` register on the stack and puts the `ax` value into `di` which is the address of the `biosregs` structure. Next is the `movzbl` instruction, which copies the `dl` value to the low 2 bytes of the `eax` register. The remaining 2 high bytes  of `eax` will be filled with zeros.

The next instruction multiplies `eax` with `0x01010101`. It needs to because `memset` will copy 4 bytes at the same time. For example, we need to fill a structure with `0x7` with memset. `eax` will contain `0x00000007` value in this case. So if we multiply `eax` with `0x01010101`, we will get `0x07070707` and now we can copy these 4 bytes into the structure. `memset` uses `rep; stosl` instructions for copying `eax` into `es:di`.

The rest of the `memset` function does almost the same as `memcpy`.

After the `biosregs` structure is filled with `memset`, `bios_putchar` calls the [0x10](http://www.ctyme.com/intr/rb-0106.htm) interrupt which prints a character. Afterwards it checks if the serial port was initialized or not and writes a character there with [serial_putchar](https://github.com/torvalds/linux/blob/master/arch/x86/boot/tty.c#L30) and `inb/outb` instructions if it was set.

ヒープの初期化
--------------------------------------------------------------------------------

After the stack and bss section were prepared in [header.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S) (see previous [part](linux-bootstrap-1.md)), the kernel needs to initialize the [heap](https://github.com/torvalds/linux/blob/master/arch/x86/boot/main.c#L116) with the [`init_heap`](https://github.com/torvalds/linux/blob/master/arch/x86/boot/main.c#L116) function.

First of all `init_heap` checks the [`CAN_USE_HEAP`](https://github.com/torvalds/linux/blob/master/arch/x86/include/uapi/asm/bootparam.h#L21) flag from the [`loadflags`](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S#L321) in the kernel setup header and calculates the end of the stack if this flag was set:

```C
    char *stack_end;

    if (boot_params.hdr.loadflags & CAN_USE_HEAP) {
        asm("leal %P1(%%esp),%0"
            : "=r" (stack_end) : "i" (-STACK_SIZE));
```

or in other words `stack_end = esp - STACK_SIZE`.

Then there is the `heap_end` calculation:
```c
    heap_end = (char *)((size_t)boot_params.hdr.heap_end_ptr + 0x200);
```
which means `heap_end_ptr` or `_end` + `512`(`0x200h`). The last check is whether `heap_end` is greater than `stack_end`. If it is then `stack_end` is assigned to `heap_end` to make them equal.

Now the heap is initialized and we can use it using the `GET_HEAP` method. We will see how it is used, how to use it and how the it is implemented in the next posts.

CPU検証
--------------------------------------------------------------------------------

The next step as we can see is cpu validation by `validate_cpu` from [arch/x86/boot/cpu.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/cpu.c).

It calls the [`check_cpu`](https://github.com/torvalds/linux/blob/master/arch/x86/boot/cpucheck.c#L102) function and passes cpu level and required cpu level to it and checks that the kernel launches on the right cpu level.
```c
check_cpu(&cpu_level, &req_level, &err_flags);
if (cpu_level < req_level) {
    ...
    return -1;
}
```

`check_cpu` checks the cpu's flags, presence of [long mode](http://en.wikipedia.org/wiki/Long_mode) in case of x86_64(64-bit) CPU, checks the processor's vendor and makes preparation for certain vendors like turning off SSE+SSE2 for AMD if they are missing, etc.

メモリ検知
--------------------------------------------------------------------------------

The next step is memory detection by the `detect_memory` function. `detect_memory` basically provides a map of available RAM to the cpu. It uses different programming interfaces for memory detection like `0xe820`, `0xe801` and `0x88`. We will see only the implementation of **0xE820** here.

Let's look into the `detect_memory_e820` implementation from the [arch/x86/boot/memory.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/memory.c) source file. First of all, the `detect_memory_e820` function initializes the `biosregs` structure as we saw above and fills registers with special values for the `0xe820` call:

```assembly
    initregs(&ireg);
    ireg.ax  = 0xe820;
    ireg.cx  = sizeof buf;
    ireg.edx = SMAP;
    ireg.di  = (size_t)&buf;
```

* `ax` contains the number of the function (0xe820 in our case)
* `cx` register contains size of the buffer which will contain data about memory
* `edx` must contain the `SMAP` magic number
* `es:di` must contain the address of the buffer which will contain memory data
* `ebx` has to be zero.

Next is a loop where data about the memory will be collected. It starts from the call of the `0x15` BIOS interrupt, which writes one line from the address allocation table. For getting the next line we need to call this interrupt again (which we do in the loop). Before the next call `ebx` must contain the value returned previously:

```C
    intcall(0x15, &ireg, &oreg);
    ireg.ebx = oreg.ebx;
```

Ultimately, it does iterations in the loop to collect data from the address allocation table and writes this data into the `e820entry` array:

* start of memory segment
* size  of memory segment
* type of memory segment (which can be reserved, usable and etc...).

You can see the result of this in the `dmesg` output, something like:

```
[    0.000000] e820: BIOS-provided physical RAM map:
[    0.000000] BIOS-e820: [mem 0x0000000000000000-0x000000000009fbff] usable
[    0.000000] BIOS-e820: [mem 0x000000000009fc00-0x000000000009ffff] reserved
[    0.000000] BIOS-e820: [mem 0x00000000000f0000-0x00000000000fffff] reserved
[    0.000000] BIOS-e820: [mem 0x0000000000100000-0x000000003ffdffff] usable
[    0.000000] BIOS-e820: [mem 0x000000003ffe0000-0x000000003fffffff] reserved
[    0.000000] BIOS-e820: [mem 0x00000000fffc0000-0x00000000ffffffff] reserved
```

キーボードの初期化
--------------------------------------------------------------------------------

The next step is the initialization of the keyboard with the call of the [`keyboard_init()`](https://github.com/torvalds/linux/blob/master/arch/x86/boot/main.c#L65) function. At first `keyboard_init` initializes registers using the `initregs` function and calling the [0x16](http://www.ctyme.com/intr/rb-1756.htm) interrupt for getting the keyboard status.
```c
    initregs(&ireg);
    ireg.ah = 0x02;     /* Get keyboard status */
    intcall(0x16, &ireg, &oreg);
    boot_params.kbd_status = oreg.al;
```
After this it calls [0x16](http://www.ctyme.com/intr/rb-1757.htm) again to set repeat rate and delay.
```c
    ireg.ax = 0x0305;   /* Set keyboard repeat rate */
    intcall(0x16, &ireg, NULL);
```

クエリ
--------------------------------------------------------------------------------

The next couple of steps are queries for different parameters. We will not dive into details about these queries, but will get back to it in later parts. Let's take a short look at these functions:

The [query_mca](https://github.com/torvalds/linux/blob/master/arch/x86/boot/mca.c#L18) routine calls the [0x15](http://www.ctyme.com/intr/rb-1594.htm) BIOS interrupt to get the machine model number, sub-model number, BIOS revision level, and other hardware-specific attributes:

```c
int query_mca(void)
{
    struct biosregs ireg, oreg;
    u16 len;

    initregs(&ireg);
    ireg.ah = 0xc0;
    intcall(0x15, &ireg, &oreg);

    if (oreg.eflags & X86_EFLAGS_CF)
        return -1;  /* No MCA present */

    set_fs(oreg.es);
    len = rdfs16(oreg.bx);

    if (len > sizeof(boot_params.sys_desc_table))
        len = sizeof(boot_params.sys_desc_table);

    copy_from_fs(&boot_params.sys_desc_table, oreg.bx, len);
    return 0;
}
```

It fills  the `ah` register with `0xc0` and calls the `0x15` BIOS interruption. After the interrupt execution it checks  the [carry flag](http://en.wikipedia.org/wiki/Carry_flag) and if it is set to 1, the BIOS doesn't support [**MCA**](https://en.wikipedia.org/wiki/Micro_Channel_architecture). If carry flag is set to 0, `ES:BX` will contain a pointer to the system information table, which looks like this:

```
Offset  Size    Description
 00h    WORD    number of bytes following
 02h    BYTE    model (see #00515)
 03h    BYTE    submodel (see #00515)
 04h    BYTE    BIOS revision: 0 for first release, 1 for 2nd, etc.
 05h    BYTE    feature byte 1 (see #00510)
 06h    BYTE    feature byte 2 (see #00511)
 07h    BYTE    feature byte 3 (see #00512)
 08h    BYTE    feature byte 4 (see #00513)
 09h    BYTE    feature byte 5 (see #00514)
---AWARD BIOS---
 0Ah  N BYTEs   AWARD copyright notice
---Phoenix BIOS---
 0Ah    BYTE    ??? (00h)
 0Bh    BYTE    major version
 0Ch    BYTE    minor version (BCD)
 0Dh  4 BYTEs   ASCIZ string "PTL" (Phoenix Technologies Ltd)
---Quadram Quad386---
 0Ah 17 BYTEs   ASCII signature string "Quadram Quad386XT"
---Toshiba (Satellite Pro 435CDS at least)---
 0Ah  7 BYTEs   signature "TOSHIBA"
 11h    BYTE    ??? (8h)
 12h    BYTE    ??? (E7h) product ID??? (guess)
 13h  3 BYTEs   "JPN"
 ```

Next we call the `set_fs` routine and pass the value of the `es` register to it. The implementation of `set_fs` is pretty simple:

```c
static inline void set_fs(u16 seg)
{
    asm volatile("movw %0,%%fs" : : "rm" (seg));
}
```

This function contains inline assembly which gets the value of the `seg` parameter and puts it into the `fs` register. There are many functions in [boot.h](https://github.com/torvalds/linux/blob/master/arch/x86/boot/boot.h) like `set_fs`, for example `set_gs`, `fs`, `gs` for reading a value in it etc...

At the end of `query_mca` it just copies the table pointed to by `es:bx` to the `boot_params.sys_desc_table`.

The next step is getting [Intel SpeedStep](http://en.wikipedia.org/wiki/SpeedStep) information by calling the `query_ist` function. First of all it checks the CPU level and if it is correct, calls `0x15` for getting info and saves the result to `boot_params`.

The following [query_apm_bios](https://github.com/torvalds/linux/blob/master/arch/x86/boot/apm.c#L21) function gets [Advanced Power Management](http://en.wikipedia.org/wiki/Advanced_Power_Management) information from the BIOS. `query_apm_bios` calls the `0x15` BIOS interruption too, but with `ah` = `0x53` to check `APM` installation. After the `0x15` execution, `query_apm_bios` functions check the `PM` signature (it must be `0x504d`), carry flag (it must be 0 if `APM` supported) and value of the `cx` register (if it's 0x02, プロテクトモード interface is supported).

Next it calls `0x15` again, but with `ax = 0x5304` for disconnecting the `APM` interface and connecting the 32-bit プロテクトモード interface. In the end it fills `boot_params.apm_bios_info` with values obtained from the BIOS.

Note that `query_apm_bios` will be executed only if `CONFIG_APM` or `CONFIG_APM_MODULE` was set in the configuration file:

```C
#if defined(CONFIG_APM) || defined(CONFIG_APM_MODULE)
    query_apm_bios();
#endif
```

The last is the [`query_edd`](https://github.com/torvalds/linux/blob/master/arch/x86/boot/edd.c#L122) function, which queries `Enhanced Disk Drive` information from the BIOS. Let's look into the `query_edd` implementation.

First of all it reads the [edd](https://github.com/torvalds/linux/blob/master/Documentation/kernel-parameters.txt#L1023) option from the kernel's command line and if it was set to `off` then `query_edd` just returns.

If EDD is enabled, `query_edd` goes over BIOS-supported hard disks and queries EDD information in the following loop:

```C
for (devno = 0x80; devno < 0x80+EDD_MBR_SIG_MAX; devno++) {
    if (!get_edd_info(devno, &ei) && boot_params.eddbuf_entries < EDDMAXNR) {
        memcpy(edp, &ei, sizeof ei);
        edp++;
        boot_params.eddbuf_entries++;
    }
    ...
    ...
    }
```

where `0x80` is the first hard drive and the value of `EDD_MBR_SIG_MAX` macro is 16. It collects data into the array of [edd_info](https://github.com/torvalds/linux/blob/master/include/uapi/linux/edd.h#L172) structures. `get_edd_info` checks that EDD is present by invoking the `0x13` interrupt with `ah` as `0x41` and if EDD is present, `get_edd_info` again calls the `0x13` interrupt, but with `ah` as `0x48` and `si` containing the address of the buffer where EDD information will be stored.

まとめ
--------------------------------------------------------------------------------

これでLinuxカーネルインターナルに関する記事のパート2は終わりです。次のパートではビデオモード設定とプロテクトモード移行の前に必要な残りの準備、そしてそのまま移行について見ていきましょう。
もし質問や提案があれば Twitter [0xAX](https://twitter.com/0xAX) や [email](anotherworldofworld@gmail.com) で連絡していただくか、Issueを作成してください。

**ご注意：英語は私の第一言語ではないことをご承知おきください。誤りを見つけた方は[linux-insides](https://github.com/0xAX/linux-internals)に、プルリクエストを送ってください。**


--------------------------------------------------------------------------------

* [Protected mode](http://en.wikipedia.org/wiki/Protected_mode)
* [Protected mode](http://wiki.osdev.org/Protected_Mode)
* [Long mode](http://en.wikipedia.org/wiki/Long_mode)
* [Nice explanation of CPU Modes with code](http://www.codeproject.com/Articles/45788/The-Real-Protected-Long-mode-assembly-tutorial-for)
* [How to Use Expand Down Segments on Intel 386 and Later CPUs](http://www.sudleyplace.com/dpmione/expanddown.html)
* [earlyprintk documentation](http://lxr.free-electrons.com/source/Documentation/x86/earlyprintk.txt)
* [Kernel Parameters](https://github.com/torvalds/linux/blob/master/Documentation/kernel-parameters.txt)
* [Serial console](https://github.com/torvalds/linux/blob/master/Documentation/serial-console.txt)
* [Intel SpeedStep](http://en.wikipedia.org/wiki/SpeedStep)
* [APM](https://en.wikipedia.org/wiki/Advanced_Power_Management)
* [EDD specification](http://www.t13.org/documents/UploadedDocuments/docs2004/d1572r3-EDD3.pdf)
* [TLDP documentation for Linux Boot Process](http://www.tldp.org/HOWTO/Linux-i386-Boot-Code-HOWTO/setup.html) (old)
* [1つ前のパート](linux-bootstrap-1.md)
