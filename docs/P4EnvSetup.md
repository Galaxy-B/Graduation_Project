# P4 代码仿真运行环境配置手册

论文中设计的 P4 语言相关实验均通过软件仿真运行的方式实现。若希望在本地复现各项实验的结果，必需一套基于以下组件构建起的 P4 代码仿真运行环境：

+ P4 Runtime（PI）

+ Behavioral Model (bmv2)

+ P4 Compiler (P4C)

+ Mininet

本手册将以一台操作系统为 Ubuntu 24.04 的 Linux 机器为例，分步骤介绍如何安装、配置上述组件，从而搭建起 P4 代码仿真运行所需的环境。

## Step 1. 安装系统基础工具

```sh
sudo apt-get --yes install git autoconf automake libtool curl make g++ unzip pkg-config python3-pip python3-venv
```

## Step 2. 创建 Python 虚拟环境

推荐使用 Python 虚拟环境或 Conda 来管理 P4 仿真运行所需要的各项依赖。

```sh
python3 -m venv P4Simulation
source P4Simulation/bin/activate
```

## Step 3. 安装 GRPC & Protobuf

GRPC 是由 Google 开发的高性能远程过程调用（RPC）框架；Protobuf 则是 Google 开发的序列化库，通常作为 GRPC 的默认接口定义和数据交换格式。二者是构建 P4Runtime 与 PI 所必需的依赖。

```sh
sudo apt-get --yes install libprotobuf-dev protobuf-compiler protobuf-compiler-grpc libgrpc-dev libgrpc++-dev
pip3 install protobuf==4.21.12
```

为防止错误的 Protobuf 版本导致后续 bmv2 组件的编译失败，可在 Protobuf 安装完成后运行以下命令，若输出结果为 2 则可继续进行后续配置。

```sh
objdump -t /usr/local/lib/libprotobuf.a | grep hidden | sort | uniq -c | wc -l
```

## Step 4. 安装 PI

PI 是通过 P4Runtime 编写控制平面逻辑、与数据平面进行交互的重要组件，需要通过 Github 获取完整的源码来进行构建与安装。

```sh
sudo apt-get --yes install libreadline-dev valgrind libtool-bin libboost-dev libboost-system-dev libboost-thread-dev
git clone https://github.com/p4lang/PI
cd PI
git submodule update --init --recursive
./autogen.sh
./configure --with-proto --without-internal-rpc --without-cli --without-bmv2 --with-python_prefix=P4Simulation
make
sudo make install
make clean
```

## Step 5. 安装 Behavioral Model

Behavioral Model 为实际运行 P4 代码的虚拟交换机模型，同样需要通过 Github 获取完整的源码来进行构建与安装。

Behavioral Model 仓库中的构建脚本需要应用补丁来确保配置的正常运行，补丁文件可以从 [https://github.com/jafingerhut/p4-guide](https://github.com/jafingerhut/p4-guide) 仓库中获取。

```sh
git clone https://github.com/p4lang/behavioral-model.git
cd behavioral-model
git pull
patch -p1 < ../p4-guide/bin/patches/behavioral-model-support-venv.patch
./install_deps.sh
./autogen.sh
./configure --with-pi --with-thrift --with-python_prefix=P4Simulation 'CXXFLAGS=-O0 -g'
make
sudo make install-strip
sudo ldconfig
make clean
```

## Step 6. 安装 P4C

P4C 为 P4 语言的编译器，可将 P4 代码转换为 Behavioral Model 所能处理的 JSON 格式输入，同样需要通过 Github 获取完整的源码来进行构建与安装。

P4C 的编译需要消耗大量的系统资源。如果初次编译失败，可尝试逐渐降低 make 命令的并行规模，并重新编译。

```sh
sudo apt-get --yes install g++ git automake libtool libgc-dev bison flex libfl-dev libgmp-dev libboost-dev libboost-iostreams-dev libboost-graph-dev llvm pkg-config python3-pip tcpdump libelf-dev clang
pip3 install scapy==2.5.0 ply
git clone https://github.com/p4lang/p4c.git
cd p4c
git pull
git submodule update --init --recursive
mkdir build
cd build
cmake .. -DCMAKE_BUILD_TYPE=Release -DENABLE_TEST_TOOLS=ON
make -j4
sudo make install/strip
sudo ldconfig
rm -rf build
```

## Step 7. 安装 Mininet

Mininet 并不是运行 P4 程序所必需的组件，但对于调试 P4 代码、运行论文中设计的各项实验来说至关重要，同样需要通过 Github 获取完整的源码来进行构建与安装。

Mininet 仓库中的构建脚本也需要应用补丁来确保配置的正常运行，补丁文件可以从 [https://github.com/jafingerhut/p4-guide](https://github.com/jafingerhut/p4-guide) 仓库中获取。

```sh
git clone https://github.com/mininet/mininet mininet
cd mininet
git checkout 6eb8973c0bfd13c25c244a3871130c5e36b5fbd7
patch -p1 < ../p4-guide/bin/patches/mininet-patch-for-2024-sep-enable-venv.patch
PYTHON=python3 ./util/install.sh -nw
```

## Step 8. 安装杂项依赖

最后一步是安装运行 P4 代码过程中可能会使用到的杂项依赖，包括数据平面的测试框架、P4 Runtime 的终端 CLI 等等。

P4Runtime Shell 仓库中的构建脚本也需要应用补丁来确保配置的正常运行，补丁文件可以从 [https://github.com/jafingerhut/p4-guide](https://github.com/jafingerhut/p4-guide) 仓库中获取。

```sh
git clone https://github.com/p4lang/ptf
cd ptf
pip install .
cd ..

sudo apt-get --yes install libgflags-dev net-tools
pip3 install psutil crcmod wheel
pip3 install grpcio==1.59.3

git clone https://github.com/p4lang/p4runtime-shell
cd p4runtime-shell
patch -p1 < ../p4-guide/bin/patches/p4runtime-shell-2023-changes.patch
pip3 install .
```
