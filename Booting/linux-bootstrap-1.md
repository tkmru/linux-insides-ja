Kernel booting process. Part 1.
================================================================================

ブートローダーからカーネルまで
--------------------------------------------------------------------------------


もし私のブログの[記事](http://0xax.blogspot.com/search/label/asm)を読まれた方はご存じかと思いますが、ちょっと前から低レイヤーのプログラミングを行っています。
Linux用x86_64アセンブリによるプログラミングについて記事を書いていて、Linuxのソースコードにも触れるようになりました。
低レイヤーがどのように機能しているのか、コンピュータでプログラムがどのように実行されるのか、どのようにメモリに配置されるのか、カーネルがどのようにプロセスとメモリを扱うのか、低レイヤーでネットワークスタックがどのように動くのか等、多くのことを理解しようととても興味が湧いています。
それで、**x86_64** のLinux カーネルについてのシリーズを書こうと決心しました。

私はプロのカーネルプログラマではないことと、仕事でもカーネルのコードを書いていないことをご了承ください。
ただの趣味です。私は低レイヤーが単に好きで、どのようにして動いているのかとても興味があります。もし何か困惑した点や、ご質問やご意見がありましたら、Twitter [0xAX](https://twitter.com/0xAX) や [email](anotherworldofworld@gmail.com) でお知らせいただくか、[issue](https://github.com/0xAX/linux-insides/issues/new)を作成してください。
そうしてくれると助かります。全ての記事は [linux-insides](https://github.com/0xAX/linux-insides) からアクセスでき、私の英文が間違っていたり内容に問題があったりした場合は、気軽にプルリクエストを送ってください。

*これは正式なドキュメントではありません。あくまでも学習のためや知識共有のためのものですのでご注意ください。*

**必要な知識**

* Cコードの理解
* アセンブリ(AT&T記法)の理解

ツールについて学び始めている人のために、この記事とつづく記事の中で説明を入れようと思います。さて、簡単な導入はここで終わりにして、今からカーネルと低レイヤーにダイブしましょう。

全てのコードはカーネル 3.18のものです。変更があった場合は、私はそれに応じて更新します。

魔法の電源ボタンの次はなにが起こるのか？
--------------------------------------------------------------------------------

本連載はLinux カーネルついてのシリーズですが、カーネルのコードからは始めません。 - 少なくともこの段落では。ラップトップやデスクトップコンピューターは魔法の電源ボタンを押すと起動します。
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
しかし、16ビットのレジスタしかなく、16ビットのレジスタが使用できるアドレスは最大で `2^16-1`、 または `0xffff`(64KB)までです。[Memory segmentation](http://en.wikipedia.org/wiki/Memory_segmentation)は、利用可能なアドレス空間すべてを利用するために用いられる方法です。
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

実際のブートセクタの場合、この続きは多くの0たちや感嘆符ではなく、起動処理とパーティションテーブルになります。これ以降はBIOSからブートローダーに動作が移ります。

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

[GRUB2](https://www.gnu.org/software/grub/) や [syslinux](http://www.syslinux.org/wiki/index.php/The_Syslinux_Project) のような、Linuxを起動させることができるブートローダーは数多くあります。
Linuxカーネルは、Linuxサポートを実行するためのブートローダーに必要な条件を指定する[Boot protocol](https://github.com/torvalds/linux/blob/master/Documentation/x86/boot.txt)を持っています。
ここでは例として GRUB2 について述べます。

BIOSはブートデバイスを選んで、ブートセクタコードに対する制御を伝達し、[boot.img](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/boot/i386/pc/boot.S;hb=HEAD)から実行を開始します。
このコードは、利用可能な空間が限られているため非常にシンプルで、GRUB2 のコアイメージの位置へジャンプするためのポインタを含んでいます。
コアイメージは [diskboot.img](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/boot/i386/pc/diskboot.S;hb=HEAD) で始まりますが、最初のパーティションの前の未使用のスペースにある最初のセクタの直後に格納されます。
上記のコードは残りのコアイメージをメモリにロードしますが、それには GRUB2 のカーネルとファイルシステムを取り扱うためのドライバを含んでいます。
残りのコアイメージをロードした後に、[grub_main](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/kern/main.c)を実行します。

`grub_main` は、コンソールの初期化、モジュールのためのベースアドレスの取得、ルートデバイスの設定、GRUB設定ファイルの ロード/パース、モジュールのロードなどを行います。実行の最後には、`grub_main` がGRUBを通常モードへ移動させます。
`grub_normal_execute`（`grub-core/normal/main.c`）が最後の準備を完了させ、OSを選択するためのメニューを表示します。
GRUBメニューのエントリの１つを選択するとき、`grub_menu_execute_entry` が実行され、grub`boot`コマンドを実行して、選択したOSをブートします。

カーネルのブートプロトコルを見て分かるように、ブートローダーはカーネルのセットアップヘッダを読み込み、いくつかのフィールドを満たさなければいけません。
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

ブートローダーは、これと、（[この例](https://github.com/torvalds/linux/blob/master/Documentation/x86/boot.txt#L354)のようなLinuxブートプロトコルのwriteでマークされている）残りのヘッダを、コマンドラインまたは計算し求めた値で埋める必要があります。
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

ブートローダーがカーネルに制御を移したとき、以下のアドレスで開始されます。:

```
X + sizeof(KernelBootSector) + 1
```

`X` がカーネルのブートセクタがロードされている位置を示します。この場合は、`X` が `0x10000` で、メモリダンプに見て取れます。:

![kernel first address](http://oi57.tinypic.com/16bkco2.jpg)

ブートローダーはLinuxカーネルをメモリへロードし、ヘッダのフィールドを埋め、該当のメモリアドレスへジャンプします。
今、われわれはカーネルのセットアップコードへ直接移動することができます。

Kernelの設定を始める
--------------------------------------------------------------------------------

われわれは、ついにカーネルまでたどり着きました。しかし、カーネルはまだ起動しません。
最初に、カーネルとメモリ管理、プロセス管理などの設定が必要になります。
カーネルのセットアップの実行は[_start](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S#L293)で
[arch/x86/boot/header.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S)から開始します。
いくつかの命令が手前にあって、最初は少し奇妙に見えるかもしれません。

昔はLinuxカーネルが自前でブートローダーを持っていました。しかし、今は実行すると例のようになります。

```
qemu-system-x86_64 vmlinuz-3.18-generic
```

次のような結果が見られるはずです。:

![Try vmlinuz in qemu](http://oi60.tinypic.com/r02xkz.jpg)

実際は（画像にある）[MZ](https://en.wikipedia.org/wiki/DOS_MZ_executable)からheader.Sが開始され、[PE](https://en.wikipedia.org/wiki/Portable_Executable)ヘッダに続いて、エラーメッセージが表示されます。:

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

これには[UEFI](https://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface)モードでOSを起動することが必要です。
にこれが内部で動作するかどうか確認しませんが、続く章の中の1つで見ていきましょう。

これがカーネルセットアップのエントリポイントです。:

```assembly
// header.S line 292
.globl _start
_start:
```

ブートローダー（grub2など）はこのポイント（`MZ`からのオフセット`0x200`）を知っています。
`header.S` がエラーメッセージが表示される。`bstext`セクションから始まっているにも関わらず、このエントリポイントへ直接ジャンプします。:

```
//
// arch/x86/boot/setup.ld
//
. = 0;                    // current position
.bstext : { *(.bstext) }  // put .bstext section to position 0
.bsdata : { *(.bsdata) }
```

カーネルセットアップのエントリポイントはこちらです。:

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

ここでは`start_of_setup-1f`のポイントにジャンプする`jmp`命令のオペコード `0xeb`を見ることが出来ます。
`Nf`表記が意味するところは、`2f`が次のローカル`2:`ラベルを表しているということです。この場合、ジャンプした直後に行くのがラベル`1`です。
そこには残りの[セットアップヘッダ](https://github.com/torvalds/linux/blob/master/Documentation/x86/boot.txt#L156)も含まれます。セットアップヘッダのすぐ後に、`start_of_setup` ラベルで開始される`.entrytext`があります。

実際にはこれが（さっきのジャンプ命令を除いて）最初に実行するコードです。
カーネルセットアップにブートローダーから制御を移された後に、最初の`jmp`命令がカーネルのリアルモードの開始からオフセット`0x200`（最初の512Byteの後）に格納されます。
これは次のLinux カーネルブートプロトコルとgrub2のソースコードを見て分かります。:

```C
segment = grub_linux_real_target >> 4;
state.gs = state.fs = state.es = state.ds = state.ss = segment;
state.cs = segment + 0x20;
```

カーネルセットアップが始まった後、セグメントレジスタが以下の値を持つことを意味します。:

```
gs = fs = es = ds = ss = 0x1000
cs = 0x1020
```

この場合は、カーネルが`0x10000`に置かれます。

`start_of_setup`にジャンプした後は、カーネルが以下の作業をする必要があります。:

* すべてのセグメントレジスタの値が同じか確認する。
* 必要にであれば、正しくスタックをセットアップする。
* [bss](https://en.wikipedia.org/wiki/.bss)をセットアップする。
* [main.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/main.c)のCコードにジャンプする。

次は実装を見ていきましょう。


セグメントレジスタのアライメント
--------------------------------------------------------------------------------

まず、セグメントレジスタ `ds`と`es`が同じアドレスを指すようにし、次に`cld`命令を実行してdirection flagをクリアします。:

```assembly
    movw    %ds, %ax
    movw    %ax, %es
    cld
```

前述したとおり、grub2はカーネルのセットアップコードをアドレス`0x10000`に、`cs`に`0x1020`をロードします。
なぜなら、ファイルの冒頭から実行されるのではなく、以下のコードから実行されるからです。

```assembly
_start:
    .byte 0xeb
    .byte start_of_setup-1f
```

`jump`命令は[4d 5a](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S#L47)から512 Byte離れたところにあります。
また、他の全てのセグメントレジスタと同じように、`cs`を`0x1020`から`0x10000`までアラインする必要があります。それが終わったらスタックを設定します。:

```assembly
    pushw   %ds
    pushw   $6f
    lretw
```

`ds`の値をスタックにプッシュし、ラベル[6](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S#L494)のアドレスもスタックにプッシュすると、`lretw`命令が実行されます。
`lretw`命令を呼び出すと、ラベル6のアドレスが[instruction pointer](https://en.wikipedia.org/wiki/Program_counter)レジスタにロードされ、`ds`の値が`cs`にロードされます。
それが完了すると、`ds`と`cs`は同じ値を持つようになります。

スタックの設定
--------------------------------------------------------------------------------

リアルモードでだいたい全てのセットアップコードは、C言語の開発環境を作る準備となります。次の[ステップ](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S#L467)では`ss`レジスタの値をチェックし、もし`ss`が間違っている場合は正しいスタックを設定します。:

```assembly
    movw    %ss, %dx
    cmpw    %ax, %dx
    movw    %sp, %dx
    je      2f
```

これは、異なる3つのシナリオを導くことが可能です。:

* `ss`が有効値0x10000を持つ（`cs`を除く全てのセグメントレジスタと同様）
* `ss`は無効で、`CAN_USE_HEAP`フラグがセットされている（下記参照）
* `ss`は無効で、`CAN_USE_HEAP`フラグがセットされていない（下記参照）

3つのすべてのシナリオを全て見てみましょう。

* `ss`は正しいアドレス（0x10000）を持つ。この場合、ラベル[2](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S#L481)へと飛びます。:

```assembly
2:  andw    $~3, %dx
    jnz     3f
    movw    $0xfffc, %dx
3:  movw    %ax, %ss
    movzwl  %dx, %esp
    sti
```

ここで、`dx`（ブートローダーによって与えられる`sp`を含みます）が4Byte にアライメントされ、0になっているかどうか確認できます。
もし0の場合は`0xfffc`（最大のセグメントサイズの64KBより前で4Byteにアラインされたアドレス）を`dx`に代入します。
0でない場合は、引き続きブートローダーから与えられたsp（この例では0xf7f4）を使います。
正しいセグメントアドレス`0x10000`を格納しているssにaxの値を代入した後で、正しいspの値を設定します。これで正しくスタックを設定できました。:

![stack](http://oi58.tinypic.com/16iwcis.jpg)

* 2つ目のシナリオでは（`ss` != `ds`）となります。最初に、[_end](https://github.com/torvalds/linux/blob/master/arch/x86/boot/setup.ld#L52)(セットアップコードの最後のアドレス)の値をdxに置き、`loadflags`のヘッダフィールドを`testb`命令を使ってチェックし、ヒープ領域を使えるかどうかを確認します。[loadflags](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S#L321)は、以下のように定義されるビットマスクヘッダです。:

```C
#define LOADED_HIGH     (1<<0)
#define QUIET_FLAG      (1<<5)
#define KEEP_SEGMENTS   (1<<6)
#define CAN_USE_HEAP    (1<<7)
```

そしてブートプロトコルを読むと、以下のように書かれています。

```
Field name: loadflags

  This field is a bitmask.

  Bit 7 (write): CAN_USE_HEAP
    Set this bit to 1 to indicate that the value entered in the
    heap_end_ptr is valid.  If this field is clear, some setup code
    functionality will be disabled.
```

`CAN_USE_HEAP`のbitがセットされたときは、`_end`を指す`dx`に`heap_end_ptr`を置き、そこに`STACK_SIZE`（最小のスタックのサイズは512Byte）を加えます。
これ以降、dxがキャリーされていない場合（キャリーされてなければ、dx = _end + 512となる）、ラベル`2`(前のケースと同じように)にジャンプし、正しいスタックを作ります。

![stack](http://oi62.tinypic.com/dr7b5w.jpg)

* `CAN_USE_HEAP`がセットされてないとき、`_end`から`_end + STACK_SIZE`までの最小のスタックを使います。:

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
そして `cx` は2ビット右シフトすることで、4（word長）で割られ、`stosl` 命令を繰り返し`di`が指すアドレスに `eax` の値（0）を格納して、`di` は自動的に4ずつ増加し、`cx` が0になるまで繰り返されます。
このコードの効果は、`__bss_start` から `_end`まで、メモリ内にある全てのWordを通して、0が書きこむことです。:

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
もし質問や提案があれば Twitter [0xAX](https://twitter.com/0xAX) や [email](anotherworldofworld@gmail.com) で連絡していただくか、Issueを作成してください。
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
