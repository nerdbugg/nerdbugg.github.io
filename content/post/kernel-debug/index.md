---
title: "Linux Kernel Debug环境搭建"
description: 
date: 2024-04-29T19:51:11+08:00
image: 
math: 
license: 
comments: true
categories:
    - Kernel
tags:
    - Kernel
weight: 1
---
## 前言

构建调试环境可以帮助程序员更好的了解程序运行时的行为，分析清楚模块间的交互关系。本文将使用QEMU+GDB的组合构建一个可以调试内核的环境，并且被调试的内核在虚拟机中可正常执行指令等操作。

本环境搭建流程主要特点是：不依赖GUI，可在ssh会话中搭建；被调试的虚拟机包含完整发行版和系统，可安装包可执行指令。

在此对环境搭建步骤进行记录，方面后续使用，也希望能帮到需要类似环境的同学。

## 方案概览

在普通的用户态代码调试中，GDB进程在调用ptrace系统调用Attach至被调试进程后，由Linux内核中ptrace系统调用的实现支撑GDB读写被调试进程的内存，修改被调试进程的寄存器等操作，对用户提供断点、观察变量值这些操作。

那么自然，在对内核的调试中，同样需要一个更高级的组件能够观察修改内核的状态，并将其提供给GDB。据笔者了解到的可选项包含QEMU模拟器提供的GDB Stub，以及Linux内核提供的KGDB两种选项。

- QEMU模拟器提供GDB Stub，由QEMU Hypervisor对虚拟机内的内核状态进行监控与修改，并通过TCP连接对外暴露服务
- KGDB是Linux内核中的模块，可通过串口等方式对外暴露GDB Server服务

本文中将以QEMU+GDB方式搭建环境，KGDB方式可参考[相关资料](https://sergioprado.blog/debugging-the-linux-kernel-with-gdb/)。

## 方案步骤

### 设置虚拟机

由于笔者使用的环境为ssh会话，不提供GUI显示，以下的步骤均在该环境下进行。

通常使用中断提供显示界面时，需要为Linux内核添加 `console=ttyS0` 参数，使内核向该tty设备中输出中断字节流，并由当前中断连接该设备，提供命令行界面。

在大多数的发行版安装ISO中，内核并未配置相关参数。因此，必须借助GUI或者VNC等工具完成系统安装。

但是，Debian发行版提供了单独的内核与ISO的安装介质，因此可以在安装过程中配置内核参数，从而可通过命令行安装系统。

使用virt-install工具完成虚拟机操作系统安装，命令如下（参考[KVM - Debian Wiki](https://wiki.debian.org/KVM#Creating_a_new_guest)）：

```bash
virt-install --virt-type kvm --name bookworm-amd64 \
--location https://deb.debian.org/debian/dists/bookworm/main/installer-amd64/ \
--os-variant debian12 \
--disk size=10 --memory 1024 \
--graphics none \
--console pty,target_type=serial \
--extra-args "console=ttyS0"
```

完成安装后，可通过 `virsh start bookworm-amd64` 命令启动虚拟机，使用  `virtsh console bookworm-amd64` 命令连接虚拟机。

为了进一步简化操作，便于后续设置Qemu GDB参数，编写合适的qemu命令行来启动该虚拟机。一个例子如下：

```bash
# IMAGE_PATH=path/to/vm_disk_image
qemu-system-x86_64 \
  --enable-kvm \
  -m 2048 \
  -smp 2 \
  -drive file=$IMAGE_PATH,format=qcow2 \
  -netdev user,id=net0,ipv6=off,hostfwd=tcp::8022-:22 -device virtio-net-pci,netdev=net0 \
  -nographic
```

该例子中仅为虚拟机配置磁盘与网络两种设备，并且网络使用最简单的User-mode-Networking，配置了主机8022至虚拟机22的端口转发用于ssh登录。

### 编译内核

在Debian发行版中， `/boot` 目录下会保存当前内核的config、内核、以及init ramfs。应尽可能使用相近的内核配置，避免虚拟机内配置不兼容出现问题。

在下载对应版本内核源码后，通过上一步的ssh连接将虚拟机内的内核config拷贝下来，并打开 `CONFIG_GDB_SCRIPTS` 配置，关闭 `CONFIG_DEBUG_INFO_REDUCED` 配置。

命令如下：

```bash
# 使用12线程编译，并通过bear生成compile_commands.json便于后续IDE识别
# 由于包含了很多驱动程序模块，时间在半小时-1小时不等
bear -- make -j12
```

在完成编译后，得到被压缩的内核文件： `./arch/x86/boot/bzImage` ，原始内核文件： `./vmlinux`

使用 `sshfs` 将虚拟机内的根文件目录挂载至某一文件夹下（ssh连接使用root用户），例如 `~/vm` 。

下面需要将编译好的内核模块安装至虚拟机中，命令如下：

```bash
# ~/vm 为sshfs挂载的虚拟机根目录
make modules_install INSTALL_MOD_PATH=~/vm
```

最后，将内核编译使用的配置文件拷贝至虚拟机 `/boot` 目录下。并通过Debian中的 `update-initramfs` 命令根据配置生成iniframfs文件。

```bash
update-initramfs -c -k 6.1.0
```

这里的6.1.0为内核的后缀，可在编译时的config中进行设置，内核加载目录、生成initramfs的命令行中均应保持一致。

最后，将内核文件、initramfs拷出备用。

### 使用编译内核启动虚拟机

最后一次使用虚拟机的原始配置登陆虚拟机，通过 `cat /proc/cmdline` 命令获取虚拟机内核启动参数。

使用上一步得到的内核文件、initramfs启动虚拟机。

```bash
qemu-system-x86_64 \
  --enable-kvm \
  -m 2048 \
  -smp 2 \
  -kernel $KERNEL_PATH \
  -initrd $INITRD_PATH \
  -append "root=UUID=6b911d85-9296-46bc-b71d-0facb65f92a2 rw console=ttyS0 nokaslr"\
  -drive file=$IMAGE_PATH,format=qcow2 \
  -netdev user,id=net0,ipv6=off,hostfwd=tcp::8022-:22 -device virtio-net-pci,netdev=net0 \
  -nographic 

```

这里的 `-append` 选项即为上述得到的内核参数，在后续添加 `console=ttyS0` 参数来在终端访问， `nokaslr` 参数用来避免断点失效。

### 配置GDB

最后，配置QEMU gdb stub选项，启动虚拟机。

```bash
qemu-system-x86_64 \
  --enable-kvm \
  -m 2048 \
  -smp 2 \
  -kernel $KERNEL_PATH \
  -initrd $INITRD_PATH \
  -append "root=UUID=6b911d85-9296-46bc-b71d-0facb65f92a2 rw console=ttyS0 nokaslr"\
  -drive file=$IMAGE_PATH,format=qcow2 \
  -netdev user,id=net0,ipv6=off,hostfwd=tcp::8022-:22 -device virtio-net-pci,netdev=net0 \
  -nographic \
  -S -gdb tcp::26002
```

这里最后一行参数指定QEMU提供的gdb server监听26002端口，可使用gdb连接该端口进行调试。

一个样例如下：

```bash
# under kernel source root
gdb vmlinux
```

```bash
# enter gdb shell
(gdb) target remote :26002
```

完成上述步骤后，gdb在连接至gdb server后应当已经处于中断执行的状态，可通过 `hbreak` 指令对内核符号下断点进行调试。（使用 `break` 指令会提示无法访问地址）

至此，一个纯命令行环境下的Linux内核调试环境就已经搭建好了。

如果有GUI环境的话，为虚拟机安装操作系统的步骤会简化很多，使用Libvirt的虚拟机管理器图形界面即可。

## 参考资料

- [KVM - Debian Wiki](https://wiki.debian.org/KVM#Creating_a_new_guest)
- [Kernel/Traditional compilation - Arch Wiki](https://wiki.archlinux.org/title/Kernel/Traditional_compilation)
- [Debugging kernel and modules via gdb — The Linux Kernel documentation](https://www.kernel.org/doc/html/v4.14/dev-tools/gdb-kernel-debugging.html)
- [Booting a Custom Linux Kernel in QEMU and Debugging It With GDB](https://nickdesaulniers.github.io/blog/2018/10/24/booting-a-custom-linux-kernel-in-qemu-and-debugging-it-with-gdb/)
- [Debugging the Linux kernel with GDB](https://sergioprado.blog/debugging-the-linux-kernel-with-gdb/)
