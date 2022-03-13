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

