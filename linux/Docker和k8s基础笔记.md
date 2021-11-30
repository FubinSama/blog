# Docker和k8s基础冲刺班笔记

## Docker

### Namespace

`Linux Namespace`是一种`Linux Kernel`提供的资源隔离方案：系统可以为进程分配不同的Namespace，并保证不同的`Namespace`资源独立分配、进程彼此隔离，即不同的Namespace下的进程互不干扰。

```c
// 进程的数据结构
struct task_struct {
    // ...
    /* namespaces */
    struct nsproxy *nsproxy;
}
// Namespace数据结构
struct nsproxy {
    atomic_t count;
    struct uts_namespace *uts_ns;
    struct ipc_namespace *ipc_ns;
    struct mnt_namespace *mnt_ns;
    struct pid_namespace *pid_ns_for_children;
    struct net *net_ns;
}
```
