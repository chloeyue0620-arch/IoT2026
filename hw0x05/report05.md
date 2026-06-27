# 实验五：固件 Patch 与动态调试实验报告

## 一、实验目的

- 掌握固件 Patch 的基本方法，理解通过修改二进制指令绕过异常逻辑的技术思路。
- 以 Tenda AC9 原始固件为对象，完成固件下载、解包、文件系统提取和用户态仿真。
- 观察原始未 Patch 固件在 QEMU 用户态仿真中的“假死”或 Web 无法访问现象，并分析问题原因。
- 使用 Ghidra 定位并修改 `httpd` 中的关键指令，使固件 Web 服务能够在仿真环境中正常启动。
- 结合汇编代码和反编译伪代码分析 Patch 原理，理解固件初始化阶段硬件检测逻辑对仿真的影响。

## 二、实验环境

| 项目 | 配置 |
|------|------|
| 操作系统 | Ubuntu 虚拟机 |
| 固件对象 | Tenda AC9 原始固件 |
| 固件版本 | `ac9_kf_V15.03.05.19(6318_)_cn` |
| 目标程序 | `bin/httpd` |
| 解包工具 | binwalk |
| 仿真工具 | QEMU user、QEMU user static、QEMU system arm、qemu-utils |
| 网络工具 | bridge-utils、uml-utilities、ifupdown、net-tools |
| Patch 工具 | Ghidra |
| Web 访问地址 | `http://10.0.2.16/main.html` |

## 三、实验原理与基础知识

### （一）用户态固件仿真与“假死”问题

用户态固件仿真是在宿主机系统中直接运行固件文件系统里的用户态程序。例如，本实验中的 Tenda AC9 Web 服务程序 `bin/httpd` 是 ARM 架构 ELF 文件，需要借助 `qemu-arm` 或 `qemu-arm-static` 在 Ubuntu 虚拟机中运行。

原始路由器固件通常假设自己运行在真实硬件设备上，启动过程中会访问硬件状态、NVRAM 配置、内部守护进程或厂商自定义服务。当这些资源在 QEMU 用户态环境中不存在时，程序可能一直等待、连接失败或进入异常分支，表现为终端卡住、报错、Web 页面无法访问，即实验手册中所说的“假死”现象。

### （二）二进制 Patch 原理

二进制 Patch 是指不重新编译源代码，而是直接修改可执行文件中的机器指令。本实验使用 Ghidra 对 `bin/httpd` 进行静态分析，并在地址 `0x0002E47C` 修改 ARM 指令：

```asm
cpy r3,r0
```

修改为：

```asm
MOV R3, 1
```

对应机器码为：

```text
01 30 A0 E3
```

该修改将原本依赖函数返回值的判断结果固定为成功值 `1`，使程序跳过仿真环境中无法通过的硬件连接检测逻辑，从而继续执行 Web 服务初始化流程。

### （三）Ghidra Patch 流程

Ghidra 可以导入固件中的 ARM ELF 程序，完成自动分析、反汇编、反编译和指令修改。常见流程如下：

- 导入目标二进制文件 `bin/httpd`。
- 执行自动分析，识别函数、字符串和交叉引用。
- 跳转到手册指定地址 `0x0002E47C`。
- 使用 `Patch Instruction` 修改汇编指令。
- 以 `Original File` 格式导出 Patch 后的二进制文件。
- 将 Patch 后的文件替换回固件文件系统，并重新运行验证效果。

## 四、实验内容

### （一）仿真原始 Tenda AC9 固件

#### 1. 实验背景

上一实验使用的是经过特殊处理的 Tenda AC9 固件，可以较顺利地完成用户态仿真。本次实验改为运行原始未 Patch 固件，目的是观察其在仿真环境中的异常表现，并说明后续 Patch 的必要性。

原始固件在 QEMU 用户态环境中容易出现问题，主要原因包括：

- 固件启动时会执行真实路由器上的硬件初始化逻辑。
- 仿真环境中不存在对应硬件设备或厂商内部服务。
- `httpd` 访问这些资源时可能阻塞或报错。
- 最终表现为程序卡住、报错或 Web 页面无法访问。

#### 2. 准备实验目录

将下载得到的 Tenda AC9 原始固件放入实验目录：

```bash
mkdir -p ~/iot_0x05
cd ~/iot_0x05
```

#### 3. 安装相关工具

由于当前 Ubuntu 软件源中没有单独可用的旧式 `qemu` 总包，因此安装 QEMU 具体组件和网络工具：

```bash
sudo apt update
sudo apt install -y binwalk qemu-user qemu-user-static qemu-system-arm qemu-utils bridge-utils uml-utilities ifupdown net-tools
```

工具作用如下：

- `binwalk`：提取固件中的 SquashFS 文件系统。
- `qemu-user`、`qemu-user-static`：在宿主机上运行 ARM 用户态程序。
- `qemu-system-arm`、`qemu-utils`：提供 ARM 仿真和镜像处理相关组件。
- `bridge-utils`、`uml-utilities`、`ifupdown`：配置 QEMU 和固件 Web 服务访问所需网络环境。
- `net-tools`：使用 `netstat` 等命令检查端口和网络状态。

#### 4. 提取固件文件系统

使用 `binwalk -Me` 递归提取固件：

```bash
binwalk -Me ./ac9_kf_V15.03.05.19\(6318_\)_cn.zip
find . -maxdepth 4 -type d -name "squashfs-root"
```

![固件解包过程](image/1-0.png)

截图显示 binwalk 正在识别并提取固件中的压缩文件和文件系统内容，这是后续运行 `httpd` 的基础。

![提取出的 squashfs-root 目录](image/1-1.png)

截图显示已经成功找到 `squashfs-root` 目录，说明固件文件系统提取完成，可以进入该目录准备用户态仿真。

#### 5. 检查端口占用

启动 Web 服务前，先检查 80 端口是否被占用：

```bash
sudo netstat -tulnp | grep ':80'
```

本次实验未发现明显端口冲突，因此继续进行网络配置。如果 80 端口已被其他服务占用，需要先停止对应进程或改用其他端口。

#### 6. 配置仿真网络

通过 `ip a` 查看当前虚拟机网卡名称：

```bash
ip a
```

![查看网卡名称](image/1-3-8.png)

截图显示当前主要网卡为 `enp0s3`。因此配置桥接网络时，需要将实验手册中的示例网卡名调整为本机实际网卡 `enp0s3`。

编辑 `/etc/network/interfaces`：

```bash
sudo nano /etc/network/interfaces
```

写入如下内容：

```text
auto lo
iface lo inet loopback

auto enp0s3
iface enp0s3 inet dhcp

auto br0
iface br0 inet dhcp
bridge_ports enp0s3
bridge_maxwait 0
```

![配置 interfaces 文件](image/1-3-1.png)

截图展示了 `br0` 桥接接口配置，其中 `bridge_ports enp0s3` 表示将实际网卡 `enp0s3` 加入桥接接口。

编辑 `/etc/qemu-ifup`：

```bash
sudo nano /etc/qemu-ifup
```

写入如下内容：

```sh
#! /bin/sh
echo "Executing /etc/qemu/ifup"
echo "Bringing up $1 bridged mode..."
sudo /sbin/ifconfig $1 0.0.0.0 promisc up
echo "Adding $1 to br0..."
sudo /sbin/brctl addif br0 $1
sleep 3
```

![配置 qemu-ifup 文件](image/1-3-2.png)

截图展示了 QEMU 网络启动脚本内容。该脚本用于在 QEMU 创建网络接口后，将其设置为混杂模式并加入 `br0`。

保存后添加执行权限，并重启网络服务：

```bash
sudo chmod a+x /etc/qemu-ifup
sudo systemctl daemon-reload
sudo systemctl restart systemd-networkd.service
sudo ifdown br0 && sudo ifup br0
```

实验中执行 `ifdown br0` 时出现 `interface br0 not configured`，说明 `br0` 此前尚未启用；随后 `ifup br0` 成功执行，并通过 DHCP 获取到 `10.0.2.16/24`。

![br0 获取地址](image/1-3-4.png)

截图显示 `br0` 已成功获取 IP 地址 `10.0.2.16/24`，为后续访问固件 Web 服务提供网络基础。

#### 7. 准备固件运行环境

进入提取出的固件根目录，复制 QEMU ARM 静态程序，并修复 Web 目录：

```bash
cd _ac9_kf_V15.03.05.19\(6318_\)_cn.zip.extracted/_ac9_kf_V15.03.05.19\(6318_\)_cn.bin.extracted/squashfs-root
sudo cp /usr/bin/qemu-arm-static qemu-arm-static
cp -r webroot_ro/* webroot/
```

检查并设置 `httpd` 的执行权限：

```bash
ls -l bin/httpd
chmod +x bin/httpd
```

![准备 httpd 运行环境](image/1-3-5.png)

截图显示 `qemu-arm-static` 已复制到固件根目录，`bin/httpd` 也具备执行权限。此时文件系统已经具备启动 Web 服务的基本条件。

#### 8. 启动原始 httpd

使用 QEMU 用户态方式启动原始固件中的 `httpd`：

```bash
sudo qemu-arm -L ./ bin/httpd
```

程序启动后，终端输出 `init_core_dump`、`WeLoveLinux`、`Welcome to ...` 等信息，说明 `httpd` 已经开始执行。但随后出现如下连接失败信息：

```text
connect: No such file or directory
connect to server failed
connect cfm failed
```

![启动 httpd 后报错](image/1-3-6.png)

截图显示原始 `httpd` 启动后卡在连接确认相关逻辑，无法继续进入正常 Web 服务监听状态。

此时在浏览器中访问：

```text
http://10.0.2.16/main.html
```

![浏览器访问失败](image/1-3-7.png)

截图显示浏览器无法连接到目标页面，说明原始未 Patch 固件虽然可以开始执行，但无法在当前用户态仿真环境中正常提供 Web 服务。

#### 9. 第一部分结果分析

从终端输出和浏览器访问结果可以看出：

- `httpd` 能够被 QEMU 启动，说明文件权限和基本运行路径已经配置成功。
- 程序启动后卡在运行时依赖连接阶段，说明异常不是简单命令错误。
- `connect cfm failed` 表明程序尝试连接固件内部服务或硬件相关组件失败。
- 浏览器无法访问 Web 页面，说明原始固件尚不能在当前用户态仿真环境中正常运行。

该现象与实验手册描述一致：原始 Tenda AC9 固件在初始化过程中依赖真实硬件或专有系统服务，QEMU 用户态环境无法完整模拟这些资源，因此需要通过 Patch 修改关键判断逻辑。

### （二）对原始二进制文件进行 Patch

#### 1. 安装并启动 Ghidra

在 Ubuntu 虚拟机中安装并启动 Ghidra：

```bash
sudo snap install ghidra
ghidra
```

![启动 Ghidra](image/2-1-1.png)

截图显示 Ghidra 已启动，后续可用于导入和分析 `httpd` 二进制文件。

启动后新建工程：

- 工程类型选择 `Non-Shared Project`。
- 工程名称设置为 `iot`。
- 创建完成后进入 Ghidra 主界面。

![创建 Ghidra 工程](image/2-1-2.png)

截图显示 Ghidra 工程创建完成，后续分析文件会被导入到该工程中。

#### 2. 导入并分析 httpd

在 Ghidra 中选择 `File -> Import File`，导入固件文件系统中的目标程序：

```text
squashfs-root/bin/httpd
```

导入完成后，双击 `httpd` 进入 CodeBrowser，并使用默认选项进行自动分析。

![导入 httpd 并完成分析](image/3-1-4.png)

截图显示 `httpd` 已成功导入并完成初步分析。由于该文件是 ARM 架构二进制程序，Ghidra 会识别其指令集、程序入口和函数结构。

#### 3. 定位并修改关键指令

根据实验手册，关键 Patch 地址为：

```text
0x0002E47C
```

跳转到该地址后，可以看到原始指令为：

```asm
0002e47c    00 30 a0 e1    cpy    r3,r0
```

该指令会将 `r0` 的值复制到 `r3`。结合运行现象可知，该位置附近与连接确认或硬件检测逻辑有关。在仿真环境中，该检测容易失败，导致程序无法继续正常提供 Web 服务。

在该地址右键选择 `Patch Instruction`，将指令修改为：

```asm
MOV R3, 1
```

修改后对应机器码为：

```text
01 30 A0 E3
```

Ghidra 中显示的新指令为：

```asm
0002e47c    01 30 a0 e3    mov    r3,#0x1
```

![修改关键指令](image/3-1-5.png)

截图显示关键地址处的指令已经被修改为 `mov r3,#0x1`。该 Patch 的作用是将检测结果固定为成功值，使程序不再完全依赖原本的连接检测返回值。

#### 4. 导出并替换 httpd

完成指令修改后，在 Ghidra 中选择导出文件，格式保持为 `Original File`，导出路径设置为：

```text
/home/chloe/iot_0x05/httpd_patch
```

![导出 Patch 后的 httpd](image/3-1-6.png)

截图显示 Patch 后的二进制文件已导出。随后回到固件文件系统目录，先备份原始文件，再使用 Patch 后的文件进行替换：

```bash
cp bin/httpd bin/httpd_original
cp ~/iot_0x05/httpd_patch bin/httpd
chmod +x bin/httpd
```

这样可以保留原始 `httpd`，便于后续对比 Patch 前后的运行结果。

#### 5. 重新运行 Patch 后的 httpd

替换完成后，在 `squashfs-root` 目录下重新启动 `httpd`：

```bash
sudo qemu-arm -L ./ bin/httpd
```

运行后，终端仍然出现了部分连接失败信息，例如：

```text
connect: No such file or directory
Connect to server failed
create socket fail -1
```

但与 Patch 前不同的是，程序没有停留在失败状态，而是继续向后执行，并成功输出：

```text
httpd listen ip = 10.0.2.16 port = 80
webs: Listening for HTTP requests at address 10.0.2.16
```

![Patch 后 httpd 成功监听](image/3-2-1.png)

截图显示 Patch 后的 `httpd` 已成功监听 `10.0.2.16:80`，说明 Web 服务已经启动。

随后在浏览器中访问：

```text
http://10.0.2.16/main.html
```

![Patch 后成功访问 Web 页面](image/3-2-2.png)

截图显示浏览器成功进入 Tenda 路由器 Web 管理界面，说明 Patch 后的固件 Web 服务已经可以在 QEMU 用户态环境中正常访问。

#### 6. 第二部分小结

本部分完成了对原始 `httpd` 的二进制 Patch：

- 在 Ghidra 中定位到关键地址 `0x0002E47C`。
- 将原始指令 `cpy r3,r0` 修改为 `mov r3,#0x1`。
- 导出 Patch 后的 `httpd` 并替换固件文件系统中的原始文件。
- 重新运行后，`httpd` 成功监听 `10.0.2.16:80`。
- 浏览器可以正常访问 Tenda Web 管理页面。

Patch 前后的对比说明，原始固件运行失败的关键原因在于初始化过程中存在不适合仿真环境的检测逻辑。修改该判断后，程序能够跳过失败分支并继续启动 Web 服务。

### （三）逆向分析 Patch 原理与原因

#### 1. 原始问题定位

在第一部分中，原始 `httpd` 启动后出现如下现象：

- 程序能够输出 `WeLoveLinux` 和 `Welcome to ...`，说明 `httpd` 已经开始执行。
- 随后出现 `connect cfm failed` 等错误。
- 浏览器无法访问 `10.0.2.16/main.html`。

这说明问题不在于 `httpd` 无法被 QEMU 加载，而是在初始化过程中某个连接确认或硬件检测逻辑没有通过，导致程序无法继续监听 Web 请求。

#### 2. 修改前后的汇编逻辑

在 Ghidra 中定位到地址 `0x0002E47C` 后，可以看到该位置位于 `ConnectCfm` 调用之后。原始逻辑会使用 `ConnectCfm` 的返回值继续判断程序是否能够向后执行。

Patch 后的关键汇编如下：

```asm
0002e470    mov    r0,#0x1
0002e474    bl     sleep
0002e478    bl     ConnectCfm
0002e47c    mov    r3,#0x1
0002e480    cmp    r3,#0x0
```

其中最关键的修改是：

```asm
mov r3,#0x1
```

即将原来依赖函数返回值的结果直接改为固定成功值 `1`。

![Patch 后关键汇编逻辑](image/4-1.png)

截图显示 Patch 后关键位置已经使用常量 `1` 参与后续比较。这样即使仿真环境中缺少真实硬件或相关服务，也不会因为连接检测失败而卡住。

#### 3. 反编译伪代码分析

从 Ghidra 的反编译结果可以看到，`httpd` 启动阶段会执行网络检测和连接确认相关逻辑，例如：

- 通过 `check_network` 等函数等待网络状态。
- 调用 `ConnectCfm()` 进行连接确认。
- 后续读取 Web 服务配置项，例如 `lan.webiplanssen`、`lan.webport`、`lan.webipen`、`lan.ip` 等。

![Patch 后反编译逻辑](image/4-2.png)

截图中的反编译逻辑说明，`ConnectCfm()` 所在位置处于 Web 服务初始化之前。如果这里的检测失败，后面的监听 IP、监听端口和 Web 页面服务就无法正常建立。

本次 Patch 的思路不是完整模拟真实路由器硬件，而是将关键检测结果强制置为成功，使程序跳过不适合在 QEMU 用户态环境中执行的失败分支。

#### 4. Patch 后运行验证

重新运行 Patch 后的 `httpd`，终端输出显示程序最终成功进入 Web 服务监听阶段：

```text
httpd listen ip = 10.0.2.16 port = 80
webs: Listening for HTTP requests at address 10.0.2.16
```

![Patch 后 httpd 成功监听](image/3-2-1.png)

随后访问：

```text
http://10.0.2.16/main.html
```

![Patch 后成功访问 Web 页面](image/3-2-2.png)

因此可以确认，本次 Patch 已经绕过导致仿真失败的关键检测逻辑，使原始固件的 Web 服务能够在 QEMU 用户态环境中运行。

## 五、实验问题与解决方法

### （一）Ubuntu 中 qemu 总包不可直接安装

实验手册中给出了 `sudo apt-get install qemu` 的安装方式，但当前 Ubuntu 环境中直接安装 `qemu` 总包并不适用，可能出现找不到包或安装内容不完整的问题。

解决时没有继续依赖 `qemu` 总包，而是改为安装实验实际需要的具体组件：

```bash
sudo apt install -y qemu-user qemu-user-static qemu-system-arm qemu-utils
```

这样可以同时满足 ARM 用户态程序运行和相关 QEMU 组件调用需求。


### （二）原始 httpd 启动后 Web 页面无法访问

原始 `httpd` 能够通过 QEMU 启动，但终端出现：

```text
connect cfm failed
```

浏览器访问 `http://10.0.2.16/main.html` 时无法建立连接。

原始 Tenda AC9 固件会在初始化阶段进行硬件连接或内部服务检测。QEMU 用户态环境中不存在真实路由器硬件和相关厂商服务，因此检测失败后程序无法继续完成 Web 服务初始化。

解决方法是使用 Ghidra 定位地址 `0x0002E47C`，将原始指令：

```asm
cpy r3,r0
```

修改为：

```asm
mov r3,#0x1
```

该修改使程序跳过失败检测结果，继续执行后续 Web 初始化逻辑。

Patch 后重新运行 `httpd`，终端输出：

```text
httpd listen ip = 10.0.2.16 port = 80
webs: Listening for HTTP requests at address 10.0.2.16
```

浏览器能够正常访问 Tenda Web 管理页面，说明问题解决。



## 六、实验总结

本次实验完整完成了 Tenda AC9 原始固件从解包、仿真、问题复现、二进制 Patch 到逆向分析的完整流程。原始 `httpd` 在 QEMU 用户态环境中因硬件检测失败而无法提供 Web 服务，通过 Ghidra 定位关键地址 `0x0002E47C` 并修改指令后，问题得到解决，浏览器成功访问 Tenda Web 管理页面。

通过本次实验，加深了对以下内容的理解：

- **固件解包与用户态仿真**：使用 `binwalk -Me` 提取固件中的 SquashFS 文件系统，通过 `qemu-arm` 在宿主机上运行 ARM 架构的 `httpd` 程序，掌握了固件从获取到运行的基础流程。
- **”假死”问题根因分析**：原始固件在初始化阶段会调用 `ConnectCfm()` 等函数进行硬件连接确认，QEMU 用户态环境中不存在真实硬件资源，导致检测失败后程序卡住，无法进入 Web 服务监听状态。
- **二进制 Patch 方法**：在 Ghidra 中将 `cpy r3,r0` 修改为 `mov r3,#0x1`（机器码 `01 30 A0 E3`），通过固定检测返回值绕过仿真环境中的失败分支。这一思路说明固件仿真的障碍不一定需要完整模拟硬件，有时精确修改关键判断逻辑即可解决。
- **动静结合的分析思路**：Patch 前后的运行现象（终端输出、浏览器访问结果）与 Ghidra 静态反编译伪代码相互印证，动态运行现象帮助定位问题阶段，静态逆向分析揭示具体代码逻辑，两者结合才能准确确定 Patch 位置和修改方案。

本次实验为后续固件安全分析中遇到类似”假死”问题时提供了可参考的定位和 Patch 思路。

## 参考资料

1. 《固件 Patch 与动态调试实验手册》0x05，课程实验资料，2026-05-09。
2. Tenda AC9 固件下载页面：https://www.tenda.com.cn/material/show/102682
3. Ghidra 官方项目：https://github.com/NationalSecurityAgency/ghidra
4. binwalk 项目仓库：https://github.com/ReFirmLabs/binwalk
5. QEMU 官方文档：https://www.qemu.org/docs/master/
