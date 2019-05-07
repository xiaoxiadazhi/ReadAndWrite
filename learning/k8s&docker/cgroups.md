## cgroups

---

### What are cgroups?

```
cgroup是内核的一部分，是一种分配系统资源的机制。它以分层的方式组织进程，并且沿着层级结构来分配资源。
cgroup只要由两部分组成：
- core
	- 核心部分主要负责分层组织进程
- controllers
	- cgroup控制器通常负责沿着层次结构分发特定类型的系统资源

* cgroup = control group
* Linux上的资源管理系统
* 目录层级在/sys/fs/cgroup
```



### cgroup v1

### cgroup v2

### v1 VS v2

### v2改进的点

page cache统计