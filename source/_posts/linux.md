---
title: Linux performance optimization
date: 2019-12-24 16:07:17
tags: tools
categories: Linux
---
This blog record some useful tools for linux.
<!--more-->
# reference
1. [一文读懂Linux进程、进程组、会话、僵尸](https://mp.weixin.qq.com/s/5_90fSpp-JeD-ffZixqc9g)
2. [Linux中父进程为何要苦苦地知道子进程的死亡原因？](https://mp.weixin.qq.com/s/QBF2ozxqy03Cprh_ugi3NA)
3. [浅谈 Linux 高负载的系统化分析](https://mp.weixin.qq.com/s/2Sx52ggm_D-oFVAhmNTbHw)
4. [CPU 使用率低高负载的原因](https://mp.weixin.qq.com/s/TQZ3anPiStzgI1B--uTGlQ)
5. [Linux系统文件系统及文件基础篇](https://mp.weixin.qq.com/s/Bv1j41dZJBXy-yfVwvtJPw)
6. [如何在Linux中使用awk工具详解](https://mp.weixin.qq.com/s/Ilc2s4ReBiMVWQDi8aH4Tw)
7. [通过10个例子掌握Linux下lsof命令](https://mp.weixin.qq.com/s/XWy_pgxOG0tspvHBDpYafg) 
8. [学习操作系统](https://mp.weixin.qq.com/s/oGc79uYz05wDfRSivMpJUA)

Linux performance tools 
![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20191225190537.png)
# CPU
What is the Linux system load ? Can refer this [article](https://mp.weixin.qq.com/s/AhZw_k5U_Pf2k9AOvruFcQ) or [this](http://www.brendangregg.com/blog/2017-08-08/linux-load-averages.html)  
Linux load averages are "system load averages" that show the running thread (task) demand on the system as an average number of running plus waiting threads. R + D state: Running + Uninterruptible Sleep
## stress sysstat

We can do some test by command `stress`
```bash
apt install stress sysstat

# stress i cpu
stress --cpu 1 --timeout 600

# check uptime load
watch -d uptime

# check ALL cpu status
mpstat -P ALL 5

# check every process or thread cpu usage 
pidstat -u 5 1
```
A thread is the basic unit of scheduling, and the process is the basic unit of resource owners
+ Thread
1. smallest sequence of programming instructions that can be managed independently by a scheduler
2. Has its own register e.g. PC (program counter), SP (stack pointer)
+ Process
1. instance of a computer porgram that is being executed
2. A process can have one or multiple thread
3. Most programs are single threaded
+ Parallel computing
1. Run program currently on one or more CPUs
2. Multi-threading (shared-memory)
3. Multi-processing (independent-memory)  
## context switch

```bash
# interval 5s output 1 row data
$ vmstat 5
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy  id  wa st
 0  0     0 7005360  91564 818900    0    0     0     0   25   33  0  0  100  0  0

# cs（context switch）
# in（interrupt）
# r（Running or Runnable）
# b（Blocked） 
```
`vmstat` only provide overall context switch of system, `pidstat -w` shows process context switch

```bash
# interval 5s output 1 row data
$ pidstat -w 5
Linux 4.15.0 (ubuntu)  09/23/18  _x86_64_  (2 CPU)
 
08:18:26      UID       PID   cswch/s nvcswch/s  Command
08:18:31        0         1      0.20      0.00  systemd
08:18:31        0         8      5.40      0.00  rcu_sched

# cswch: voluntary context switches
# nvcswch: non voluntary context switches
```

we use `sysbench` to simulate a system multi-threaded scheduling switch.

```bash
sysbench --threads=10 --max-time=300 threads run

# check all context switch
vmstat 1
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs     us sy id wa st
 6  0      0 6487428 118240 1292772  0    0     0     0 9019  1398830 16 84  0  0  0
 8  0      0 6487428 118240 1292772  0    0     0     0 10191 1392312 16 84  0  0  0

# check which process cause high cs 1398830
# -w  Report  task switching activity  
# -u  Report CPU utilization.
pidstat -w -u 1
08:06:33      UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
08:06:34        0     10488   30.00  100.00    0.00    0.00  100.00     0  sysbench
08:06:34        0     26326    0.00    1.00    0.00    0.00    1.00     0  kworker/u4:2
 
08:06:33      UID       PID   cswch/s nvcswch/s  Command
08:06:34        0         8     11.00      0.00  rcu_sched
08:06:34        0        16      1.00      0.00  ksoftirqd/1
08:06:34        0       471      1.00      0.00  hv_balloon
08:06:34        0      1230      1.00      0.00  iscsid
08:06:34        0      4089      1.00      0.00  kworker/1:5
08:06:34        0      4333      1.00      0.00  kworker/0:3
08:06:34        0     10499      1.00    224.00  pidstat
08:06:34        0     26326    236.00      0.00  kworker/u4:2
08:06:34     1000     26784    223.00      0.00  sshd

# -t display statistics for  threads  associated  with  selected tasks.
pidstat -wt 1

08:14:05      UID      TGID       TID   cswch/s nvcswch/s  Command
...
08:14:05        0     10551         -      6.00      0.00  sysbench
08:14:05        0         -     10551      6.00      0.00  |__sysbench
08:14:05        0         -     10552  18911.00 103740.00  |__sysbench
08:14:05        0         -     10553  18915.00 100955.00  |__sysbench
08:14:05        0         -     10554  18827.00 103954.00  |__sysbench


# check /proc/interrupts

watch -d cat /proc/interrupts

          CPU0       CPU1
...
RES:    2450431    5279697   Rescheduling interrupts
...
```
## cpu utilization 

```bash
cat /proc/stat | grep ^cpu
cpu  280580 7407 286084 172900810 83602 0 583 0 0 0
cpu0 144745 4181 176701 86423902 52076 0 301 0 0 0
cpu1 135834 3226 109383 86476907 31525 0 282 0 0 0

# perf is a good tools
perf top

perf record # Ctrl+C stop

perf report # show record

# Best practice how to perf container process
# perf record in host then perf report in container
# install perf in host 
apt-get install -y linux-tools-common linux-tools-generic linux-tools-$(uname -r)）

perf record -g -p < pid>

docker cp perf.data test:/tmp 
docker exec -i -t test bash
cd /tmp/ 
apt-get update && apt-get install -y linux-tools linux-perf procps
perf_4.9 report


# pstress display a tree of processes
pstree | grep stress
```

## many Z (zombie) state

```bash
top

top - 05:56:23 up 17 days, 16:45,  2 users,  load average: 2.00, 1.68, 1.39
Tasks: 247 total,   1 running,  79 sleeping,   0 stopped, 115 zombie
%Cpu0  :  0.0 us,  0.7 sy,  0.0 ni, 38.9 id, 60.5 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu1  :  0.0 us,  0.7 sy,  0.0 ni,  4.7 id, 94.6 wa,  0.0 hi,  0.0 si,  0.0 st
...
 
  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
 4340 root      20   0   44676   4048   3432 R   0.3  0.0   0:00.05 top
 4345 root      20   0   37280  33624    860 D   0.3  0.0   0:00.01 app
 4344 root      20   0   37280  33624    860 D   0.3  0.4   0:00.01 app
    1 root      20   0  160072   9416   6752 S   0.0  0.1   0:38.59 systemd
...


# dstat check cpu and IO
dstat 1 10

You did not select any stats, using -cdngy by default.
--total-cpu-usage-- -dsk/total- -net/total- ---paging-- ---system--
usr sys idl wai stl| read  writ| recv  send|  in   out | int   csw
  0   0  96   4   0|1219k  408k|   0     0 |   0     0 |  42   885
  0   0   2  98   0|  34M    0 | 198B  790B|   0     0 |  42   138
  0   0   0 100   0|  34M    0 |  66B  342B|   0     0 |  42   135
  0   0  84  16   0|5633k    0 |  66B  342B|   0     0 |  52   177
  0   3  39  58   0|  22M    0 |  66B  342B|   0     0 |  43   144
  0   0   0 100   0|  34M    0 | 200B  450B|   0     0 |  46   147
  0   0   2  98   0|  34M    0 |  66B  342B|   0     0 |  45   134
  0   0   0 100   0|  34M    0 |  66B  342B|   0     0 |  39   131
  0   0  83  17   0|5633k    0 |  66B  342B|   0     0 |  46   168
  0   3  39  59   0|  22M    0 |  66B  342B|   0     0 |  37   134

# find the D pid
top
...
  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
 4340 root      20   0   44676   4048   3432 R   0.3  0.0   0:00.05 top
 4345 root      20   0   37280  33624    860 D   0.3  0.0   0:00.01 app
 4344 root      20   0   37280  33624    860 D   0.3  0.4   0:00.01 app
...

# for example pid = 4344
pidstat -d -p 4344 1 3
06:38:50      UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s iodelay  Command
06:38:51        0      4344      0.00      0.00      0.00       0  app
06:38:52        0      4344      0.00      0.00      0.00       0  app
06:38:53        0      4344      0.00      0.00      0.00       0  app

# check all process, find 6082
pidstat -d 1 20
...
06:48:46      UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s iodelay  Command
06:48:47        0      4615      0.00      0.00      0.00       1  kworker/u4:1
06:48:47        0      6080  32768.00      0.00      0.00     170  app
06:48:47        0      6081  32768.00      0.00      0.00     184  app
 
06:48:47      UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s iodelay  Command
06:48:48        0      6080      0.00      0.00      0.00     110  app
 
06:48:48      UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s iodelay  Command
06:48:49        0      6081      0.00      0.00      0.00     191  app
 
06:48:49      UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s iodelay  Command
 
06:48:50      UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s iodelay  Command
06:48:51        0      6082  32768.00      0.00      0.00       0  app
06:48:51        0      6083  32768.00      0.00      0.00       0  app
 
06:48:51      UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s iodelay  Command
06:48:52        0      6082  32768.00      0.00      0.00     184  app
06:48:52        0      6083  32768.00      0.00      0.00     175  app
 
06:48:52      UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s iodelay  Command
06:48:53        0      6083      0.00      0.00      0.00     105  app
...

# use strace
strace -p 6082
strace: attach: ptrace(PTRACE_SEIZE, 6082): Operation not permitted

ps aux | grep 6082
root      6082  0.0  0.0      0     0 pts/0    Z+   13:43   0:00 [app] <defunct>

# find blkdev_direct_IO 
perf record -g
perf report
```
![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20191225210140.png)

```bash
# find father of 3084
pstree -aps 3084
systemd,1
  └─dockerd,15006 -H fd://
      └─docker-containe,15024 --config /var/run/docker/containerd/containerd.toml
          └─docker-containe,3991 -namespace moby -workdir...
              └─app,4009
                  └─(app,3084)
```

## soft interrupt
Linux divides the interrupt handling process into two phases, the upper half and the lower half: The first part is used to quickly handle interrupts. It runs in interrupt disable mode and mainly deals with hardware-related or time-sensitive tasks. The second part is used to delay processing of the unfinished work in the upper half, usually running as a kernel thread.

```bash
# /proc/softirqs provides the operation of soft interrupts;

# /proc/interrupts provides the operation of hard interrupts.

cat /proc/softirqs
                    CPU0       CPU1
          HI:          0          0
       TIMER:     811613    1972736
      NET_TX:         49          7
      NET_RX:    1136736    1506885
       BLOCK:          0          0
    IRQ_POLL:          0          0
     TASKLET:     304787       3691
       SCHED:     689718    1897539
     HRTIMER:          0          0
         RCU:    1330771    1354737

ps aux | grep softirq
root   7  0.0  0.0     0    0 ?    S    Oct10   0:01 [ksoftirqd/0]
root  16  0.0  0.0     0    0 ?    S    Oct10   0:01 [ksoftirqd/1]

# sar - Collect, report, or save system activity information.

sar -n DEV 1
15:03:46        IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s   rxcmp/s   txcmp/s  rxmcst/s   %ifutil
15:03:47         eth0  12607.00   6304.00    664.86    358.11      0.00      0.00      0.00      0.01
15:03:47      docker0   6302.00  12604.00    270.79    664.66      0.00      0.00      0.00      0.00
15:03:47           lo      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
15:03:47    veth9f6bbcd 6302.00  12604.00    356.95    664.66      0.00      0.00      0.00      0.05

# check TCP package SYN ddos attack
tcpdump -i eth0 -n tcp port 80
15:11:32.678966 IP 192.168.0.2.18238 > 192.168.0.30.80: Flags [S], seq 458303614, win 512, length 0
...
```

## summary 
![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20191225215653.png)
![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20191225215708.png)
![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20191225215717.png)
![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20191225215723.png)
![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20191225215732.png)

# Memory
![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20191227165532.png)  
![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20191227165536.png)  
![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20191227165539.png)  
![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20191227165542.png)

## buffer & cache
Buffers are temporary storage of raw disk blocks, that is, used to cache data on the disk, usually not particularly large (about 20MB). In this way, the kernel can centralize scattered writes and optimize disk writes in a unified manner. For example, multiple small writes can be combined into a single large write.  

Cached is a page cache that reads files from disk, that is, it is used to cache data read from files. This way, the next time you access these file data, you can quickly get it directly from memory without having to access the slow disk again.  

SReclaimable is part of Slab. Slab consists of two parts, the recyclable part is recorded with SReclaimable; the non-recyclable part is recorded with SUnreclaim.  

```bash
# Before using cachestat and cachetop, we must first install the bcc package
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 4052245BD4284CDD
echo "deb https://repo.iovisor.org/apt/xenial xenial main" | sudo tee /etc/apt/sources.list.d/iovisor.list
sudo apt-get update
sudo apt-get install -y bcc-tools libbcc-examples linux-headers-$(uname -r)

export PATH=$PATH:/usr/share/bcc/tools

#cachestat provides read and write hits for the entire system cache.

# 1s interval 3 row data
cachestat 1 3
   TOTAL   MISSES     HITS  DIRTIES   BUFFERS_MB  CACHED_MB
    2        0        2        1           17        279
    2        0        2        1           17        279
    2        0        2        1           17        279 

#cachetop provides cache hits for each process.

cachetop
11:58:50 Buffers MB: 258 / Cached MB: 347 / Sort: HITS / Order: ascending
PID      UID      CMD              HITS     MISSES   DIRTIES  READ_HIT%  WRITE_HIT%
13029 root     python                 1        0        0     100.0%       0.0%
```
Check the specify file cache, we can use [pcstat](https://github.com/tobert/pcstat)
```bash
export GOPATH=~/go
export PATH=~/go/bin:$PATH
go get golang.org/x/sys/unix
go get github.com/tobert/pcstat/pcstat

# first try
pcstat /bin/ls
+---------+----------------+------------+-----------+---------+
| Name    | Size (bytes)   | Pages      | Cached    | Percent |
|---------+----------------+------------+-----------+---------|
| /bin/ls | 133792         | 33         | 0         | 000.000 |
+---------+----------------+------------+-----------+---------+

# second try
ls
pcstat /bin/ls
+---------+----------------+------------+-----------+---------+
| Name    | Size (bytes)   | Pages      | Cached    | Percent |
|---------+----------------+------------+-----------+---------|
| /bin/ls | 133792         | 33         | 33        | 100.000 |
+---------+----------------+------------+-----------+---------+
```
## memory leak
1. Check the whole memory by `vmstat`
```bash
vmstat 3
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
0  0    0 6601824  97620 1098784    0    0     0     0   62  322  0  0 100  0  0
0  0    0 6601700  97620 1098788    0    0     0     0   57  251  0  0 100  0  0
0  0    0 6601320  97620 1098788    0    0     0     3   52  306  0  0 100  0  0
0  0    0 6601452  97628 1098788    0    0     0    27   63  326  0  0 100  0  0
2  0    0 6601328  97628 1098788    0    0     0    44   52  299  0  0 100  0  0
0  0    0 6601080  97628 1098792    0    0     0     0   56  285  0  0 100  0  0 
```
The free column of memory is constantly changing and is in a downward trend; the buffer and cache remain basically unchanged.

2. `memleak` is a tool in the bcc package
```bash
/usr/share/bcc/tools/memleak -a -p $(pidof app)
WARNING: Couldn't find .text section in /app
WARNING: BCC can't handle sym look ups for /app
    addr = 7f8f704732b0 size = 8192
    addr = 7f8f704772d0 size = 8192
    addr = 7f8f704712a0 size = 8192
    addr = 7f8f704752c0 size = 8192
    32768 bytes in 4 allocations from stack
        [unknown] [app]
        [unknown] [app]
        start_thread+0xdb [libpthread-2.27.so] 
# [unknown] is app is in docker

# copy exec file from docker, then to check
docker cp app:/app /app
/usr/share/bcc/tools/memleak -p $(pidof app) -a
Attaching to pid 12512, Ctrl+C to quit.
[03:00:41] Top 10 stacks with outstanding allocations:
    addr = 7f8f70863220 size = 8192
    addr = 7f8f70861210 size = 8192
    addr = 7f8f7085b1e0 size = 8192
    addr = 7f8f7085f200 size = 8192
    addr = 7f8f7085d1f0 size = 8192
    40960 bytes in 5 allocations from stack
        fibonacci+0x1f [app]
        child+0x4f [app]
        start_thread+0xdb [libpthread-2.27.so] 
```

## swap
Data that has been modified by the application and has not yet been written to the disk (that is, dirty pages) must be written to the disk before memory can be released.These dirty pages can generally be written to disk in two ways.
1. You can use the system call fsync in your application to synchronize dirty pages to disk;
2. It can also be left to the system, and the kernel thread pdflush is responsible for refreshing these dirty pages.
Swap writes these infrequently accessed memory to disk, then releases this memory for use by other more needed processes. When the memory is accessed again, it is sufficient to re-read it from disk.
![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20191227180106.png)
In the NUMA architecture, multiple processors are divided into different nodes, and each node has its own local memory space.
![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20191227180111.png)

```bash
numactl --hardware
available: 2 nodes (0-1)
node 0 cpus: 0 2 4 6 8 10 12 14 16 18 20 22 24 26 28 30
node 0 size: 32674 MB
node 0 free: 418 MB
node 1 cpus: 1 3 5 7 9 11 13 15 17 19 21 23 25 27 29 31
node 1 size: 32768 MB
node 1 free: 212 MB
node distances:
node   0   1 
  0:  10  21 
  1:  21  10 

cat /proc/zoneinfo
...
Node 0, zone   Normal
 pages free     227894
       min      14896
       low      18620
       high     22344
...
     nr_free_pages 227894
     nr_zone_inactive_anon 11082
     nr_zone_active_anon 14024
     nr_zone_inactive_file 539024
     nr_zone_active_file 923986
...


# -r memory，-S  Swap 
$ sar -r -S 1
04:39:56    kbmemfree   kbavail kbmemused  %memused kbbuffers  kbcached  kbcommit   %commit  kbactive   kbinact   kbdirty
04:39:57      6249676   6839824   1919632     23.50    740512     67316   1691736     10.22    815156    841868         4
 
04:39:56    kbswpfree kbswpused  %swpused  kbswpcad   %swpcad
04:39:57      8388604         0      0.00         0      0.00
 
04:39:57    kbmemfree   kbavail kbmemused  %memused kbbuffers  kbcached  kbcommit   %commit  kbactive   kbinact   kbdirty
04:39:58      6184472   6807064   1984836     24.30    772768     67380   1691736     10.22    847932    874224        20
 
04:39:57    kbswpfree kbswpused  %swpused  kbswpcad   %swpcad
04:39:58      8388604         0      0.00         0      0.00
 
…
 
04:44:06    kbmemfree   kbavail kbmemused  %memused kbbuffers  kbcached  kbcommit   %commit  kbactive   kbinact   kbdirty
04:44:07       152780   6525716   8016528     98.13   6530440     51316   1691736     10.22    867124   6869332         0
 
04:44:06    kbswpfree kbswpused  %swpused  kbswpcad   %swpcad
04:44:07      8384508      4096      0.05        52      1.27

# check cachetop 50 hit rate

cachetop 5
12:28:28 Buffers MB: 6349 / Cached MB: 87 / Sort: HITS / Order: ascending
PID      UID      CMD              HITS     MISSES   DIRTIES  READ_HIT%  WRITE_HIT%
18280   root     python              22        0        0     100.0%       0.0%
18279   root     dd              41088    41022        0      50.0%      50.0%


# pages_free fluctuation
watch -d grep -A 15 'Normal' /proc/zoneinfo
Node 0, zone   Normal
  pages free     21328
        min      14896
        low      18620
        high     22344
        spanned  1835008
        present  1835008
        managed  1796710
        protection: (0, 0, 0, 0, 0)
      nr_free_pages 21328
      nr_zone_inactive_anon 79776
      nr_zone_active_anon 206854
      nr_zone_inactive_file 918561
      nr_zone_active_file 496695
      nr_zone_unevictable 2251
      nr_zone_write_pending 0


# sort by VmSwap
for file in /proc/*/status ; do awk '/VmSwap|Name|^Pid/{printf $2 " " $3}END{ print ""}' $file; done | sort -k 3 -n -r | head
dockerd 2226 10728 kB
docker-containe 2251 8516 kB
snapd 936 4020 kB
networkd-dispat 911 836 kB
polkitd 1004 44 kB
```

## summary
![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20191227232819.png)  
![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20191227232823.png)  
![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20191227232828.png)  
![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20191227232831.png)

What's the difference between system and disk ?
A disk is a storage device (to be exact, a block device) that can be divided into different disk partitions. On a disk or disk partition, you can also create a file system and mount it in a directory on the system. In this way, the system can read and write files through this mount directory.

In other words, a disk is a block device that stores data and is the carrier of a file system. Therefore, the file system still needs to ensure the persistent storage of data through disk.

You will see this sentence in many places, everything in Linux is a file. In other words, you can access disks and files through the same file interface (such as open, read, write, close, etc.).

What we usually mean by "documents" is actually ordinary documents.

The disk or partition refers to the block device file.

# I-O
The Linux file system allocates two data structures for each file, an `index node` and a `directory entry`. They are mainly used to record the meta information and directory structure of files.
![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20191228204621.png)  
![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20191228204848.png)

```bash
# slabtop - display kernel slab cache information in real time

slabtop

Active / Total Objects (% used)    : 503302 / 506242 (99.4%)
 Active / Total Slabs (% used)      : 10730 / 10730 (100.0%)
 Active / Total Caches (% used)     : 100 / 143 (69.9%)
 Active / Total Size (% used)       : 153659.58K / 155417.98K (98.9%)
 Minimum / Average / Maximum Object : 0.01K / 0.31K / 8.00K

  OBJS ACTIVE  USE OBJ SIZE  SLABS OBJ/SLAB CACHE SIZE NAME                   
103782 103782 100%    0.19K   2471       42     19768K dentry
 73749  73749 100%    0.10K   1891       39      7564K buffer_head
 45900  45900 100%    0.13K    765       60      6120K kernfs_node_cache
 39985  39985 100%    0.58K    727       55     23264K inode_cache
 34830  34830 100%    1.05K   1161       30     37152K ext4_inode_cache
 18462  18462 100%    0.04K    181      102       724K ext4_extent_status
 17784  17784 100%    0.20K    456       39      3648K vm_area_struct
 15456  15456 100%    0.69K    336       46     10752K squashfs_inode_cache
 14336  14144   0%    0.50K    224       64      7168K kmalloc-512

```
![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20191228233800.png)

## check io
```bash
iostat -d -x 1 
Device            r/s     w/s     rkB/s     wkB/s   rrqm/s   wrqm/s  %rrqm  %wrqm r_await w_await aqu-sz rareq-sz wareq-sz  svctm  %util 
loop0            0.00    0.00      0.00      0.00     0.00     0.00   0.00   0.00    0.00    0.00   0.00     0.00     0.00   0.00   0.00 
loop1            0.00    0.00      0.00      0.00     0.00     0.00   0.00   0.00    0.00    0.00   0.00     0.00     0.00   0.00   0.00 
sda              0.00    0.00      0.00      0.00     0.00     0.00   0.00   0.00    0.00    0.00   0.00     0.00     0.00   0.00   0.00 
sdb              0.00    0.00      0.00      0.00     0.00     0.00   0.00   0.00    0.00    0.00   0.00     0.00     0.00   0.00   0.00 
```
![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20191228234202.png)
* % util is the disk I / O usage 
* r/s + w/s is IOPS
* rkB/s + wkB/s is the throughput
* r_await + w_await is the response time

```bash
# check process
pidstat -d 1 
13:39:51      UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s iodelay  Command 
13:39:52      102       916      0.00      4.00      0.00       0  rsyslogd

iotop
Total DISK READ :       0.00 B/s | Total DISK WRITE :       7.85 K/s 
Actual DISK READ:       0.00 B/s | Actual DISK WRITE:       0.00 B/s 
  TID  PRIO  USER     DISK READ  DISK WRITE  SWAPIN     IO>    COMMAND 
15055 be/3 root        0.00 B/s    7.85 K/s  0.00 %  0.00 % systemd-journald 

```

## write issue
```bash
 top 
top - 14:43:43 up 1 day,  1:39,  2 users,  load average: 2.48, 1.09, 0.63 
Tasks: 130 total,   2 running,  74 sleeping,   0 stopped,   0 zombie 
%Cpu0  :  0.7 us,  6.0 sy,  0.0 ni,  0.7 id, 92.7 wa,  0.0 hi,  0.0 si,  0.0 st 
%Cpu1  :  0.0 us,  0.3 sy,  0.0 ni, 92.3 id,  7.3 wa,  0.0 hi,  0.0 si,  0.0 st 
KiB Mem :  8169308 total,   747684 free,   741336 used,  6680288 buff/cache 
KiB Swap:        0 total,        0 free,        0 used.  7113124 avail Mem 
 
  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND 
18940 root      20   0  656108 355740   5236 R   6.3  4.4   0:12.56 python 
1312 root      20   0  236532  24116   9648 S   0.3  0.3   9:29.80 python3 

# 92.7 io_wait 
# memroy most is used for buff/cache 

# check which device
iostat -x -d 1 
Device            r/s     w/s     rkB/s     wkB/s   rrqm/s   wrqm/s  %rrqm  %wrqm r_await w_await aqu-sz rareq-sz wareq-sz  svctm  %util 
loop0            0.00    0.00      0.00      0.00     0.00     0.00   0.00   0.00    0.00    0.00   0.00     0.00     0.00   0.00   0.00 
sdb              0.00    0.00      0.00      0.00     0.00     0.00   0.00   0.00    0.00    0.00   0.00     0.00     0.00   0.00   0.00 
sda              0.00   64.00      0.00  32768.00     0.00     0.00   0.00   0.00    0.00 7270.44 1102.18     0.00   512.00  15.50  99.20

# check which process
pidstat -d 1 
 
15:08:35      UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s iodelay  Command 
15:08:36        0     18940      0.00  45816.00      0.00      96  python 

15:08:36      UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s iodelay  Command 
15:08:37        0       354      0.00      0.00      0.00     350  jbd2/sda1-8 
15:08:37        0     18940      0.00  46000.00      0.00      96  python 
15:08:37        0     20065      0.00      0.00      0.00    1503  kworker/u4:2 

# check pid 18940
strace -p 18940 
strace: Process 18940 attached 
...
mmap(NULL, 314576896, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f0f7aee9000 
mmap(NULL, 314576896, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f0f682e8000 
write(3, "2018-12-05 15:23:01,709 - __main"..., 314572844 
) = 314572844 
munmap(0x7f0f682e8000, 314576896)       = 0 
write(3, "\n", 1)                       = 1 
munmap(0x7f0f7aee9000, 314576896)       = 0 
close(3)                                = 0 
stat("/tmp/logtest.txt.1", {st_mode=S_IFREG|0644, st_size=943718535, ...}) = 0 


# check pid 18940 open files
lsof -p 18940 
COMMAND   PID USER   FD   TYPE DEVICE  SIZE/OFF    NODE NAME 
python  18940 root  cwd    DIR   0,50      4096 1549389 / 
python  18940 root  rtd    DIR   0,50      4096 1549389 / 
… 
python  18940 root    2u   CHR  136,0       0t0       3 /dev/pts/0 
python  18940 root    3w   REG    8,1 117944320     303 /tmp/logtest.txt 

# root cause is python wirte 300MB/s to /tmp/logtest.txt 
```

## io_wait issue

```bash
top 
top - 14:27:02 up 10:30,  1 user,  load average: 1.82, 1.26, 0.76 
Tasks: 129 total,   1 running,  74 sleeping,   0 stopped,   0 zombie 
%Cpu0  :  3.5 us,  2.1 sy,  0.0 ni,  0.0 id, 94.4 wa,  0.0 hi,  0.0 si,  0.0 st 
%Cpu1  :  2.4 us,  0.7 sy,  0.0 ni, 70.4 id, 26.5 wa,  0.0 hi,  0.0 si,  0.0 st 
KiB Mem :  8169300 total,  3323248 free,   436748 used,  4409304 buff/cache 
KiB Swap:        0 total,        0 free,        0 used.  7412556 avail Mem 
 
  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND 
12280 root      20   0  103304  28824   7276 S  14.0  0.4   0:08.77 python 
   16 root      20   0       0      0      0 S   0.3  0.0   0:09.22 ksoftirqd/1 
1549 root      20   0  236712  24480   9864 S   0.3  0.3   3:31.38 python3 

# iowait is high, memory is ok.

# util = 98%
iostat -d -x 1
Device            r/s     w/s     rkB/s     wkB/s   rrqm/s   wrqm/s  %rrqm  %wrqm r_await w_await aqu-sz rareq-sz wareq-sz  svctm  %util 
loop0            0.00    0.00      0.00      0.00     0.00     0.00   0.00   0.00    0.00    0.00   0.00     0.00     0.00   0.00   0.00 
sda              0.00   71.00      0.00  32912.00     0.00     0.00   0.00   0.00    0.00 18118.31 241.89     0.00   463.55  13.86  98.40 


# check which porcess
pidstat -d 1 
14:39:14      UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s iodelay  Command 
14:39:15        0     12280      0.00 335716.00      0.00       0  python 


strace -p 12280 
strace: Process 12280 attached 
select(0, NULL, NULL, NULL, {tv_sec=0, tv_usec=567708}) = 0 (Timeout) 
stat("/usr/local/lib/python3.7/importlib/_bootstrap.py", {st_mode=S_IFREG|0644, st_size=39278, ...}) = 0 
stat("/usr/local/lib/python3.7/importlib/_bootstrap.py", {st_mode=S_IFREG|0644, st_size=39278, ...}) = 0 

# filter result
 strace -p 12280 2>&1 | grep write 

# introduce new tool bcc
# install bcc
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 4052245BD4284CDD 
echo "deb https://repo.iovisor.org/apt/$(lsb_release -cs) $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/iovisor.list 
sudo apt-get update 
sudo apt-get install bcc-tools libbcc-examples linux-headers-$(uname -r)

cd /usr/share/bcc/tools
./filetop -C 
 
TID    COMM             READS  WRITES R_Kb    W_Kb    T FILE 
514    python           0      1      0       2832    R 669.txt 
514    python           0      1      0       2490    R 667.txt 
514    python           0      1      0       2685    R 671.txt 
514    python           0      1      0       2392    R 670.txt 
514    python           0      1      0       2050    R 672.txt 
 
...
 
TID    COMM             READS  WRITES R_Kb    W_Kb    T FILE 
514    python           2      0      5957    0       R 651.txt 
514    python           2      0      5371    0       R 112.txt 
514    python           2      0      4785    0       R 861.txt 
514    python           2      0      4736    0       R 213.txt 
514    python           2      0      4443    0       R 45.txt 

# -T thread check thread 514
ps -efT | grep 514
root     12280  514 14626 33 14:47 pts/0    00:00:05 /usr/local/bin/python /app.py 


# new tool opensnoop
opensnoop 
12280  python              6   0 /tmp/9046db9e-fe25-11e8-b13f-0242ac110002/650.txt 
12280  python              6   0 /tmp/9046db9e-fe25-11e8-b13f-0242ac110002/651.txt 
12280  python              6   0 /tmp/9046db9e-fe25-11e8-b13f-0242ac110002/652.txt 
```

## sql slow
```bash
# use top to check overview
top
top - 12:02:15 up 6 days,  8:05,  1 user,  load average: 0.66, 0.72, 0.59
Tasks: 137 total,   1 running,  81 sleeping,   0 stopped,   0 zombie
%Cpu0  :  0.7 us,  1.3 sy,  0.0 ni, 35.9 id, 62.1 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu1  :  0.3 us,  0.7 sy,  0.0 ni, 84.7 id, 14.3 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem :  8169300 total,  7238472 free,   546132 used,   384696 buff/cache
KiB Swap:        0 total,        0 free,        0 used.  7316952 avail Mem
 
  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
27458 999       20   0  833852  57968  13176 S   1.7  0.7   0:12.40 mysqld
27617 root      20   0   24348   9216   4692 S   1.0  0.1   0:04.40 python
 1549 root      20   0  236716  24568   9864 S   0.3  0.3  51:46.57 python3
22421 root      20   0       0      0      0 I   0.3  0.0   0:01.16 kworker/u

# io wait is high, so we check io

iostat -d -x 1
Device            r/s     w/s     rkB/s     wkB/s   rrqm/s   wrqm/s  %rrqm  %wrqm r_await w_await aqu-sz rareq-sz wareq-sz  svctm  %util
...
sda            273.00    0.00  32568.00      0.00     0.00     0.00   0.00   0.00    7.90    0.00   1.16   119.30     0.00   3.56  97.20

# rkB/s is 32 MB, util is 97.2%

# find the pid of high IO
pidstat -d 1
12:04:11      UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s iodelay  Command
12:04:12      999     27458  32640.00      0.00      0.00       0  mysqld
12:04:12        0     27617      4.00      4.00      0.00       3  python
12:04:12        0     27864      0.00      4.00      0.00       0  systemd-journal

# next use strace+ lsof 
strace -f -p 27458
[pid 28014] read(38, "934EiwT363aak7VtqF1mHGa4LL4Dhbks"..., 131072) = 131072
[pid 28014] read(38, "hSs7KBDepBqA6m4ce6i6iUfFTeG9Ot9z"..., 20480) = 20480
[pid 28014] read(38, "NRhRjCSsLLBjTfdqiBRLvN9K6FRfqqLm"..., 131072) = 131072
[pid 28014] read(38, "AKgsik4BilLb7y6OkwQUjjqGeCTQTaRl"..., 24576) = 24576
[pid 28014] read(38, "hFMHx7FzUSqfFI22fQxWCpSnDmRjamaW"..., 131072) = 131072
[pid 28014] read(38, "ajUzLmKqivcDJSkiw7QWf2ETLgvQIpfC"..., 20480) = 20480

# Thread 28014 is reading a large amount of data, and the descriptor read is 38. 
# We can execute the following lsof command and specify the thread number 28014, 
# specifically view this suspicious thread and suspicious file:
lsof -p 28014
# no output previous command 

# check the previous command, failed
echo $?
1

#-t thread  -a show command
pstree -t -a -p 27458
mysqld,27458 --log_bin=on --sync_binlog=1
...
  ├─{mysqld},27922
  ├─{mysqld},27923
  └─{mysqld},28014

# use pid not thread id
lsof -p 27458
COMMAND  PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
...
​mysqld  27458      999   38u   REG    8,1 512440000 2601895 /var/lib/mysql/test/products.MYD

```

## slow redis
```bash
top
top - 12:46:18 up 11 days,  8:49,  1 user,  load average: 1.36, 1.36, 1.04
Tasks: 137 total,   1 running,  79 sleeping,   0 stopped,   0 zombie
%Cpu0  :  6.0 us,  2.7 sy,  0.0 ni,  5.7 id, 84.7 wa,  0.0 hi,  1.0 si,  0.0 st
%Cpu1  :  1.0 us,  3.0 sy,  0.0 ni, 94.7 id,  0.0 wa,  0.0 hi,  1.3 si,  0.0 st
KiB Mem :  8169300 total,  7342244 free,   432912 used,   394144 buff/cache
KiB Swap:        0 total,        0 free,        0 used.  7478748 avail Mem
 
  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
 9181 root      20   0  193004  27304   8716 S   8.6  0.3   0:07.15 python
 9085 systemd+  20   0   28352   9760   1860 D   5.0  0.1   0:04.34 redis-server
  368 root      20   0       0      0      0 D   1.0  0.0   0:33.88 jbd2/sda1-8
  149 root       0 -20       0      0      0 I   0.3  0.0   0:10.63 kworker/0:1H
 1549 root      20   0  236716  24576   9864 S   0.3  0.3  91:37.30 python3

 # high io wait 84.7%

iostat -d -x 1
Device            r/s     w/s     rkB/s     wkB/s   rrqm/s   wrqm/s  %rrqm  %wrqm r_await w_await aqu-sz rareq-sz wareq-sz  svctm  %util
...
sda              0.00  492.00      0.00   2672.00     0.00   176.00   0.00  26.35    0.00    1.76   0.00     0.00     5.43   0.00   0.00

# why has write operation ? redis is just read
pidstat -d 1
12:49:35      UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s iodelay  Command
12:49:36        0       368      0.00     16.00      0.00      86  jbd2/sda1-8
12:49:36      100      9085      0.00    636.00      0.00       1  redis-server

# -f follow process and thread，-T system call time，-tt show time
strace -f -T -tt -p 9085
[pid  9085] 14:20:16.826131 epoll_pwait(5, [{EPOLLIN, {u32=8, u64=8}}], 10128, 65, NULL, 8) = 1 <0.000055>
[pid  9085] 14:20:16.826301 read(8, "*2\r\n$3\r\nGET\r\n$41\r\nuuid:5b2e76cc-"..., 16384) = 61 <0.000071>
[pid  9085] 14:20:16.826477 read(3, 0x7fff366a5747, 1) = -1 EAGAIN (Resource temporarily unavailable) <0.000063>
[pid  9085] 14:20:16.826645 write(8, "$3\r\nbad\r\n", 9) = 9 <0.000173>
[pid  9085] 14:20:16.826907 epoll_pwait(5, [{EPOLLIN, {u32=8, u64=8}}], 10128, 65, NULL, 8) = 1 <0.000032>
[pid  9085] 14:20:16.827030 read(8, "*2\r\n$3\r\nGET\r\n$41\r\nuuid:55862ada-"..., 16384) = 61 <0.000044>
[pid  9085] 14:20:16.827149 read(3, 0x7fff366a5747, 1) = -1 EAGAIN (Resource temporarily unavailable) <0.000043>
[pid  9085] 14:20:16.827285 write(8, "$3\r\nbad\r\n", 9) = 9 <0.000141>
[pid  9085] 14:20:16.827514 epoll_pwait(5, [{EPOLLIN, {u32=8, u64=8}}], 10128, 64, NULL, 8) = 1 <0.000049>
[pid  9085] 14:20:16.827641 read(8, "*2\r\n$3\r\nGET\r\n$41\r\nuuid:53522908-"..., 16384) = 61 <0.000043>
[pid  9085] 14:20:16.827784 read(3, 0x7fff366a5747, 1) = -1 EAGAIN (Resource temporarily unavailable) <0.000034>
[pid  9085] 14:20:16.827945 write(8, "$4\r\ngood\r\n", 10) = 10 <0.000288>
[pid  9085] 14:20:16.828339 epoll_pwait(5, [{EPOLLIN, {u32=8, u64=8}}], 10128, 63, NULL, 8) = 1 <0.000057>
[pid  9085] 14:20:16.828486 read(8, "*3\r\n$4\r\nSADD\r\n$4\r\ngood\r\n$36\r\n535"..., 16384) = 67 <0.000040>
[pid  9085] 14:20:16.828623 read(3, 0x7fff366a5747, 1) = -1 EAGAIN (Resource temporarily unavailable) <0.000052>
[pid  9085] 14:20:16.828760 write(7, "*3\r\n$4\r\nSADD\r\n$4\r\ngood\r\n$36\r\n535"..., 67) = 67 <0.000060>
[pid  9085] 14:20:16.828970 fdatasync(7) = 0 <0.005415>
[pid  9085] 14:20:16.834493 write(8, ":1\r\n", 4) = 4 <0.000250>


lsof -p 9085
redis-ser 9085 systemd-network    3r     FIFO   0,12      0t0 15447970 pipe
redis-ser 9085 systemd-network    4w     FIFO   0,12      0t0 15447970 pipe
redis-ser 9085 systemd-network    5u  a_inode   0,13        0    10179 [eventpoll]
redis-ser 9085 systemd-network    6u     sock    0,9      0t0 15447972 protocol: TCP
redis-ser 9085 systemd-network    7w      REG    8,1  8830146  2838532 /data/appendonly.aof
redis-ser 9085 systemd-network    8u     sock    0,9      0t0 15448709 protocol: TCP

# Descriptor number 3 is a pipe, number 5 is an eventpoll
# number 7 is an ordinary file, and number 8 is a TCP socket.
# Only the 7th ordinary file will generate disk write, 
# and the file path it operates on is /data/appendonly.aof. 
# The corresponding system calls include write and fdatasync.


strace -f -p 9085 -T -tt -e fdatasync
strace: Process 9085 attached with 4 threads
[pid  9085] 14:22:52.013547 fdatasync(7) = 0 <0.007112>
[pid  9085] 14:22:52.022467 fdatasync(7) = 0 <0.008572>
[pid  9085] 14:22:52.032223 fdatasync(7) = 0 <0.006769>
...
[pid  9085] 14:22:52.139629 fdatasync(7) = 0 <0.008183>

# final check the process in docker
PID=$(docker inspect --format {{.State.Pid}} app)
nsenter --target $PID --net -- lsof -i
COMMAND    PID            USER   FD   TYPE   DEVICE SIZE/OFF NODE NAME
redis-ser 9085 systemd-network    6u  IPv4 15447972      0t0  TCP localhost:6379 (LISTEN)
redis-ser 9085 systemd-network    8u  IPv4 15448709      0t0  TCP localhost:6379->localhost:32996 (ESTABLISHED)
python    9181            root    3u  IPv4 15448677      0t0  TCP *:http (LISTEN)
python    9181            root    5u  IPv4 15449632      0t0  TCP localhost:32996->localhost:6379 (ESTABLISHED)
 
```
## summary
![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20191229183130.png)
![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20191229183134.png)
![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20191229183137.png)
![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20191229183323.png)

# Network
![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20191229202720.png)  
![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20191229202724.png)
* Bandwidth, which indicates the maximum transmission rate of the link. The unit is usually b/s (bits per second).
* Throughput, which indicates the amount of data successfully transmitted per unit of time. The unit is usually b/s (bits/second) or B/s (bytes/second). Throughput is limited by bandwidth, and throughput/bandwidth is the utilization of the network.
* Delay means the delay from the time the network request is sent until the remote response is received.
* PPS is the abbreviation of Packet Per Second (packet per second), which means the transmission rate in network packets.

## basic command
```bash
root@ubuntu:/home/feiyang# ifconfig 
ens33: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.17.3  netmask 255.255.255.0  broadcast 192.168.17.255
        inet6 fe80::20c:29ff:fe35:6403  prefixlen 64  scopeid 0x20<link>
        ether 00:0c:29:35:64:03  txqueuelen 1000  (Ethernet)
        RX packets 2193  bytes 150818 (150.8 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 203  bytes 24234 (24.2 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 17791  bytes 1264539 (1.2 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 17791  bytes 1264539 (1.2 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```
* errors indicates the number of packets with errors, such as check errors, frame synchronization errors, etc.
* dropped indicates the number of dropped packets, that is, the packet has received the Ring Buffer, but the packet was lost due to insufficient memory and other reasons;
* overruns indicates the number of overrun packets, that is, the network I / O speed is too fast, causing the packets in the Ring Buffer to be too late to be processed (the queue is full) and the packet loss;
* carrier indicates the number of packets with carrirer errors, such as mismatch in duplex mode, problems with physical cables, etc .;
* collisions indicates the number of collision packets.

```bash
root@ubuntu:/home/feiyang# netstat -nlp | head -n 3
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN      549/systemd-resolve 

root@ubuntu:/home/feiyang# ss -lntp | head -n 3
State    Recv-Q    Send-Q        Local Address:Port        Peer Address:Port                                                                                    
LISTEN   0         128           127.0.0.53%lo:53               0.0.0.0:*        users:(("systemd-resolve",pid=549,fd=13))                                      
LISTEN   0         128                 0.0.0.0:22               0.0.0.0:*        users:(("sshd",pid=1040,fd=3)) 

```
+ When the socket is connected (Established),
Recv-Q indicates the number of bytes (that is, the length of the receive queue) that the socket buffer has not been taken away by the application.
Send-Q indicates the number of bytes (that is, the length of the send queue) that have not been acknowledged by the remote host.

+ When the socket is in the listening state (Listening),
Recv-Q represents the current value of the syn backlog.
Send-Q represents the largest syn backlog value.

The syn backlog is the length of the semi-connected queue in the TCP protocol stack, and accordingly there is also a fully connected queue (accept queue)

```bash
# netstat
root@ubuntu:/home/feiyang# netstat -s
Ip:
    Forwarding: 2
    19787 total packets received
    0 forwarded
    0 incoming packets discarded
    19785 incoming packets delivered
    19373 requests sent out
Icmp:
    0 ICMP messages received
    0 input ICMP message failed
    ICMP input histogram:
    0 ICMP messages sent
    0 ICMP messages failed
    ICMP output histogram:
Tcp:
    4 active connection openings
    1 passive connection openings
    4 failed connection attempts
    0 connection resets received
    1 connections established
    294 segments received
    195 segments sent out
    0 segments retransmitted
    0 bad segments received
    4 resets sent
Udp:
    19388 packets received
    0 packets to unknown port received
    0 packet receive errors
    19180 packets sent
    0 receive buffer errors
    0 send buffer errors
    IgnoredMulti: 107
UdpLite:
TcpExt:
    3 delayed acks sent
    47 packet headers predicted
    2 acknowledgments not containing data payload received
    170 predicted acknowledgments
    TCPBacklogCoalesce: 9
    TCPAutoCorking: 12
    TCPOrigDataSent: 180
    TCPDelivered: 180
IpExt:
    InMcastPkts: 279
    OutMcastPkts: 66
    InBcastPkts: 107
    OutBcastPkts: 7
    InOctets: 1405727
    OutOctets: 1393405
    InMcastOctets: 19672
    OutMcastOctets: 6290
    InBcastOctets: 9737
    OutBcastOctets: 327
    InNoECTPkts: 19788

# ss
root@ubuntu:/home/feiyang# ss -s
Total: 958 (kernel 2224)
TCP:   6 (estab 1, closed 0, orphaned 0, synrecv 0, timewait 0/0), ports 0

Transport  Total     IP        IPv6
*	         2224      -         -        
RAW	       0         0         0        
UDP	       6         4         2        
TCP	       6         4         2        
INET	     12        8         4        
FRAG	     0         0         0 


# sar
root@ubuntu:/home/feiyang# sar -n DEV 1
Linux 5.0.0-37-generic (ubuntu) 	12/29/2019 	_x86_64_	(2 CPU)

08:42:20 PM     IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s   rxcmp/s   txcmp/s  rxmcst/s   %ifutil
08:42:21 PM     ens33      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
08:42:21 PM        lo     80.00     80.00      5.55      5.55      0.00      0.00      0.00      0.00

08:42:21 PM     IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s   rxcmp/s   txcmp/s  rxmcst/s   %ifutil
08:42:22 PM     ens33     34.00     34.00      1.99      3.25      0.00      0.00      0.00      0.00
08:42:22 PM        lo      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00

# ping
root@ubuntu:/home/feiyang# ping -c3 114.114.114.114
PING 114.114.114.114 (114.114.114.114) 56(84) bytes of data.
64 bytes from 114.114.114.114: icmp_seq=1 ttl=128 time=222 ms
64 bytes from 114.114.114.114: icmp_seq=2 ttl=128 time=221 ms
64 bytes from 114.114.114.114: icmp_seq=3 ttl=128 time=222 ms

root@ubuntu:/home/feiyang# ethtool ens33 | grep Speed
	Speed: 1000Mb/s
```
##  C10K and C1000K
1. select or poll Apache
2. epoll Nginx 
3. Asynchronous I/O


1. one master, multiple worker
2. multi-process listen same port, need enable SO_REUSEPORT
3. C1000K's solution is essentially built on epoll's non-blocking I / O model.
4. C10M: To solve this problem, the most important thing is to skip the lengthy path of the kernel protocol stack and send the network packets directly to the application to be processed. There are two common mechanisms here, DPDK and XDP.
+ DPDK is the standard for user mode networks. It skips the kernel protocol stack and directly processes the network reception by the user mode process through polling.
![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20191229210141.png)  
+ XDP (eXpress Data Path) is a high-performance network data path provided by the Linux kernel. It allows network packets to be processed before entering the kernel protocol stack, which can also bring higher performance. The bottom layer of XDP, like the bcc-tools we used before, is implemented based on the eBPF mechanism of the Linux kernel.
![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20191229210414.png)

## performance test 

```bash
# enable pktgen
root@ubuntu:/home/feiyang# modprobe pktgen
root@ubuntu:/home/feiyang# ps -ef | grep pktgen | grep -v grep
root       2478      2  0 22:25 ?        00:00:00 [kpktgend_0]
root       2480      2  0 22:25 ?        00:00:00 [kpktgend_1]
root       2481      2  0 22:25 ?        00:00:00 [kpktgend_0]
root       2482      2  0 22:25 ?        00:00:00 [kpktgend_1]
root@ubuntu:/home/feiyang# ls /proc/net/pktgen/
kpktgend_0  kpktgend_1  pgctrl
```
do some test

```bash
# define function
function pgset() {
    local result
    echo $1 > $PGDEV
 
    result=`cat $PGDEV | fgrep "Result: OK:"`
    if [ "$result" = "" ]; then
         cat $PGDEV | fgrep Result:
    fi
}
 
# bind 0 thread to eth0
PGDEV=/proc/net/pktgen/kpktgend_0
pgset "rem_device_all"   # clear bind
pgset "add_device eth0"  # add eth0 
 
# config eth0 
PGDEV=/proc/net/pktgen/eth0
pgset "count 1000000"    # total send package
pgset "delay 5000"       # interval 
pgset "clone_skb 0"      # SKB package duplicate
pgset "pkt_size 64"      # package size
pgset "dst 192.168.0.30" # dest IP
pgset "dst_mac 11:11:11:11:11:11"  # dest MAC
 
# start test
PGDEV=/proc/net/pktgen/pgctrl
pgset "start"


cat /proc/net/pktgen/eth0
Params: count 1000000  min_pkt_size: 64  max_pkt_size: 64
     frags: 0  delay: 0  clone_skb: 0  ifname: eth0
     flows: 0 flowlen: 0
...
Current:
     pkts-sofar: 1000000  errors: 0
     started: 1534853256071us  stopped: 1534861576098us idle: 70673us
...
Result: OK: 8320027(c8249354+d70673) usec, 1000000 (64byte,0frags)
  120191pps 61Mb/sec (61537792bps) errors: 0
```
TCP test

```bash
# Ubuntu
apt-get install iperf3
# CentOS
yum install iperf3


root@ubuntu:/home/feiyang# iperf3 -s -i 1 -p 10000
-----------------------------------------------------------
Server listening on 10000
-----------------------------------------------------------


iperf3 -c 192.168.0.30 -b 1G -t 15 -P 2 -p 10000

[ ID] Interval           Transfer     Bandwidth
...
[SUM]   0.00-15.04  sec  0.00 Bytes  0.00 bits/sec                  sender
[SUM]   0.00-15.04  sec  1.51 GBytes   860 Mbits/sec                  receiver

```
HTTP test

```
Ubuntu
apt-get install -y apache2-utils
# CentOS
yum install -y httpd-tools


ab -c 1000 -n 10000 http://192.168.0.30/
...
Server Software:        nginx/1.15.8
Server Hostname:        192.168.0.30
Server Port:            80
 
...
 
Requests per second:    1078.54 [#/sec] (mean)
Time per request:       927.183 [ms] (mean)
Time per request:       0.927 [ms] (mean, across all concurrent requests)
Transfer rate:          890.00 [Kbytes/sec] received
 
Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0   27 152.1      1    1038
Processing:     9  207 843.0     22    9242
Waiting:        8  207 843.0     22    9242
Total:         15  233 857.7     23    9268
 
Percentage of the requests served within a certain time (ms)
  50%     23
  66%     24
  75%     24
  80%     26
  90%    274
  95%   1195
  98%   2335
  99%   4663
 100%   9268 (longest request)
```
App test
```bash
git clone https://github.com/wg/wrk
cd wrk
apt-get install build-essential -y
make
sudo cp wrk /usr/local/bin/


wrk -c 1000 -t 2 http://192.168.0.30/
Running 10s test @ http://192.168.0.30/
  2 threads and 1000 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    65.83ms  174.06ms   1.99s    95.85%
    Req/Sec     4.87k   628.73     6.78k    69.00%
  96954 requests in 10.06s, 78.59MB read
  Socket errors: connect 0, read 0, write 0, timeout 179
Requests/sec:   9641.31
Transfer/sec:      7.82MB

# setup add parameter
-- example script that demonstrates response handling and
-- retrieving an authentication token to set on all future
-- requests
 
token = nil
path  = "/authenticate"
 
request = function()
   return wrk.format("GET", path)
end
 
response = function(status, headers, body)
   if not token and status == 200 then
      token = headers["X-Token"]
      path  = "/resource"
      wrk.headers["X-Token"] = token
   end
end

wrk -c 1000 -t 2 -s auth.lua http://192.168.0.30/

```

## DNS slow
+ A record, used to translate the domain name into an IP address;
+ CNAME record for creating aliases;
+ The NS record indicates the name server address corresponding to the domain name.


```bash
root@ubuntu:/home/feiyang/wrk# dig +trace +nodnssec feiyang233.club
root@ubuntu:/home/feiyang/wrk# nslookup feiyang233.club
Server:		1.1.1.1
Address:	1.1.1.1#53

Non-authoritative answer:
feiyang233.club	canonical name = fainyang.github.io.
Name:	fainyang.github.io
Address: 185.199.109.153
Name:	fainyang.github.io
Address: 185.199.110.153
Name:	fainyang.github.io
Address: 185.199.111.153
Name:	fainyang.github.io
Address: 185.199.108.153


root@ubuntu:/home/feiyang/wrk# dig +trace +nodnssec feiyang233.club
; <<>> DiG 9.11.3-1ubuntu1.11-Ubuntu <<>> +trace +nodnssec feiyang233.club
;; global options: +cmd
.			7974	IN	NS	i.root-servers.net.
.			7974	IN	NS	j.root-servers.net.
.			7974	IN	NS	k.root-servers.net.
.			7974	IN	NS	l.root-servers.net.
.			7974	IN	NS	m.root-servers.net.
.			7974	IN	NS	a.root-servers.net.
.			7974	IN	NS	b.root-servers.net.
.			7974	IN	NS	c.root-servers.net.
.			7974	IN	NS	d.root-servers.net.
.			7974	IN	NS	e.root-servers.net.
.			7974	IN	NS	f.root-servers.net.
.			7974	IN	NS	g.root-servers.net.
.			7974	IN	NS	h.root-servers.net.
;; Received 431 bytes from 1.1.1.1#53(1.1.1.1) in 8 ms

club.			172800	IN	NS	ns4.dns.nic.club.
club.			172800	IN	NS	ns1.dns.nic.club.
club.			172800	IN	NS	ns6.dns.nic.club.
club.			172800	IN	NS	ns2.dns.nic.club.
club.			172800	IN	NS	ns3.dns.nic.club.
club.			172800	IN	NS	ns5.dns.nic.club.
;; Received 456 bytes from 192.36.148.17#53(i.root-servers.net) in 104 ms

feiyang233.club.	3600	IN	NS	f1g1ns1.dnspod.net.
feiyang233.club.	3600	IN	NS	f1g1ns2.dnspod.net.
;; Received 98 bytes from 156.154.144.215#53(ns1.dns.nic.club) in 189 ms

;; Received 44 bytes from 182.140.167.166#53(f1g1ns1.dnspod.net) in 404 ms
```
1. no /etc/resolv.conf 
```bash
nslookup feiyang233.club
;; connection timed out; no servers could be reached

# add DNS
echo "nameserver 1.1.1.1" > /etc/resolv.conf
```
2. DNS unstable
```bash
time nslookup time.geekbang.org
Server:		8.8.8.8
Address:	8.8.8.8#53
 
Non-authoritative answer:
Name:	time.geekbang.org
Address: 39.106.233.176
 
real	0m10.349s
user	0m0.004s
sys	0m0.0


time nslookup time.geekbang.org
;; connection timed out; no servers could be reached
 
real	0m15.011s
user	0m0.006s
sys	0m0.006s
```
The DNS server itself has problems, the response is slow and unstable;

The network delay from the client to the DNS server is relatively large;

DNS request or response packets are lost by network devices on the link in some cases.

```bash
ping -c3 8.8.8.8
PING 8.8.8.8 (8.8.8.8): 56 data bytes
64 bytes from 8.8.8.8: icmp_seq=0 ttl=31 time=137.637 ms
64 bytes from 8.8.8.8: icmp_seq=1 ttl=31 time=144.743 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=31 time=138.576 ms
--- 8.8.8.8 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max/stddev = 137.637/140.319/144.743/3.152 ms

# try other DNS

ping -c3 114.114.114.114
PING 114.114.114.114 (114.114.114.114): 56 data bytes
64 bytes from 114.114.114.114: icmp_seq=0 ttl=67 time=31.130 ms
64 bytes from 114.114.114.114: icmp_seq=1 ttl=56 time=31.302 ms
64 bytes from 114.114.114.114: icmp_seq=2 ttl=56 time=31.250 ms
--- 114.114.114.114 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max/stddev = 31.130/31.227/31.302/0.072 ms


echo nameserver 114.114.114.114 > /etc/resolv.conf
time nslookup time.geekbang.org
Server:		114.114.114.114
Address:	114.114.114.114#53
 
Non-authoritative answer:
Name:	time.geekbang.org
Address: 39.106.233.176
 
real    0m0.064s
user    0m0.007s
sys     0m0.006s

```
## dump traffic
```bash

# Ubuntu
apt-get install tcpdump wireshark
 
# CentOS
yum install -y tcpdump wireshark

tcpdump -nn udp port 53 or host 35.190.27.188

tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
14:02:31.100564 IP 172.16.3.4.56669 > 114.114.114.114.53: 36909+ A? geektime.org. (30)
14:02:31.507699 IP 114.114.114.114.53 > 172.16.3.4.56669: 36909 1/0/0 A 35.190.27.188 (46)
14:02:31.508164 IP 172.16.3.4 > 35.190.27.188: ICMP echo request, id 4356, seq 1, length 64
14:02:31.539667 IP 35.190.27.188 > 172.16.3.4: ICMP echo reply, id 4356, seq 1, length 64
14:02:31.539995 IP 172.16.3.4.60254 > 114.114.114.114.53: 49932+ PTR? 188.27.190.35.in-addr.arpa. (44)
14:02:36.545104 IP 172.16.3.4.60254 > 114.114.114.114.53: 49932+ PTR? 188.27.190.35.in-addr.arpa. (44)
14:02:41.551284 IP 172.16.3.4 > 35.190.27.188: ICMP echo request, id 4356, seq 2, length 64
14:02:41.582363 IP 35.190.27.188 > 172.16.3.4: ICMP echo reply, id 4356, seq 2, length 64
14:02:42.552506 IP 172.16.3.4 > 35.190.27.188: ICMP echo request, id 4356, seq 3, length 64
14:02:42.583646 IP 35.190.27.188 > 172.16.3.4: ICMP echo reply, id 4356, seq 3, length 64
```
The purpose of PTR reverse address resolution is to find out the domain name from the IP address, but in fact, not all IP addresses will define PTR records, so PTR queries are likely to fail.

```bash
# check PTR

nslookup -type=PTR 35.190.27.188 8.8.8.8
Server:	8.8.8.8
Address:	8.8.8.8#53
Non-authoritative answer:
188.27.190.35.in-addr.arpa	name = 188.27.190.35.bc.googleusercontent.com.
Authoritative answers can be found from:

# test
dig +short example.com
93.184.216.34
tcpdump -nn host 93.184.216.34 -w web.pcap

# in another tty
curl http://example.com

```
![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20191231091850.png)
![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20191231091854.png)  
```bash
#tcpdump output format
Timestamp Protocol Source Address Source Port> Destination Address Destination Port Network Packet Details
时间戳 协议 源地址 源端口 > 目的地址 目的端口 网络包详细信息

```
![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20191231091857.png)  
![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20191231091900.png)

## anti-DDoS
DDoS（Distributed Denial of Service） 
+ Running out of bandwidth
+ Running out of operating system resources
+ Running out of application resources

```bash
# -S set TCP  SYN，-p port 80
# -i u10  10us interval
hping3 -S -p 80 -i u10 192.168.0.30


# -w only output HTTP status total time -o redirect to /dev/null
curl -s -w 'Http code: %{http_code}\nTotal time:%{time_total}s\n' -o /dev/null http://192.168.0.30/
...
Http code: 200
Total time:0.002s


sar -n DEV 1
08:55:49        IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s   rxcmp/s   txcmp/s  rxmcst/s   %ifutil
08:55:50      docker0      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
08:55:50         eth0  22274.00    629.00   1174.64     37.78      0.00      0.00      0.00      0.02
08:55:50           lo      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00

# small package 54B （1174*1024/22274=54）


# -i eth0 nic，-n Don't convert addresses (i.e., host addresses, port numbers, etc.) to names.
# tcp port 80 
$ tcpdump -i eth0 -n tcp port 80
09:15:48.287047 IP 192.168.0.2.27095 > 192.168.0.30: Flags [S], seq 1288268370, win 512, length 0
09:15:48.287050 IP 192.168.0.2.27131 > 192.168.0.30: Flags [S], seq 2084255254, win 512, length 0
09:15:48.287052 IP 192.168.0.2.27116 > 192.168.0.30: Flags [S], seq 677393791, win 512, length 0
09:15:48.287055 IP 192.168.0.2.27141 > 192.168.0.30: Flags [S], seq 1276451587, win 512, length 0
09:15:48.287068 IP 192.168.0.2.27154 > 192.168.0.30: Flags [S], seq 1851495339, win 512, length 0
...

```
![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20191231164029.png)
![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20191231164122.png)

```bash
# check SYN_REC
netstat -n -p | grep SYN_REC
tcp        0      0 192.168.0.30:80          192.168.0.2:12503      SYN_RECV    -
tcp        0      0 192.168.0.30:80          192.168.0.2:13502      SYN_RECV    -
tcp        0      0 192.168.0.30:80          192.168.0.2:15256      SYN_RECV    -
tcp        0      0 192.168.0.30:80          192.168.0.2:18117      SYN_RECV    -
...

netstat -n -p | grep SYN_REC | wc -l
193

# method for SYN DDoS
# limit SYN 1/s
$ iptables -A INPUT -p tcp --syn -m limit --limit 1/s -j ACCEPT
 
# limit every IP only establish 10 connections during 60s
$ iptables -I INPUT -p tcp --dport 80 --syn -m recent --name SYN_FLOOD --update --seconds 60 --hitcount 10 -j REJECT

sysctl net.ipv4.tcp_max_syn_backlog
net.ipv4.tcp_max_syn_backlog = 256

# increase max
sysctl -w net.ipv4.tcp_max_syn_backlog=1024
net.ipv4.tcp_max_syn_backlog = 1024

# decrease retry time from default 5 to 1
sysctl -w net.ipv4.tcp_synack_retries=1
net.ipv4.tcp_synack_retries = 1

# enable TCP SYN Cookies
sysctl -w net.ipv4.tcp_syncookies=1
net.ipv4.tcp_syncookies = 1

# above set is temporary, for persistent
cat /etc/sysctl.conf
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_synack_retries = 1
net.ipv4.tcp_max_syn_backlog = 1024
```
In a Linux server, you can increase the anti-attack capability of the server and reduce the impact of DDoS on normal services through various methods such as kernel tuning, DPDK, and XDP. In the application, you can use various levels of caching, WAF, CDN and other methods to mitigate the impact of DDoS on the application.

## network slow
```bash
traceroute --tcp -p 80 -n baidu.com
traceroute to baidu.com (123.125.115.110), 30 hops max, 60 byte packets
 1  * * *
 2  * * *
 3  * * *
 4  * * *
 5  * * *
 6  * * *
 7  * * *
 8  * * *
 9  * * *
10  * * *
11  * * *
12  * * *
13  * * *
14  123.125.115.110  20.684 ms *  20.798 ms


# install wrk
git clone https://github.com/wg/wrk
cd wrk
apt-get install build-essential -y
make
sudo cp wrk /usr/local/bin/

# create 2 nginx
docker run --network=host --name=good -itd nginx

docker run --name nginx --network=host -itd feisky/nginx:latency

# curl port 80 and 8080
# 80 ok
curl http://192.168.0.30
<!DOCTYPE html>
<html>
...
<p><em>Thank you for using nginx.</em></p>
</body>
</html>
 
# 8080 ok
curl http://192.168.0.30:8080
...
<p><em>Thank you for using nginx.</em></p>
</body>
</html>

# 80 ok 
hping3 -c 3 -S -p 80 192.168.0.30
HPING 192.168.0.30 (eth0 192.168.0.30): S set, 40 headers + 0 data bytes
len=44 ip=192.168.0.30 ttl=64 DF id=0 sport=80 flags=SA seq=0 win=29200 rtt=7.8 ms
len=44 ip=192.168.0.30 ttl=64 DF id=0 sport=80 flags=SA seq=1 win=29200 rtt=7.7 ms
len=44 ip=192.168.0.30 ttl=64 DF id=0 sport=80 flags=SA seq=2 win=29200 rtt=7.6 ms
 
--- 192.168.0.30 hping statistic ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 7.6/7.7/7.8 ms

# 8080 ok
hping3 -c 3 -S -p 8080 192.168.0.30
HPING 192.168.0.30 (eth0 192.168.0.30): S set, 40 headers + 0 data bytes
len=44 ip=192.168.0.30 ttl=64 DF id=0 sport=8080 flags=SA seq=0 win=29200 rtt=7.7 ms
len=44 ip=192.168.0.30 ttl=64 DF id=0 sport=8080 flags=SA seq=1 win=29200 rtt=7.6 ms
len=44 ip=192.168.0.30 ttl=64 DF id=0 sport=8080 flags=SA seq=2 win=29200 rtt=7.3 ms
 
--- 192.168.0.30 hping statistic ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 7.3/7.6/7.7 ms

# test 80 
测试 80 端口性能
$ # wrk --latency -c 100 -t 2 --timeout 2 http://192.168.0.30/
Running 10s test @ http://192.168.0.30/
  2 threads and 100 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     9.19ms   12.32ms 319.61ms   97.80%
    Req/Sec     6.20k   426.80     8.25k    85.50%
  Latency Distribution
     50%    7.78ms
     75%    8.22ms
     90%    9.14ms
     99%   50.53ms
  123558 requests in 10.01s, 100.15MB read
Requests/sec:  12340.91
Transfer/sec:     10.00MB

# test 8080, this nginx is very slow
wrk --latency -c 100 -t 2 --timeout 2 http://192.168.0.30:8080/
Running 10s test @ http://192.168.0.30:8080/
  2 threads and 100 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    43.60ms    6.41ms  56.58ms   97.06%
    Req/Sec     1.15k   120.29     1.92k    88.50%
  Latency Distribution
     50%   44.02ms
     75%   44.33ms
     90%   47.62ms
     99%   48.88ms
  22853 requests in 10.01s, 18.55MB read
Requests/sec:   2283.31
Transfer/sec:      1.85MB

# dump the traffic

tcpdump -nn tcp port 8080 -w nginx.pcap
wrk --latency -c 100 -t 2 --timeout 2 http://192.168.0.30:8080/

```
Open this nginx.pcap in Wireshark, Statics -> Flow Graph，select “Limit to display filter” and setup Flow type to “TCP Flows”：
![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20200101110429.png)
Blue area is very slow costs 40ms, 40ms is minimum timeout for TCP delayed acknowledgement (Delayed ACK).  
An optimization mechanism for TCP ACK, that is, instead of sending an ACK for each request, you wait for a while (such as 40ms). If there are other packets that need to be sent during this period, send them with the ACK. Of course, if you can't wait for other packets, then send ACK separately after timeout.

```bash
# TCP_QUICKACK (since Linux 2.4.4)
# Enable  quickack mode if set or disable quickack mode if cleared.  In quickack mode, 
# acks are sent imme‐diately, rather than delayed if needed in accordance to normal TCP operation.
# This flag is  not  perma‐nent,  it only enables a switch to or from quickack mode.  
# Subsequent operation of the TCP protocol will once again enter/leave quickack mode
# depending on internal  protocol  processing  and  factors  such  as delayed ack timeouts occurring 
# and data transfer.  This option should not be used in code intended to be portable.
```

[The Nagle algorithm](https://en.wikipedia.org/wiki/Nagle%27s_algorithm) is an optimization algorithm used in the TCP protocol to reduce the number of small packets sent, in order to improve the utilization of the actual bandwidth. The Nagle algorithm specifies that there can be at most one unconfirmed outstanding packet on a TCP connection; no other packets are sent until an ACK for this packet is received.

```bash
#TCP_NODELAY
# If set, disable the Nagle algorithm.  This means that segments are always sent as soon as possible,
# even if there is only a small amount of data.  When not set, data is buffered until there is a 
# sufficient amount to send out, thereby avoiding the frequent sending of small packets, 
# which results in poor uti‐lization of the network.  
# This option is overridden by TCP_CORK; however, setting this option forces an explicit flush of
# pending output, even if TCP_CORK is currently set.

# check tcp_nodelay
cat /etc/nginx/nginx.conf | grep tcp_nodelay
    tcp_nodelay    off;
```
## NAT
Network Address and Port Translation
![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20200101112538.png)
```bash
# SNAT
# MASQUERADE, change out ip to Linux wan ip
iptables -t nat -A POSTROUTING -s 192.168.0.0/16 -j MASQUERADE

iptables -t nat -A POSTROUTING -s 192.168.0.2 -j SNAT --to-source 100.100.100.100

# DNAT
iptables -t nat -A PREROUTING -d 100.100.100.100 -j DNAT --to-destination 192.168.0.2

iptables -t nat -A POSTROUTING -s 192.168.0.2 -j SNAT --to-source 100.100.100.100
iptables -t nat -A PREROUTING -d 100.100.100.100 -j DNAT --to-destination 192.168.0.2

# enable forward function
# sysctl net.ipv4.ip_forward
# net.ipv4.ip_forward = 1
sysctl -w net.ipv4.ip_forward=1

# open files
ulimit -n
1024

# increase to 66535
ulimit -n 65536


ab -c 5000 -n 100000 -r -s 2 http://192.168.0.30/
...
Requests per second:    6576.21 [#/sec] (mean)
Time per request:       760.317 [ms] (mean)
Time per request:       0.152 [ms] (mean, across all concurrent requests)
Transfer rate:          5390.19 [Kbytes/sec] received
 
Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0  177 714.3      9    7338
Processing:     0   27  39.8     19     961
Waiting:        0   23  39.5     16     951
Total:          1  204 716.3     28    7349

# run new test container
docker run --name nginx --privileged -p 8080:8080 -itd feisky/nginx:nat

iptables -nL -t nat
Chain PREROUTING (policy ACCEPT)
target     prot opt source               destination
DOCKER     all  --  0.0.0.0/0            0.0.0.0/0            ADDRTYPE match dst-type LOCAL
 
...
 
Chain DOCKER (2 references)
target     prot opt source               destination
RETURN     all  --  0.0.0.0/0            0.0.0.0/0
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:8080 to:172.17.0.2:8080

# test again
ab -c 5000 -n 100000 -r -s 2 http://192.168.0.30:8080/
...
apr_pollset_poll: The timeout specified has expired (70007)
Total of 5602 requests completed

# set timeout is 30s
ab -c 5000 -n 10000 -r -s 30 http://192.168.0.30:8080/
...
Requests per second:    76.47 [#/sec] (mean)
Time per request:       65380.868 [ms] (mean)
Time per request:       13.076 [ms] (mean, across all concurrent requests)
Transfer rate:          44.79 [Kbytes/sec] received
 
Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0 1300 5578.0      1   65184
Processing:     0 37916 59283.2      1  130682
Waiting:        0    2   8.7      1     414
Total:          1 39216 58711.6   1021  130682
...
```

we create a script to follow this iptable path
```bash
#! /usr/bin/env stap
 
############################################################
# Dropwatch.stp
# Author: Neil Horman <nhorman@redhat.com>
# An example script to mimic the behavior of the dropwatch utility
# http://fedorahosted.org/dropwatch
############################################################
 
# Array to hold the list of drop points we find
global locations
 
# Note when we turn the monitor on and off
probe begin { printf("Monitoring for dropped packets\n") }
probe end { printf("Stopping dropped packet monitor\n") }
 
# increment a drop counter for every location we drop at
probe kernel.trace("kfree_skb") { locations[$location] <<< 1 }
 
# Every 5 seconds report our drop locations
probe timer.sec(5)
{
  printf("\n")
  foreach (l in locations-) {
    printf("%d packets dropped at %s\n",
           @count(locations[l]), symname(l))
  }
  delete locations
}
```

run this script
```bash

stap --all-modules dropwatch.stp
Monitoring for dropped packets

10031 packets dropped at nf_hook_slow
676 packets dropped at tcp_v4_rcv
 
7284 packets dropped at nf_hook_slow
268 packets dropped at tcp_v4_rcv


# use perf to check 
# record 30s crtl + c
$ perf record -a -g -- sleep 30
 
# print report
$ perf report -g graph,0
```
![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20200101140853.png)

Slow in 3 point
1. ipv4_conntrack_in
2. br_nf_pre_routing
3. iptable_nat_ipv4_in

```bash
# check conntrack
sysctl -a | grep conntrack
net.netfilter.nf_conntrack_count = 180
net.netfilter.nf_conntrack_max = 1000
net.netfilter.nf_conntrack_buckets = 65536
net.netfilter.nf_conntrack_tcp_timeout_syn_recv = 60
net.netfilter.nf_conntrack_tcp_timeout_syn_sent = 120
net.netfilter.nf_conntrack_tcp_timeout_time_wait = 120
...

# net.netfilter.nf_conntrack_max = 1000 is to small
dmesg | tail
[104235.156774] nf_conntrack: nf_conntrack: table full, dropping packet
[104243.800401] net_ratelimit: 3939 callbacks suppressed
[104243.800401] nf_conntrack: nf_conntrack: table full, dropping packet
[104262.962157] nf_conntrack: nf_conntrack: table full, dropping packet


The connection tracking object size is 376, and the list item size is 16
nf_conntrack_max * connection tracking object size + nf_conntrack_buckets * list item size
= 1000 * 376 + 65536 * 16 B
= 1.4 MB

# increase conntrack
sysctl -w net.netfilter.nf_conntrack_max=131072
sysctl -w net.netfilter.nf_conntrack_buckets=65536

# test again, now delay is ok
ab -c 5000 -n 100000 -r -s 2 http://192.168.0.30:8080/
...
Requests per second:    6315.99 [#/sec] (mean)
Time per request:       791.641 [ms] (mean)
Time per request:       0.158 [ms] (mean, across all concurrent requests)
Transfer rate:          4985.15 [Kbytes/sec] received
 
Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0  355 793.7     29    7352
Processing:     8  311 855.9     51   14481
Waiting:        0  292 851.5     36   14481
Total:         15  666 1216.3    148   14645


conntrack -L -o extended | head
ipv4     2 tcp      6 7 TIME_WAIT src=192.168.0.2 dst=192.168.0.96 sport=51744 dport=8080 src=172.17.0.2 dst=192.168.0.2 sport=8080 dport=51744 [ASSURED] mark=0 use=1
ipv4     2 tcp      6 6 TIME_WAIT src=192.168.0.2 dst=192.168.0.96 sport=51524 dport=8080 src=172.17.0.2 dst=192.168.0.2 sport=8080 dport=51524 [ASSURED] mark=0 use=1


conntrack -L -o extended | wc -l
14289

# collecte all states
conntrack -L -o extended | awk '/^.*tcp.*$/ {sum[$6]++} END {for(i in sum) print i, sum[i]}'
SYN_RECV 4
CLOSE_WAIT 9
ESTABLISHED 2877
FIN_WAIT 3
SYN_SENT 2113
TIME_WAIT 9283

# collect each source IP
conntrack -L -o extended | awk '{print $7}' | cut -d "=" -f 2 | sort | uniq -c | sort -nr | head -n 10
  14116 192.168.0.2
    172 192.168.0.96
```
## important 
```bash
# tcp time_wait settings check 
sysctl net.netfilter.nf_conntrack_tcp_timeout_time_wait
net.netfilter.nf_conntrack_tcp_timeout_time_wait = 120
```
## summary
After setting TCP_NODELAY for the TCP connection, you can disable the Nagle algorithm;

After TCP_CORK is enabled for a TCP connection, small packets can be aggregated into large packets before being sent (note that it will block the sending of small packets);

With SO_SNDBUF and SO_RCVBUF, you can adjust the size of the socket send buffer and receive buffer, respectively.
![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20200101141930.png)  
![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20200101141933.png)  
![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20200101141936.png)
The three values of tcp_rmem and tcp_wmem are min, default, and max respectively. The system will automatically adjust the size of the TCP receive / send buffer according to these settings.

The three values of udp_mem are min, pressure, max. The system will automatically adjust the size of the UDP send buffer according to these settings.

![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20200101215350.png)
# Best Practice 

## docker

```bash
# check running  docker
docker ps

# check logs
docker logs -f [container_id]


# check container config in json format
docker inspect [container_id] -f '{{json .State}}' | jq
{
  "Status": "exited",
  "Running": false,
  "Paused": false,
  "Restarting": false,
  "OOMKilled": true,
  "Dead": false,
  "Pid": 0,
  "ExitCode": 137,
  "Error": "",
  ...
}

# OMM Kill, we can check in host
dmesg
[193038.106393] java invoked oom-killer: gfp_mask=0x14000c0(GFP_KERNEL), nodemask=(null), order=0, oom_score_adj=0
[193038.106396] java cpuset=0f2b3fcdd2578165ea77266cdc7b1ad43e75877b0ac1889ecda30a78cb78bd53 mems_allowed=0
[193038.106402] CPU: 0 PID: 27424 Comm: java Tainted: G  OE    4.15.0-1037 #39-Ubuntu
[193038.106404] Hardware name: Microsoft Corporation Virtual Machine/Virtual Machine, BIOS 090007  06/02/2017
[193038.106405] Call Trace:
[193038.106414]  dump_stack+0x63/0x89
[193038.106419]  dump_header+0x71/0x285
[193038.106422]  oom_kill_process+0x220/0x440
[193038.106424]  out_of_memory+0x2d1/0x4f0
[193038.106429]  mem_cgroup_out_of_memory+0x4b/0x80
[193038.106432]  mem_cgroup_oom_synchronize+0x2e8/0x320
[193038.106435]  ? mem_cgroup_css_online+0x40/0x40
[193038.106437]  pagefault_out_of_memory+0x36/0x7b
[193038.106443]  mm_fault_error+0x90/0x180
[193038.106445]  __do_page_fault+0x4a5/0x4d0
[193038.106448]  do_page_fault+0x2e/0xe0
[193038.106454]  ? page_fault+0x2f/0x50
[193038.106456]  page_fault+0x45/0x50
[193038.106459] RIP: 0033:0x7fa053e5a20d
[193038.106460] RSP: 002b:00007fa0060159e8 EFLAGS: 00010206
[193038.106462] RAX: 0000000000000000 RBX: 00007fa04c4b3000 RCX: 0000000009187440
[193038.106463] RDX: 00000000943aa440 RSI: 0000000000000000 RDI: 000000009b223000
[193038.106464] RBP: 00007fa006015a60 R08: 0000000002000002 R09: 00007fa053d0a8a1
[193038.106465] R10: 00007fa04c018b80 R11: 0000000000000206 R12: 0000000100000768
[193038.106466] R13: 00007fa04c4b3000 R14: 0000000100000768 R15: 0000000010000000
[193038.106468] Task in /docker/0f2b3fcdd2578165ea77266cdc7b1ad43e75877b0ac1889ecda30a78cb78bd53
killed as a result of limit of /docker/
0f2b3fcdd2578165ea77266cdc7b1ad43e75877b0ac1889ecda30a78cb78bd53
[193038.106478] memory: usage 524288kB, limit 524288kB, failcnt 77
[193038.106480] memory+swap: usage 0kB, limit 9007199254740988kB, failcnt 0
[193038.106481] kmem: usage 3708kB, limit 9007199254740988kB, failcnt 0
[193038.106481] Memory cgroup stats for /docker/0f2b3fcdd2578165ea77266cdc7b1ad43e75877b0ac1889ecda30a78cb78bd53: cache:0KB rss:520580KB rss_huge:450560KB shmem:0KB mapped_file:0KB dirty:0KB writeback:0KB inactive_anon:0KB active_anon:520580KB inactive_file:0KB active_file:0KB unevictable:0KB
[193038.106494] [ pid ]   uid  tgid total_vm      rss pgtables_bytes swapents oom_score_adj name
[193038.106571] [27281]     0 27281  1153302   134371  1466368        0             0 java
[193038.106574] Memory cgroup out of memory: Kill process 27281 (java) score 1027 or sacrifice child
[193038.148334] Killed process 27281 (java) total-vm:4613208kB, anon-rss:517316kB, file-rss:20168kB, shmem-rss:0kB
[193039.607503] oom_reaper: reaped process 27281 (java), now anon-rss:0kB, file-rss:0kB, shmem-rss:0kB


# add memory limit
docker run --name tomcat --cpus 0.1 -m 512M -p 8080:8080 -itd feisky/tomcat:8

docker exec tomcat java -XX:+PrintFlagsFinal -version | grep HeapSize
uintx ErgoHeapSizeLimit                         = 0                                   {product}
uintx HeapSizePerGCThread                       = 87241520                            {product}
uintx InitialHeapSize                          := 132120576                           {product}
uintx LargePageHeapSizeThreshold                = 134217728                           {product}
uintx MaxHeapSize                              := 2092957696   


# set env to limit memory
docker run --name tomcat --cpus 0.1 -m 512M -e JAVA_OPTS='-Xmx512m -Xms512m' -p 8080:8080 -itd feisky/tomcat:8

# check thread 
PID=$(docker inspect [container_id] -f '{{.State.Pid}}')
# run pidstat
$ pidstat -t -p $PID 1
12:59:28      UID      TGID       TID    %usr %system  %guest   %wait    %CPU   CPU  Command
12:59:29        0     29850         -   10.00    0.00    0.00    0.00   10.00     0  java
12:59:29        0         -     29850    0.00    0.00    0.00    0.00    0.00     0  |__java
12:59:29        0         -     29897    5.00    1.00    0.00   86.00    6.00     1  |__java
...
12:59:29        0         -     29905    3.00    0.00    0.00   97.00    3.00     0  |__java
12:59:29        0         -     29906    2.00    0.00    0.00   49.00    2.00     1  |__java
12:59:29        0         -     29908    0.00    0.00    0.00   45.00    0.00     0  |__java

# io_wait is high
```
## packet loss
```bash
# check max connection
sysctl net.netfilter.nf_conntrack_max
net.netfilter.nf_conntrack_max = 262144

sysctl net.netfilter.nf_conntrack_count
net.netfilter.nf_conntrack_count = 182

# execute in container
root@nginx:/ iptables -t filter -nvL
Chain INPUT (policy ACCEPT 25 packets, 1000 bytes)
 pkts bytes target     prot opt in     out     source               destination
    6   240 DROP       all  --  *      *       0.0.0.0/0            0.0.0.0/0            statistic mode random probability 0.29999999981
 
Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
 
Chain OUTPUT (policy ACCEPT 15 packets, 660 bytes)
 pkts bytes target     prot opt in     out     source               destination
    6   264 DROP       all  --  *      *       0.0.0.0/0            0.0.0.0/0            statistic mode random probability 0.29999999981

# check MTU
netstat -i
Kernel Interface table
Iface      MTU    RX-OK RX-ERR RX-DRP RX-OVR    TX-OK TX-ERR TX-DRP TX-OVR Flg
eth0       100      157      0    344 0            94      0      0      0 BMRU
lo       65536        0      0      0 0             0      0      0      0 LRU
```
During Linux startup, there are three special processes, that is, the three processes with the smallest PID numbers.

Process 0 is an idle process. This is also the first process created by the system. After initializing processes 1 and 2, it becomes an idle task. It runs when no other tasks are executing on the CPU.

Process 1 is the init process, which is usually the systemd process. It runs in user mode and is used to manage other user mode processes.

Process 2 is a kthreadd process, which runs in kernel mode and is used to manage kernel threads.

```bash

ps -f --ppid 2 -p 2
UID         PID   PPID  C STIME TTY          TIME CMD
root          2      0  0 12:02 ?        00:00:01 [kthreadd]
root          9      2  0 12:02 ?        00:00:21 [ksoftirqd/0]
root         10      2  0 12:02 ?        00:11:47 [rcu_sched]
root         11      2  0 12:02 ?        00:00:18 [migration/0]
...
root      11094      2  0 14:20 ?        00:00:00 [kworker/1:0-eve]
root      11647      2  0 14:27 ?        00:00:00 [kworker/0:2-cgr]

# Kernel thread names (CMD) are in square brackets []
ps -ef | grep "\[.*\]"
root         2     0  0 08:14 ?        00:00:00 [kthreadd]
root         3     2  0 08:14 ?        00:00:00 [rcu_gp]
root         4     2  0 08:14 ?        00:00:00 [rcu_par_gp]
...

# test -S: SYN  -p port, -i interval 10us
hping3 -S -p 80 -i u10 192.168.0.30

# top
top
top - 08:31:43 up 17 min,  1 user,  load average: 0.00, 0.00, 0.02
Tasks: 128 total,   1 running,  69 sleeping,   0 stopped,   0 zombie
%Cpu0  :  0.3 us,  0.3 sy,  0.0 ni, 66.8 id,  0.3 wa,  0.0 hi, 32.4 si,  0.0 st
%Cpu1  :  0.0 us,  0.3 sy,  0.0 ni, 65.2 id,  0.0 wa,  0.0 hi, 34.5 si,  0.0 st
KiB Mem :  8167040 total,  7234236 free,   358976 used,   573828 buff/cache
KiB Swap:        0 total,        0 free,        0 used.  7560460 avail Mem
 
  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
    9 root      20   0       0      0      0 S   7.0  0.0   0:00.48 ksoftirqd/0
   18 root      20   0       0      0      0 S   6.9  0.0   0:00.56 ksoftirqd/1
 2489 root      20   0  876896  38408  21520 S   0.3  0.5   0:01.50 docker-containe
 3008 root      20   0   44536   3936   3304 R   0.3  0.0   0:00.09 top
    1 root      20   0   78116   9000   6432 S   0.0  0.1   0:11.77 systemd


# check ksoftirqd pid is 9
pstack 9
Could not attach to target 9: Operation not permitted.
detach: No such process

cat /proc/9/stack
[<0>] smpboot_thread_fn+0x166/0x170
[<0>] kthread+0x121/0x140
[<0>] ret_from_fork+0x35/0x40
[<0>] 0xffffffffffffffff

# perf 
perf record -a -g -p 9 -- sleep 30
perf report

# install flamegraph
git clone https://github.com/brendangregg/FlameGraph
cd FlameGraph

perf script -i /root/perf.data | ./stackcollapse-perf.pl --all |  ./flamegraph.pl > ksoftirqd.svg

```
## Dynamic Tracing

```bash
cd /sys/kernel/debug/tracing

# if not exist this directory
mount -t debugfs nodev /sys/kernel/debug

cat available_tracers
hwlat blk mmiotrace function_graph wakeup_dl wakeup_rt wakeup function nop

cat available_filter_functions
cat available_events

# test
echo do_sys_open > set_graph_function
echo function_graph > current_tracer
echo funcgraph-proc > trace_options

# enable trace
echo 1 > tracing_on
ls
# disable trace
echo 0 > tracing_on

cat trace
# tracer: function_graph
#
# CPU  TASK/PID         DURATION                  FUNCTION CALLS
# |     |    |           |   |                     |   |   |   |
 0)    ls-12276    |               |  do_sys_open() {
 0)    ls-12276    |               |    getname() {
 0)    ls-12276    |               |      getname_flags() {
 0)    ls-12276    |               |        kmem_cache_alloc() {
 0)    ls-12276    |               |          _cond_resched() {
 0)    ls-12276    |   0.049 us    |            rcu_all_qs();
 0)    ls-12276    |   0.791 us    |          }
 0)    ls-12276    |   0.041 us    |          should_failslab();
 0)    ls-12276    |   0.040 us    |          prefetch_freepointer();
 0)    ls-12276    |   0.039 us    |          memcg_kmem_put_cache();
 0)    ls-12276    |   2.895 us    |        }
 0)    ls-12276    |               |        __check_object_size() {
 0)    ls-12276    |   0.067 us    |          __virt_addr_valid();
 0)    ls-12276    |   0.044 us    |          __check_heap_object();
 0)    ls-12276    |   0.039 us    |          check_stack_object();
 0)    ls-12276    |   1.570 us    |        }
 0)    ls-12276    |   5.790 us    |      }
 0)    ls-12276    |   6.325 us    |    }
...

# Ubuntu
apt-get install trace-cmd
# CentOS
yum install trace-cmd


# above 5 step simplify to follow 1 step
trace-cmd record -p function_graph -g do_sys_open -O funcgraph-proc ls
$ trace-cmd report
...
  ls-12418 [000] 85558.075341: funcgraph_entry:                   |  do_sys_open() {
  ls-12418 [000] 85558.075363: funcgraph_entry:                   |    getname() {
  ls-12418 [000] 85558.075364: funcgraph_entry:                   |      getname_flags() {
  ls-12418 [000] 85558.075364: funcgraph_entry:                   |        kmem_cache_alloc() {
  ls-12418 [000] 85558.075365: funcgraph_entry:                   |          _cond_resched() {
  ls-12418 [000] 85558.075365: funcgraph_entry:        0.074 us   |            rcu_all_qs();
  ls-12418 [000] 85558.075366: funcgraph_exit:         1.143 us   |          }
  ls-12418 [000] 85558.075366: funcgraph_entry:        0.064 us   |          should_failslab();
  ls-12418 [000] 85558.075367: funcgraph_entry:        0.075 us   |          prefetch_freepointer();
  ls-12418 [000] 85558.075368: funcgraph_entry:        0.085 us   |          memcg_kmem_put_cache();
  ls-12418 [000] 85558.075369: funcgraph_exit:         4.447 us   |        }
  ls-12418 [000] 85558.075369: funcgraph_entry:                   |        __check_object_size() {
  ls-12418 [000] 85558.075370: funcgraph_entry:        0.132 us   |          __virt_addr_valid();
  ls-12418 [000] 85558.075370: funcgraph_entry:        0.093 us   |          __check_heap_object();
  ls-12418 [000] 85558.075371: funcgraph_entry:        0.059 us   |          check_stack_object();
  ls-12418 [000] 85558.075372: funcgraph_exit:         2.323 us   |        }
  ls-12418 [000] 85558.075372: funcgraph_exit:         8.411 us   |      }
  ls-12418 [000] 85558.075373: funcgraph_exit:         9.195 us   |    }
...
```

```bash
perf probe --add do_sys_open
Added new event:
  probe:do_sys_open    (on do_sys_open)
You can now use it in all perf tools, such as:
    perf record -e probe:do_sys_open -aR sleep 1


perf probe -V do_sys_open
Available variables at do_sys_open
        @<do_sys_open+0>
                char*   filename
                int     dfd
                int     flags
                struct open_flags     op
                umode_t mode


# delete probe
perf probe --del probe:do_sys_open

perf probe --add 'do_sys_open filename:string'
Added new event:
  probe:do_sys_open    (on do_sys_open with filename:string)
You can now use it in all perf tools, such as:
    perf record -e probe:do_sys_open -aR sleep 1

# run
perf record -e probe:do_sys_open -aR ls
# result
perf script
   perf 13593 [000] 91846.053622: probe:do_sys_open: (ffffffffa807b290) filename_string="/proc/13596/status"
   ls 13596 [000] 91846.053995: probe:do_sys_open: (ffffffffa807b290) filename_string="/etc/ld.so.cache"
   ls 13596 [000] 91846.054011: probe:do_sys_open: (ffffffffa807b290) filename_string="/lib/x86_64-linux-gnu/libselinux.so.1"
   ls 13596 [000] 91846.054066: probe:do_sys_open: (ffffffffa807b290) filename_string="/lib/x86_64-linux-gnu/libc.so.6”
```

```bash
# delete probe before leave
perf probe --del probe:do_sys_open

# starce is based on ptrace
strace ls
...
access("/etc/ld.so.nohwcap", F_OK)      = -1 ENOENT (No such file or directory)
access("/etc/ld.so.preload", R_OK)      = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
...
access("/etc/ld.so.nohwcap", F_OK)      = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libselinux.so.1", O_RDONLY|O_CLOEXEC) = 3
...

# perf trace is light
perf trace ls
         ? (         ): ls/14234  ... [continued]: execve()) = 0
     0.177 ( 0.013 ms): ls/14234 brk(                                                                  ) = 0x555d96be7000
     0.224 ( 0.014 ms): ls/14234 access(filename: 0xad98082                                            ) = -1 ENOENT No such file or directory
     0.248 ( 0.009 ms): ls/14234 access(filename: 0xad9add0, mode: R                                   ) = -1 ENOENT No such file or directory
     0.267 ( 0.012 ms): ls/14234 openat(dfd: CWD, filename: 0xad98428, flags: CLOEXEC                  ) = 3
     0.288 ( 0.009 ms): ls/14234 fstat(fd: 3</usr/lib/locale/C.UTF-8/LC_NAME>, statbuf: 0x7ffd2015f230 ) = 0
     0.305 ( 0.011 ms): ls/14234 mmap(len: 45560, prot: READ, flags: PRIVATE, fd: 3                    ) = 0x7efe0af92000
     0.324 Dockerfile  test.sh
( 0.008 ms): ls/14234 close(fd: 3</usr/lib/locale/C.UTF-8/LC_NAME>                          ) = 0
     ...
```
![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20200101221820.png)

## socket

```bash 
netstat -s | grep socket
    73 resets received for embryonic SYN_RECV sockets
    308582 TCP sockets finished time wait in fast timer
    8 delayed acks further delayed because of locked socket
    290566 times the listen queue of a socket overflowed
    290566 SYNs to LISTEN sockets dropped

ss -ltnp
State     Recv-Q     Send-Q            Local Address:Port            Peer Address:Port
LISTEN    10         10                      0.0.0.0:80                   0.0.0.0:*         users:(("nginx",pid=10491,fd=6),("nginx",pid=10490,fd=6),("nginx",pid=10487,fd=6))
LISTEN    7          10                            *:9000                       *:*         users:(("php-fpm",pid=11084,fd=9),...,("php-fpm",pid=10529,fd=7))

sysctl net.ipv4.ip_local_port_range
net.ipv4.ip_local_port_range=20000 20050

sysctl -w net.ipv4.ip_local_port_range="10000 65535"
net.ipv4.ip_local_port_range = 10000 65535

# timewait is still use port, can decrase timewait time or port reuse
ss -s
TCP:   32775 (estab 1, closed 32768, orphaned 0, synrecv 0, timewait 32768/0), ports 0

sysctl net.ipv4.tcp_tw_reuse
net.ipv4.tcp_tw_reuse = 0

...
```
## monitor
![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20200101222832.png)  
![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20200101222835.png)
![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20200101222947.png)

## learn kernel
1. [kernel](https://elixir.bootlin.com/linux/latest/source)
2. [eBPF](https://github.com/iovisor/bcc)
# Additions
![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20200101224237.png)  
![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20200101224240.png)  
![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20200101224243.png)  
![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20200101224245.png)