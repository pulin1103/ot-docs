# 泰凌微电子Thread RCP和NCP方案介绍（二）

## 介绍

本教程的下半部分将演示使用[Telink LinuxBDT工具](http://wiki.telink-semi.cn/tools_and_sdk/Tools/BDT/LinuxBDT.tar.bz2)将OpenThread RCP和 NCP固件烧录到 Telink B91 开发板，并且分别与树莓派协同工作，创建和管理Thread网络的所必须的步骤。

### 学习内容

* 分别将OpenThread Co-Processor固件（ ot-ncp-ftd 和 ot-rcp ）烧录到两块Telink B91开发板上。

* 在Raspberry Pi 3B+或更高版本上，使用Docker和RCP搭建OpenThread边界路由器（OTBR）。

* 在Raspberry Pi 3B+或更高版本上，使用 `Pyspinel` 验证NCP功能。

### 所需条件

硬件：

* 2块B91开发板。

* 1台Raspberry Pi 3B+或更高版本，并安装Raspbian操作系统映像。

* 1台Linux主机，至少带有两个USB端口。

* 1个已连接互联网的交换机（或路由器）和若干条以太网电缆。

软件：

* Telink烧录和调试工具 —— LinuxBDT。

* 其他工具，比如Git和West。

## 固件设置

### 泰凌LinuxBDT设置

下载[Telink LinuxBDT工具](http://wiki.telink-semi.cn/tools_and_sdk/Tools/BDT/LinuxBDT.tar.bz2)，并将其解压到Linux主机的本地目录，例如 `~`，以允许用户将固件烧录到B91开发板。

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

### 固件烧录

如下图所示，使用USB连接线将一块Telink B91开发板连接到Telink烧录板。

<img src="img/connection_overview.png" alt="connection_overview.png" width="624.00" />

在命令行输入如下指令（以烧录ot-ncp-ftd固件为例）。

```console
$ cd ~/zephyrproject/build_ot_ncp_ftd/zephyr
$ cp zephyr.bin ~/LinuxBDT/bin/ot-ncp-ftd.bin
$ cd ~/LinuxBDT
$ sudo ./bdt 9518 ac
 Activate OK!
$ sudo ./bdt 9518 wf 0 -i bin/ot-ncp-ftd.bin
 EraseSectorsize...
 Total Time: 2181 ms
 Flash writing...
 [100%][-] [##################################################]
 File Download to Flash at address 0x000000: 491700 bytes
 Total Time: 30087 ms
```

ot-rcp 的烧录方法和 ot-ncp-ftd 的基本一样，不同之处在于固件名称。烧录完成后分别将两块B91开发板做好标记区分，烧录 ot-ncp-ftd 的开发板标记为“NCP”，烧录 ot-rcp 的开发板标记为“RCP”。

## 固件应用

本教程使用树莓派来验证RCP和NCP两种固件功能

* 树莓派安装Docker作为OTBR的Host端，验证RCP功能。

* 树莓派安装并运行Pyspinel，验证NCP功能。

### 树莓派

1. 确保写入SD卡中的是[Raspbian Bullseye Lite OS image](https://downloads.Raspberrypi.org/raspios_lite_armhf/images/raspios_lite_armhf-2023-05-03/2023-05-03-raspios-bullseye-armhf-lite.img.xz)或[Raspbian Bullseye with Desktop](https://downloads.Raspberrypi.org/raspios_armhf/images/raspios_armhf-2023-05-03/2023-05-03-raspios-bullseye-armhf.img.xz)

2. 您可以选择通过SSH连接到树莓派，也可以直接在Raspbian桌面上操作。本教程将使用SSH。

3. 在下一步安装OTBR Docker或Pyspinel之前，先更新本地代码库和软件包管理器。

```console
$ sudo apt-get update
$ sudp apt-get upgrade
```

### 无线电协处理器（RCP）

ot-rcp固件的烧录步骤参考ot-ncp-ftd烧录过程，将B91开发板连接到树莓派的USB端口上，连接方式如下图所示。

<img src="img/OTBR_overview.png" alt="OTBR_overview.png" width="624.00" />

#### 安装Docker

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

#### 配置并运行Docker

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

5. 新开一个SSH终端窗口，测试树莓派和RCP的连通性，并建立Thread网络。

```console
$ docker exec -ti otbr sh -c "sudo ot-ctl"
> state 
disabled
Done
> panid 0x1022 
Done
> ifconfig up
Done
> thread start 
Done
> state 
detached
Done
> state 
leader
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

### 网络协处理器（NCP）

关闭树莓派，拔掉RCP。将NCP连接到树莓派的USB端口上，再重新启动树莓派。连接方式如下图。

<img src="img/NCP_overview.png" alt="NCP_overview.png" width="624.00" />

#### 安装Pyspinel

在树莓派上打开一个新的SSH终端窗口。

1. 安装依赖项：

```console
$ sudo apt install python3-pip
$ pip3 install --user pyserial ipaddress
```

2. 下载 `pyspinel` 的源码到本地：

```console
$ git clone https://github.com/openthread/pyspinel
```

3. 安装Pyspinel：

```console
$ cd pyspinel
$ sudo python3 setup.py install
```

#### 验证NCP功能

1. 配置NCP连接。

```console
$ sudo chmod a+rw /dev/ttyACM0
```

2. 运行Pyspinel CLI。

```console
$ spinel-cli.py -u /dev/ttyACM0 -n 1
spinel-cli >
```

3. 查询NCP版本。

```console
spinel-cli > version
OPENTHREAD/aabbee49c; Zephyr; Aug 10 2023 14:02:37
Done
```

4. 建立Thread网络。

```console
spinel-cli > ifconfig up
Done
spinel-cli > thread start
Done
spinel-cli > state
detached
Done
spinel-cli > state
leader
Done
```

可以看到NCP已成为leader，Thread网络被成功创建。

可选用的spinel-cli命令：

* 查看帮助菜单获取可用命令。

```console
spinel-cli > help

Available commands (type help <name> for more information):
============================================================
bufferinfo         extaddr       ncp-filter        releaserouterid
channel            extpanid      ncp-ll64          reset
child              h             ncp-ml64          rloc16
childmax           help          ncp-raw           route
childtimeout       history       ncp-tun           router
clear              ifconfig      netdata           routerdowngradethreshold
commissioner       ipaddr        networkidtimeout  routerselectionjitter
contextreusedelay  joiner        networkkey        routerupgradethreshold
counters           keysequence   networkname       scan
debug              leaderdata    panid             state
debug-mem          leaderweight  parent            thread
diag               mac           ping              txpower
discover           macfilter     prefix            v
eidcache           mfg           q                 vendor
exit               mode          quit              version
```

## 总结

您现在已经知道：

* 如何搭建并使用Telink Zephyr开发环境。
* 如何构建 `ot-ncp-ftd` 和 `ot-rcp` 两种二进制文件并将其烧录到B91开发板。
* 如何使用Docker和RCP将Raspberry Pi 3B+ 或更高版本设置为OpenThread边界路由器（OTBR）。
* 在Raspberry Pi 3B+或更高版本上，使用 `Pyspinel` 验证NCP功能。

由此可见，RCP和NCP方案都可以实现OTBR的功能。不过，从目前的社区的支持力度看，RCP更适合用于OTBR的开发。

### 深入阅读

查看[openthread.io](https://openthread.io/)和[GitHub](https://github.com/openthread)，了解各种OpenThread资源，包括：

* [Supported Platforms](https://openthread.io/platforms/)
    — discover all the platforms that support OpenThread
* [Build OpenThread](https://openthread.io/guides)
    — further details on building and configuring OpenThread
* [Thread Primer](https://openthread.io/guides/thread-primer)
    — covers all the Thread concepts featured in this codelab

参考文档:

* [OpenThread Co-Processor Designs](https://openthread.io/platforms/co-processor)
* [OpenThread Pyspinel](https://openthread.io/guides/pyspinel)
* [OpenThread Border Router](https://openthread.io/guides/border-router)
