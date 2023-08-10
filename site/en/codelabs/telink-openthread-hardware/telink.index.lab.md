---
id: telink-openthread-hardware
summary: 在本教程中，你将在实际的硬件设备上配置出OT边界路由器，创建并管理一个Thread网络，并且在节点间传输信息。
status: [draft]
authors: zhenghuan zhang
categories: Nest
tags: web
feedback link: https://github.com/openthread/ot-docs/issues
project: /_project.yaml
book: /_book.yaml
layout: scrolling

---

# 使用B91开发板和OpenThread构建Thread网络

[Codelab Feedback](https://github.com/openthread/ot-docs/issues)

## 介绍

Duration: 3:00

<img src="img/26b7f4f6b3ea0700.png" alt="26b7f4f6b3ea0700.png" width="624.00" />

[OpenThread](https://openthread.io/)是[Thread](http://threadgroup.org/)网络协议的开源实现，它是一种专为物联网（IoT）设备设计的稳健且安全的无线网状网络协议。OpenThread是由谷歌的Nest团队开发的，并作为开源项目免费提供给开发者社区使用。

[Thread规范](http://threadgroup.org/ThreadSpec)建立了一种可靠、安全且能效高的无线通信协议，适用于资源受限的设备，常见于智能家居和商业建筑。OpenThread包含了Thread的完整网络层范围，包括IPv6、6LoWPAN、带有MAC安全性的IEEE 802.15.4、网状链路建立和网状路由等功能。

Telink已将OpenThread实现整合到Zephyr RTOS中，实现了与Telink硬件的无缝兼容。这个整合的源代码可以在[GitHub](https://github.com/telink-semi/zephyr)上方便地获取，并且还提供了软件开发工具包（SDK)。

在这个教程中，您将在实际硬件上构建OpenThread边界路由器，创建和管理一个Thread网络，并在节点之间交换信息。下图展现了教程中的硬件设置，包括一个OT边界路由器（OTBR）和一个Thread设备。

<img src="img/codelab_overview.png" alt="codelab_overview.png" width="624.00" />

### 学习内容

* 使用Telink Zephyr开发环境配置OpenThread编译环境。

* 构建OpenThread CLI示例（ `ot-cli-ftd` 和 `ot-rcp` ），并分别将其烧录到Telink B91开发板。

* 在Raspberry Pi 3B+或更高版本上，使用Docker搭建OpenThread边界路由器（OTBR）。

* 在OTBR上创建一个Thread网络。

* 使用带外调试方式向Thread网络添加设备。

* 使用CLI验证Thread网络中节点之间的连接性。

### 所需条件

硬件：

* 2块B91开发板。

* 1台Raspberry Pi 3B+或更高版本，并安装Raspbian操作系统映像。

* 1台Linux主机，至少带有两个USB端口。

* 1个已连接互联网的交换机（或路由器）和若干条以太网电缆。

软件：

* Telink烧录和调试工具 —— LinuxBDT。

* 串口终端工具，比如PuTTY。

* 其他工具，比如Git和West。

## 前提条件

Duration: 4:00

### Thread基本概念和OpenThread CLI

在进行本教程之前，建议先完成[OpenThread Simulation codelab](https://openthread.io/codelabs/openthread-simulation/#0)，以便熟悉基本的Thread概念和OpenThread CLI。

### Linux主机

Linux主机（Ubuntu v20.04 LTS或更高版本）充当构建机器，用于设置Telink Zephyr开发环境并烧录所有Thread开发板。为了完成这些任务，Linux主机需要两个可用的USB端口和互联网连接。

### 串口连接和串口终端工具

您可以直接将设备插入Linux主机的USB端口。此外，您将需要一个串口终端工具来访问这些设备。

在本教程中，终端工具PuTTY用于控制FTD Joiner和Raspberry Pi，也可以使用其他终端软件。

### Telink B91开发套件

本教程需要2块B91开发板。下面的图片展示了一个套件中所需的最少组件。

<img src="img/overview.png" alt="overview.png" width="624.00" />

本教程将使用一块B91开发板作为RCP（无线电协处理器），使用另一个B91开发板作为FTD（全功能Thread设备）。
如果您尚未拥有这块开发板，您可以从[Telink官方网站](http://wiki.telink-semi.cn/wiki/Hardware/B91_Generic_Starter_Kit_Hardware_Guide/)获取有关B91开发套件的更多详细信息。
需要用到的部分组件如下表所示：

| 标号 | 名称                                                    |
| :--- | ------------------------------------------------------ |
| 1    | Telink B91开发板                                   |
| 2    | Telink烧录板                                            |
| 3    | 2.4GHz天线                                              |
| 4    | USB电缆（USB A 转 mini USB）                            |

### 安装有Raspbian操作系统镜像的树莓派3B+或更高版本

在本教程中，需要使用带有[Raspbian Bullseye Lite OS image](https://downloads.raspberrypi.org/raspios_lite_armhf/images/raspios_lite_armhf-2023-05-03/2023-05-03-raspios-bullseye-armhf-lite.img.xz) 或[Raspbian Bullseye with Desktop](https://downloads.raspberrypi.org/raspios_armhf/images/raspios_armhf-2023-05-03/2023-05-03-raspios-bullseye-armhf.img.xz)的树莓派3B+或更高版本。
它通过以太网连接到互联网，并将配置为OpenThread边界路由器（OTBR）的主机。

### 网络连接

本教程需要一个已连接互联网的交换机（或路由器）和若干条以太网电缆。
它们用于将Raspberry Pi与Linux主机连接起来，便于用户通过主机对Raspberry Pi进行配置。

### LinuxBDT

Telink [烧录和调试工具 (BDT)](http://wiki.telink-semi.cn/wiki/IDE-and-Tools/Burning-and-Debugging-Tools-for-all-Series/) 适用于所有Telink芯片系列，可用于擦除和烧录OpenThread固件到Telink B91开发套件上。
在您的Linux主机上安装基于X86架构的[LinuxBDT](http://wiki.telink-semi.cn/tools_and_sdk/Tools/BDT/LinuxBDT.tar.bz2)。

### 其他

* Git，用于设置Telink Zephyr开发环境。
* West， 用于管理Zephyr项目并构建OpenThread二进制文件。

## 固件设置

Duration: 12:00

### Telink Zephyr开发环境设置

在Linux主机上打开命令行，执行以下命令，以确保您的APT软件包管理器是最新的。

```console
$ sudo apt update
$ sudo apt upgrade
```

完成后，继续执行以下步骤。

1. 安装依赖项。

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

     在执行后续步骤之前，验证系统上安装的版本。如果版本不对，将 APT 镜像切换到稳定且最新的镜像，或手动更新这些依赖项。

2. 安装west。

     ```console
     $ pip3 install --user -U west
     $ echo 'export PATH=~/.local/bin:"$PATH"' >> ~/.bashrc
     $ source ~/.bashrc
     ```

     确保 `~/.local/bin` 包含在 `$PATH` 环境变量中。

3. 获取Zephyr项目的源码。

     ```console
     $ west init ~/zephyrproject
     $ cd ~/zephyrproject
     $ west update
     $ west blobs fetch hal_telink
     $ west zephyr-export
     ```

     在中国大陆，使用 `west init ~/zephyrproject` 和 `west update` 获取 Zephyr 源代码，通常需要花费额外的时间。此外，某些项目可能无法从国外服务器更新，寻找其他方法来下载最新的源代码。

4. 为 Zephyr 安装额外的 Python 依赖项。

     ```console
     $ pip3 install --user -r ~/zephyrproject/zephyr/scripts/requirements.txt
     ```

5. 设置 Zephyr 工具链。下载 Zephyr 工具链（大约 1~2 GB）到本地目录中，以允许您烧录固件到开发板。在中国大陆境内，该步骤可能需要花费额外时间。

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

     其中 [-x.y.z] 可以是任何文本的可选项，例如 -0.13.2。SDK安装后不能移动该目录。接着安装Zephyr工具链。

     ```console
     $ tar xvf zephyr-sdk-0.16.1_linux-x86_64.tar.xz
     $ cd zephyr-sdk-0.16.1
     $ ./setup.sh -t riscv64-zephyr-elf -h -c
     ```

6. 构建Hello World示例。使用Hello World示例验证官方Zephyr项目配置是否正确，然后再继续设置自定义项目。

     ```console
     $ cd ~/zephyrproject/zephyr
     $ west build -p auto -b tlsr9518adk80d samples/hello_world
     ```

     使用west build命令从Zephyr存储库的根目录构建hello_world示例。您可以在 `build/zephyr` 目录下找到名为 `zephyr.bin` 的固件。

7. 将Zephyr环境脚本添加到 `~/.bashrc`。在bash中执行一下命令。

     ```console
     $ echo "source ~/zephyrproject/zephyr/zephyr-env.sh" >> ~/.bashrc
     $ source ~/.bashrc
     ```

8. 添加Telink Zephyr远程库。下载Telink repo到本地作为开发分支并更新该分支。

     ```console
     $ cd ~/zephyrproject/zephyr
     $ git remote add telink-semi https://github.com/telink-semi/zephyr
     $ git fetch telink develop
     $ git checkout develop
     $ cd ..
     $ west update
     $ west blobs fetch hal_telink
     ```

     更多信息参考：[Zephyr Doc -- Getting Started Guide](https://docs.zephyrproject.org/latest/getting_started/index.html)

### 泰凌LinuxBDT设置

下载Telink Linux BDT烧录工具，并将其解压到Linux主机的本地目录，例如 `~`，以允许用户将固件烧录到B91开发板。

```console
$ cd ~
$ wget http://wiki.telink-semi.cn/tools_and_sdk/Tools/BDT/LinuxBDT.tar.bz2
$ tar -vxf LinuxBDT.tar.bz2 
```

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

能搜索到Telink Web Debugger v3.6，代表BDT烧录器顺利连接到Linux主机。

### 固件编译

本教程中将构建两种固件：

* `ot-cli-ftd`
* `ot-rcp`

编译方法如下：

1. 无线电协处理器（ot-rcp）

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

如下图所示，使用USB连接线将一块Telink B91开发板连接到Telink烧录板。

<img src="img/connection_overview.png" alt="connection_overview.png" width="624.00" />

在命令行输入如下指令（以烧录ot-cli-ftd固件为例）。

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

ot-rcp的烧录方法和ot-cli-ftd的基本一样，不同之处在于固件名称。烧录完成后分别将两块B91开发板做好标记区分，烧录ot-cli-ftd的开发板标记为“FTD Joiner”，烧录ot-rcp的开发板标记为“RCP”。

## 为FTD Joiner设备配置串口控制台

Duration: 3:00

如图所示，直接将FTD Joiner插入计算机的USB端口。

<img src="img/usb_connection.png" alt="usb_connection.png" width="624.00" />

将FTD Joiner设备连接到Linux主机后，打开PuTTY，新建terminal，设置串口信息后打开串口。

<img src="img/uart_console.png" alt="uart_console.png" width="624.00" />

 OpenThread的命令行参考[OpenThread CLI Reference](https://github.com/openthread/openthread/blob/f7690fe7e9d638341921808cba6a3e695ec0131e/src/cli/README.md), 使用所有命令的时候，务必加上`ot`前缀。  

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

Duration: 9:00

OpenThread Border Router是由两个主要部件所组成的设备：

* **Raspberry Pi** 包含充当边界路由器（BR）所需的所有服务和固件。

* **RCP** 负责Thread的射频通信。

### 无线电协处理器（RCP）

ot-rcp固件的烧录步骤参考ot-cli-ftd烧录过程，将B91开发板连接到树莓派的USB端口上，连接方式如下图所示。

<img src="img/OTBR_overview.png" alt="OTBR_overview.png" width="624.00" />

### 树莓派

1. 确保写入SD卡中的是[Raspbian Bullseye Lite OS image](https://downloads.Raspberrypi.org/raspios_lite_armhf/images/raspios_lite_armhf-2023-05-03/2023-05-03-raspios-bullseye-armhf-lite.img.xz)或[Raspbian Bullseye with Desktop](https://downloads.Raspberrypi.org/raspios_armhf/images/raspios_armhf-2023-05-03/2023-05-03-raspios-bullseye-armhf.img.xz)

2. 您可以选择通过SSH连接到树莓派，也可以直接在Raspbian桌面上操作。本教程将使用SSH。

3. 在下一步安装OTBR Docker之前，先更新本地代码库和软件包管理器。

     ```console
     $ sudo apt-get update
     $ sudp apt-get upgrade
     ```

### 安装Docker

重新启动树莓派并打开一个SSH终端窗口。

1. 安装Docker：

     ```console
     $ curl -sSL https://get.docker.com | sh
     ```

2. 将当前用户添加到Docker组中，授予权限，这样在每个命令前都不需要加上`sudo`。

     ```console
     $ sudo usermod -aG docker $USER
     ```

     你需要重启树莓派来使改动生效。

3. 若Docker尚未启动，请将其启动：

     ```console
     $ sudo dockerd
     ```

4. OTBR 防火墙脚本在 Docker 容器内创建规则。运行 modprobe 以加载 iptables 的内核模块。

     ```console
     $ sudo modprobe ip6table_filter
     ```

### 配置并运行Docker

本教程直接从[OpenThread Docker Hub](https://hub.docker.com/u/openthread/)拉取OTBR Docker镜像，该镜像已经过OpenThread团队的测试和验证。

1. 拉取镜像：

     ```console
     $ docker pull openthread/otbr:latest
     ```

2. 查看Docker容器中的镜像列表：

     ```console
     $ docker images
     REPOSITORY        TAG       IMAGE ID       CREATED      SIZE
     openthread/otbr   latest    db081f4de15f   6 days ago   766MB
     ```

3. 通过检查 `/dev` 确定RCP设备的串行端口名称, 出现 `ttyACM0` 表示RCP正确连接。

     ```console
     $ ls /dev/tty*
     ...
     /dev/ttyACM0
     ... 
     ```

4. 第一次运行OTBR Docker, 并引用RCP的串行端口（ttyACM0），此后若要继续使用该OTBR Docker，请使用命令 **docker start otbr**。

     ```console
     $ docker run --name "otbr" --sysctl "net.ipv6.conf.all.disable_ipv6=0 net.ipv4.conf.all.forwarding=1 net.ipv6.conf.all.forwarding=1" -p 8080:80 --dns=127.0.0.1 -it --volume /dev/ttyACM0:/dev/ttyACM0 --privileged openthread/otbr --radio-url spinel+hdlc+uart:///dev/ttyACM0
     ```

5. 新开一个SSH终端窗口，测试树莓派和RCP的连通性。

     ```console
     $ docker exec -ti otbr sh -c "sudo ot-ctl"
     > state 
     disabled
     Done
     ```

可选用的Docker命令：

* 获取正在运行的Docker容器信息：

     ```console
     $ docker ps -aq
     ```

* 停止OTBR Docker：

     ```console
     $ docker stop otbr
     ```

* 移除OTBR Docker：

     ```console
     $ docker rm otbr
     ```

* 重新加载OTBR Docker：

     ```console
     $ docker restart otbr
     ```

此时，FTD Joiner和OTBR都已经准备好，可以进行下一步构建Thread网络。

## 创建一个Thread网络

Duration: 6:00

### 在RCP上创建一个Thread网络

我们在OTBR上使用 **ot-ctl** shell 来建立Thread网络。如果您在上一节退出了shell，在SSH终端中输入以下命令来重新启动它：

```console
$ docker exec -ti otbr sh -c "sudo ot-ctl"
```

然后按下表顺序输入命令，并确认每一步达到预期结果后继续下一步。

| 序号  | 命令                    |命令简介                                                           | 预期响应 |
| :---- | ------------------------|-------------------------------------------------------------------------- | ------------------ |
| 1     | `dataset init new`      | 创建一个新的随机网络数据集。                                          | Done               |
| 2     | `dataset commit active` | 将新的数据集提交到非易失性存储器中的活动运行数据集。 | Done               |
| 3     | `ifconfig up`           | 启动IPv6接口。                                                  | Done               |
| 4     | `thread start`          | 启用Thread协议并连接到一个Thread网络。              | Done               |
|       |                         |                          | 等待10秒，直到Thread接口启动并建立一个Thread网络。                    |
| 5     | `state`                 | 检查设备状态。可以多次调用此命令，直到设备成为Leader并继续进行下一步操作。                                                    | leader<br/>Done               |
| 6     | `dataset active`        | 检查完整的活动运行数据集，并记住网络密钥。                                                  | Active Timestamp: 1<br/>Channel: 13<br/>Channel Mask: 0x07fff800<br/>Ext PAN ID: b07476e168eda4fc<br/>Mesh Local Prefix: fd8c:60bc:a98:c7ba::/64<br/>Network Key: c312485187484ceb5992d2343baaf93d<br/>Network Name: OpenThread-599c<br/>PAN ID: 0x599c<br/>PSKc: 04f79ad752e8401a1933486c95299f60<br/>Security Policy: 672 onrc 0<br/>Done               |

OTBR在创建网络过程中随机生成的 `networkkey` 将在 FTD Joiner 加入这个Thread网络时被用到。

### FTD Joiner通过带外调试方式加入网络

带外调试是指通过非无线方式（例如在OpenThread CLI中手动输入）传输网络凭据给待入网设备。在串口控制台中，向FTD Joiner按顺序输入如下命令。

| 序号  | 命令                                                       |命令简介                                                            | 预期响应 |
| :---- | --------------------------------------------------------   | ----------------------------------------------------------------------------- | ------------------ |
| 1     | `ot dataset networkkey c312485187484ceb5992d2343baaf93d`   | 设备要连接到Thread网络，只需要网络密钥。 | Done               |
| 2     | `ot dataset commit active`                                 | 将新的数据集提交到非易失性存储器中的活动运行数据集。  | Done               |
| 3     | `ot ifconfig up`                                           | 启动IPv6接口。                                                   | Done               |
| 4     | `ot thread start`                                          | 启用Thread协议并连接到一个Thread网络。               | Done               |
|       |                                                            |                   | 等待20秒，让设备自行配置参数并加入Thread网络。                   |
| 5     | `ot state`                                                 | 检查设备状态。                                                        | child/router<br/>Done               |

> aside positive
>
> **注意：** FTD Joiner设备刚开始是child，一段时间之后会转变为router，这都是正常状态。

### 拓扑图

在SSH终端中输入 `ipaddr`、`child table`、`router table` 等命令，得到如下响应。

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

OTBR的 `RLOC16` 是 `0xb000`，而FTD Joiner最初的 `RLOC16` 是 `0xb001`。然后，在获得路由器ID后，FTD Joiner的 `RLOC16` 变成了 `0x8400`。可以看出FTD Joiner设备从child升级为router。

> aside positive
>
> **Note:** RLOC 全称是 Routing Locator，根据Thread设备在网络拓扑中的位置来标识该设备，是Thread设备的几个IPv6地址之一，具体介绍查阅[IPv6 Addressing](https://openthread.io/guides/thread-primer/ipv6-addressing#routing-locator-rloc)。

现在的Thread网络包含两个节点，拓扑如下图所示。

<img src="img/topology.png" alt="topology.png" width="400.00" />

## Thread设备之间的通信

Duration: 6:00

### ICMPv6通信

我们使用 `ping` 命令来检查同一网络的Thread设备能否相互通信。先用 `ipaddr` 命令获取设备的RLOC。

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

OpenThread提供的应用服务中还包括UDP，可以使用UDP API在Thread网络中的节点之间传递信息，或者通过边界路由器向外部网络传递信息。OpenThread的UDP API在[OpenThread CLI - UDP Example](https://github.com/openthread/openthread/blob/f7690fe7e9d638341921808cba6a3e695ec0131e/src/cli/README_UDP.md)中有详细介绍，此教程将使用其中的部分API在OTBR和FTD Joiner之间传输信息。

首先获取OTBR的Mesh-Local EID，该地址也是Thread设备的一个IPv6地址，与网络拓扑无关，可用于访问同一Thread网络分区内的Thread设备。

```console
> ipaddr mleid
fd8c:60bc:a98:c7ba:5249:34ab:26d1:aff6
Done
```

在SSH终端输入如下指令，启用OTBR UDP，绑定设备的1022端口。

```console
> udp open
Done
> udp bind :: 1022
Done
```

在串口控制台输入如下命令，启用FTD Joiner设备的UDP，绑定设备的1022端口，然后向OTBR发送一个5字节的`hello`信息。

```console
> ot udp open 
Done
> ot udp bind :: 1022
Done
> ot udp send fd8c:60bc:a98:c7ba:5249:34ab:26d1:aff6 1022 hello
Done
```

SSH终端输出如下信息，OTBR收到来自FTD Joiner设备的`hello`信息，UDP通信成功。

```console
> 5 bytes from fd8c:60bc:a98:c7ba:9386:63cf:19d7:5a61 1022 hello
```

## 恭喜

您已经创建了一个简单的Thread网络，并验证了该网络内的通信。

您现在已经知道：

* 如何搭建并使用Telink Zephyr开发环境。
* 如何构建 `ot-cli-ftd` 和 `ot-rcp` 两种二进制文件并将其烧录到B91开发板。
* 如何使用Docker将Raspberry Pi 3B+ 或更高版本设置为OpenThread边界路由器（OTBR）。
* 如何在OTBR上创建Thread网络。
* 通过带外调试方式将设备添加到Thread网络。
* 如何验证Thread网络节点之间的连通性。

### 深入阅读

查看[openthread.io](https://openthread.io/)和[GitHub](https://github.com/openthread)，了解各种OpenThread资源，包括：

* [Supported Platforms](https://openthread.io/platforms/)
    — discover all the platforms that support OpenThread
* [Build OpenThread](../../guides/build/index.md)
    — further details on building and configuring OpenThread
* [Thread Primer](../../guides/thread-primer/index.md)
    — covers all the Thread concepts featured in this codelab

参考文档:

* [OpenThread CLI reference](https://github.com/openthread/openthread/blob/main/src/cli/README.md)
* [OpenThread UDP CLI reference](https://github.com/openthread/openthread/blob/main/src/cli/README_UDP.md)
* [OpenThread UDP API reference](https://openthread.io/reference/group/api-udp)

## License

Copyright (c) 2021-2023, The OpenThread Authors.
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
