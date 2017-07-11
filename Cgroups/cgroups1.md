Control Groups
================================================================================

はじめに
--------------------------------------------------------------------------------

これは、[linux insides](http://0xax.gitbooks.io/linux-insides/content/)の新しい章の最初のパートです。
パートの名前からあなたが予想したとおり、このパートではLinuxカーネル内での [control groups](https://en.wikipedia.org/wiki/Cgroups) や `cgroups`の仕組みを扱います。

`Cgroups` はLinuxカーネルが提供する特別な仕組みであり、プロセスやプロセスの集合に対して、プロセッサ時間、グループあたりのプロセス数、コントロールグループあたりのメモリ量、またはそのようなリソースの組み合わせなどに対して `リソース` の種類を割り当てることができます。 
`Cgroups`は階層的に構成されています。仕組みは通常のプロセスと似ていて、階層的であるため、子の`cgroups`は親から特定のパラメータの集合を継承します。
しかし、実際には同じではありません。`cgroups` と通常のプロセスの主な違いは、通常のプロセスツリーは常に単一であるが、コントロールグループでは多くの異なる階層が同時に存在する可能性があるということです。 
各コントロールグループの階層が`サブシステム`のコントロールグループに関連付けられているため、単純なステップではありませんでした。

ある `control group subsystem` はプロセッサ時間、または [pids](https://en.wikipedia.org/wiki/Process_identifier) の数や、言い換えるなら、`control group` のプロセス数を表します。
Linux カーネルは以下の12の `control group subsystems` のサポートを提供しています:

* `cpuset` - assigns individual processor(s) and memory nodes to task(s) in a group;
* `cpu` - uses the scheduler to provide cgroup tasks access to the processor resources;
* `cpuacct` - generates reports about processor usage by a group;
* `io` - sets limit to read/write from/to [block devices](https://en.wikipedia.org/wiki/Device_file);
* `memory` - sets limit on memory usage by a task(s) from a group;
* `devices` - allows access to devices by a task(s) from a group;
* `freezer` - allows to suspend/resume for a task(s) from a group;
* `net_cls` - allows to mark network packets from task(s) from a group;
* `net_prio` - provides a way to dynamically set the priority of network traffic per network interface for a group;
* `perf_event` - provides access to [perf events](https://en.wikipedia.org/wiki/Perf_(Linux)) to a group;
* `hugetlb` - activates support for [huge pages](https://www.kernel.org/doc/Documentation/vm/hugetlbpage.txt) for a group;
* `pid` - sets limit to number of processes in a group.

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

または、[sysfs](https://en.wikipedia.org/wiki/Sysfs):

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

すでに、あなたは `control groups`メカニズムは、Linuxカーネルのニーズに直接的に開発された仕組みでなく、
主にユーザー空間のニーズに対して開発されたものだと勘づいているかもしれません。
`control groups`を使うには、最初に作成するべきです。私たちは二つの方法で `cgroup`を作成するでしょう。

最初の方法は、`sys/fs/cgroup` からサブディレクトリを作成し、サブディレクトリを作成した直後に自動的に作成される `tasks`ファイルにタスクのpidを追加することです。

2番目の方法は `libcgroup`ライブラリ（Fedoraの`libcgroup-tools`）のutilを使って `cgroups`を作成/破壊/管理する方法です。

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
デバイスを禁止するには（この場合は `/dev/tty`）、以下の行に`devices.deny`を書きこみます:

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

Now after we just saw little theory about `control groups` Linux kernel mechanism, we may start to dive into the source code of Linux kernel to acquainted with this mechanism closer. As always we will start from the initialization of `control groups`. Initialization of `cgroups` divided into two parts in the Linux kernel: early and late. In this part we will consider only `early` part and `late` part will be considered in next parts.

Early initialization of `cgroups` starts from the call of the:

```C
cgroup_init_early();
```

function in the [init/main.c](https://github.com/torvalds/linux/blob/master/init/main.c) during early initialization of the Linux kernel. This function is defined in the [kernel/cgroup.c](https://github.com/torvalds/linux/blob/master/kernel/cgroup.c) source code file and starts from the definition of two following local variables:

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

The `cgroup_sb_opts` structure defined in the same source code file and looks:

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

which represents mount options of `cgroupfs`. For example we may create named cgroup hierarchy (with name `my_cgrp`) with the `name=` option and without any subsystems:

```
$ mount -t cgroup -oname=my_cgrp,none /mnt/cgroups
```

The second variable - `ss` has type - `cgroup_subsys` structure which is defined in the [include/linux/cgroup-defs.h](https://github.com/torvalds/linux/blob/master/include/linux/cgroup-defs.h) header file and as you may guess from the name of the type, it represents a `cgroup` subsystem. This structure contains various fields and callback functions like:

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

Where for example `ccs_online` and `ccs_offline` callbacks are called after a cgroup successfully will complete all allocations and a cgroup will be before releasing respectively. The `early_init` flags marks subsystems which may/should be initialized early. The `id` and `name` fields represents unique identifier in the array of registered subsystems for a cgroup and `name` of a subsystem respectively. The last - `root` fields represents pointer to the root of of a cgroup hierarchy.

Of course the `cgroup_subsys` structure bigger and has other fields, but it is enough for now. Now as we got to know important structures related to `cgroups` mechanism, let's return to the `cgroup_init_early` function. Main purpose of this function is to do early initialization of some subsystems. As you already may guess, these `early` subsystems should have `cgroup_subsys->early_init = 1`. Let's look what subsystems may be initialized early.

After the definition of the two local variables we may see following lines of code:

```C
init_cgroup_root(&cgrp_dfl_root, &opts);
cgrp_dfl_root.cgrp.self.flags |= CSS_NO_REF;
```

Here we may see call of the `init_cgroup_root` function which will execute initialization of the default unified hierarchy and after this we set `CSS_NO_REF` flag in state of this default `cgroup` to disable reference counting for this css. The `cgrp_dfl_root` is defined in the same source code file:

```C
struct cgroup_root cgrp_dfl_root;
```

Its `cgrp` field represented by the `cgroup` structure which represents a `cgroup` as you already may guess and defined in the [include/linux/cgroup-defs.h](https://github.com/torvalds/linux/blob/master/include/linux/cgroup-defs.h) header file. We already know that a process which is represented by the `task_struct` in the Linux kernel. The `task_struct` does not contain direct link to a `cgroup` where this task is attached. But it may be reached via `ccs_set` field of the `task_struct`. This `ccs_set` structure holds pointer to the array of subsystem states:

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

And via the `cgroup_subsys_state`, a process may get a `cgroup` that this process is attached to:

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

So, the overall picture of `cgroups` related data structure is following:

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



So, the `init_cgroup_root` fills the `cgrp_dfl_root` with the default values. The next thing is assigning initial `ccs_set` to the `init_task` which represents first process in the system:

```C
RCU_INIT_POINTER(init_task.cgroups, &init_css_set);
```

And the last big thing in the `cgroup_init_early` function is initialization of `early cgroups`. Here we go over all registered subsystems and assign unique identity number, name of a subsystem and call the `cgroup_init_subsys` function for subsystems which are marked as early:

```C
for_each_subsys(ss, i) {
		ss->id = i;
		ss->name = cgroup_subsys_name[i];

        if (ss->early_init)
			cgroup_init_subsys(ss, true);
}
```

The `for_each_subsys` here is a macro which is defined in the [kernel/cgroup.c](https://github.com/torvalds/linux/blob/master/kernel/cgroup.c) source code file and just expands to the `for` loop over `cgroup_subsys` array. Definition of this array may be found in the same source code file and it looks in a little unusual way:

```C
#define SUBSYS(_x) [_x ## _cgrp_id] = &_x ## _cgrp_subsys,
    static struct cgroup_subsys *cgroup_subsys[] = {
        #include <linux/cgroup_subsys.h>
};
#undef SUBSYS
```

It is defined as `SUBSYS` macro which takes one argument (name of a subsystem) and defines `cgroup_subsys` array of cgroup subsystems. Additionally we may see that the array is initialized with content of the [linux/cgroup_subsys.h](https://github.com/torvalds/linux/blob/master/include/linux/cgroup_subsys.h) header file. If we will look inside of this header file we will see again set of the `SUBSYS` macros with the given subsystems names:

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

This works because of `#undef` statement after first definition of the `SUBSYS` macro. Look at the `&_x ## _cgrp_subsys` expression. The `##` operator concatenates right and left expression in a `C` macro. So as we passed `cpuset`, `cpu` and etc., to the `SUBSYS` macro, somewhere `cpuset_cgrp_subsys`, `cp_cgrp_subsys` should be defined. And that's true. If you will look in the [kernel/cpuset.c](https://github.com/torvalds/linux/blob/master/kernel/cpuset.c) source code file, you will see this definition:

```C
struct cgroup_subsys cpuset_cgrp_subsys = {
    ...
    ...
    ...
	.early_init	= true,
};
```

So the last step in the `cgroup_init_early` function is initialization of early subsystems with the call of the `cgroup_init_subsys` function. Following early subsystems will be initialized:

* `cpuset`;
* `cpu`;
* `cpuacct`.

The `cgroup_init_subsys` function does initialization of the given subsystem with the default values. For example sets root of hierarchy, allocates space for the given subsystem with the call of the `css_alloc` callback function, link a subsystem with a parent if it exists, add allocated subsystem to the initial process and etc.

That's all. From this moment early subsystems are initialized.

まとめ
--------------------------------------------------------------------------------

これでLinuxカーネルの`Control groups`の仕組みについての最初のパートは終わりです。
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
