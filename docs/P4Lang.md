# P4 语言学习笔记

P4 官方教程代码仓库：[https://github.com/p4lang/tutorials](https://github.com/p4lang/tutorials/tree/master)

P4 Language Cheat Sheet：[https://drive.google.com/file](https://drive.google.com/file/d/1Z8woKyElFAOP6bMd8tRa_Q4SA1cd_Uva/view)

## 1. 基础语法

+ P4 语言的四大工具：计数器、计量器、寄存器、主动上报机制

+ P4 语言自带 Hash 散列函数

### 1.1. 计数器

```cpp
counter(bit<32> size, CounterType type);
```

声明一个最大计数值为 `size` 的计数器，按照 `type` 单位进行计数，包括 `CounterType.packets`（数据包）、`CounterType.bytes`（字节数）与 `CounterType.packets_and_bytes`（数据包与字节数同时计数）

```cpp
void count(in bit<32> index);
```

计数器由数据平面更新，但只能由控制平面读取；可通过该接口为计数器设置触发规则。

```cpp
direct_counter(CounterType type);
```

声明一个与表关联的计数器，每次表被应用并且表条目被匹配时触发计数，计数单位 `type` 同上

### 1.2. 计量器

```cpp
meter(bit<32> size, MeterType type);
```

声明一个大小为 `size` 的计量器数组，用于记录至多 `size` 个网络流的状态信息。计量的单位为 `type`，包括 `CounterType.packets`（数据包）与 `CounterType.bytes`（字节数）两种类型

计量器主要有以下四项属性：

+ 峰值信息速率（Peak Information Rate，PIR）：该网络流可达到的最高传输速率
  
+ 峰值突发大小（Peak Burst Size，PBS）：该网络流可容许的数据包大小
  
+ 承诺信息速率（Committed Information Rate，CIR）：该网络流保证可靠的最低传输速率

+ 承诺突发大小（Committed Burst Size，CBS）：该网络流保证可靠的数据包大小

按照上述四种属性与当前数据包大小 $S$ 区分，数据包将被标记为以下三种状态：

1. 若 $P$ < $S$，则 P 桶已满，数据包被标记为红色，有较高概率被丢弃

2. 若 $P$ > $S$ 且 $C$ < $S$，则 P 桶未满但 C 桶已满，数据包被标记为黄色，会尽可能地被转发

3. 若 $P$ > $S$ 且 $C$ > $S$，则 P / C 桶都未满，数据包被标记为绿色，只有很小概率被丢弃

### 1.3. 寄存器

```cpp
register<T>(bit<32> size);
```

声明一个内部类型为 `T`，大小为 `size` 的数组，可调用接口对指定下标的元素进行读写

```cpp
void read(out T result, in bit<32> index);
void write(in bit<32> index, in T value);
```

### 1.4. 主动上报机制

```cpp
extern void digest<T>(in bit<32> receiver, in T data);
```

将 `data` 中携带的信息上报至控制平面，以便进一步处理

### 1.5. 内置 Hash 函数

```cpp
void hash(out R result, in HashAlgorithm algo, in T base, in D data, in M max);
```

+ `result`：哈希函数的输出结果

+ `algo`：使用的哈希算法，P4 目前支持 crc32 / crc32_custom / crc16 / crc16_custom /random /identity / csum16 / xor16 等算法

+ `base`：函数依赖的 base 值，通常为需额外指定位数的 0，如 `32w0`

+ `data`：需要求取哈希值的目标数据，必须是一个元组类型

+ `max`：通过取模限制函数输出的最大哈希值

## 2. 运行方式

+ 运行 P4 代码依赖的组件：PI / P4C / BMv2 / MiniNet

### 2.1. 代码仿真步骤

1. 编写网络拓扑 `topology.json` 与交换机流表 `sN_runtime.json` 的配置文件

2. 使用 P4C 编译 P4 代码，V1Model 架构下生成产物为若干 json 文件

3. 解析 `topology.json`，构建相应的 MiniNet 仿真拓扑，按照该拓扑启动若干台 BMv2 交换机以及 Host

4. 将 P4 代码编译生成的 json 文件导入所有 BMv2 交换机中

5. 解析 `sN_runtime.json`，分别载入对应交换机 sN 的流表中

6. 通过 MiniSet CLI 启动日志与 pcap 文件的记录流程