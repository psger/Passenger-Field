### 如何用 Cgroup 来限制一个进程的 CPU 资源
#### Linux Cgroups 的全称是 Linux Control Group。它最主要的作用，就是限制一个进程组能够使用的资源上限，包括 CPU、内存、磁盘、网络带宽等等。

- Cgroup 暴露给用户的操作接口是文件系统，各种资源的限制在 `/sys/fs/cgroup/` 目录下以文件的形式呈现
```shell
➜  ~ mount -t cgroup
cgroup on /sys/fs/cgroup/systemd type cgroup (rw,nosuid,nodev,noexec,relatime,xattr,name=systemd)
cgroup on /sys/fs/cgroup/cpu,cpuacct type cgroup (rw,nosuid,nodev,noexec,relatime,cpu,cpuacct)
cgroup on /sys/fs/cgroup/pids type cgroup (rw,nosuid,nodev,noexec,relatime,pids)
cgroup on /sys/fs/cgroup/cpuset type cgroup (rw,nosuid,nodev,noexec,relatime,cpuset)
cgroup on /sys/fs/cgroup/freezer type cgroup (rw,nosuid,nodev,noexec,relatime,freezer)
cgroup on /sys/fs/cgroup/perf_event type cgroup (rw,nosuid,nodev,noexec,relatime,perf_event)
cgroup on /sys/fs/cgroup/net_cls,net_prio type cgroup (rw,nosuid,nodev,noexec,relatime,net_cls,net_prio)
cgroup on /sys/fs/cgroup/hugetlb type cgroup (rw,nosuid,nodev,noexec,relatime,hugetlb)
cgroup on /sys/fs/cgroup/memory type cgroup (rw,nosuid,nodev,noexec,relatime,memory)
cgroup on /sys/fs/cgroup/devices type cgroup (rw,nosuid,nodev,noexec,relatime,devices)
cgroup on /sys/fs/cgroup/blkio type cgroup (rw,nosuid,nodev,noexec,relatime,blkio)
cgroup on /sys/fs/cgroup/rdma type cgroup (rw,nosuid,nodev,noexec,relatime,rdma)
```
- 查看 CPU 资源的限制
```shell
levi-lei# pwd
/sys/fs/cgroup/cpu
levi-lei# ls
cgroup.clone_children  cpuacct.usage_percpu_sys   docker
cgroup.procs	       cpuacct.usage_percpu_user  notify_on_release
cgroup.sane_behavior   cpuacct.usage_sys	  release_agent
container	       cpuacct.usage_user	  system.slice
cpuacct.stat	       cpu.cfs_period_us	  tasks
cpuacct.usage	       cpu.cfs_quota_us		  user.slice
cpuacct.usage_all      cpu.shares
cpuacct.usage_percpu   cpu.stat
```
关注三个文件
```shell
// 配额为 -1 表示还没有限制
levi-lei# cat cpu.cfs_quota_us
-1
// CPU period 则是默认的 100 ms（100000 us）
levi-lei# cat cpu.cfs_period_us
100000
// tasks 表示受限制的进程 ID
levi-lei# cat tasks 
1
2
3
4
```
#### 使用 Cgroup 来限制一个进程的 CPU 资源
- 制造一个死循环把 CPU 打满
```shell
➜  cpu,cpuacct while : ; do : ; done &
[2] 10618
```
可以看到进程 10618 的 CPU 使用率已经达到 100% 了
```shell
➜  cpu,cpuacct top

top - 17:05:31 up 15 days,  8:49,  1 user,  load average: 1.58, 2.09,
Tasks: 753 total,   3 running, 690 sleeping,   0 stopped,   2 zombie
%Cpu(s):  3.2 us,  7.2 sy, 14.3 ni, 72.0 id,  0.1 wa,  0.0 hi,  3.3 s
KiB Mem : 16265232 total,   798992 free, 10586072 used,  4880168 buff
KiB Swap:  2097148 total,   919912 free,  1177236 used.  4147868 avai

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ 
10618 user      25   5  176656   3016      0 R 100.0  0.0   0:26.91 
12313 user      25   5  176656   3016      0 R  19.6  0.0   7:53.49 
```
- 在 `/sys/fs/cgroup/cpu/` 下创建一个新的文件夹 `container`
```shell
mkdir container
levi-lei# pwd
/sys/fs/cgroup/cpu/container
levi-lei# ls
cgroup.clone_children  cpuacct.usage_percpu_sys   cpu.shares
cgroup.procs	       cpuacct.usage_percpu_user  cpu.stat
cpuacct.stat	       cpuacct.usage_sys	  notify_on_release
cpuacct.usage	       cpuacct.usage_user	  tasks
cpuacct.usage_all      cpu.cfs_period_us
cpuacct.usage_percpu   cpu.cfs_quota_us
```
这个目录就称为一个“控制组”。你会发现，操作系统会在你新创建的 container 目录下，自动生成该子系统对应的资源限制文件。
- 增大使用 CPU 的时间间隔
```shell
echo 20000 > /sys/fs/cgroup/cpu/container/cpu.cfs_quota_us
```
- 将进程加入到限制组中
```shell
echo 10618 > /sys/fs/cgroup/cpu/container/tasks
```
- 在次查看该进程的 CPU 使用率
```shell
28442 systemd+  20   0   67016  39404   8616 S  13.1  0.2 365:09.13 
10618 user      25   5  176656   3016      0 R  10.2  0.0   3:43.42
```