---
id: telink-openthread-hardware
summary: In this Codelab, you'll program OpenThread on real hardware, create and manage a Thread network, and pass messages between nodes.
status: [final]
authors: zhenghuan zhang
categories: Nest
tags: web
feedback link: https://github.com/openthread/ot-docs/issues
project: /_project.yaml
book: /_book.yaml

---

# 使用TLSR9518开发板和OpenThread构建Thread网络

[Codelab Feedback](https://github.com/openthread/ot-docs/issues)

## 介绍

<img src="img/26b7f4f6b3ea0700.png" alt="26b7f4f6b3ea0700.png" width="624.00" />

由Google发布的[OpenThread](https://openthread.io/)是[Thread®](http://threadgroup.org/)网络协议的开源实现。Google Nest将OpenThread提供给开发者，以便通过提供访问Nest产品中使用的技术来促进连接家庭产品的开发。

[Thread规范](http://threadgroup.org/ThreadSpec)为家庭应用程序建立了可靠、安全和节能的无线通信协议，利用IPv6。OpenThread包括Thread内的完整网络层范围，例如IPv6、6LoWPAN、具有MAC安全性的IEEE 802.15.4、网状链路建立和网状路由。

在这个Codelab中，您将在实际硬件上编程OpenThread，创建和管理一个Thread网络，并在节点之间交换消息。

<img src="img/codelab_overview.png" alt="codelab_overview.png" width="624.00" />

### 学习内容

* 如何搭建并使用Telink Zephyr开发环境
* 如何构建ot-cli-ftd和ot-rcp两种二进制文件并将其刷写到TLSR9518开发板
* 如何使用Docker将Raspberry Pi 3B+ 或更高版本设置为OpenThread边界路由器（OTBR）
* 如何在OTBR上创建Thread网络。
* 通过带外调试方式将设备添加到Thread网络
* 如何CLI验证Thread网络节点之间的连通性

## 环境准备

### 硬件:

1. 2块TLSR9518开发板， 此Codelab使用一个TLSR9518作为RCP，使用一个TLSR9518作为FTD。
如果您刚开始接触新板，可以从[Telink官网](http://wiki.telink-semi.cn/wiki/Hardware/B91_Generic_Starter_Kit_Hardware_Guide/)获取TLSR9518开发套件，如下图所示。

     <img src="img/overview.jpg" alt="overview.jpg" width="624.00" />

     | Index | Name                                                    |
     | :--- | -------------------------------------------------------- |
     | 1    | Telink TLSR9518EVB                                       |
     | 2    | Telink Burning Board                                     |
     | 3    | 2.4Ghz antenna                                           |
     | 4&5  | USB cable (USB A to mini USB)                            |
     | 6    | UART to USB tool                                         |

2. Raspberry Pi 3B+或更高版本，能通过以太网连接到互联网。在此Codelab中将其配置为OpenThread Border Router的host端。

3. Linux主机系统采用基于Debian的发行版（如Ubuntu v20.04 LTS及更高版本），将被用作构建机器。Linux主机需要两个USB端口并连到互联网。

4. 一台连接到互联网的交换机和若干以太网电缆，电缆将Raspberry Pi和Linux主机连接到交换机，便于用户通过主机配置Raspberry Pi。

### 软件:

1. Telink的LinuxBDT，用于擦除、烧录固件。

2. PuTTY，用于控制FTD Joiner，配置Raspberry Pi。

## 固件设置

### Telink Zephyr开发环境设置
在Linux主机上打开命令行，先执行APT更新和升级，然后再执行以下步骤。

```console
$ sudo apt update
$ sudo apt upgrade
```

1. 安装依赖项：

     ```console
     $ wget https://apt.kitware.com/kitware-archive.sh
     $ sudo bash kitware-archive.sh
     $ sudo apt install --no-install-recommends git cmake ninja-build gperf \
     ccache dfu-util device-tree-compiler \
     python3-dev python3-pip python3-setuptools python3-tk python3-wheel xz-utils file \
     make gcc gcc-multilib g++-multilib libsdl2-dev
     ```

     Zephyr目前需要主要依赖项的最低版本，例如 CMake (3.20.0)、Python3 (3.6)、Devicetree 编译器 (1.4.6)。

     ```console
     $ cmake --version
     $ python3 --version
     $ dtc --version
     ```

     在执行后续步骤之前，请验证系统上安装的版本。如果版本不对，请将 APT 镜像切换到稳定且最新的镜像，或手动更新这些依赖项。

2. 安装west：

     ```console
     $ pip3 install --user -U west
     $ echo 'export PATH=~/.local/bin:"$PATH"' >> ~/.bashrc
     $ source ~/.bashrc
     ```

     确保~/.local/bin在$PATH环境变量上

3. 获取Telink Zephyr源码：

     ```console
     $ west init ~/zephyrproject
     $ cd ~/zephyrproject
     $ west update
     $ west blobs fetch hal_telink
     $ west zephyr-export
     ```

     在中国大陆，使用west init ~/zephyrproject和west update获取 Zephyr 源代码通常需要花费额外的时间。此外，某些项目可能无法从国外服务器更新。请寻找其他方法来下载最新的源代码。

4. 为 Zephyr 安装额外的 Python 依赖项：

     ```console
     $ pip3 install --user -r ~/zephyrproject/zephyr/scripts/requirements.txt
     ```

5. 设置Zephyr工具链：

     下载 Zephyr 工具链（大约 1~2 GB）到本地目录中，以允许您刷新大多数板。中国大陆境内可能需要额外时间。

     ```console
     $ wget https://github.com/zephyrproject-rtos/sdk-ng/releases/download/v0.16.1/zephyr-sdk-0.16.1_linux-x86_64.tar.xz
     $ wget -O - https://github.com/zephyrproject-rtos/sdk-ng/releases/download/v0.16.1/sha256.sum | shasum --check --ignore-missing
     ```
     下载Zephyr SDK并将其放置在推荐路径中，如下所示。

     ```console
     $HOME/zephyr-sdk[-x.y.z]
     $HOME/.local/zephyr-sdk[-x.y.z]
     $HOME/.local/opt/zephyr-sdk[-x.y.z]
     $HOME/bin/zephyr-sdk[-x.y.z]
     /opt/zephyr-sdk[-x.y.z]
     /usr/zephyr-sdk[-x.y.z]
     /usr/local/zephyr-sdk[-x.y.z]
     ```

     其中 [-xyz] 是可选文本，可以是任何文本，例如 -0.13.2。SDK安装后不能移动该目录。接着安装Zephyr工具链：

     ```console
     $ tar xvf zephyr-sdk-0.16.1_linux-x86_64.tar.xz
     $ cd zephyr-sdk-0.16.1
     $ ./setup.sh -t riscv64-zephyr-elf -h -c
     ```

6. 构建Hello World示例

     请先使用Hello World示例验证官方Zephyr项目配置是否正确，然后再继续设置自定义项目。

     ```console
     $ cd ~/zephyrproject/zephyr
     $ west build -p auto -b tlsr9518adk80d samples/hello_world
     ```

     使用west build命令从Zephyr存储库的根目录构建hello_world示例。您可以在build/zephyr目录下找到名为zephyr.bin的固件。

7. 将Zephyr环境脚本添加到~/.bashrc

     在bash中执行一下命令：

     ```console
     $ echo "source ~/zephyrproject/zephyr/zephyr-env.sh" >> ~/.bashrc
     $ source ~/.bashrc
     ```

8. 添加Telink Zephyr远程库：

     下载Telink repo到本地作为开发分支并更新该分支。

     ```console
     $ cd ~/zephyrproject/zephyr
     $ git remote add telink-semi https://github.com/telink-semi/zephyr
     $ git fetch telink develop
     $ git checkout develop
     $ cd ..
     $ west update
     $ west blobs fetch hal_telink
     ```

     更多信息参考：[https://docs.zephyrproject.org/latest/getting_started/index.html](https://docs.zephyrproject.org/latest/getting_started/index.html)

### 泰凌工具设置

下载Telink Linux BDT烧录工具并将其解压到Linux主机的本地目录，例如~，以允许用户将固件烧录到TLSR9518开发板。

```console
$ cd ~
$ wget http://wiki.telink-semi.cn/tools_and_sdk/Tools/BDT/LinuxBDT.tar.bz2
$ tar -vxf LinuxBDT.tar.bz2 
```
> aside positive
>
> **Note:** 在中国大陆以外的地区下载可能会花费额外的时间。

将BDT通过USB接口连接到Linux主机上，在命令行输入如下指令。

```console
$ cd LinuxBDT
$ sudo ./bdt lsusb -v
Bus 002 Device 001: ID 1d6b:0003 xHCI Host Controller
Bus 001 Device 003: ID 0bda:565a Integrated_Webcam_HD
Bus 001 Device 023: ID 413c:301a Dell MS116 USB Optical Mouse
Bus 001 Device 037: ID 248a:826a Telink Web Debugger v3.6
Bus 001 Device 001: ID 1d6b:0002 xHCI Host Controller
```

能搜索到Telink Web Debugger v3.6，代表BDT烧录器顺利连接到Linux，可以进行下一步固件烧录

### 固件编译

此Codelab中会用到ot-cli-ftd和ot-rcp两种固件，编译方法如下：

1. 无线电协处理器（ot-rcp)

     ```console
     $ cd ~/zephyrproject
     $ rm -rf build_ot_coprocessor
     $ west build -b tlsr9518adk80d -d build_ot_coprocessor zephyr/samples/net/openthread/coprocessor -- -DDTC_OVERLAY_FILE="usb.overlay" -DOVERLAY_CONFIG=overlay-rcp-usb-telink.conf
     ```

2. 带交互命令行的全功能Thread设备（ot-cli-ftd）

     ```console
     $ cd ~/zephyrproject
     $ rm -rf build_ot_cli_ftd
     $ west build -b tlsr9518adk80d -d build_ot_cli_ftd zephyr/samples/net/openthread/cli -- -DOVERLAY_CONFIG=overlay-telink-fixed-mac.conf -DCONFIG_OPENTHREAD_FTD=y
     ```

### 固件烧录

将TLSR9518开发板与BDT如下图连接

<img src="img/connection_overview.jpg" alt="connection_overview.jpg" width="624.00" />

在命令行输入如下指令（以烧录ot-cli-ftd固件为例）

```console
$ cd ~/zephyrproject/build_ot_cli_ftd/zephyr
$ cp zephyr.bin ~/LinuxBDT/bin/ot-cli-ftd.bin
$ cd ~/LinuxBDT
$ sudo ./bdt 9518 ac
 Activate OK!
$ sudo ./bdt 9518 wf 0 -i bin/ot-cli-ftd.bin
 EraseSectorsize...
 Total Time: 2181 ms
 Flash writing...
 [100%][-] [##################################################]
 File Download to Flash at address 0x000000: 491700 bytes
 Total Time: 30087 ms
```

ot-rcp的烧录方法和ot-cli-ftd的基本一样，不同之处在于固件名称。烧录完成后分别将两块TLSR9518开发板做好标记区分，烧录ot-cli-ftd的板子标记为FTD Joiner，烧录ot-rcp的板子标记为RCP。

## 为FTD Joiner设备配置串口控制台

要想通过命令行功能控制FTD Joiner设备，请将UART连接到以下引脚：

| Name  | Pin                                                    |
| :---- | ------------------------------------------------------ |
| RX    | PB3 (pin 15 of J34)                                    |
| TX    | PB2 (pin 18 of J34)                                    |
| GND   | GND (pin 23 of J50)                                    |

> aside positive
>
> **Note:** 波特率：115200 bits/s

如图连接好设备后，打开PuTTY，新建terminal，设置串口信息后打开串口。

<img src="img/uart_console.png" alt="uart_console.png" width="624.00" />

OpenThread的命令行参考[OpenThread CLI Reference](https://github.com/openthread/openthread/blob/f7690fe7e9d638341921808cba6a3e695ec0131e/src/cli/README.md), 使用的时候务必加上`ot`前缀。  

例子：
```bash
> ot state
disabled
Done
> ot channel
11
Done
>
```

## OpenThread Border Router（OTBR）设置

OpenThread Border Router是由两个主要部分组成的设备：
* **Raspberry Pi** 包含充当边界路由器（BR）所需的所有服务和固件。
* **RCP** 负责Thread通信。

### 无线电协处理器（RCP）

ot-rcp固件的烧录步骤参考ot-cli-ftd烧录过程，将TLSR9518开发板连接到树莓派的USB端口上，连接方式如图

<img src="img/OTBR_overview.png" alt="OTBR_overview.png" width="624.00" />

### 树莓派

1. 确保写入SD卡中的是[Raspbian Bullseye Lite OS image](https://downloads.Raspberrypi.org/raspios_lite_armhf/images/raspios_lite_armhf-2023-05-03/2023-05-03-raspios-bullseye-armhf-lite.img.xz)或[Raspbian Bullseye with Desktop](https://downloads.Raspberrypi.org/raspios_armhf/images/raspios_armhf-2023-05-03/2023-05-03-raspios-bullseye-armhf.img.xz)
2. 用户可以通过SSH连接到Raspberry Pi。
3. 在下一步安装OTBR Docker之前，先更新本地代码库和软件包管理器

     ```console
     $ sudo apt-get update
     $ sudp apt-get upgrade
     ```

### 安装Docker

重新启动树莓派并打开一个SSH终端窗口。

1. 安装Docker

     ```console
     $ curl -sSL https://get.docker.com | sh
     ```

2. 修改Docker用户设置，使每个命令前面都不用再添加sudo，需要重新启动。

     ```console
     $ sudo usermod -aG docker $USER
     ```

3. 安装git

     ```console
     $ sudo apt install git
     ```

4. 若Docker尚未启动，请将其启动

     ```console
     $ sudo dockerd
     ```

5. OTBR 防火墙脚本在 Docker 容器内创建规则。运行 modprobe 以加载 iptables 的内核模块

     ```console
     $ sudo modprobe ip6table_filter
     ```

### 配置并运行Docker

此Codelab直接从[OpenThread Docker Hub](https://hub.docker.com/u/openthread/)拉取OTBR Docker镜像，该镜像已经过OpenThread团队的测试和验证。

1. 拉取镜像

     ```console
     $ docker pull openthread/otbr:latest
     ```

2. 查看Docker容器中的镜像列表

     ```console
     $ docker images
     REPOSITORY        TAG       IMAGE ID       CREATED      SIZE
     openthread/otbr   latest    db081f4de15f   6 days ago   766MB
     ```

3. 通过检查/dev确定RCP设备的串行端口名称, 出现ttyACM0表示RCP正确连接

     ```console
     $ ls /dev/tty*
     ...
     /dev/ttyACM0
     ... 
     ```

4. 第一次运行OTBR Docker, 并引用RCP的串行端口（ttyACM0），此后若要继续使用该OTBR Docker，请使用命令**docker start otbr**

     ```console
     $ docker run --name "otbr" --sysctl "net.ipv6.conf.all.disable_ipv6=0 net.ipv4.conf.all.forwarding=1 net.ipv6.conf.all.forwarding=1" -p 8080:80 --dns=127.0.0.1 -it --volume /dev/ttyACM0:/dev/ttyACM0 --privileged openthread/otbr --radio-url spinel+hdlc+uart:///dev/ttyACM0
     ```

5. 新开一个SSH终端窗口，测试树莓派和RCP的连通性

     ```console
     $ docker exec -ti otbr sh -c "sudo ot-ctl"
     > state 
     disabled
     Done
     ```

可选内容
* 获取正在运行的Docker容器信息

     ```console
     $ docker ps -aq
     ```

* 停止OTBR Docker

     ```console
     $ docker stop otbr
     ```

* 移除OTBR Docker

     ```console
     $ docker rm otbr
     ```

* 重新加载OTBR Docker

     ```console
     $ docker restart otbr
     ```

此时，FTD Joiner和OTBR都已经准备好，可以进行下一步构建Thread网络。

## 创建Thread网络

### OTBR创建Thread网络

我们使用OTBR的**ot-ctl** shell来建立Thread网络。在SSH终端输入：

```console
$ docker exec -ti otbr sh -c "sudo ot-ctl"
```

然后按下表顺序输入命令，并确认每一步达到预期结果后继续下一步

| Index | Command               |Simple introduction                                                           | Expected Responses |
| :---- | --------------------- | ---------------------------------------------------------------------------- | ------------------ |
| 1     | dataset init new      | Create a new random network dataset                                          | Done               |
| 2     | dataset commit active | Commit new dataset to the Active Operational Dataset in non-volatile storage | Done               |
| 3     | ifconfig up           | Bring up the IPv6 interface                                                  | Done               |
| 4     | thread start          | Enable Thread protocol operation and attach to a Thread network              | Done               |
| 5     | state                 | Check the device state.This command can be called multiple times until it becomes the leader and moves on to the next step                                                    | leader<br/>Done               |
| 6     | dataset active        | Check the complete Active Operational Dataset, please remember networkkey                                                  | Active Timestamp: 1<br/>Channel: 13<br/>Channel Mask: 0x07fff800<br/>Ext PAN ID: b07476e168eda4fc<br/>Mesh Local Prefix: fd8c:60bc:a98:c7ba::/64<br/>Network Key: c312485187484ceb5992d2343baaf93d<br/>Network Name: OpenThread-599c<br/>PAN ID: 0x599c<br/>PSKc: 04f79ad752e8401a1933486c95299f60<br/>Security Policy: 672 onrc 0<br/>Done               |

OTBR在创建网络过程中随机生成的networkkey将在其他设备加入这个Thread网络时被用到。
### FTD Joiner通过带外调试方式加入网络

带外调试是指通过非无线方式（例如在OpenThread CLI中手动输入）传输网络凭据给待入网设备。在串口控制台中向FTD Joiner按顺序输入如下命令

| Index | Command                                                  |Simple introduction                                                            | Expected Responses |
| :---- | -------------------------------------------------------- | ----------------------------------------------------------------------------- | ------------------ |
| 1     | ot dataset networkkey c312485187484ceb5992d2343baaf93d   | Only the Network Key is necessary for a device to connect to a Thread network | Done               |
| 2     | ot dataset commit active                                 | Commit new dataset to the Active Operational Dataset in non-volatile storage  | Done               |
| 3     | ot ifconfig up                                           | Bring up the IPv6 interface                                                   | Done               |
| 4     | ot thread start                                          | Enable Thread protocol operation and attach to a Thread network               | Done               |
| 5     | ot state                                                 | Check the device state                                                        | child/router<br/>Done               |

> aside positive
>
> **Note:** FTD Joiner设备刚开始是child，一段时间之后会转变为router，这都是正常状态。
### 拓扑图

在SSH终端中输入`ipaddr`、`child table`、`router table`等命令，得到如图响应

```console
> ipaddr rloc
fd8c:60bc:a98:c7ba:0:ff:fe00:b000
Done
> child table
| ID  | RLOC16 | Timeout    | Age        | LQ In | C_VN |R|D|N|Ver|CSL|QMsgCnt|Suprvsn| Extended MAC     |
+-----+--------+------------+------------+-------+------+-+-+-+---+---+-------+-------+------------------+
|   1 | 0xb001 |        240 |         23 |     3 |   51 |1|1|1|  3| 0 |     0 |   129 | 82bc12fbe783468e |

Done
> router table
| ID | RLOC16 | Next Hop | Path Cost | LQ In | LQ Out | Age | Extended MAC     | Link |
+----+--------+----------+-----------+-------+--------+-----+------------------+------+
| 44 | 0xb000 |       63 |         0 |     0 |      0 |   0 | 7ae354109d611f7e |    0 |

Done
...
> child table
| ID  | RLOC16 | Timeout    | Age        | LQ In | C_VN |R|D|N|Ver|CSL|QMsgCnt|Suprvsn| Extended MAC     |
+-----+--------+------------+------------+-------+------+-+-+-+---+---+-------+-------+------------------+

Done
> router table
| ID | RLOC16 | Next Hop | Path Cost | LQ In | LQ Out | Age | Extended MAC     | Link |
+----+--------+----------+-----------+-------+--------+-----+------------------+------+
| 33 | 0x8400 |       63 |         0 |     3 |      3 |  13 | e61487c1cda940a6 |    1 |
| 44 | 0xb000 |       63 |         0 |     0 |      0 |   0 | 7ae354109d611f7e |    0 |

Done
```

RLOC16为0xb000的是OTBR，FTD Joiner设备的RLOC16一开始是0xb001，后来获得Router ID后变成0x8400，可以看出FTD Joiner设备从child升级为router。

> aside positive
>
> **Note:** RLOC全称是Routing Locator，根据Thread设备在网络拓扑中的位置来标识该设备，是Thread设备的几个IPv6地址之一，具体介绍查阅[IPv6 Addressing](https://openthread.io/guides/thread-primer/ipv6-addressing#routing-locator-rloc)

现在的Thread网络包含两个节点，拓扑如下图所示。

<img src="img/topology.png" alt="topology.png" width="400.00" />

## Thread设备之间的通信

### ICMPv6通信

我们使用`ping`命令来检查同一网络的Thread设备能否相互通信。先用`ipaddr`命令获取设备的RLOC。

```console
> ipaddr
fd8c:60bc:a98:c7ba:0:ff:fe00:fc11
fdbd:7274:649c:1:1d19:9613:f705:a5af
fd8c:60bc:a98:c7ba:0:ff:fe00:fc10
fd8c:60bc:a98:c7ba:0:ff:fe00:fc38
fd8c:60bc:a98:c7ba:0:ff:fe00:fc00
fd8c:60bc:a98:c7ba:0:ff:fe00:b000
fd8c:60bc:a98:c7ba:5249:34ab:26d1:aff6
fe80:0:0:0:78e3:5410:9d61:1f7e
Done
```

在FTD Joiner设备的串口控制台输入如下指令执行ping操作。

```console
> ot ping fd8c:60bc:a98:c7ba:0:ff:fe00:b000
16 bytes from fd8c:60bc:a98:c7ba:0:ff:fe00:b000: icmp_seq=1 hlim=64 time=19ms
1 packets transmitted, 1 packets received. Packet loss = 0.0%. Round-trip min/avg/max = 19/19.0/19 ms.
Done
```

串口输出的响应说明OTBR端收到了ping request，FTD Joiner设备收到了OTBR返回的ping response，两个设备之间通信正常。

### UDP通信

OpenThread提供的应用服务中还包括UDP，可以使用UDP API在Thread网络中的节点之间传递信息，或者通过边界路由器向外部网络传递信息。OpenThread的UDP API在[OpenThread CLI - UDP Example](https://github.com/openthread/openthread/blob/f7690fe7e9d638341921808cba6a3e695ec0131e/src/cli/README_UDP.md)中有详细介绍，此Codelab将使用其中的部分API在OTBR和FTD Joiner之间传输信息。

首先获取OTBR的Mesh-Local EID，该地址也是Thread设备的一个IPv6地址，与网络拓扑无关，可用于访问同一Thread网络分区内的Thread设备。

```console
> ipaddr mleid
fd8c:60bc:a98:c7ba:5249:34ab:26d1:aff6
Done
```

在SSH终端输入如下指令，启用OTBR UDP，绑定设备的1022端口

```console
> udp open
Done
> udp bind :: 1022
Done
```

在串口控制台输入如下命令，启用FTD Joiner设备的UDP，绑定设备的1022端口，然后向OTBR发送一个5字节的`hello`信息

```console
> ot udp open 
Done
> ot udp bind :: 1022
Done
> ot udp send fd8c:60bc:a98:c7ba:5249:34ab:26d1:aff6 1022 hello
Done
```

SSH终端输出如下信息，OTBR收到来自FTD Joiner设备的`hello`信息，UDP通信成功

```console
> 5 bytes from fd8c:60bc:a98:c7ba:9386:63cf:19d7:5a61 1022 hello
```

## 恭喜

一个简单的Thread网络被创建了，网络的通信功能也得到了验证。

您现在已经知道：

* 如何搭建并使用Telink Zephyr开发环境
* 如何构建ot-cli-ftd和ot-rcp两种二进制文件并将其刷写到TLSR9518开发板
* 如何使用Docker将Raspberry Pi 3B+ 或更高版本设置为OpenThread边界路由器（OTBR）
* 如何在OTBR上创建Thread网络。
* 通过带外调试方式将设备添加到Thread网络
* 如何验证Thread网络节点之间的连通性

### 深入阅读

查看[openthread.io](https://openthread.io/)和[GitHub](https://github.com/openthread)，了解各种OpenThread资源，包括：

*  [Supported Platforms](https://openthread.io/platforms/)
    — discover all the platforms that support OpenThread
*  [Build OpenThread](../../guides/build/index.md)
    — further details on building and configuring OpenThread
*  [Thread Primer](../../guides/thread-primer/index.md)
    — covers all the Thread concepts featured in this Codelab

参考文档:

*  [OpenThread CLI reference](https://github.com/openthread/openthread/blob/main/src/cli/README.md)
*  [OpenThread UDP CLI reference](https://github.com/openthread/openthread/blob/main/src/cli/README_UDP.md)
*  [OpenThread Daemon reference](https://openthread.io/platforms/co-processor/ot-daemon)
*  [OpenThread UDP API reference](https://openthread.io/reference/group/api-udp)
*  [GNU Screen quick reference](http://aperiodic.net/screen/quick_reference)

## License

Copyright (c) 2021-2022, The OpenThread Authors.
All rights reserved.

Redistribution and use in source and binary forms, with or without
modification, are permitted provided that the following conditions are met:

1. Redistributions of source code must retain the above copyright
   notice, this list of conditions and the following disclaimer.
2. Redistributions in binary form must reproduce the above copyright
   notice, this list of conditions and the following disclaimer in the
   documentation and/or other materials provided with the distribution.
3. Neither the name of the copyright holder nor the
   names of its contributors may be used to endorse or promote products
   derived from this software without specific prior written permission.

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS"
AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE
ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE
LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR
CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF
SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS
INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN
CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE)
ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE
POSSIBILITY OF SUCH DAMAGE.


