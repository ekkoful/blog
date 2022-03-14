# 前言
《Kubernetes 网络权威指南》--杜军著

觉得还是需要多研究研究 Kubernetes 网络相关的知识，所以看了这本书，并记录相关笔记。以便后续学习，总结。

# 第一章--Linux网络虚拟化
这一章作为本书的第一章，也是吸引我的原因，因为 Kubernetes 网络，本质上就是 Linux 虚拟网络，所以在第一章，熟悉和学习 Linux 虚拟网络相关的知识，便于后续的章节的阅读和学习。
## 1.1 网络虚拟化的基石：Network namespace
Linux 的 namespace 组要作用是隔离内核资源，对于进程来说，如果想要使用 namespace 里面的资源，首先要进入到这个 namespace 之中，而且无法跨 namespace 访问资源，Linux 的 namespace 给里面的进程造成了两个错觉，1：它是系统中的唯一进程，2：它独享系统的所有资源。  
- 回顾知识点1: Linux 操作系统中7个 namespace，可以使用 `man namespaces` 查看相关含义，Linux中这7个 namesapce 本质是 docker 实现的主要原理，同样的 Kubernetes 中的 Pod 通过对这些 namespace 进行不同的程度的共享和完全隔离，来实现 Pod 中容器能够共享网络等主要功能。
```bash
# man namespaces
Namespace Types:
Namespace Flag            Page                  Isolates
Cgroup    CLONE_NEWCGROUP cgroup_namespaces(7)  Cgroup root directory
IPC       CLONE_NEWIPC    ipc_namespaces(7)     System V IPC,
                                                POSIX message queues
Network   CLONE_NEWNET    network_namespaces(7) Network devices,
                                                stacks, ports, etc.
Mount     CLONE_NEWNS     mount_namespaces(7)   Mount points
PID       CLONE_NEWPID    pid_namespaces(7)     Process IDs
User      CLONE_NEWUSER   user_namespaces(7)    User and group IDs
UTS       CLONE_NEWUTS    uts_namespaces(7)     Hostname and NIS
                                                domain name
```
### 1.1.1 初识 network namespace
network namespace 通过系统调用来实现，在使用 Linux 中 clone() 系统调用的时候，传入 CLONE_NEWNET 参数创建一个network namespace。与其他 namespace 需要通过代码调用系统调用 API不同，network namespace的增删改查功能已经集成到了 Linux 的 `ip` 命令的子命令 `netns` 命令中。
下面是简单的命令使用
```bash
# 创建一个名为 netns1 的 network namespace
# ip netns add netns1
# 进入一个 network namespace
# ip netns exec netns1 ip link list
# 列出系统中的 network namespace
# ip netns list
# 删除 network namespace
# ip netns delete netns1
```
- 当 ip 命令创建了一个 network namespace 时，系统会在 `/var/run/netns` 路径下面生成一个挂载点。挂载点的作用一方面时方便对 namespace 进行管理，另一方面是使 namespace 即使没有进程运行也能继续存在。
- `ip netns delete netns1` 同样也没有删除 netns1 这个 network namespace，它知识移除了这个 network namespace 对应的挂载点。只要里面还有进程在运行着，network namespace 便会一直存在。
### 1.1.2 配置 network namespace
一个全新的 namespace 会附带创建一个本地回环地址，除此之外，这个新的 network namespace 没有任何其他的网络设备，而且这个 network namespace 自带的 lo 设备状态还是 DOWN 未开启的。
但是如果想要和外界进行通信，就需要在 network namespace 再创建一堆虚拟的网卡，即所谓的 veth pair。veth pair 总是成对出现的且相互连接的，就像 Linux 系统的双向管道，报文从 veth pair 一端进去就会从另外一端收到。
可以使用以下命令进行 network namespace 的一些配置
```bash
# 创建 veth0 和 veth1 一对网卡
[root@localhost ~]# ip link add veth0 type veth peer name veth1

# 将 veth1 移动到 netns1 这个 network namespace 中
[root@localhost ~]# ip link set veth1 netns netns1

# 查看 ip 地址，可以看到 veth0 没有地址和状态为 DOWN
[root@localhost ~]# ip add list
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 52:54:00:4d:77:d3 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global noprefixroute dynamic eth0
       valid_lft 85190sec preferred_lft 85190sec
    inet6 fe80::5054:ff:fe4d:77d3/64 scope link 
       valid_lft forever preferred_lft forever
4: veth0@if3: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 3a:99:ee:7d:bd:0e brd ff:ff:ff:ff:ff:ff link-netnsid 0

# 查看 netns1 network namespace 的网络 link
[root@localhost ~]# ip netns exec netns1 ip link list
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
3: veth1@if4: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 0e:2e:00:5f:ea:b0 brd ff:ff:ff:ff:ff:ff link-netnsid 0

# 配置 netns1 network namespace veth1 网卡状态 up
[root@localhost ~]# ip netns exec netns1 ip link set veth1 up
[root@localhost ~]# ip netns exec netns1 ip link list
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
3: veth1@if4: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state LOWERLAYERDOWN mode DEFAULT group default qlen 1000
    link/ether 0e:2e:00:5f:ea:b0 brd ff:ff:ff:ff:ff:ff link-netnsid 0

# 配置 netns1 network namespace veth1 网卡网卡地址
[root@localhost ~]# ip netns exec netns1 ip addr add 10.1.1.1/24 dev veth1
[root@localhost ~]# ip netns exec netns1 ip addr list
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
3: veth1@if4: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state LOWERLAYERDOWN group default qlen 1000
    link/ether 0e:2e:00:5f:ea:b0 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.1.1.1/24 scope global veth1
       valid_lft forever preferred_lft forever

# 查看本机 network namespace 网卡的 ip 地址
[root@localhost ~]# ip addr list
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 52:54:00:4d:77:d3 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global noprefixroute dynamic eth0
       valid_lft 78824sec preferred_lft 78824sec
    inet6 fe80::5054:ff:fe4d:77d3/64 scope link 
       valid_lft forever preferred_lft forever
4: veth0@if3: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 3a:99:ee:7d:bd:0e brd ff:ff:ff:ff:ff:ff link-netnsid 0

# 配置本机 network namespace veth0 网卡 ip 地址
[root@localhost ~]# ip addr add 10.1.1.2/24 dev veth0
[root@localhost ~]# ip addr list
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 52:54:00:4d:77:d3 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global noprefixroute dynamic eth0
       valid_lft 78784sec preferred_lft 78784sec
    inet6 fe80::5054:ff:fe4d:77d3/64 scope link 
       valid_lft forever preferred_lft forever
4: veth0@if3: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 3a:99:ee:7d:bd:0e brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.1.1.2/24 scope global veth0
       valid_lft forever preferred_lft forever

# 配置本机 network namespace veth0 网卡状态 up
[root@localhost ~]# ip link set veth0 up

# 测试 ping netns1 network namespace 下面 veth1 网卡的 ip 地址 
[root@localhost ~]# ping 10.1.1.1
PING 10.1.1.1 (10.1.1.1) 56(84) bytes of data.
64 bytes from 10.1.1.1: icmp_seq=1 ttl=64 time=0.101 ms
64 bytes from 10.1.1.1: icmp_seq=2 ttl=64 time=0.027 ms
````
