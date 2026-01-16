# Driver

## User

假设你是一个精通 GPU 相关的高级研究人员, 在使用 GPU 之前需要按照驱动, nvidia 的驱动下载后看起来只是一个 shell 脚本, 但是我并不能打开脚本看看里面到底有什么, 安装驱动到底要执行哪些操作, 所以我想知道是否有办法打开这个 shell 脚本, 或者在哪里可以看到执行这个脚本的详细日志, 例如添加参数 +x 执行脚本

## Model

你好! 作为一名在 GPU 架构和驱动底层浸淫多年的研究人员, 我很理解你的好奇心. NVIDIA 的 .run 安装包确实看起来像个"黑盒", 但实际上它是一个自解压的可执行脚本(Self-extracting archive), 通常是基于 makeself 构建的.

你完全可以拆解它、观察它, 甚至调试它. 以下是深度探索 NVIDIA 驱动安装包的几种专业方法:

1. 如何"打开"并查看脚本内容?

虽然你用 vim 或 cat 打开它会看到大量乱码(因为脚本后面紧跟着压缩的二进制数据), 但你可以通过参数将其内容解压出来.

解压而不安装:

执行以下命令, 它会将所有内容提取到当前目录下的一个同名文件夹中:

```Bash
sh ./NVIDIA-Linux-x86_64-xxx.yy.run -x
```

或者使用:

```Bash
./NVIDIA-Linux-x86_64-xxx.yy.run --extract-only
```

解压后你会看到什么?

进入解压后的目录, 你会发现:

* nvidia-installer: 真正的 C 语言编写的安装程序二进制文件.

* kernel/: 驱动内核模块(.ko)的源代码. 这是最有价值的部分, 你可以看到 NVIDIA 如何与 Linux 内核接口(如 nvidia.ko, nvidia-modeset.ko, nvidia-uvm.ko)进行交互.

* libglvnd, libcuda.so.*: 预编译的用户态动态库.

* html/: 详细的离线文档.

2. 运行时的详细日志在哪里?

如果你想知道安装过程中每一步到底发生了什么, 有几个渠道:

官方安装日志(最权威):

无论安装成功还是失败, NVIDIA 都会记录详细日志到:

```Bash
/var/log/nvidia-installer.log
```

这个日志会记录它检测到的内核版本、调用的编译器命令、复制文件的路径以及遇到的错误.

使用详细模式运行:

在安装时添加 -v (verbose) 参数:

```Bash
sudo ./NVIDIA-Linux-x86_64-xxx.yy.run --verbose
```

Shell 脚本层面的调试:

如果你想看 .run 包装脚本(那个 shell 头部)是如何运行的, 可以使用 shell 的调试模式:

```Bash
sh -x ./NVIDIA-Linux-x86_64-xxx.yy.run [其他参数]
```

注意: 这只能让你看到解压和启动 nvidia-installer 之前的逻辑, 真正的安装逻辑是在解压出来的 nvidia-installer 二进制文件里.

3. 高级研究员视角: 安装驱动到底在执行什么操作?

从底层视角来看, 执行这个脚本主要完成了以下五个核心步骤:

内核接口构建 (Kernel Interface Build):
由于 Linux 内核没有稳定的 ABI, 驱动必须针对你当前的内核版本现场编译. 脚本会进入 kernel/ 目录, 调用 gcc 和 make, 利用内核头文件(kernel-headers)生成 .ko 文件.

内核模块加载:
它会尝试将生成的模块加载到内核(insmod 或 modprobe). 如果原有的驱动(如开源的 Nouveau)未卸载, 这步会报错.

用户态库拷贝:
将 libcuda.so, libGL.so, libnvidia-encode.so 等库文件拷贝到系统的标准库路径(如 /usr/lib64 或 /usr/lib/x86_64-linux-gnu).

配置动态链接器:
运行 ldconfig 更新缓存, 并创建符号链接. 它还会安装 nvidia-smi 这种管理工具.

更新 Initramfs(可选但关键):
在某些发行版上, 它会触发 dracut 或 update-initramfs, 确保在系统启动早期就能加载驱动.

4. 几个专业建议

DKMS(Dynamic Kernel Module Support): 强烈建议在安装时加上 --dkms 参数. 这样当你以后更新系统内核时, 驱动会自动重新编译, 不需要你手动再跑一遍 .run 脚本.

依赖检查: 在运行脚本前, 确保你已经安装了 gcc, make 和 kernel-devel (或 kernel-headers).

卸载残留: 如果你想清理环境, 可以使用 --uninstall 参数.

总结:

你想看内部逻辑, 请使用 --extract-only; 你想看运行细节, 请查阅 /var/log/nvidia-installer.log. 如果你对 GPU 内核态和用户态的通信感兴趣, 解压后的 kernel/ 源码目录是最好的教科书.

## User

下载好驱动文件 NVIDIA-Linux-x86_64-xxx.yy.run 之后, 如何查看它有哪些运行的参数, 例如 --extract-only 这种

## Model

作为一个研究人员, 探索工具的最佳入口永远是它的帮助文档(Help Manual). NVIDIA 的 .run 文件虽然是一个打包好的二进制+脚本混合体, 但它内置了非常详细的命令行选项说明.

你可以通过以下几种方式查看它支持的所有参数:

1. 最直接的方法: 查看基础帮助

在终端中输入以下命令:

```Bash
./NVIDIA-Linux-x86_64-xxx.yy.run --help
```

或者使用简写:

```Bash
./NVIDIA-Linux-x86_64-xxx.yy.run -h
```

这将列出最常用的基础参数, 比如安装路径设置、静默安装等.

2. 核心进阶: 查看"高级"选项(研究员必备)

基础帮助通常只展示 20% 的常用命令. 作为高级用户, 你一定想看剩下的 80% 隐藏参数, 这需要使用:

```Bash
./NVIDIA-Linux-x86_64-xxx.yy.run --advanced-options
```

或者简写:

```Bash
./NVIDIA-Linux-x86_64-xxx.yy.run -A
```

这个命令会展示极其详尽的参数列表, 涵盖了内核模块编译、兼容性层、专家级调试等.

3. 你可能会用到的"硬核"参数分类

为了方便你研究, 我整理了几个在研究和深度调试中经常用到的参数:

A. 提取与自检类

--extract-only: 仅解压, 不执行安装. 这是研究其目录结构的第一步.

--check: 检查安装包的完整性(校验 MD5/SHA).

--info: 显示该安装包的版本、构建日期等元数据.

B. 自动化与静默类 (常用于服务器集群部署)

-s / --silent: 静默安装, 不弹出任何交互式界面.

-a / --accept-license: 自动接受用户许可协议(EULA).

-X / --run-nvidia-xconfig: 安装后自动修改 X11 配置(在服务器/计算卡环境下通常不需要).

C. 内核与编译类 (底层调试核心)

--dkms: 强烈推荐. 将驱动注册到 DKMS. 这样当 Linux 内核更新时, 驱动会自动重新编译, 避免系统升级后掉驱动.

--no-kernel-module: 只安装用户态库(如 CUDA 运行库), 不安装内核驱动. 常用于容器环境.

--kernel-source-path=<path>: 手动指定内核源码路径, 如果你的内核是自定义编译的, 这个非常有用.

--no-precompiled-interface: 强制从源码编译内核接口, 而不是尝试使用预编译的二进制.

D. 路径与清理类

--prefix=<path>: 指定安装前缀(默认是 /usr).

--uninstall: 卸载驱动.

4. 技巧: 如何离线阅读这些参数?

如果你觉得终端翻页不方便, 可以将这些参数输出到一个文本文件中慢慢研究:

```Bash
./NVIDIA-Linux-x86_64-xxx.yy.run -A > nvidia_options.txt
```

5. 补充: 如果文件没有执行权限

如果你刚下载下来执行报错, 记得先给它权限:

```Bash
chmod +x NVIDIA-Linux-x86_64-xxx.yy.run
```

或者直接用 sh 调用:

```Bash
sh ./NVIDIA-Linux-x86_64-xxx.yy.run --advanced-options
```

专家提示:

如果你在研究驱动对特定 Linux 内核版本的兼容性, 建议配合 grep 使用. 例如想看所有和内核(kernel)相关的参数:

```bash
./NVIDIA-Linux-x86_64-xxx.yy.run -A | grep -i kernel
```

## User

在驱动安装过程中还会安装 nvidia-smi 这种管理工具, 在哪里可以找到驱动安装过程中安装的所有工具

## Model

在 NVIDIA 驱动安装完成后, 你可以通过多种方式找到它安装的"所有工具"和相关文件. 由于 NVIDIA 驱动不仅包含 `nvidia-smi` 这种命令行工具, 还包含大量的动态库(`.so`)、内核模块(`.ko`)和配置文件, 以下是作为研究员最常用的几种"盘点"方法:

1. 查看安装日志(最精准的清单)

NVIDIA 安装程序执行的所有文件复制操作都会详细记录在日志文件中. 这是查看你的系统上到底安装了什么的权威依据.

路径: `/var/log/nvidia-installer.log`
操作: 你可以使用 `grep` 搜索关键词 `installing` 或直接查看日志结尾部分:

```bash
cat /var/log/nvidia-installer.log | grep "Installing"
```

日志会清楚地记录每个文件从安装包内部路径到系统路径(如 `/usr/bin`、`/usr/lib64`)的映射关系.

2. 核心工具的安装路径

绝大多数用户态工具都会被安装到系统的标准可执行路径中, 通常在 `/usr/bin` 或 `/usr/local/bin`.

你可以通过以下命令一次性找到它们:

```bash
ls /usr/bin/nvidia-*
ls /usr/local/bin/nvidia-*

$ ls -al /usr/bin/nvidia-*
-rwxr-xr-x 1 root root   40769 Sep 11 23:25 /usr/bin/nvidia-bug-report.sh
-rwxr-xr-x 1 root root 5292232 Dec  4  2024 /usr/bin/nvidia-cdi-hook
-rwxr-xr-x 1 root root   51568 Dec  4  2024 /usr/bin/nvidia-container-cli
-rwxr-xr-x 1 root root 5652808 Dec  4  2024 /usr/bin/nvidia-container-runtime
-rwxr-xr-x 1 root root 4127624 Dec  4  2024 /usr/bin/nvidia-container-runtime-hook
lrwxrwxrwx 1 root root      38 Jul 25 15:50 /usr/bin/nvidia-container-toolkit -> /usr/bin/nvidia-container-runtime-hook
-rwxr-xr-x 1 root root 9306608 Dec  4  2024 /usr/bin/nvidia-ctk
-rwxr-xr-x 1 root root   54088 Sep 11 23:25 /usr/bin/nvidia-cuda-mps-control
-rwxr-xr-x 1 root root   18656 Sep 11 23:25 /usr/bin/nvidia-cuda-mps-server
-rwxr-xr-x 1 root root  142064 Sep 11 23:25 /usr/bin/nvidia-debugdump
-rwxr-xr-x 1 root root     270 Oct  7  2024 /usr/bin/nvidia-detector
-rwxr-xr-x 1 root root  372128 Sep 11 23:25 /usr/bin/nvidia-installer
-rwsr-xr-x 1 root root   39248 Sep 11 23:25 /usr/bin/nvidia-modprobe
-rwxr-xr-x 1 root root 5043392 Sep 11 23:25 /usr/bin/nvidia-ngx-updater
-rwxr-xr-x 1 root root  208336 Sep 11 23:25 /usr/bin/nvidia-persistenced
-rwxr-xr-x 1 root root  229888 Sep 11 23:25 /usr/bin/nvidia-powerd
-rwxr-xr-x 1 root root  306312 Sep 11 23:25 /usr/bin/nvidia-settings
-rwxr-xr-x 1 root root     900 Sep 11 23:25 /usr/bin/nvidia-sleep.sh
-rwxr-xr-x 1 root root 1068640 Sep 11 23:25 /usr/bin/nvidia-smi
lrwxrwxrwx 1 root root      16 Sep 11 23:25 /usr/bin/nvidia-uninstall -> nvidia-installer
-rwxr-xr-x 1 root root  207424 Sep 11 23:25 /usr/bin/nvidia-xconfig
$

```

常见的工具包括:

* `nvidia-smi`: 监控与管理(查询功率、显存、进程).
* `nvidia-settings`: 桌面环境下用于调节亮度和显示配置的 GUI 工具.
* `nvidia-xconfig`: 用于自动生成或修改 X11 配置文件.
* `nvidia-modprobe`: 用于在用户态自动加载内核模块并创建设备节点.
* `nvidia-debugdump`: 专家级工具, 用于捕获 GPU 核心转储(Core Dump).
* `nvidia-cuda-mps-control`: 用于多进程服务(MPS)的控制工具.

3. 查看安装包解压后的"蓝图"

如果你想在安装**之前**就知道会有哪些工具, 可以使用我们之前提到的解压参数:

```bash
sh ./NVIDIA-Linux-x86_64-xxx.yy.run -x

cd NVIDIA-Linux-x86_64-xxx.yy/
```

在这个目录下, 你会看到:

* 所有二进制工具的前身(通常是直接的可执行文件).
* `nvidia-installer`: 驱动安装程序本身.
* `html/` 目录: 里面有详细的官方文档(`index.html`), 其中专门有一章叫 "Listing of Installed Components", 会列出该版本驱动包含的所有文件及其默认安装位置.

4. 库文件与内核模块

除了"工具", 驱动的大部分核心其实是在下面这两个地方:

* 共享库 (Libraries): 搜索 `/usr/lib64` 或 `/usr/lib/x86_64-linux-gnu` 下以 `libnvidia-` 开头的文件.

```bash
ls /usr/lib64/libnvidia-* /usr/lib64/libcuda.so*
```
这里你会找到 `libnvidia-ml.so` (NVML 库, `nvidia-smi` 的底层依赖) 和 `libcuda.so` (CUDA 核心驱动库).

* 内核模块 (Kernel Modules): 驱动的核心组件, 安装在:

```bash
ls /lib/modules/$(uname -r)/kernel/drivers/video/nvidia*.ko
```

5. 快速卸载与清理

如果你想知道安装了什么是为了后续能卸载干净, 驱动自带了自清理功能:

```bash
nvidia-installer --uninstall
```

或者直接运行:

```bash
/usr/bin/nvidia-uninstall
```

总结:

如果你只想找可执行的管理工具, 直接看 `/usr/bin/nvidia-*`. 如果你需要完整的文件清单, 请查阅 `/var/log/nvidia-installer.log`.

## User

在进行 cuda 编程时, 需要安装 cuda 相关的库和工具, 在 Ubuntu 发行版中使用包管理工具 apt 安装 cuda 时, 会自动安装相关的 gpu 驱动吗? cuda 应该会依赖于 gpu 驱动

## Model

作为一个深耕 GPU 计算领域的研究人员, 我可以非常肯定地回答你: 在大多数情况下, 是的.

当你使用 NVIDIA 官方提供的 apt 源安装 cuda 这个"元包(Meta-package)"时, 它会自动触发驱动程序的安装.

为了让你在研究过程中少走弯路, 我需要拆解一下这背后的包依赖逻辑以及常见的陷阱.

1. 核心依赖链(为什么会自动安装)

在 Debian/Ubuntu 的包管理体系中, cuda 包并不是一个包含代码的实体, 而是一个容器(元包). 它的依赖关系通常如下:

cuda (元包)
    依赖 cuda-12-x (具体版本的元包)
        依赖 cuda-runtime-12-x (运行环境)
            依赖 cuda-drivers (驱动元包)
                依赖 nvidia-driver-xxx (真正的驱动程序)
        依赖 cuda-toolkit-12-x (编译器 nvcc、头文件、工具)

当你执行 sudo apt install cuda 时, apt 会解析这个长长的链条, 发现系统中没有满足要求的 nvidia-driver, 因此会自动将其加入待安装列表.

2. 关键: cuda vs cuda-toolkit (研究员必须分清)

这是很多初学者甚至高级开发者都会踩的坑. 根据你的需求, 你应该选择不同的包:

如果你执行 sudo apt install cuda:

结果: 安装 CUDA 所有的库、头文件、示例, 以及配套的最新驱动.

风险: 如果你系统里已经手动跑过 .run 脚本安装了驱动, apt 会再次安装驱动, 导致驱动版本冲突, 甚至出现内核模块加载失败.

如果你执行 sudo apt install cuda-toolkit:

结果: 只安装编译器(nvcc)、开发库(CUBLAS, CURAND 等)和头文件. 它不依赖驱动.

场景: 这种方式最适合"驱动已经装好了"或者"在 Docker 容器内编译程序"的情况.

3. 安装过程中的三个"冷知识"

作为高级研究员, 你需要关注这些底层细节:

A. 驱动版本的前向兼容性(Forward Compatibility)

以前, CUDA 版本必须配合特定版本的驱动(比如 CUDA 11 必须驱动 > 450). 但从 CUDA 11 开始, NVIDIA 引入了 CUDA Forward Compatibility SDK. 这意味着在数据中心卡(如 A100/H100)上, 即使驱动版本较低, 通过安装特定的兼容性包, 也能运行高版本的 CUDA 程序.

B. 解决"驱动不匹配"的风险

如果你使用 apt 安装, 驱动会被安装到系统的标准内核模块路径. 如果以后你又尝试用 .run 脚本更新驱动, 系统会变得非常混乱.

专家建议: 既然你选择了 Ubuntu, 尽量全程使用 apt 管理驱动和 CUDA, 不要混用 .run 脚本.

C. nvidia-utils 与 nvidia-smi

当你通过 apt 安装 cuda 导致驱动被安装时, nvidia-smi 这个工具通常被包含在 nvidia-utils-xxx 包里. apt 会通过依赖关系自动把这个管理工具也装上.

4. 最佳实践流程

如果你正在配置一台新的 GPU 服务器, 我建议的流程是:

添加 NVIDIA 官方仓库:

按照官方文档, 下载 cuda-keyring.deb 并安装.

更新索引:

sudo apt update

有选择地安装:

全家桶(含驱动): sudo apt install cuda

只装驱动: sudo apt install nvidia-driver-570(根据最新版本号)

只装工具链(驱动已有时): sudo apt install cuda-toolkit-12-6

总结

apt install cuda 会自动安装驱动.

它通过 cuda-drivers 这个包作为中间层, 确保你安装的 CUDA 版本有一个能正常工作的驱动支撑. 但如果你不希望 apt 动你的驱动, 请务必只安装 cuda-toolkit.

另外提醒一点: 安装完 apt 版驱动后, 通常需要重启系统才能让内核加载新的 nvidia.ko 模块. 你可以通过 lsmod | grep nvidia 来检查内核模块是否已成功加载.


```bash
$ ./NVIDIA-Linux-x86_64-575.57.08.run --help

./NVIDIA-Linux-x86_64-575.57.08.run [options]

This program will install the NVIDIA Accelerated Graphics Driver
for Linux-x86_64 575.57.08 by unpacking the embedded tarball and
executing the ./nvidia-installer  installation utility.

Below are the most common options; for a complete list use
'--advanced-options'.

--info
  Print embedded info (title, default target directory) and exit.

--check
  Check integrity of the archive and exit.

-x, --extract-only
  Extract the contents of ./NVIDIA-Linux-x86_64-575.57.08.run, but do not
  run 'nvidia-installer'.


The following arguments will be passed on to the ./nvidia-installer
utility:

  -v, --version
      Print the nvidia-installer version and exit.

  -h, --help
      Print usage information for the common commandline options and exit.

  -A, --advanced-options
      Print usage information for the common commandline options as well as the advanced options, and then exit.


$ ./NVIDIA-Linux-x86_64-575.57.08.run --info

  Identification    : NVIDIA Accelerated Graphics Driver for Linux-x86_64 575.57.08
  Target directory  : NVIDIA-Linux-x86_64-575.57.08
  Uncompressed size : 1522661 KB
  Compression       : zstd
  Date of packaging : Sat May 24 09:42:38 UTC 2025
  Application run after extraction : ./nvidia-installer

  The directory NVIDIA-Linux-x86_64-575.57.08 will be removed after extraction.

$
$ ./NVIDIA-Linux-x86_64-575.57.08.run --advanced-options

./NVIDIA-Linux-x86_64-575.57.08.run [options]

This program will install the NVIDIA Accelerated Graphics Driver
for Linux-x86_64 575.57.08 by unpacking the embedded tarball and
executing the ./nvidia-installer  installation utility.

--info
  Print embedded info (title, default target directory) and exit.

--lsm
  Print embedded lsm entry (or no LSM) and exit.

--pkg-history
  Print the package history of this file and exit.

--list
  Print the list of files in the archive and exit.

--check
  Check integrity of the archive and exit.

-x, --extract-only
  Extract the contents of ./NVIDIA-Linux-x86_64-575.57.08.run, but do not
  run 'nvidia-installer'.

--add-this-kernel
  Build a precompiled kernel interface for the currently running
  kernel and repackage the .run file to include this newly built
  precompiled kernel interface.  The new .run file will be placed
  in the current directory and the string "-custom" appended
  to its name, unless already present, to distinguish it from the
  original .run file.

--apply-patch [Patch]
  Apply the patch 'Patch' to the kernel interface files included
  in the .run file, remove any precompiled kernel interfaces
  and then repackage the .run file.  The new .run file will be
  placed in the current directory and the string "-custom"
  appended to its name, unless already present, to distinguish it
  from the original .run file.

--keep
  Do not delete target directory when done.

--target [NewDirectory]
  Extract contents in 'NewDirectory'

--extract-decompress
  Extract the embedded decompression program to stdout


The following arguments will be passed on to the ./nvidia-installer
utility:

COMMON OPTIONS:

  -v, --version
      Print the nvidia-installer version and exit.

  -h, --help
      Print usage information for the common commandline options and exit.

  -A, --advanced-options
      Print usage information for the common commandline options as well as the advanced options, and then exit.


ADVANCED OPTIONS:

  -v, --version
      Print the nvidia-installer version and exit.

  -h, --help
      Print usage information for the common commandline options and exit.

  -A, --advanced-options
      Print usage information for the common commandline options as well as the advanced options, and then exit.

  -a, --accept-license
      This option is obsolete and ignored by nvidia-installer.  It is provided for compatibility with older versions of nvidia-installer, which required this option for explicit license acceptance. Use of the NVIDIA driver implies acceptance of the NVIDIA Software License Agreement, contained in the file 'LICENSE' (in the top level directory of the driver package).

  -i, --driver-info
      Print information about the currently installed NVIDIA driver version.

  --uninstall
      Uninstall the currently installed NVIDIA driver.

  --skip-module-unload
      When uninstalling the driver, skip unloading of the NVIDIA kernel module. This option is ignored when the driver is being installed.

  --skip-module-load
      Skip the test load of the NVIDIA kernel modules after the modules are built, and skip loading them after installation is complete.

  --sanity
      Perform basic sanity tests on an existing NVIDIA driver installation.

  -e, --expert
      Enable 'expert' installation mode; more detailed questions will be asked, and more verbose output will be printed; intended for expert users.  The questions may be suppressed with the '--no-questions' commandline option.

  -q, --no-questions
      Do not ask any questions; the default (normally 'yes') is assumed for all yes/no questions, and the default string is assumed in any situation where the user is prompted for string input.

  -s, --silent
      Run silently; no questions are asked and no output is printed, except for error messages to stderr.  This option implies '--ui=none --no-questions'.

  --x-prefix=X-PREFIX
      The prefix under which the X components of the NVIDIA driver will be installed; the default is '/usr/X11R6' unless nvidia-installer detects that X.Org >= 7.0 is installed, in which case the default is '/usr'.  Only under rare circumstances should this option be used.

  --xfree86-prefix=XFREE86-PREFIX
      This is a deprecated synonym for --x-prefix.

  --x-module-path=X-MODULE-PATH
      The path under which the NVIDIA X server modules will be installed.  If this option is not specified, nvidia-installer uses the following search order and selects the first valid directory it finds: 1) `X -showDefaultModulePath`, 2) `pkg-config --variable=moduledir xorg-server`, or 3) the X library path (see the '--x-library-path' option) plus either 'modules' (for X servers older than X.Org 7.0) or 'xorg/modules' (for X.Org 7.0 or later).

  --x-library-path=X-LIBRARY-PATH
      The path under which the NVIDIA X libraries will be installed.  If this option is not specified, nvidia-installer uses the following search order and selects the first valid directory it finds: 1) `X -showDefaultLibPath`, 2) `pkg-config --variable=libdir xorg-server`, or 3) the X prefix (see the '--x-prefix' option) plus 'lib' on 32bit systems, and either 'lib64' or 'lib' on 64bit systems, depending on the installed Linux distribution.

  --x-sysconfig-path=X-SYSCONFIG-PATH
      The path under which X system configuration files will be installed.  If this option is not specified, nvidia-installer uses the following search order and selects the first valid directory it finds: 1) `pkg-config --variable=sysconfigdir xorg-server`, or 2) /usr/share/X11/xorg.conf.d.

  --opengl-prefix=OPENGL-PREFIX
      The prefix under which the OpenGL components of the NVIDIA driver will be installed; the default is: '/usr'.  Only under rare circumstances should this option be used.  The Linux OpenGL ABI (http://oss.sgi.com/projects/ogl-sample/ABI/) mandates this default value.

  --opengl-libdir=OPENGL-LIBDIR
      The path relative to the OpenGL library installation prefix under which the NVIDIA OpenGL components will be installed.  The default is 'lib' on 32bit systems, and 'lib64' or 'lib' on 64bit systems, depending on the installed Linux distribution.  Only under very rare circumstances should this option be used.

  --wine-prefix=WINE-PREFIX
      The prefix under which Wine components of the NVIDIA driver will be installed.  The default is '/usr'.

  --wine-libdir=WINE-LIBDIR
      The path relative to the Wine installation prefix under which Wine components of the NVIDIA driver will be installed.  The default is '<opengl-libdir>/nvidia/wine', where <opengl-libdir> is the path determined for --opengl-libdir above.

  --installer-prefix=INSTALLER-PREFIX
      The prefix under which the installer binary will be installed; the default is: '/usr'.  Note: please use the '--utility-prefix' option instead.

  --utility-prefix=UTILITY-PREFIX
      The prefix under which the NVIDIA utilities (nvidia-installer, nvidia-settings, nvidia-xconfig, nvidia-bug-report.sh) and the NVIDIA utility libraries will be installed; the default is: '/usr'.

  --utility-libdir=UTILITY-LIBDIR
      The path relative to the utility installation prefix under which the NVIDIA utility libraries will be installed.  The default is 'lib' on 32bit systems, and 'lib64' or 'lib' on 64bit systems, depending on the installed Linux distribution.

  --xdg-data-dir=XDG-DATA-DIR
      The prefix under which XDG data files (such as application .desktop files and icons) will be installed.  The default is '/usr/share'.

  --gbm-backend-dir=GBM-BACKEND-DIR
      The path relative to the OpenGL library installation prefix under which the NVIDIA GBM backend library will be installed.  The default is '<libdir>/gbm', where '<libdir>' is 'lib64' or 'lib', depending on the installed Linux distribution.  Only under very rare circumstances should this option be used.

  --gbm-backend-dir32=GBM-BACKEND-DIR32
      The path relative to the 32-bit OpenGL library installation prefix under which the 32-bit NVIDIA GBM backend library will be installed.  The default is '<libdir>/gbm', where <libdir>' is 'lib' or 'lib32', depending on the installed Linux distribution.  Only under very rare circumstances should this option be used.

  --documentation-prefix=DOCUMENTATION-PREFIX
      The prefix under which the documentation files for the NVIDIA driver will be installed.  The default is: '/usr'.

  --application-profile-path=APPLICATION-PROFILE-PATH
      The directory under which default application profiles for the NVIDIA driver will be installed. The default is: '/usr/share/nvidia'.

  --kernel-include-path=KERNEL-INCLUDE-PATH
      The directory containing the kernel include files that should be used when compiling the NVIDIA kernel module.  This option is deprecated; please use '--kernel-source-path' instead.

  --kernel-source-path=KERNEL-SOURCE-PATH
      The directory containing the kernel source files that should be used when compiling the NVIDIA kernel module.  When not specified, the installer will use '/lib/modules/`uname -r`/build', if that directory exists.  Otherwise, it will use '/usr/src/linux'.

  --kernel-output-path=KERNEL-OUTPUT-PATH
      The directory containing any KBUILD output files if either one of the 'KBUILD_OUTPUT' or 'O' parameters were passed to KBUILD when building the kernel image/modules.  When not specified, the installer will assume that no separate output directory was used.

  --kernel-install-path=KERNEL-INSTALL-PATH
      The directory in which the NVIDIA kernel module should be installed.  The default value is either '/lib/modules/`uname -r`/kernel/drivers/video' (if '/lib/modules/`uname -r`/kernel' exists) or '/lib/modules/`uname -r`/video'.

  --proc-mount-point=PROC-MOUNT-POINT
      The mount point for the proc file system; if not specified, then this value defaults to '/proc' (which is normally correct).  The mount point of the proc filesystem is needed because the contents of '<proc filesystem>/version' is used when identifying if a precompiled kernel interface is available for the currently running kernel.  This option should only be needed in very rare circumstances.

  --log-file-name=LOG-FILE-NAME
      File name of the installation log file (the default is: '/var/log/nvidia-installer.log').

  --tmpdir=TMPDIR
      Use the specified directory as a temporary directory when generating transient files used by the installer; if not given, then the following list will be searched, and the first one that exists will be used: $TMPDIR, /tmp, ., $HOME.

  --ui=UI
      Specify what user interface to use, if available.  Valid values for UI are 'ncurses' (the default) or 'none'. If the ncurses interface fails to initialize, or 'none' is specified, then a simple printf/scanf interface will be used.

  -c, --no-ncurses-color
      Disable use of color in the ncurses user interface.

  --no-nvidia-modprobe
      Skip installation of 'nvidia-modprobe', a setuid root utility which nvidia-installer installs by default.  nvidia-modprobe can be used by user-space NVIDIA driver components to load the NVIDIA kernel module, create the NVIDIA device files, and configure certain runtime settings in the kernel when those components run without sufficient privileges to do so on their own, e.g., the CUDA driver run within the permissions of a non-privileged user.  This utility is only needed if other means of performing these privileged operations are unavailable. See the nvidia-modprobe(1) man page for full details on the tasks performed by this utility.

  -k KERNEL-NAME, --kernel-name=KERNEL-NAME
      Build and install the NVIDIA kernel module for the non-running kernel specified by KERNEL-NAME (KERNEL-NAME should be the output of `uname -r` when the target kernel is actually running).  This option implies '--no-precompiled-interface'.  If the options '--kernel-install-path' and '--kernel-source-path' are not given, then they will be inferred from KERNEL-NAME; eg: '/lib/modules/KERNEL-NAME/kernel/drivers/video/' and '/lib/modules/KERNEL-NAME/build/', respectively.

  -n, --no-precompiled-interface
      Disable use of precompiled kernel interfaces.

  --no-abi-note
      The NVIDIA OpenGL libraries contain an OS ABI note tag, which identifies the minimum kernel version needed to use the library.  This option causes the installer to remove this note from the OpenGL libraries during installation.

  --no-rpms
      Normally, the installer will check for several rpms that conflict with the driver (specifically: NVIDIA_GLX and NVIDIA_kernel), and remove them if present.  This option disables this check.

  -b, --no-backup
      During driver installation, conflicting files are backed up, so that they can be restored when the driver is uninstalled.  This option causes the installer to simply delete conflicting files, rather than back them up.

  -r, --no-recursion
      Normally, nvidia-installer will recursively search for potentially conflicting libraries under the default OpenGL and X server installation locations.  With this option set, the installer will only search in the top-level directories.

  -K, --kernel-modules-only
      Install the kernel modules only, and do not uninstall the existing driver.  This is intended to be used to install kernel modules for additional kernels (in cases where you might boot between several different kernels).  To use this option, you must already have a driver installed, and the version of the installed driver must match the version of these kernel modules.

  --no-kernel-modules
      Install everything but the kernel modules, and do not remove any existing, possibly conflicting, kernel modules.  This can be useful in some debug environments, or when installing the open kernel modules from source.  If you use this option, you must be careful to ensure that NVIDIA kernel modules matching this driver version are installed separately.

  --no-x-check
      Do not abort the installation if nvidia-installer detects that an X server is running.  Only under very rare circumstances should this option be used.

  --precompiled-kernel-interfaces-path=PRECOMPILED-KERNEL-INTERFACES-PATH
      Before searching for a precompiled kernel interface in the .run file, search in the specified directory.

  -z, --no-nouveau-check
      Normally, nvidia-installer aborts installation if the nouveau kernel driver is in use.  Use this option to disable this check.

  -Z, --disable-nouveau, --no-disable-nouveau
      nvidia-installer will attempt to disable the nouveau kernel driver by default, if it is in use during installation. Use '--no-disable-nouveau to prevent nvidia-installer from disabling nouveau by default.

  -X, --run-nvidia-xconfig
      nvidia-installer can optionally invoke the nvidia-xconfig utility.  This will update the system X configuration file so that the NVIDIA X driver is used.  The pre-existing X configuration file will be backed up.  At the end of installation, nvidia-installer will ask the user if they wish to run nvidia-xconfig; the default response is 'no'.  Use this option to make the default response 'yes'.  This is useful with the '--no-questions' or '--silent' options, which assume the default values for all questions.

  --force-selinux=FORCE-SELINUX
      Linux installations using SELinux (Security-Enhanced Linux) require that the security type of all shared libraries be set to 'shlib_t' or 'textrel_shlib_t', depending on the distribution. nvidia-installer will detect when to set the security type, and set it using chcon(1) on the shared libraries it installs.  If the execstack(8) system utility is present, nvidia-installer will use it to also clear the executable stack flag of the libraries.  Use this option to override nvidia-installer's detection of when to set the security type.  Valid values for FORCE-SELINUX are 'yes' (force setting of the security type), 'no' (prevent setting of the security type), and 'default' (let nvidia-installer decide when to set the security type).

  --selinux-chcon-type=SELINUX-CHCON-TYPE
      When SELinux support is enabled, nvidia-installer will try to determine which chcon argument to use by first trying 'textrel_shlib_t', then 'texrel_shlib_t', then 'shlib_t'.  Use this option to override this detection logic.

  --no-sigwinch-workaround
      Normally, nvidia-installer ignores the SIGWINCH signal before it forks to execute commands, e.g. to build the kernel module, and restores the SIGWINCH signal handler after the child process has terminated.  This option disables this behavior.

  --no-distro-scripts
      Normally, nvidia-installer will run scripts from /usr/lib/nvidia before and after installing or uninstalling the driver.  Use this option to disable execution of these scripts.

  --no-opengl-files
      Do not install any of the OpenGL-related driver files.

  --no-wine-files
      Do not install any of the Wine driver files.

  --kernel-module-source-prefix=KERNEL-MODULE-SOURCE-PREFIX
      Specify a path where the source directory for the kernel module will be installed. Default: install source directory at /usr/src

  --kernel-module-source-dir=KERNEL-MODULE-SOURCE-DIR
      Specify the name of the directory where the kernel module sources will be installed. Default: directory name is nvidia-VERSION

  --no-kernel-module-source
      Skip installation of the kernel module source.

  --dkms, --no-dkms
      nvidia-installer can optionally register the NVIDIA kernel module sources, if installed, with DKMS, then build and install a kernel module using the DKMS-registered sources.  This will allow the DKMS infrastructure to automatically build a new kernel module when changing kernels.  During installation, if DKMS is detected, nvidia-installer will ask the user if they wish to register the module with DKMS; the default response is 'yes' unless the --no-dkms option is set, in which case the default response is 'no'.

  --module-signing-secret-key=MODULE-SIGNING-SECRET-KEY
      Specify a path to a private key to use for signing the NVIDIA kernel module. The corresponding public key must also be provided.

  --module-signing-public-key=MODULE-SIGNING-PUBLIC-KEY
      Specify a path to a public key to use for verifying the signature of the NVIDIA kernel module. The corresponding private key must also be provided.

  --module-signing-script=MODULE-SIGNING-SCRIPT
      Specify a path to a program to use for signing the NVIDIA kernel module. The program will be called with the arguments: program-name <HASH> <PRIVATEKEY> <PUBLICKEY> <MODULE>; if the program returns an error status, it will be called again with the arguments: program-name <PRIVATEKEY> <PUBLICKEY> <MODULE>. Default: use the sign-file script in the kernel source directory.

  --module-signing-key-path=MODULE-SIGNING-KEY-PATH
      Specify a path where signing keys generated by nvidia-installer will be installed. Default: install keys to '/usr/share/nvidia'.

  --module-signing-hash=MODULE-SIGNING-HASH
      Specify a cryptographic hash algorithm to use for signing kernel modules. This requires a module signing tool that allows explicit selection of the hash algorithm, and the hash algorithm name must be recognizable by the module signing tool. Default: select a hash algorithm automatically, based on the kernel's configuration.

  --module-signing-x509-hash=MODULE-SIGNING-X509-HASH
      Specify a cryptographic hash algorithm to use for signing X.509 certificates generated by nvidia-installer. The hash algorithm name must be one of the message digest algorithms recognized by the x509(1) command.

  --no-check-for-alternate-installs
      Maintainers of alternate driver installation methods can report the presence and/or availability of an alternate driver installation to nvidia-installer. Setting this option skips the check for alternate driver installations.

  --no-unified-memory
      Do not install the NVIDIA Unified Memory kernel module. This kernel module is required for CUDA on 64-bit systems, and if it is not installed, the CUDA driver and CUDA applications will not be able to run. The '--no-unified-memory' option should only be used to work around failures to build or install the Unified Memory kernel module on systems that do not need to run CUDA.

  --no-drm
      Do not install the nvidia-drm kernel module. This kernel module provides several features, including X11 autoconfiguration, support for PRIME, and DRM-KMS. The latter is used to support modesetting on windowing systems that run independently of X11. The '--no-drm' option should only be used to work around failures to build or install the nvidia-drm kernel module on systems that do not need these features.

  --no-peermem
      Do not install the nvidia-peermem kernel module. This kernel module provides support for peer-to-peer memory sharing with Mellanox HCAs (Host Channel Adapters) via GPUDirect RDMA (Remote Direct Memory Access).

  -j CONCURRENCY-LEVEL, --concurrency-level=CONCURRENCY-LEVEL
      Set the concurrency level for operations such as building the kernel module which may be parallelized on SMP systems. By default, this will be set to the number of detected CPUs, or to '1', if nvidia-installer fails to detect the number of CPUs. Systems with a large number of CPUs will have the default concurrency level limited to 32; setting a higher level on the command line will override this limit.

  --force-libglx-indirect
      Always install a libGLX_indirect.so.0 symlink, overwriting one if it exists.

  --no-libglx-indirect
      Do not install a libGLX_indirect.so.0 symlink.

  --install-libglvnd, --no-install-libglvnd
      The installer will try to determine whether the libglvnd libraries are already available, and will install them if they're not. Use --install-libglvnd to always install the libglvnd libraries, overwriting any that already exist.  Use --no-install-libglvnd to exclude the libglvnd libraries, even if they appear to be missing.

  --glvnd-egl-config-path=GLVND-EGL-CONFIG-PATH
      Install the EGL vendor library config file to this directory. If the libglvnd libraries are already present, then by default the installer will try to determine the path by running `pkg-config --variable=datadir libglvnd`. If that fails, then it will default to /usr/share/glvnd/egl_vendor.d.

  --egl-external-platform-config-path=EGL-EXTERNAL-PLATFORM-CONFIG-PATH
      If the package includes an EGL external platform library, then install the EGL external platform library config file to this directory. Defaults to /usr/share/egl/egl_external_platform.d.

  --override-file-type-destination=OVERRIDE-FILE-TYPE-DESTINATION
      Override the default destination for a file type. This option takes an argument in the form of '<FILE_TYPE>:<destination>', where <FILE_TYPE> is a file type from the installer .manifest file, and <destination> is an absolute path to the directory where files of that type should be installed. This option may be given multiple times in order to override the destinations for multiple file types. Use of this option takes precedence over any other options that might otherwise influence the destination of the specified file type.

  --skip-depmod
      Don't run the depmod(1) utility after modifying kernel modules.  This should only be used in cases where depmod(1) will be run separately after running nvidia-installer.

  --systemd, --no-systemd
      By default, the installer will install systemd unit files if systemctl is detected. Specifying --no-systemd will disable installation of systemd units.

  --systemd-unit-prefix=SYSTEMD-UNIT-PREFIX
      The path to which systemd unit files should be installed. By default, the installer uses `pkg-config --variable=systemdsystemunitdir systemd` to determine the location to install systemd unit files, or '/usr/lib/systemd/system' if pkg-config is not available. Ignored if --no-systemd is specified.

  --systemd-sleep-prefix=SYSTEMD-SLEEP-PREFIX
      The path to which systemd-sleep script files should be installed. By default, the installer uses `pkg-config --variable=systemdsleepdir systemd` to determine the location to install systemd-sleep script files, or '/usr/lib/systemd/system-sleep' if pkg-config is not available. Ignored if --no-systemd is specified.

  --systemd-sysconf-prefix=SYSTEMD-SYSCONF-PREFIX
      The path to which systemd unit enable symlinks should be installed. By default, the installer uses `pkg-config --variable=systemdsystemconfdir systemd` to determine the location to install these symlinks, or '/etc/systemd/system' if pkg-config is not available. Ignored if --no-systemd is specified.

  -M KERNEL-MODULE-TYPE, --kernel-module-type=KERNEL-MODULE-TYPE
      The type of kernel modules to build and install. Valid values are open and proprietary.

  -m KERNEL-MODULE-BUILD-DIRECTORY, --kernel-module-build-directory=KERNEL-MODULE-BUILD-DIRECTORY
      Directly set the directory within the package from which to build the kernel modules. This option is deprecated; use --kernel-module-type instead.

  --print-recommended-kernel-module-type
      Print if open or proprietary modules should be used based on currently attached GPUs and exit. NOTE: A kernel module type may be recommended even if it is not included in this driver package.

  --allow-installation-with-running-driver, --no-allow-installation-with-running-driver
      Proceed with installation even if an NVIDIA driver is already installed and running.

  --rebuild-initramfs, --no-rebuild-initramfs
      Rebuild the initramfs after installation is complete, regardless of the default action recommended by nvidia-installer.  --no-rebuild-initramfs skips rebuilding the initramfs after installation is complete.  These options are useful for non-interactive installations when a specific behavior is desired, regardless of what nvidia-installer would recommend by default in an interactive installation.


$
# 只能在 Linux 操作系统中操作
$ ./NVIDIA-Linux-x86_64-575.57.08.run --extract-only
Creating directory NVIDIA-Linux-x86_64-575.57.08
Verifying archive integrity... OK
Uncompressing NVIDIA Accelerated Graphics Driver for Linux-x86_64 575.57.08....................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................
$ ls -al
total 380448
drwxr-xr-x  3 root root      4096 Jan 15 18:55 .
drwxr-xr-x 16 root root      4096 Jan 15 18:42 ..
drwxr-xr-x 10 root root      4096 May 24  2025 NVIDIA-Linux-x86_64-575.57.08
-rwxr-xr-x  1 root root 389558524 Jan 15 18:52 NVIDIA-Linux-x86_64-575.57.08.run
$
```

