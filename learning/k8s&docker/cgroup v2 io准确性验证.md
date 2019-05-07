### 背景

​	验证cgroup v2对普通进程io的统计是否正确



### 命令

​	dd if=/dev/zero of=/xfs/file1 bs=128M count=1



### 基本信息

- 10G普通云硬盘

- 操作系统与内核版本：Linux version      4.14.15-041415-generic (kernel@tangerine) (gcc version 7.2.0 (Ubuntu      7.2.0-8ubuntu3)) #201801231530 SMP Tue Jan 23 20:33:21 UTC 2018

- Cgroup v2的io.stat如下所示

  ```shell
  root@VM-16-35-ubuntu:/xfs/cgroup2/cg3# cat io.stat 
  252:16 rbytes=0 wbytes=184549376 rios=0 wios=128
  ```



### 使用Cgroup v2的步骤

- 禁用cgroup v1

- - 在/boot/grub/grub.cfg添加内核启动参数：cgroup_no_v1=all，如下所示

  - reboot

    ```shell
    linux   /boot/vmlinuz-4.14.15-041415-generic root=UUID=971546b4-fe6b-4f81-9cbb-9186ff0454ea ro net.ifnames=0 biosdevname=0 console=ttyS0,115200 console=tty0 panic=5 crashkernel=auto  crashkernel=384M-:128M cgroup_no_v1=all
    ```

- 挂载cgroup v2

  - 为了不受其他进程影响，我申请了一块新盘，/xfs是挂载点
  - mount -t cgroup2 nodev       /xfs/cgroup2

- 使用I/O controller

  - 创建新cgroup

    ```shell
    mkdir /xfs/cgroup2/cg3
    ```

  - 为了可以在新创建的cgroup上使用I/O controller来编辑I/O限制，需要在subtree_control中写上io

    ```shell
    echo "+io" > /xfs/cgroup2/cgroup.subtree_control
    ```

  - 限制I/O（8:0是磁盘的主次设备号,1048576代表限制I/O为1MB/s）

    ```shell
    echo "8:0 wbps=1048576" > /xfs/cgroup2/cg3/io.max
    ```

  - 把bash session添加到cg3这个cgroup

    ```shell
    echo $$ > /cgroup2/cg2/cgroup.procs
    ```

  - 使用`dd`来生成一些I/O负载

    ```shell
    dd if=/dev/zero of=/tmp/file1 bs=512M count=1
    ```



### 结论

- 三个问题
  1. 使用dd写新文件时，wbytes不增加
  2. 在未加入cgroup的进程写的同时，使用dd写时（无论新旧文件），wbytes不增加
  3. 每次使用dd命令写128M数据时，wbytes增加184549376
- 正确的地方

  - 每次使用dd写128M到已有文件时，wbytes增加固定值