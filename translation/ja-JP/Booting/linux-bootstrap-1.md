# カーネルのブートプロセス（#1）

## ブートローダーからカーネルまで

私の以前の記事「[blog posts](https://0xax.github.io/categories/assembler/)」をお読みになったことがあれば、私が低レベルプログラミングに関わっていた時期があることにお気づきかと思います。私はこれまで`x86_64` Linux向けのアセンブリプログラミングについてブログ記事をいくつか書いたことがあり、それと同時にLinuxカーネルのソースコードを追いかけ始めました。

私は低レベル部分の動作や、プログラムがコンピュータ上で実行される仕組みや、プログラムがメモリに割り当てられる仕組みや、カーネルがプロセスやメモリを管理する仕組みや、ネットワークスタックが低レベルでどのように動くか、といったさまざまなことにたまらなく興味を覚えました。そして私は、**x86_64**アーキテクチャ向けのLinuxカーネルについてさらに記事を書こうと決めたのです。

なお、私はプロのカーネルハッカーでもなければ仕事でカーネルコードを書いたこともありません。まったくの趣味です。私は低レベル周りが好きでたまらず、その仕組みに心を惹かれています。読んでいてこんがらがってきたり、質問やフィードバックがありましたら、Twitter）[0xAX](https://twitter.com/0xAX)）か[メール](anotherworldofworld@gmail.com)でお知らせいただくか、あるいは[issue](https://github.com/0xAX/linux-insides/issues/new)を立ててください。よろしくお願いします。

すべての記事は[GitHubリポジトリ](https://github.com/0xAX/linux-insides)でも読めます。英文や内容に誤りがありましたら、お気軽にプルリクを送ってください。

*本書はLinuxの公式ドキュメントではありません。学習と知識を共有するためのものです。*

**必要な知識**

- C言語のコードを理解できること
- アセンブリコード（AT&T構文）を理解できること

いずれにせよ、こうしたツールを学び始めたばかりの方向けに、今後なるべくある程度の解説を加える予定です。おっと、手短ですが導入部はこのぐらいにしておきましょう。ここから先はLinuxカーネルや低レベルの世界に飛び込むことになります。

本書を書き始めたときのLinuxカーネルのバージョンは`3.18`でしたが、それ以来相当な変更を加えてきました。今後もLinuxカーネルに更新があり次第反映する予定です。

## 魔法の電源ボタンを押すと次に何が起きるか

この文書はLinuxカーネルに関するシリーズ記事ですが、いきなりカーネルのコードから始めたりはしません（少なくともこのパラグラフでは）。ノートPCなりデスクトップPCなりの魔法の電源ボタンを押すと、たちまち何かが動き出します。マザーボードから[パワーサプライ（電源ユニット）](https://en.wikipedia.org/wiki/Power_supply)機器に信号を1つ送信し、パワーサプライが信号を受け取ると、コンピュータに電力が供給されます。マザーボードが[power goodシグナル](https://en.wikipedia.org/wiki/Power_good_signal)を受け取ると、CPUの起動を開始します。CPUはレジスタ内に残ったデータをすべてリセットして、各レジスタに定義済みの値をセットします。

[80386](https://en.wikipedia.org/wiki/Intel_80386) CPU以降のCPUでは、コンピュータのリセット後に以下の定義済みのデータを定義します。

```
IP          0xfff0
CS selector 0xf000
CS base     0xffff0000
```

するとプロセッサは[realモード](https://en.wikipedia.org/wiki/Real_mode)で動き出します。ここで少し前に戻って、realモードの[メモリセグメンテーション](https://en.wikipedia.org/wiki/Memory_segmentation)について理解してみましょう。realモードは、[8086](https://en.wikipedia.org/wiki/Intel_8086) CPUからモダンなIntel 64ビットCPUに至るまで、x86互換のあらゆるプロセッサでサポートされています。`8086`プロセッサのアドレスバスは20ビットなので、`0-0xFFFFF`すなわち`1メガバイト`のアドレス空間を持ちます。しかしレジスタは`16ビット`しかないので、最大アドレス空間は`2^16 - 1`すなわち`0xffff` （64キロバイト）です。

[メモリセグメンテーション](https://en.wikipedia.org/wiki/Memory_segmentation)は、このアドレス空間をすべて使うために用いられます。全メモリは、`65536`バイト（64 KB）の小さな固定サイズのセグメントに分割されます。16ビットレジスタでは`64 KB`を超えるメモリのアドレッシングができないので、別の方法を用います。

1つのアドレスは2つの部分からなります。「セグメントセレクタ」はベースアドレスを1つ持ち、「オフセット」はベースアドレスからのオフセットを1つ持ちます。realモードの場合、1つのセグメントセレクタに関連するベースアドレスは`セグメントセレクタ * 16`になります。つまり、メモリの物理アドレスを取得するには、次のようにこのセグメントセレクタを16倍してオフセットをそこに加えます。

```
PhysicalAddress = Segment Selector * 16 + Offset
```

たとえば、`CS:IP`が「`0x2000:0x0010`」の場合、対応する物理アドレスは以下のようになります。

```python
>>> hex((0x2000 << 4) + 0x0010)
'0x20010'
```

しかし、セグメントセレクタとオフセットを最大に取ると（`0xffff:0xffff`）、得られる物理アドレスは次のようになります。

```python
>>> hex((0xffff << 4) + 0xffff)
'0x10ffef'
```

これは最初のメガバイトから`65520`バイト先の位置になります。realモードでアクセスできるのは1メガバイトだけなので、`0x10ffef`は`0x00ffef`となり、[A20 line](https://en.wikipedia.org/wiki/A20_line)は無効になります。

それでは、realモードとメモリのアドレッシングについて少し理解が進んだところで少し前に戻り、リセット後のレジスタの値について説明します。

`CS`レジスタは（可視の）「セグメントセレクタ」と、（非可視の）「ベースアドレス」の2つでできています。通常のベースアドレスは、セグメントセレクタを16倍する形になっていますが、ハードウェアのリセット中のセグメントセレクタは、CSレジスタが`0xf000`で読み込まれ、ベースアドレスが`0xffff0000`で読み込まれます。プロセッサは、`CS`が変更されるまでの間この特殊なベースアドレスを使います。

開始アドレスは、次のようにEIPレジスタの値にベースアドレスを加えた形になります。

```python
>>> 0xffff0000 + 0xfff0
'0xfffffff0'
```

`0xfffffff0`という値が得られますが、これは4GB先の16バイトを指します。この部分は[リセットベクタ](https://en.wikipedia.org/wiki/Reset_vector)と呼ばれ、リセット後にCPUが最初のインストラクションをここで見つけられることが期待されます。リセットベクタには[jump](https://en.wikipedia.org/wiki/JMP_%28x86_instruction%29)（`jmp`）インストラクションが含まれ、これはBIOSのエントリポイントを指すのが普通です。[coreboot](https://www.coreboot.org/)のソースコード（`src/cpu/x86/16bit/reset16.inc`）は以下のようになっています。

```assembly
    .section ".reset", "ax", %progbits
    .code16
.globl	_start
_start:
    .byte  0xe9
    .int   _start16bit - ( . + 2 )
    ...
```

この部分の`jmp`インストラクションの[opcode](http://ref.x86asm.net/coder32.html#xE9)は`0xe9`となっており、その行き先アドレスは`_start16bit - ( . + 2)`です。

`reset`セクションは`16`バイトで、`0xfffffff0`というアドレスから開始されるようコンパイルされています（`src/cpu/x86/16bit/reset16.ld`）。

```
SECTIONS {
    /* Trigger an error if I have an unuseable start address */
    _bogus = ASSERT(_start16bit >= 0xffff0000, "_start16bit too low. Please report.");
    _ROMTOP = 0xfffffff0;
    . = _ROMTOP;
    .reset . : {
        *(.reset);
        . = 15;
        BYTE(0x00);
    }
}
```

ここでBIOSが起動しました。ハードウェアの初期化とチェックが完了すると、BIOSはブート可能なデバイスを検索する必要があります。ブートの順序はBIOS設定に保存され、BIOSがどのデバイスから起動を試みるかを制御します。ハードディスクからの起動を試みる場合、BIOSはブートセクタを探そうとします。ハードディスクのパーティショニングが[MBRパーティションレイアウト](https://en.wikipedia.org/wiki/Master_boot_record)になっている場合、ブートセクタは最初のセクタの`446`バイトに保存されます。なお各セクタのサイズは`512`バイトです。最初のセクタの最後の2バイトは`0x55`と`0xaa`になっており、これはそのデバイスがブート可能であることを示します。

以下の例をご覧ください。

```assembly
;
; Note: this example is written in Intel Assembly syntax
;
[BITS 16]

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

これを以下のコマンドでビルドして実行します。

```
nasm -f bin boot.nasm && qemu-system-x86_64 boot
```

このコマンドは、あらかじめディスクイメージとしてビルドしておいた`boot`というバイナリを用いるよう[QEMU](http://qemu.org)に指示します。前述のアセンブリコードで生成されたバイナリはブートセクタとしての要件を満たしている（元は`0x7c00`にセットされており、マジックシーケンスで終わっています）ので、QEMUはこのバイナリをディスクイメージのマスタブートレコード（MBR）として扱います。

実行結果は次のとおりです。

![Simple bootloader which prints only ](http://oi60.tinypic.com/2qbwup0.jpg)

この例では、コードが`16-bit`のrealモードで実行され、メモリの`0x7c00`が開始地点になります。開始後に[0x10](http://www.ctyme.com/intr/rb-0106.htm)割り込みが呼び出されますが、これは`!`記号を出力するだけのものです。残りの`510`バイトはゼロで埋められ、末尾は`0xaa`と`0x55`になります。

`objdump`ユーティリティで次のようにバイナリをダンプできます。

```
nasm -f bin boot.nasm
objdump -D -b binary -mi386 -Maddr16,data16,intel boot
```

現実のブートセクタでは、ゼロの連続と感嘆符1つの代わりに、ブートプロセスの続きやパーティションテーブルが置かれます☺️。BIOSはこの時点で制御をブートローダーに渡します。

**メモ**: ここでは前述のようにCPUがrealモードで動作しているので、メモリの物理アドレスは以下のように算出されます。

```
PhysicalAddress = Segment Selector * 16 + Offset
```

前述のとおり、このモードでは16ビットの汎用レジスタしか使えません。16ビットレジスタの最大値は`0xffff`なので、値を最大にすると結果は以下のようになります。

```python
>>> hex((0xffff * 16) + 0xffff)
'0x10ffef'
```

`0x10ffef`は`1MB + 64KB - 16b`に等しい値です。[8086](https://en.wikipedia.org/wiki/Intel_8086)プロセッサ（realモードを備えた最初のプロセッサ）のアドレス線は20ビットです。`2^20 = 1048576`は1MBなので、実際に利用可能なメモリは1MBになります。

realモードの一般的なメモリマップは以下のようになります。

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

本記事の冒頭で、CPUが最初に実行するインストラクションが置かれるアドレスは`0xFFFFFFF0`と説明しましたが、これは`0xFFFFF`（1MB）よりも遥かに巨大な空間です。どうすればrealモードのCPUがこのアドレスにアクセスできるのでしょうか？その答えは[coreboot](https://www.coreboot.org/Developer_Manual/Memory_map)のドキュメントにあります。

```
0xFFFE_0000 - 0xFFFF_FFFF: 128 kilobyte ROM mapped into address space
```

実行開始の段階では、BIOSがRAMではなくROMに置かれているからです。

## ブートローダー

Linuxをブートできるブートローダーは1つではなく、[GRUB 2](https://www.gnu.org/software/grub/)や[syslinux](http://www.syslinux.org/wiki/index.php/The_Syslinux_Project)などさまざまなものがあります。Linuxカーネルには[Bootプロトコル](https://github.com/torvalds/linux/blob/v4.16/Documentation/x86/boot.txt)という仕様が定められており、Linuxサポートを実装するのに必要な要件が指定されています。本記事の例ではGRUB 2について解説します。

さっきの続きです。`BIOS`でブートデバイスが選択済みで、制御がブートセクタのコードに移ると、今度は[boot.img](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/boot/i386/pc/boot.S;hb=HEAD)の実行に進みます。このコードは（使える容量が制限されているので）きわめてシンプルで、GRUB 2のコアイメージが置かれている場所へのジャンプに使うポインタを1つ含みます。コアイメージは[diskboot.img](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/boot/i386/pc/diskboot.S;hb=HEAD)から始まります。これは通常、最初のパーティションより前にある未使用スペースの最初のセクタの直後に保存されています。上述のコードによってコアイメージの残り（GRUB 2のカーネルやファイルシステム操作用ドライバを含む）がメモリに読み込まれます。コアイメージの残りを読み込み終わると、[grub_main](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/kern/main.c)関数を実行します。

`grub_main`関数は、コンソールの初期化、さまざまなモジュールのベースアドレス取得、ルートデバイスの設定、grubの設定ファイルの読み込みと解析（parse）、モジュールの読み込みなどを行います。`grub_main`の実行が終了するときに、grubをnormalモードに変更します。`grub_normal_execute`関数（ソースファイルは`grub-core/normal/main.c`）は最後の準備を完了して、OSを選択するメニューを表示します。grubメニューのエントリを1つ選択すると、`grub_menu_execute_entry`が実行され、指定のOSをブートするためのgrbの`boot`コマンドを実行します。

カーネルのブートプロトコルに記載されているように、ブートローダーはカーネルセットアップヘッダーの一部のフィールドを読み込み、そこに値を設定しなければなりません。このヘッダーの開始場所は、カーネルのセットアップコードからのオフセット`0x01f1`になります。このオフセット値は、ブートの[linkerスクリプト](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/setup.ld)で確認できます。カーネルヘッダー[arch/x86/boot/header.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S)の冒頭は以下のようになっています。

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

ブートローダーは、このヘッダーの残りの値（Linuxブートプロトコルでは、[この例](https://github.com/torvalds/linux/blob/v4.16/Documentation/x86/boot.txt#L354)のように`write`とだけマーキングされています）に、コマンドラインで受け取った値またはブート中に算出された値のいずれかを設定しなければなりません。（カーネルのセットアップヘッダーのすべてのフィールドについて今は詳しく説明しませんが、カーネルで使われるフィールドについてはその都度説明します: すべてのフィールドの説明については[bootプロトコル](https://github.com/torvalds/linux/blob/v4.16/Documentation/x86/boot.txt#L156)をご覧ください）。

カーネルのブートプロトコルに記載されているように、カーネルが読み込まれた後のメモリマッピングは以下のようになります。

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

つまり、ブートローダーが制御をカーネルに渡すときの開始位置は以下のようになります。

```
X + sizeof(KernelBootSector) + 1
```

`X`は、読み込まれるカーネルブートセクタのアドレスを表します。この場合の`X`は、以下のメモリダンプでわかるように`0x10000`になります。

![kernel first address](http://oi57.tinypic.com/16bkco2.jpg)

この時点で、ブートローダはLinuxカーネルをメモリに読み込み完了し、ヘッダーフィールドに値をセットしてから、対応するメモリアドレスにジャンプします。ここでめでたくカーネルのセットアップコードに直接移動できるようになりました。

## カーネルのセットアップステージの冒頭部分

ついにカーネルまでたどり着きました（まだカーネルは実行されていませんが）。まずはカーネルのセットアップで、解凍（decompressor）やメモリ管理絡みの部分を設定しなければなりません。これらが完了すると、カーネルのセットアップ部分で実際のカーネルを解凍してそこにジャンプします。セットアップ部分の実行の冒頭は、[arch/x86/boot/header.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S)の[_start](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L292)シンボルになります。

この部分の前にはインストラクションがいくつも置かれているので、最初ちょっと奇妙に思えるかもしれません。実を言うと、はるか昔のLinuxカーネルでは、ここに独自のブートローダーが置かれていたのです。それはさておき、たとえば以下を実行したとしましょう。

```
qemu-system-x86_64 vmlinuz-3.18-generic
```

すると以下が表示されます。

![Try vmlinuz in qemu](http://oi60.tinypic.com/r02xkz.jpg)

実は、`header.S`ファイルの冒頭にはマジックナンバー[MZ](https://en.wikipedia.org/wiki/DOS_MZ_executable)（前述のイメージを参照）があり、表示する若干のエラーメッセージがそれに続き、さらに[PE](https://en.wikipedia.org/wiki/Portable_Executable)ヘッダーがあります。

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

ここはOSを[UEFI](https://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface)サポート付きで読み込むのに必要です。今は詳細に立ち入りませんが、この後の章でカバーする予定です。

カーネルセットアップの実際のエントリポイントは以下になります。

```assembly
// header.S line 292
.globl _start
_start:
```

ブートローダー（grub2など）はこの位置（`MZ`からのオフセット`0x200`）を認識してそこに直接ジャンプしますが、実は`header.S`が`.bstext`セクションから始まっているせいで、エラーメッセージが出力されます。

```
//
// arch/x86/boot/setup.ld
//
. = 0;                    // current position
.bstext : { *(.bstext) }  // put .bstext section to position 0
.bsdata : { *(.bsdata) }
```

この場合、カーネルセットアップのエントリポイントは次のようになります。

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

上の`jmp`インストラクションとopcode `0xeb`は、`start_of_setup-1f`の地点にジャンプします。`Nf`記法では、たとえば`2f`はローカルラベル`2:`を参照しますが、この場合はジャンプの直後に現れるラベル`1`になり、ここにはセットアップ[ヘッダー](https://github.com/torvalds/linux/blob/v4.16/Documentation/x86/boot.txt#L156)の残りが含まれます。セットアップヘッダーの直後には`.entrytext`セクションがあり、その冒頭には`start_of_setup`ラベルがあります。

ここが実際に実行される最初のコードです（もちろん先のjumpインストラクションを別にすればですが）。カーネルのセットアップ部分がブートローダーから制御を受け取った後の最初の`jmp`インストラクションの位置は、カーネルのrealモードの開始地点からのオフセット`0x200`になります（最初の512バイト以降など）。この部分は、Linuxカーネルのブートプロトコルとgrub2のソースコードの両方に記載されています。

```C
segment = grub_linux_real_target >> 4;
state.gs = state.fs = state.es = state.ds = state.ss = segment;
state.cs = segment + 0x20;
```

この場合、カーネルは物理アドレス`0x10000`に読み込まれます。これは、カーネルのセットアップが開始した後でセグメントレジスタに次の値を設定するということです。

```
gs = fs = es = ds = ss = 0x1000
cs = 0x1020
```

`start_of_setup`にジャンプした後、カーネルは以下を行う必要があります。

- どのセグメントレジスタも値が等しいことを確認する
- 必要に応じて適切なスタックをセットアップする
- [bss](https://en.wikipedia.org/wiki/.bss)（Block Started by Symbol）をセットアップする
- [arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c)のC言語コードにジャンプする

実装を見てみましょう。

## セグメントレジスタのアラインメント

最初に、カーネルは`ds`セグメントレジスタと`es`セグメントレジスタが同じアドレスを指していることを確認します。続いて、`cld` インストラクションでdirectionフラグをクリアします。

```assembly
    movw    %ds, %ax
    movw    %ax, %es
    cld
```

前述したように、`grub2`はデフォルトではアドレス`0x10000`にあるカーネルセットアップコードを読み込んで`0x1020`の`cs`を読み込みます。理由は、実行開始地点はファイルの冒頭ではなく、以下のジャンプが開始地点になるからです。

```assembly
_start:
    .byte 0xeb
    .byte start_of_setup-1f
```

ここは[4d 5a](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L46)からのオフセット`512`バイトになります。さらに、他のすべてのセグメントレジスタと同様に`cs`も`0x1020`から`0x1000`にアラインする必要があります。それが終わったら、スタックをセットアップします。

```assembly
    pushw   %ds
    pushw   $6f
    lretw
```

上は、`ds`の値をスタックにプッシュし、続いて[6](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L602)ラベルのアドレスもスタックにプッシュしてから、`lretw`インストラクションを実行します。`lretw`インストラクションが呼ばれると、ラベル`6`のアドレスを[instruction pointer](https://en.wikipedia.org/wiki/Program_counter)レジスタに読み込んでから、`cs`を`ds`の値で読み込みます。これで、`ds`と`cs`の値が同じになります。

## スタックのセットアップ

セットアップコードのほぼすべては、realモードでのC言語環境を設定することに使われます。次の[ステップ](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L575)では、`ss`レジスタの値をチェックし、`ss`が正しくない場合は正しいスタックをセットアップします。

```assembly
    movw    %ss, %dx
    cmpw    %ax, %dx
    movw    %sp, %dx
    je      2f
```

ここでは3つの異なるシナリオがありえます。

- `ss`に正しい値`0x1000`がある（`cs`以外のすべてのレジスタで行ったのと同様に）
- `ss`の値が無効、かつ`CAN_USE_HEAP`フラグがセットされている（以下を参照）
- `ss`の値が無効、かつ`CAN_USE_HEAP`フラグがセットされていない（以下を参照）

それでは3つのシナリオをすべて順に見てみましょう。

- `ss`に正しいアドレス（`0x1000`）がある。この場合、ラベル[2](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L589)に進みます。

```assembly
2:  andw    $~3, %dx
    jnz     3f
    movw    $0xfffc, %dx
3:  movw    %ax, %ss
    movzwl  %dx, %esp
    sti
```

ここでは`dx`（ブートローダーから渡された`sp`の値を含む）のアラインメントを`4`バイトにセットし、ゼロかどうかをチェックします。ゼロの場合は`dx`に`0xfffc`（64KBセグメントのアラインされた末尾の4バイト）をセットします。ゼロでない場合は、ブートローダーから渡された`sp`の値（ここでは`0xf7f4`）を引き続き用います。それが終わったら、`ax`の値を`ss`に渡します（`ss`の値は`0x1000`）。これで以下のように正しいスタックができました。

![stack](http://oi58.tinypic.com/16iwcis.jpg)

- 次はシナリオ2（`ss` != `ds`）の解説です。まず[_end](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/setup.ld)の値（セットアップコードの末尾のアドレス）を`dx`に置き、続いて`testb`インストラクションで`loadflags`ヘッダーフィールドをチェックすることでヒープが利用可能かどうかを調べます。[loadflags](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L320)は以下のように定義されているビットマスクヘッダーです。

```C
#define LOADED_HIGH     (1<<0)
#define QUIET_FLAG      (1<<5)
#define KEEP_SEGMENTS   (1<<6)
#define CAN_USE_HEAP    (1<<7)
```

そしてブートプロトコルには以下の記述があります。

```
Field name: loadflags

  This field is a bitmask.

  Bit 7 (write): CAN_USE_HEAP
    Set this bit to 1 to indicate that the value entered in the
    heap_end_ptr is valid.  If this field is clear, some setup code
    functionality will be disabled.
```

`CAN_USE_HEAP`をセットすると、`heap_end_ptr`を`dx`に置き（これは`_end`を指します）、`STACK_SIZE`（最大スタックサイズである`1024`バイト）を追加します。これで、`dx`がキャリーオーバーしなければ（`dx = _end + 1024`はキャリーオーバーしない）、先ほどと同様にラベル`2`にジャンプし、正しいスタックが作られます。

![stack](http://oi62.tinypic.com/dr7b5w.jpg)

- `CAN_USE_HEAP`がセットされていない場合は、`_end`から`_end + STACK_SIZE`までの最小限のスタックを使います。

![minimal stack](http://oi60.tinypic.com/28w051y.jpg)

## BSSのセットアップ

メインのC言語コードにジャンプできるようになるまでに、まだステップが2つ残っています。[BSS](https://en.wikipedia.org/wiki/.bss)領域のセットアップと、「マジック」シグネチャの設定です。最初はシグネチャのチェックです。

```assembly
    cmpl    $0x5a5aaa55, setup_sig
    jne     setup_bad
```

上は単純に[setup_sig](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/setup.ld)をマジックナンバー`0x5a5aaa55`と比較しています。等しくない場合はfatal errorを出力します。

マジックナンバーが一致すれば、正しいセグメントレジスタのセットとスタックは既にわかっているので、Cコードにジャンプするのに必要なのはBSSセクションのセットアップだけとなります。

BSSセクションは、静的にアロケーションされた、初期化されていないデータの保存に使われます。Linuxは、以下のコードを用いてメモリのこの領域を注意深くゼロで埋めます。

```assembly
    movw    $__bss_start, %di
    movw    $_end+3, %cx
    xorl    %eax, %eax
    subw    %di, %cx
    shrw    $2, %cx
    rep; stosl
```

最初に、[__bss_start](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/setup.ld)アドレスを`di`に置きます。次に、`_end + 3`アドレス（+3で4バイトにアラインする）を`cx`に置きます。`xor`インストラクションで`eax`レジスタをゼロクリアして、bssセクションサイズ（`cx`-`di`）を算出し、`cx`に置きます。続いて、`cx`を4（1ワードのサイズ）で割り、`stosl`インストラクションを繰り返し使い、`di`が指すアドレスに`eax`の値（ゼロ）を置くと、`di`が4ずつ自動インクリメントされ、`cx`がゼロに達するまでこれを繰り返します。このコードの実際の効用は、`__bss_start`から`_end`までのメモリの全ワードをゼロで埋めることです。

![bss](http://oi59.tinypic.com/29m2eyr.jpg)

## mainにジャンプする

以上でスタックとBSSができたので、`main()` C関数にジャンプできるようになりました。

```assembly
    calll main
```

`main()`関数は[arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c)にあります。次のセクションでは、このコードの動作を解明します。

## 締めくくり

以上で、Linuxカーネル内部の最初の部分はおしまいです。ご意見ご質問がありましたら、Twitter（[0xAX](https://twitter.com/0xAX)）または[メール](anotherworldofworld@gmail.com)、もしくは[GitHub issue](https://github.com/0xAX/linux-internals/issues/new)にてお知らせください。次は、Linuxカーネルのセットアップで実行される最初のCコードと、`memset`や`memcpy`や `earlyprintk`、初期段階のコンソールの実装や初期化などさまざまな部分を見ていくことにします。

**私は英語ネイティブではありませんので、読みにくいところがありましたら恐縮です。英語の誤りがありましたら、[linux-insides](https://github.com/0xAX/linux-internals)までプルリクを送ってください。**

## リンク

- [Intel 80386 programmer's reference manual 1986](http://css.csail.mit.edu/6.858/2014/readings/i386.pdf)
- [Minimal Boot Loader for Intel® Architecture](https://www.cs.cmu.edu/%7E410/doc/minimal_boot.pdf)
- [8086](https://en.wikipedia.org/wiki/Intel_8086)
- [80386](https://en.wikipedia.org/wiki/Intel_80386)
- [Reset vector](https://en.wikipedia.org/wiki/Reset_vector)
- [Real mode](https://en.wikipedia.org/wiki/Real_mode)
- [Linux kernel boot protocol](https://www.kernel.org/doc/Documentation/x86/boot.txt)
- [coreboot developer manual](https://www.coreboot.org/Developer_Manual)
- [Ralf Brown's Interrupt List](http://www.ctyme.com/intr/int.htm)
- [Power supply](https://en.wikipedia.org/wiki/Power_supply)
- [Power good signal](https://en.wikipedia.org/wiki/Power_good_signal)
