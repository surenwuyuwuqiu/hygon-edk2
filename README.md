#  hygon-edk2编译OVMF

云计算IaaS虚拟化平台需要针对国产信创进行定制化改造；
适配海光C86 CPU，为云服务器编译OVMF，定制efi固件，设置特定Logo;

## **1.背景**

​     需求来源：突显云平台和hygon的深入集成度，对外展示专用虚拟机。
​     技术背景：EDK II 是一个功能丰富的现代化跨平台固件开发环境，旨在为 UEFI (Unified Extensible Firmware Interface) 固件开发提供一套工具和框架, 该固件提供了操作系统bootloader以及其他与硬件交互的基础功能。
​                       OVMF 是一个基于 EDK II 的项目，用于为虚拟机提供 UEFI 支持。OVMF 包含用于 QEMU 和 KVM 的 UEFI 固件示例。

## **2.技术（资源）概述**

​    -efi 固件的定制：ovmf开发源码版本：https://gitee.com/anolis/hygon-edk2
​     当前版本的编译工具参考 : https://gitee.com/anolis/hygon-devkit/blob/master/csv/build_edk2.sh
​     对应的tianocore开源社区及文档wiki：[github.com/tianocore/edk2/tree/master/OvmfPkg](https://github.com/tianocore/edk2/tree/master/OvmfPkg) (实际参考)

注意事项

​     Note : 当前edk2的版本在编译完成后支持csv, 设置sev参数后也支持迁移操作
​     Note : 支持Hygon 和Intel 处理器设置Logo
​     将云平台OVMF_CODE.fd 和OVMF_VARS.fd替换为定制Logo的版本

## 3.方案调研

### 3.1 OVMF / Logo功能设计:

[edk2/OvmfPkg at master · tianocore/edk2](https://github.com/tianocore/edk2/tree/master/OvmfPkg)
  OVMF 概览：
  •	项目目标： 提供支持使用 edk2 代码库的虚拟机的固件。
  •	更多信息： OVMF 项目主页
  当前能力：
  •	架构： IA32 和 X64。
  •	支持的虚拟机： QEMU（1.7.1 版本或更高，带有 1.7 版或更高的机器类型）。
  •	UEFI 启动： 支持在 QEMU 上运行 UEFI Linux 和 UEFI Windows。
  构建 OVMF：
  •	先决条件： 具备构建 edk2 MdeModulePkg 的构建环境。
  •	配置 ASL 编译器： 可以使用 Intel ASL 编译器或 Microsoft ASL 编译器。
  •	NASM： 下载 NASM。
        构建步骤包括更新 Conf/target.txt 中的 ACTIVE_PLATFORM （目标Pkg）和 TARGET_ARCH（目标架构），然后执行 edk2 的构建过程。
  在 QEMU运行 OVMF：
  •	使用 QEMU 的 -pflash、-bios 或 -L 参数指定 OVMF 固件。
  •	支持 UEFI Shell，并自动运行，如果在可移动介质上找不到 UEFI时， 启动应用程序。
  •	在 Linux 上，某些新版本的 QEMU 可能启用了 KVM 特性，可以使用 -no-kvm 选项解决 OVMF 无法启动的问题。
  •	可以捕获 OVMF 的调试消息。
  SMM 支持：
  •	需要 QEMU 2.5 以上版本。
  •	支持 S3 挂起和恢复基础结构，以及 UEFI 变量驱动程序堆栈。
  网络支持：
  •	提供默认的 UEFI 网络堆栈，支持 NIC 驱动程序，支持 iPXE 驱动程序。
  •	支持 VirtioNetDxe 驱动程序。
  •	支持 Intel 的 E1000 NIC 驱动程序。
  HTTPS 启动：
  •	HTTPS 启动是替代 PXE 的解决方案，通过 HTTPS 服务器替换 TFTP 服务器。
  •	启用 HTTPS 启动需要构建 OVMF 时使用 -D NETWORK_HTTP_BOOT_ENABLE 和 -D NETWORK_TLS_ENABLE 选项。

### 3.2 MdeModulePkg 结构

```
  [MdeModulePkg]# tree -L 1
  ├── Application
  ├── Bus
  ├── Core
  ├── Include
  ├── Library
  ├── Logo
  ├── MdeModulePkg.ci.yaml
  ├── MdeModulePkg.dec
  ├── MdeModulePkg.dsc
  ├── MdeModulePkgExtra.uni
  ├── MdeModulePkg.uni
  ├── Test
  └── Universal
```

  MdeModulePkg 是一个包含多个模块的 EDK II (EFI Development Kit II) 模块包（Package）。这个pkg包含了一些应用程序、驱动程序、库以及与 UEFI 固件开发相关的模块。下面是对这些子目录的简要说明：
  •	Application: 包含一些应用程序，如 Boot 管理器、Capsule 应用等。
  •	Bus: 包含总线驱动模块，涵盖了一些总线协议，比如 ATA、I2C、PCI、USB 等。
  •	Core: 包含一些核心模块，涉及到 DXE、PEI、SMM 等。
  •	Include: 包含一些头文件，可能是一些协议、PPI 或者库的定义。
  •	Library: 包含一系列库模块，提供了各种功能，比如认证变量库、显示处理库、文件系统库等。
  •	Logo: 包含了与 Logo 相关的模块，如 Logo 显示处理模块、Logo 的定义文件等。
  •	MdeModulePkg.ci.yaml: CI (Continuous Integration) 配置文件，用于自动化构建和测试。
  •	MdeModulePkg.dec: 包含了 MdeModulePkg 模块的声明文件，定义了该模块的基本信息。
  •	MdeModulePkg.dsc: DSC (Dec Specification) 文件，包含模块的详细配置信息，包括模块的依赖关系、编译选项等。
  •	MdeModulePkgExtra.uni: 包含了额外的国际化字符串信息。
  •	MdeModulePkg.uni: 包含了 MdeModulePkg 模块的国际化字符串。
  •	Test: 包含了测试相关的模块。
  •	Universal: 包含了一些通用的模块，如 ACPI、BdsDxe、DebugPortDxe、FileExplorerDxe 等。
  在这个文件树中，每个模块都有自己的源代码文件、头文件、声明文件（.dec）、配置文件（.dsc）、国际化字符串文件（.uni）、信息文件（.inf）等。这些文件共同构成了一个 EDK II 模块包，用于支持 UEFI 固件的开发。
  如果有一个新的 Logo 文件（如 Logo_new.bmp），可能需要进行以下步骤：

    1.	将新的 Logo 文件（Logo_new.bmp）放入 Logo 目录中。
    2.	修改 Logo.inf 文件，将 Logo_new.bmp 添加到 [Binaries] 部分。
    3.	执行编译和构建步骤，确保新的 Logo 被正确地集成到 UEFI 固件中。

### 3.3 Logo结构

  1. ### 

     ```
        [Logo]# tree -L 1
       .
       ├── Logo.bmp
       ├── Logo.c
       ├── LogoDxeExtra.uni
       ├── LogoDxe.inf
       ├── LogoDxe.uni
       ├── LogoExtra.uni
       ├── Logo.idf
       ├── Logo.inf
       └── Logo.uni
     ```

       Logo 目录中，包含了 Logo 模块的各种文件。以下是每个文件的简要说明：
       •	Logo.bmp: Logo 图像文件，是显示在设置屏幕上的默认 Logo 图片。——图片格式需指定位256位BMP
       •	Logo.c: Logo 模块的源代码文件，实现了与 Logo 相关的功能。
       •	LogoDxeExtra.uni: 包含 Logo 模块的本地化字符串和内容。
       •	LogoDxe.inf: Logo 模块的信息文件，定义了模块的属性、依赖关系等。
       •	LogoDxe.uni: 包含 Logo 模块的国际化字符串，提供了关于模块的描述信息。
       •	LogoExtra.uni: Logo 模块的附加国际化字符串文件。
       •	Logo.idf: Logo 图像的定义文件，指定了 Logo 图像的文件名和标识符。
       •	Logo.inf: Logo 模块的信息文件，类似于 LogoDxe.inf，包含了 Logo 图像的信息。
       •	Logo.uni: 包含 Logo 模块的国际化字符串，提供了关于模块的描述信息。

<img width="461" height="387" alt="image" src="https://github.com/user-attachments/assets/7f4bf522-0b77-4547-b0e5-8019d2b7f1c8" />
<img width="504" height="452" alt="image" src="https://github.com/user-attachments/assets/d81218f2-fc1c-4617-9207-9fd6b485c3a0" />

8位：  2^8 = 2^2(B) 2^3(G) 2^3(R) = 256  (256色)    可以总共显示256种颜色
16位：2^16 = 2^5(B) 2^6(G) 2^5(R) =  65536    可以总共显示65536种颜色
24位：2^24 = 2^8(B) 2^8(G) 2^8(R) =  16777216    可以总共显示16777216种颜色
32位：Alpha透明度 + 24位
  当8/16位深度时，单个原始颜色 （R/G/B）最大只能表示为（0~2^3）/（0~2^6）， 无法满足（0~0xff）的范围，所以显示的颜色范围有限。
  当24位深度时，使用24bit显示一个像素点， 由8bit Red 8bit Green 8bit Blue组合颜色而成，每一个原始颜色（R/G/B）都可以完全显示（0~0xff），所以24位及以上，我们就叫做真彩色
  当32位深度时，与24位相同，可以显示所有的颜色，同时多了一个透明度值。

## 4. OVMF环境搭建及编译

### 4.1 创建编译环境

在本地创建Ubuntu虚拟机，在Linux环境下安装编译环境：Common instructions · tianocore/tianocore.github.io Wiki

1. clone 海光对应的edk2，建议通过git clone方式，而不是下载edk2.zip手动解压，否则无法正确执行Step 3

2. cd到该项目目录

3. git 初始化该项目的子模块

```
  git clone https://gitee.com/anolis/hygon-edk2.git 
  cd hygon-edk2/
  git submodule update --init --recursive
```

  应看到子模块下载和注册的提示：代表子模块正常下载

  可能遇到的问题： "Failed to connect to github.com port 443"
  原因：本地可能开启代理，建议配置git环境变量
  解决：为Git单独配置代理，或者配置系统的环境变量
  解决方案1：

```
  git config --global http.proxy http://{ip}:{port}
  git config --global https.proxy http://{ip}:{port}
```

再次执行git clone / git submodule update 即可恢复正常

解决方案2

```
  export http_proxy=http://proxy.domain.com:proxy_port
  export ftp_proxy=$http_proxy
```

​    

### **4.2 配置工具**

  1. Install required software from apt
     #为 EDK II 设置构建环境需要使用几个 Ubuntu 软件包。以下命令将安装所有需要的软件包：

     ```
     sudo apt install build-essential uuid-dev iasl git  nasm  python-is-python3
     ```

  2. Compile build tools

     ```
     cd ~/src/edk2
     make -C BaseTools
     . edksetup.sh
     ```

       完成以上步骤后，就可以在 edk2 目录下进行代码开发了。

  3. 设置构建 shell 环境和sources.list

     ```
       cd ~/workspace/hygon-edk2
       export EDK_TOOLS_PATH=$HOME/workspace/hygon-edk2/BaseTools
       . edksetup.sh BaseTools
     ```

       Ubuntu20.04的gcc版本默认为9.0+, tianno社区的教程&&Hygon的脚本按Ubuntu16.04 的 gcc 5+ / gcc 4 +。
       实操中默认的GCC9 未正确识别，要安装gcc5 + ,需配置对应的source.list： 

     ```
       vi /etc/apt/sources.list
     ```

       在默认追加以下内容：

     ```
       deb http://mirrors.aliyun.com/ubuntu/ xenial main
       deb-src http://mirrors.aliyun.com/ubuntu/ xenial main
     
       deb http://mirrors.aliyun.com/ubuntu/ xenial-updates main
       deb-src http://mirrors.aliyun.com/ubuntu/ xenial-updates main
     
       deb http://mirrors.aliyun.com/ubuntu/ xenial universe
       deb-src http://mirrors.aliyun.com/ubuntu/ xenial universe
       deb http://mirrors.aliyun.com/ubuntu/ xenial-updates universe
       deb-src http://mirrors.aliyun.com/ubuntu/ xenial-updates universe
     
       deb http://mirrors.aliyun.com/ubuntu/ xenial-security main
       deb-src http://mirrors.aliyun.com/ubuntu/ xenial-security main
       deb http://mirrors.aliyun.com/ubuntu/ xenial-security universe
       deb-src http://mirrors.aliyun.com/ubuntu/ xenial-security universe
     ```

     

  4. 安装gcc5

     ```
     sudo apt-get update
     ```

输入命令，查看gcc5可选的版本

```
  apt-cache policy gcc-5
  sudo apt install g++-5 gcc-5
```

通过命令查询本机gcc已安装的版本

```
ls /usr/bin/gcc*
```

可以看到有gcc9和gcc5

这个时候需要管理多版本的gcc，使我们想要的gcc5成为默认版本

输入命令

```
  sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-5 40
  sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-9 50
  sudo update-alternatives --config gcc
```

根据提示，选择gcc-5对应的编号1 回车即可

最后gcc -v查看默认gcc版本，应当已经切换为gcc5

如果要对g++的多版本进行管理，只需将上面命令行中的gcc替换为g++

  5. 编辑edk2配置文件

     ```
     vi /hygon-edk2/Conf/target.txt
     ```

     ![image-20251107102316452](D:\File Storage\TyporaStorage\image-20251107102316452.png)

### **4.3 定制Logo.bmp 并build，改造点**

```
  cd hygon-edk2/OvmfPkg/
  sudo ./build.sh
```

  build会创建3各不同镜像:

  OVMF_VARS.fd : 这是持久化的UEFI变量的firmware卷，即firmware存储所有配置(引导条目和引导顺序、安全引导密钥等)。通常这个文件用作空变量存储的模板，每个VM都有自己的私有副本。例如 Libvirt虚拟机管理器 将文件存储在 /var/lib/libvirt/qemu/nvram 中。

  OVMF_CODE.fd : 带有代码的firmware卷。将它和 VARS 分开可以:确保轻松更新固件、允许将只读代码映射到guest操作系统

  OVMF.fd : 是包含 CODE 和 VARS 的一体化镜像。这样就可以使用 -bios 参数直接作为ROM加载。但是有2个缺点:UEFI 变量不是持久的、不适用于 SMM_REQUIRE=TRUE 构建

### **4.4 运行QEMU测试OVMF**

  在Ubuntu上运行qemu测试效果

```
  sudo qemu-system-x86_64 -drive if=pflash,format=raw,file=/home/han/workspace/hygon-edk2/Build/OvmfX64/DEBUG_GCC5/FV/OVMF_CODE.fd

  sudo qemu-system-x86_64 -drive if=pflash,format=raw,file=/home/han/workspace/hygon-edk2/Build/OvmfX64/DEBUG_GCC5/FV/OVMF_VARS.fd,if=pflash,format=raw,file=/home/han/workspace/hygon-edk2/Build/OvmfX64/DEBUG_GCC5/FV/OVMF_CODE.fd
```

  **使用virt-manager 创建VM并修改对应的xml文件，可观测到VM启动时有该界面**
   <img width="624" height="377" alt="image" src="https://github.com/user-attachments/assets/cecc77a8-3d9e-4f8f-bd30-69f93ae61e1c" />

### **4.5 edk2 编译支持中文  // 暂不支持，无中文字体库**

  <img width="624" height="545" alt="image" src="https://github.com/user-attachments/assets/8d88cf6b-ced5-4ae5-871c-30fc3b779946" />
  <img width="624" height="549" alt="image" src="https://github.com/user-attachments/assets/ae9c1326-051e-4d79-ad33-e54c0bcdf9aa" />
UEFI中采用UCS-2的编码方式，即固定两个字节存一个字符的编码，需要显示该字符到显示器时，会通过字符编码来查询它的位图数据，从而实现字符显示。

edk2源码中默认采用SimpleFont格式的点阵字体，字体点阵数据位于

```
 MdeModulePkg\Universal\Console\GraphicsConsoleDxe\ LaffStd.c
```

从点阵数据中可以看出，字符编码为0x0020的点阵数据为全0，因为0x20是空格的ascii码，空格什么也不用显示，所以为全0。
字符编码为0x0021对应为感叹号的ascii字符，如下图所示，将它的数据展开，由1组成的位图显示正好是个感叹号。
  <img width="624" height="436" alt="image" src="https://github.com/user-attachments/assets/c67cb6d2-75f1-460b-b440-2e9cc56b59f0" />

  <img width="509" height="647" alt="image" src="https://github.com/user-attachments/assets/abf0a05e-af1c-4cbd-8d35-fcfffcb8fefb" />

中文没有显示出来，这是因为UEFI采用位图（点阵图像）的方式进行字符显示，而edk2源码中默认只有英文和法文字库的位图数据。中文需要用到16x19的宽体才能够容纳显示。
当前支持中文的教程仅有修改UEFI和BIOS，未发现有修改OVMF的部分。
