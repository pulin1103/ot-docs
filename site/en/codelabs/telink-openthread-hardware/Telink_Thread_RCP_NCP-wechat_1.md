
# 泰凌微电子Thread RCP和NCP方案介绍（一）

## 介绍

[Thread规范](http://threadgroup.org/ThreadSpec)建立了一种可靠、安全且能效高的无线通信协议，适用于资源受限的设备，常见于智能家居和商业建筑。OpenThread包含了Thread的完整网络层范围，包括IPv6、6LoWPAN、带有MAC安全性的IEEE 802.15.4、网状链路建立和网状路由等功能。

Telink已将由谷歌的团队开发的OpenThread实现整合到Zephyr RTOS中，实现了与Telink硬件的无缝兼容。这个整合的源代码可以在[GitHub](https://github.com/telink-semi/zephyr)上方便地获取，并且还提供了软件开发工具包（SDK)。

这个教程共分为上下两个部分。在本上半部分中，您将在Telink Zephyr开发环境上构建OpenThread NCP和RCP固件；在下半部分中，您可以使用这些固件分别与树莓派协同工作，创建和管理Thread网络。

### 学习内容

* 使用Telink Zephyr开发环境配置OpenThread编译环境。

* 构建OpenThread Co-Processor固件（ `ot-ncp-ftd` 和 `ot-rcp` ）。

### 所需条件

硬件：

* 2块B91开发板。

* 1台Raspberry Pi 3B+或更高版本，并安装Raspbian操作系统映像。

* 1台Linux主机，至少带有两个USB端口。

* 1个已连接互联网的交换机（或路由器）和若干条以太网电缆。

软件：

* Telink烧录和调试工具 —— LinuxBDT。

* 其他工具，比如Git和West。

## 前提条件

### Thread基本概念和OpenThread Co-Processor

在进行本教程之前，建议先完成[OpenThread Simulation codelab](https://openthread.io/codelabs/openthread-simulation/#0)并阅读[OpenThread Co-Processor Designs](https://openthread.io/platforms/co-processor)，以便熟悉基本的Thread概念和OpenThread Co-Processor架构，对RCP和NCP两种设备有一个简单了解。

### Linux主机

Linux主机（Ubuntu v20.04 LTS或更高版本）充当构建机器，用于设置Telink Zephyr开发环境并烧录所有Thread开发板。为了完成这些任务，Linux主机需要两个可用的USB端口和互联网连接。

### Telink B91开发套件

本教程需要2块B91开发板。下面的图片展示了一个套件中所需的最少组件。

<img src="img/overview.png" alt="overview.png" width="624.00" />

本教程将使用一块B91开发板作为RCP（无线电协处理器），使用另一个B91开发板作为NCP（网络协处理器）。
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

### 固件编译

本教程中将构建两种固件：

* `ot-ncp-ftd`
* `ot-rcp`

编译方法如下：

1. 无线电协处理器（ot-rcp）

```console
$ cd ~/zephyrproject
$ rm -rf build_ot_coprocessor
$ west build -b tlsr9518adk80d -d build_ot_coprocessor zephyr/samples/net/openthread/coprocessor -- -DDTC_OVERLAY_FILE="usb.overlay" -DOVERLAY_CONFIG=overlay-rcp-usb-telink.conf
```

2. 网络协处理器（ot-ncp-ftd）

打开位于 `zephyr/samples/net/openthread/coprocessor/overlay-rcp-usb-telink.conf` 文件，按如下示范进行修改。

```console
# Telink RCP USB-CDC-ACM

CONFIG_OPENTHREAD_COPROCESSOR_NCP=y
CONFIG_OPENTHREAD_COPROCESSOR_RCP=n
...
CONFIG_USB_DEVICE_PRODUCT="OpenThread CoProcessor NCP"
```

完成后打开位于 `zephyr/samples/net/openthread/coprocessor/boards/tlsr9518adk80d.conf` 文件，按如下示范进行修改。

```console
CONFIG_OPENTHREAD_NUM_MESSAGE_BUFFERS=256
```

然后执行以下命令编译 `ot-ncp-ftd` 固件。

```console
$ cd ~/zephyrproject
$ rm -rf build_ot_ncp_ftd
$ west build -b tlsr9518adk80d -d build_ot_ncp_ftd zephyr/samples/net/openthread/coprocessor -- -DDTC_OVERLAY_FILE="usb.overlay" -DOVERLAY_CONFIG=overlay-rcp-usb-telink.conf
```

## 小结

您现在已经知道：

* 如何搭建并使用Telink Zephyr开发环境。
* 如何构建 `ot-ncp-ftd` 和 `ot-rcp` 两种二进制文件。

在下半部分中，我们将介绍使用这两种固件分别与树莓派协同工作，创建和管理Thread网络的详细步骤。
