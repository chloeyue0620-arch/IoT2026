# 作业二：基于 QEMU 的 OpenWrt 仿真实验报告

姓名：平阅

学号：202312063021

## 一、 实验目的

本实验旨在基于 QEMU 搭建 OpenWrt 仿真环境，在虚拟化条件下运行适用于 ARM 架构的 OpenWrt 镜像，验证系统能否正常启动与运行；在此基础上，通过安装并访问 LuCI 管理界面，验证 OpenWrt 的 Web 服务功能是否正常；同时，利用 SSH 方式远程连接 OpenWrt 系统，验证其远程管理服务的可用性，从而加深对嵌入式 Linux 系统仿真、网络服务配置及远程管理方法的理解。

---

## 二、 实验环境

- 计算机型号：Apple MacBook Air
- 处理器型号：Apple M4
- 内存容量：24 GB
- 宿主机操作系统：macOS Sequoia 15.6.1
- 虚拟机操作系统：Kali Linux
- 虚拟化软件：QEMU

---

## 三、实验步骤

### 1. 镜像下载与准备

新建一个专门存放 OpenWrt 镜像和启动文件的目录，避免后面文件混乱：
```bash
┌──(parallels㉿kali-linux-2024-2)-[~]
└─$ mkdir ~/openwrt                                                                    
                                                                                          
┌──(parallels㉿kali-linux-2024-2)-[~]
└─$ cd ~/openwrt
```

下载 OpenWrt 镜像并解压，接着下载引导文件 u-boot.bin：
```bash
wget https://downloads.openwrt.org/releases/23.05.2/targets/armsr/armv8/openwrt-23.05.2-armsr-armv8-generic-squashfs-combined.img.gz
gzip -d openwrt-23.05.2-armsr-armv8-generic-squashfs-combined.img.gz
wget https://mirror-03.infra.openwrt.org/releases/23.05.2/targets/armsr/armv8/u-boot-qemu_armv8/u-boot.bin
```

![alt text](image/3.1.1.png)

检查文件是否下载成功:
```bash
┌──(parallels㉿kali-linux-2024-2)-[~/openwrt]
└─$ ls -lh
```
![alt text](image/3.1.2.png)

✅ 镜像准备成功

### 2. QEMU 环境配置

先检查 QEMU 是否安装成功：
```bash
qemu-system-aarch64 --version
```
![alt text](image/3.2.1.png)

终端能显示版本号，可以继续。

启动 OpenWrt 虚拟机：
```bash
qemu-system-aarch64 -cpu cortex-a72 -m 1024 -M virt,highmem=off -nographic \
-bios u-boot.bin \
-drive file=openwrt-23.05.2-armsr-armv8-generic-squashfs-combined.img,format=raw,if=virtio \
-device virtio-net,netdev=net0 -netdev user,id=net0,net=192.168.1.0/24,hostfwd=tcp:127.0.0.1:1122-192.168.1.1:22,hostfwd=tcp:127.0.0.1:8080-192.168.1.1:80 \
-device virtio-net,netdev=net1 -netdev user,id=net1,net=192.168.2.0/24
```

上述命令参数简要说明：
- `qemu-system-aarch64`：启动 AArch64（ARMv8 64 位）虚拟机。
- `-cpu cortex-a72`：指定虚拟 CPU 型号为 Cortex-A72。
- `-m 1024`：分配 1024 MB 内存。
- `-M virt,highmem=off`：使用通用 `virt` 机器类型；关闭 highmem（减少地址空间/设备映射复杂度，提升兼容性）。
- `-nographic`：不启用图形界面，串口输出直接显示在当前终端。
- `-bios u-boot.bin`：指定引导固件（这里用 U-Boot）用于启动系统。
- `-drive file=...,format=raw,if=virtio`：把 OpenWrt 镜像作为磁盘挂载；`raw` 为镜像格式；`virtio` 为高性能虚拟磁盘接口。
- `-netdev user,id=net0,net=192.168.1.0/24`：创建一个用户态网络后端（NAT），并指定虚拟网段。
- `-device virtio-net,netdev=net0`：添加一块 virtio 网卡并连接到 `net0`。
- `hostfwd=tcp:127.0.0.1:1122-192.168.1.1:22`：把宿主机 `127.0.0.1:1122` 转发到虚拟机 `192.168.1.1:22`（SSH）。
- `hostfwd=tcp:127.0.0.1:8080-192.168.1.1:80`：把宿主机 `127.0.0.1:8080` 转发到虚拟机 `192.168.1.1:80`（Web/LuCI）。
- 第二组 `-netdev user,id=net1,net=192.168.2.0/24` + `-device virtio-net,netdev=net1`：再添加一套用户态网络与第二块网卡（第二个虚拟网段）。

![alt text](image/3.2.2.png)

✅ OpenWrt 已经启动成功

### 3. 启动网页界面

安装 LuCI 网页管理界面：
```bash
opkg update
opkg install luci
```
![alt text](image/3.3.1.png)

检查网页服务是否已经起来:
```bash
netstat -tupan
```
可以看到 `uhttpd` 已经启动，80 端口正常监听，网页界面可以访问：
![alt text](image/3.3.2.png)

在 Kali 的浏览器 里访问：http://127.0.0.1:8080，可以看到 OpenWrt 的 LuCI 登录页面：
![alt text](image/3.3.3.png)

### 4. ssh 连接

重新启动 OpenWrt 系统中的 SSH 服务：
```bash
/etc/init.d/dropbear restart
```
在 Kali 里新开一个终端窗口，输入：
```bash
ssh root@127.0.0.1 -p 1122
```

由于是第一次连接，中间过程需要输入 yes

![alt text](image/3.4.png)

由上图可以看出SSH 连接成功

## 四、 实验总结

本次实验成功在 QEMU 上搭建了 ARMv8 架构的 OpenWrt 仿真环境，能够通过 U-Boot 正常引导系统并进入控制台，说明镜像与虚拟硬件配置匹配正确。在网络配置方面，通过 `user` 模式网络与端口转发实现了宿主机到虚拟机的访问：一方面可在浏览器通过 `127.0.0.1:8080` 打开 LuCI 管理界面，验证了 `uhttpd` Web 服务工作正常；另一方面可使用 `ssh root@127.0.0.1 -p 1122` 远程登录，验证了 Dropbear SSH 服务可用。

实验过程中加深了对 QEMU 关键参数（机器类型、CPU/内存、virtio 设备、网络与端口映射）的理解，也体会到在无图形界面（`-nographic`）条件下通过串口控制台进行系统管理的常见方式。总体而言，本实验完成了 OpenWrt 启动、Web 管理与远程登录三项核心验证，为后续在仿真环境中开展网络配置与服务部署实验打下了基础。