Control Groups
================================================================================

はじめに
--------------------------------------------------------------------------------

これは、[linux insides](http://0xax.gitbooks.io/linux-insides/content/)の新しい章の最初のパートです。
パートの名前からあなたが予想したとおり、このパートではLinuxカーネル内での [control groups](https://en.wikipedia.org/wiki/Cgroups) や `cgroups`の仕組みを扱います。

`Cgroups` はLinuxカーネルが提供する特別な仕組みであり、プロセスやプロセスの集合に対して、Processor Time、グループあたりのプロセス数、コントロールグループあたりのメモリ量、またはそのようなリソースの組み合わせなどに対して `リソース` の種類を割り当てることができます。 
`Cgroups`は階層的に構成されています。仕組みは通常のプロセスと似ていて、階層的であるため、子の`cgroups`は親から特定のパラメータの集合を継承します。
しかし、実際には同じではありません。`cgroups` と通常のプロセスの主な違いは、通常のプロセスツリーは常に単一であるがコントロールグループでは多くの異なる階層が同時に存在する可能性があるということです。 
コントロールグループの階層がそれぞれ `サブシステム` のコントロールグループに関連付けられているため、単純ではありません。

ある `control group subsystem` はProcessor Time、または [pids](https://en.wikipedia.org/wiki/Process_identifier) の数や、言い換えるなら、`control group` のプロセス数を表します。
Linux カーネルは以下の12の `control group subsystems` のサポートを提供しています:

* `cpuset` - 個々のプロセッサ（たち）とメモリノードをグループ内のタスク（たち）に割り当てます。
* `cpu` - プロセッサの資源へcgroupタスクのアクセスを提供するためにスケジューラを使用します。
* `cpuacct` - グループによるプロセッサ使用状況に関するレポートを生成します。
* `io` - [block devices](https://en.wikipedia.org/wiki/Device_file)からの読み取り/への書き込みの制限を設定します。
* `memory` - グループからのタスク（たち）によるメモリ使用制限を設定します。
* `devices` - グループからのタスク（たち）によってデバイスへのアクセスを許可します。
* `freezer` - グループからのタスク（たち）の中断/再開を許可します。
* `net_cls` - グループからのタスク（たち）からのネットワークパケットをマークすることを許可します。
* `net_prio` - グループのネットワークインタフェース毎のネットワークトラフィックの優先度を動的に設定する方法を提供します。
* `perf_event` - グループに[perf events](https://en.wikipedia.org/wiki/Perf_(Linux))へのアクセスを提供します。
* `hugetlb` - グループの[huge pages](https://www.kernel.org/doc/Documentation/vm/hugetlbpage.txt)をサポートを有効にします。
* `pid` - グループ内のプロセス数に制限を設定します。

コントロールグループのサブシステムは関連する設定オプションに依存します。 
例えば、 `cpuset` サブシステムはカーネル設定オプションの`CONFIG_CPUSETS`、
`io`サブシステムはカーネル設定オプションの `CONFIG_BLK_CGROUP` を介して有効にする必要があります。
すべてのカーネルの設定のオプションは`General setup → Control Group support`で見つかるでしょう:

![menuconfig](http://oi66.tinypic.com/2rc2a9e.jpg)

あなたのコンピュータで有効なコントロールグループを[proc](https://en.wikipedia.org/wiki/Procfs)ファイルシステムで見ることができるでしょう:

```
$ cat /proc/cgroups 
#subsys_name	hierarchy	num_cgroups	enabled
cpuset	8	1	1
cpu	7	66	1
cpuacct	7	66	1
blkio	11	66	1
memory	9	94	1
devices	6	66	1
freezer	2	1	1
net_cls	4	1	1
perf_event	3	1	1
net_prio	4	1	1
hugetlb	10	1	1
pids	5	69	1
```

または、[sysfs](https://en.wikipedia.org/wiki/Sysfs)で見ることができるでしょう:

```
$ ls -l /sys/fs/cgroup/
total 0
dr-xr-xr-x 5 root root  0 Dec  2 22:37 blkio
lrwxrwxrwx 1 root root 11 Dec  2 22:37 cpu -> cpu,cpuacct
lrwxrwxrwx 1 root root 11 Dec  2 22:37 cpuacct -> cpu,cpuacct
dr-xr-xr-x 5 root root  0 Dec  2 22:37 cpu,cpuacct
dr-xr-xr-x 2 root root  0 Dec  2 22:37 cpuset
dr-xr-xr-x 5 root root  0 Dec  2 22:37 devices
dr-xr-xr-x 2 root root  0 Dec  2 22:37 freezer
dr-xr-xr-x 2 root root  0 Dec  2 22:37 hugetlb
dr-xr-xr-x 5 root root  0 Dec  2 22:37 memory
lrwxrwxrwx 1 root root 16 Dec  2 22:37 net_cls -> net_cls,net_prio
dr-xr-xr-x 2 root root  0 Dec  2 22:37 net_cls,net_prio
lrwxrwxrwx 1 root root 16 Dec  2 22:37 net_prio -> net_cls,net_prio
dr-xr-xr-x 2 root root  0 Dec  2 22:37 perf_event
dr-xr-xr-x 5 root root  0 Dec  2 22:37 pids
dr-xr-xr-x 5 root root  0 Dec  2 22:37 systemd
```

あなたはすでに `control groups` のメカニズムは、Linuxカーネルのニーズに直接的に開発された仕組みでなく、
主にユーザー空間のニーズに対して開発されたものだと勘づいているかもしれません。
`control groups`を使うには、まず最初に `control groups` を作成する必要があります。私たちは二つの方法で `cgroup`を作成するでしょう。

1つ目の方法は、`sys/fs/cgroup` からサブディレクトリを作成し、その作成直後に自動的に作成される `tasks`ファイルにタスクのpidを追加することです。

2つ目の方法は `libcgroup`ライブラリ（Fedoraの`libcgroup-tools`）のutilを使って `cgroups`を作成/破壊/管理する方法です。

シンプルな例を考えてみましょう。 以下の[bash](https://www.gnu.org/software/bash/)スクリプトは、現在のプロセスの制御端末を表す `/dev/tty`デバイスに出力します：

```shell
#!/bin/bash

while :
do
    echo "print line" > /dev/tty
    sleep 5
done
```

このスクリプトを走らせ、結果を見ましょう:

```
$ sudo chmod +x cgroup_test_script.sh
~$ ./cgroup_test_script.sh 
print line
print line
print line
...
...
...
```

`cgroupfs`がマウントされる場所を見てみましょう。 
見たとおり、これは `/sys/fs/cgroup` ディレクトリですが、あなたがマウントしたい場所にマウント出来ます。

```
$ cd /sys/fs/cgroup
```

`cgroup`のタスクによってデバイスへのアクセスを許可/拒否するリソースの種類を表す `devices`サブディレクトリに行きましょう。:

```
# cd /devices
```

`cgroup_test_group`ディレクトリを作ります:

```
# mkdir cgroup_test_group
```

`cgroup_test_group`ディレクトリの作成直後に、以下のファイルが生成されるでしょう:

```
/sys/fs/cgroup/devices/cgroup_test_group$ ls -l
total 0
-rw-r--r-- 1 root root 0 Dec  3 22:55 cgroup.clone_children
-rw-r--r-- 1 root root 0 Dec  3 22:55 cgroup.procs
--w------- 1 root root 0 Dec  3 22:55 devices.allow
--w------- 1 root root 0 Dec  3 22:55 devices.deny
-r--r--r-- 1 root root 0 Dec  3 22:55 devices.list
-rw-r--r-- 1 root root 0 Dec  3 22:55 notify_on_release
-rw-r--r-- 1 root root 0 Dec  3 22:55 tasks
```

この瞬間、`tasks`と` devices.deny`ファイルに興味が湧きます。
最初の `tasks`ファイルには、`cgroup_test_group`に付加されるプロセスのpid(s)が含まれていなければなりません。
2番目の `devices.deny`ファイルには、拒否されたデバイスのリストが含まれています。
デフォルトでは、新しく作成されたグループにはデバイスアクセスの制限がありません。
デバイスアクセスを禁止するには（この場合は `/dev/tty`）、以下の行に`devices.deny`を書きこみます:

```
# echo "c 5:0 w" > devices.deny
```

この行を1行1行見ていきましょう。最初の`c`はデバイスの種類を表します。
このケースでは、`/dev/tty`は`char device`です。`ls`コマンドの出力からこれを確認できます:

```
~$ ls -l /dev/tty
crw-rw-rw- 1 root tty 5, 0 Dec  3 22:48 /dev/tty
```

パーミッションリストの最初の`c`を見てください。
2番目の部分は`5:0`で、これはデバイスのマイナー番号とメジャー番号です。
これらの数字は`ls`の出力でも見ることができます。
最後の`w`は指定されたデバイスに書き込む作業を禁じます。
それでは、cgroup_test_script.shスクリプトを開始しましょう:

```
~$ ./cgroup_test_script.sh 
print line
print line
print line
...
...
```

そしてこのプロセスのpidを私たちのグループの`devices/tasks`ファイルに付け足します:

```
# echo $(pidof -x cgroup_test_script.sh) > /sys/fs/cgroup/devices/cgroup_test_group/tasks
```

結果は予想どおりになります:

```
~$ ./cgroup_test_script.sh 
print line
print line
print line
print line
print line
print line
./cgroup_test_script.sh: line 5: /dev/tty: Operation not permitted
```

同様の状況として[docker](https://en.wikipedia.org/wiki/Docker_(software))コンテナを実行するときを例に挙げられます:

```
~$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                    NAMES
fa2d2085cd1c        mariadb:10          "docker-entrypoint..."   12 days ago         Up 4 minutes        0.0.0.0:3306->3306/tcp   mysql-work

~$ cat /sys/fs/cgroup/devices/docker/fa2d2085cd1c8d797002c77387d2061f56fefb470892f140d0dc511bd4d9bb61/tasks | head -3
5501
5584
5585
...
...
...
```

`docker` コンテナの起動時に、`docker`はコンテナ内のプロセスのために`cgroup`を作成します:

```
$ docker exec -it mysql-work /bin/bash
$ top
 PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND                                                                                   1 mysql     20   0  963996 101268  15744 S   0.0  0.6   0:00.46 mysqld                                                                                  71 root      20   0   20248   3028   2732 S   0.0  0.0   0:00.01 bash                                                                                    77 root      20   0   21948   2424   2056 R   0.0  0.0   0:00.00 top                                                                                  
```

ホストマシンの`cgroup`を見られるかもしれません:

```C
$ systemd-cgls

Control group /:
-.slice
├─docker
│ └─fa2d2085cd1c8d797002c77387d2061f56fefb470892f140d0dc511bd4d9bb61
│   ├─5501 mysqld
│   └─6404 /bin/bash
```

これで、われわれは、`control group`の仕組み、手動での使用方法、この仕組みの目的について少し分かりました。
Linuxカーネルのソースコードを見て、この仕組みの実装を深く見ていきます。

コントロールグループの早期の初期化
--------------------------------------------------------------------------------

`control groups`のLinuxカーネルの仕組みに関する少しの理論を見ただけで、Linuxカーネルのソースコードを知ることができます。
いつものように、私たちは `control groups`の初期化から始めます。
`cgroups`の初期化は、Linuxカーネルの「早期」と「末期」の2つの部分に分かれています。 この部分では、「早期」と「末期」の部分のみが次の章で考慮されると考えます。

`cgroups`の早期の初期化は次の関数を呼ぶことから始まります:

```C
cgroup_init_early();
```

Linuxカーネルの早期の初期化の間に[init/main.c](https://github.com/torvalds/linux/blob/master/init/main.c)の中の関数を実行してください。
この関数は、[kernel/cgroup.c](https://github.com/torvalds/linux/blob/master/kernel/cgroup.c)のソース内で定義され、次の2つのローカル変数の定義から始まります:

```C
int __init cgroup_init_early(void)
{
	static struct cgroup_sb_opts __initdata opts;
	struct cgroup_subsys *ss;
    ...
    ...
    ...
}
```

`cgroup_sb_opts`構造体は同じソース内で定義されていて、以下のように見えます:

```C
struct cgroup_sb_opts {
	u16 subsys_mask;
	unsigned int flags;
	char *release_agent;
	bool cpuset_clone_children;
	char *name;
	bool none;
};
```


これは `cgroupfs` のマウントオプションを表します。
例えば、以下のコマンドは`name =`オプションを持ち、サブシステムを持たない、名付けられた cgroupの階層（名前は `my_cgrp`）を作成します:

```
$ mount -t cgroup -oname=my_cgrp,none /mnt/cgroups
```

2番目の変数 `ss`は、[include/linux/cgroup-defs.h](https://github.com/torvalds/linux/blob/master/include/linuxblob/master/include/linux/cgroup-defs.h)で定義されているタイプの `cgroup_subsys`構造体を持っています。 ヘッダファイルを参照し、タイプの名前から推測できるように、 `cgroup`サブシステムを表します。 この構造体には、次のようなさまざまなフィールドとコールバック関数が含まれています。:

```C
struct cgroup_subsys {
    int (*css_online)(struct cgroup_subsys_state *css);
    void (*css_offline)(struct cgroup_subsys_state *css);
    ...
    ...
    ...
    bool early_init:1;
    int id;
    const char *name;
    struct cgroup_root *root;
    ...
    ...
    ...
}
```

`ccs_online`と`ccs_offline`コールバックが呼び出されると、cgroupが正常に終了した後、すべての割り当てが完了し、cgroupは解放されます。
`early_init`フラグは、早期に初期化されるかもしれない/初期化されるべきサブシステムを示す。
`id`フィールドと`name`フィールドは、それぞれcgroupの登録されたサブシステムの配列内のユニークな識別子とサブシステムの`name`を表します。 最後の - `root`フィールドは、cgroupの階層のルートへのポインタを表します。

もちろん `cgroup_subsys`構造体は大きく、他のフィールドもありますが、今は十分です。
`cgroups`メカニズムに関連する重要な構造を知れたので、`cgroup_init_early`関数に戻りましょう。
この関数の主な目的は、いくつかの早期にサブシステムの初期化を行うことです。 
あなたがすでに想像しているように、これらの初期のサブシステムは `cgroup_subsys -> early_init = 1`を持つべきです。
早期に初期化されるサブシステムを見てみましょう。

2つのローカル変数を定義した後に、次のコード行を見れます:

```C
init_cgroup_root(&cgrp_dfl_root, &opts);
cgrp_dfl_root.cgrp.self.flags |= CSS_NO_REF;
```

ここでは、デフォルトの統一された階層の初期化を実行する `init_cgroup_root`関数の呼び出しを見ることができます。
この後、このデフォルトの`cgroup`の状態で `CSS_NO_REF`フラグをセットしてこのcssの参照カウントを無効にします。
`cgrp_dfl_root`は同じソースコードファイルで定義されています:

```C
struct cgroup_root cgrp_dfl_root;
```

その `cgrp`フィールドは `cgroup`構造体によってを表されます。
これは[include/linux/cgroup-defs.h](https://github.com/torvalds/linux/blob/master/include/linux/cgroup-defs.h)内で定義されます。
われわれは、Linuxカーネルの `task_struct` で表されるプロセスをすでに知っています。
`task_struct` はこのタスクがついている `cgroup` への直接リンクを含んでいません。
しかし、`task_struct `の `ccs_set`フィールドを通してそれを知ることができます。
この `ccs_set`構造体は、サブシステム状態の配列へのポインタを保持します:


```C
struct css_set {
    ...
    ...
    ....
    struct cgroup_subsys_state *subsys[CGROUP_SUBSYS_COUNT];
    ...
    ...
    ...
}
```

そして、`cgroup_subsys_state` を通して、プロセスはアタッチする `cgroup` を取得できます:

```C
struct cgroup_subsys_state {
    ...
    ...
    ...
    struct cgroup *cgroup;
    ...
    ...
    ...
}
```

以下は`cgroups`に関連する構造体の全体像です。:

```                                                 
+-------------+         +---------------------+    +------------->+---------------------+          +----------------+
| task_struct |         |       css_set       |    |              | cgroup_subsys_state |          |     cgroup     |
+-------------+         |                     |    |              +---------------------+          +----------------+
|             |         |                     |    |              |                     |          |     flags      |
|             |         |                     |    |              +---------------------+          |  cgroup.procs  |
|             |         |                     |    |              |        cgroup       |--------->|       id       |
|             |         |                     |    |              +---------------------+          |      ....      | 
|-------------+         |---------------------+----+                                               +----------------+
|   cgroups   | ------> | cgroup_subsys_state | array of cgroup_subsys_state
|-------------+         +---------------------+------------------>+---------------------+          +----------------+
|             |         |                     |                   | cgroup_subsys_state |          |      cgroup    |
+-------------+         +---------------------+                   +---------------------+          +----------------+
                                                                  |                     |          |      flags     |
                                                                  +---------------------+          |   cgroup.procs |
                                                                  |        cgroup       |--------->|        id      |
                                                                  +---------------------+          |       ....     |
                                                                  |    cgroup_subsys    |          +----------------+
                                                                  +---------------------+
                                                                             |
                                                                             |
                                                                             ↓
                                                                  +---------------------+
                                                                  |    cgroup_subsys    |
                                                                  +---------------------+
                                                                  |         id          |
                                                                  |        name         |
                                                                  |      css_online     |
                                                                  |      css_ofline     |
                                                                  |        attach       |
                                                                  |         ....        |
                                                                  +---------------------+
```


`init_cgroup_root` は `cgrp_dfl_root` をデフォルト値で埋めます。
次に、システムの最初のプロセスを表す `init_task` に最初の `ccs_set` を割り当てます。:

```C
RCU_INIT_POINTER(init_task.cgroups, &init_css_set);
```

そして、`cgroup_init_early`関数の最後の大きなことは `early cgroups` の初期化です。
ここでは、すべての登録されたサブシステムを調べ、一意の識別番号、サブシステムの名前を割り当て、早期にマークされたサブシステムのために `cgroup_init_subsys`関数を呼び出します。:

```C
for_each_subsys(ss, i) {
		ss->id = i;
		ss->name = cgroup_subsys_name[i];

        if (ss->early_init)
			cgroup_init_subsys(ss, true);
}
```

ここの `for_each_subsys`マクロは、[kernel/cgroup.c](https://github.com/torvalds/linux/blob/master/kernel/cgroup.c)で定義されているものです。`cgroup_subsys`配列に対する`for`ループです。
この配列の定義は、同じファイルにありますが、それは少し奇妙です:

```C
#define SUBSYS(_x) [_x ## _cgrp_id] = &_x ## _cgrp_subsys,
    static struct cgroup_subsys *cgroup_subsys[] = {
        #include <linux/cgroup_subsys.h>
};
#undef SUBSYS
```

1つの引数（サブシステムの名前）をとり、cgroupサブシステムの `cgroup_subsys`配列を定義する`SUBSYS`マクロとして定義されています。
それに加えて、配列が [linux/cgroup_subsys.h](https://github.com/torvalds/linux/blob/master/include/linux/cgroup_subsys.h)ヘッダーファイルの内容で初期化されていることがわかります。
このヘッダーファイルを見てみると、指定されたサブシステムの名前を持つ`SUBSYS`マクロのセットが再び表示されます:

```C
#if IS_ENABLED(CONFIG_CPUSETS)
SUBSYS(cpuset)
#endif

#if IS_ENABLED(CONFIG_CGROUP_SCHED)
SUBSYS(cpu)
#endif
...
...
...
```

これは、`SUBSYS`マクロを最初に定義した後の`#undef`文のために働きます。
`&_x ## _cgrp_subsys`式を見てください。 `##`演算子は、`C`マクロの右と左の式を連結します。
`cpuset`、`cpu`などを `SUBSYS`マクロに渡すと、`cpuset_cgrp_subsys`、`cp_cgrp_subsys`のどこかを定義する必要があります。
それは本当です。[kernel/cpuset.c](https://github.com/torvalds/linux/blob/master/kernel/cpuset.c)のソースコードファイルを見ると、次のように定義されています。:


```C
struct cgroup_subsys cpuset_cgrp_subsys = {
    ...
    ...
    ...
	.early_init	= true,
};
```

したがって、`cgroup_init_early`関数の最後のステップでは、`cgroup_init_subsys`関数を呼び出すことによって、初期のサブシステムを初期化します。
以下の初期のサブシステムは初期化されます:

* `cpuset`;
* `cpu`;
* `cpuacct`.

`cgroup_init_subsys`関数は与えられたサブシステムをデフォルト値で初期化します。
階層のルートを設定し、`css_alloc`コールバック関数の呼び出しで与えられたサブシステムのためのスペースを割り当て、もし親プロセスが存在する場合はサブシステムを親プロセスにリンクし、割り当てられたサブシステムなどを初期プロセスに追加します。

これで全てです。この瞬間から、初期のサブシステムが初期化されます。

まとめ
--------------------------------------------------------------------------------

これでLinuxカーネルの `Control groups` の仕組みについての最初のパートは終わりです。
`control groups`の仕組みに関連する初期化の最初のステップと理論を見てきました。
次のパートでは、`control groups`の実践的な側面を見ていこうと思います。

もし質問や提案があれば [twitter](https://twitter.com/0xAX)で連絡してください。

**英語が私の母国語ではないことに留意してください、ご迷惑をおかけして申し訳ありません。 もし、何か間違いが見つかった場合は、[linux-insides](https://github.com/0xAX/linux-insides)にPRを送ってください**

リンク
--------------------------------------------------------------------------------

* [control groups](https://en.wikipedia.org/wiki/Cgroups)
* [PID](https://en.wikipedia.org/wiki/Process_identifier)
* [cpuset](http://man7.org/linux/man-pages/man7/cpuset.7.html)
* [block devices](https://en.wikipedia.org/wiki/Device_file)
* [huge pages](https://www.kernel.org/doc/Documentation/vm/hugetlbpage.txt)
* [sysfs](https://en.wikipedia.org/wiki/Sysfs)
* [proc](https://en.wikipedia.org/wiki/Procfs)
* [cgroups kernel documentation](https://www.kernel.org/doc/Documentation/cgroup-v1/cgroups.txt)
* [cgroups v2](https://www.kernel.org/doc/Documentation/cgroup-v2.txt)
* [bash](https://www.gnu.org/software/bash/)
* [docker](https://en.wikipedia.org/wiki/Docker_(software))
* [perf events](https://en.wikipedia.org/wiki/Perf_(Linux))
* [Previous chapter](https://0xax.gitbooks.io/linux-insides/content/MM/linux-mm-1.html)
