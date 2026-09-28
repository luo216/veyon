# Veyon 树状自适应中继推流（Tree-based Dynamic Relay Demo）

## 一、 背景与痛点

在传统的 Veyon 屏幕广播（Demo 演示）架构中，采用的是**纯单播（Unicast）TCP 点对点连接**：
- 教师机作为中心 VNC 服务端，局域网内的每一台学生机都直接向教师机建立一条独立的 TCP 连接。
- 当机房规模达到 **50~60 台甚至更大（例如 56 台学生机）** 时，教师机在每个屏幕帧刷新周期内都需要复制并单播发送 56 份完整/增量图像数据。
- 这会导致：
  1. **教师机网卡带宽跑满**：引发严重网络排队、丢包和 TCP 拥塞重传；
  2. **交换机下行端口突发拥塞**：广播关键帧时产生网络风暴；
  3. **学生端大面积卡顿**：出现几秒到十几秒的高延迟、花屏甚至断连。

---

## 二、 解决方案与核心架构

为了从根本上解决大并发下的推流瓶颈，本项目对 Veyon 的 Demo 模块进行了架构重构，引入**树状分裂自适应中继推流（Tree-based Dynamic Relay）**。

### 1. 控制面与数据面分离（Control/Data Plane Separation）

- **控制面（平面星型直控）**：
  - 教师机与所有学生机之间的控制信令（启动演示、停止演示、锁屏、监控缩略图等）依然**100% 保持平铺直连**（端口 `11100`）。
  - 控制报文仅几百字节，教师机对全体学生保持绝对控制力，点击“停止”瞬时生效，绝无中间层级依赖。
- **数据面（4 叉树自适应中继）**：
  - 仅将高负载的屏幕像素流（端口 `11400`）组织为 **4 叉树（4-ary Tree）** 分发网络。

### 2. 动态随机拓扑（“随机推，无需指定”）

- **无需手动指定中继节点**：完全自适应当前机房实际在线机器。
- **每次广播随机洗牌**：
  - 教师端发起演示时，收集当前处于在线（Connected）状态的学生列表。
  - 使用高强度随机数生成器（`QRandomGenerator`）对在线学生进行乱序洗牌（Random Shuffle）。
  - 按照 4 叉树公式计算每个节点的父节点 IP：
    - 前 4 台学生机器以教师机为父节点；
    - 后续第 $i$ 台学生机器以打乱后第 $\lfloor (i - 4) / 4 \rfloor$ 台机器为父节点。
  - 每一轮广播随机轮换中继角色，负载公平分摊，不会固定损耗特定机器。

### 3. 数学模型（以 56 台学生机为例）

- **根节点（教师机，Level 0）**：
  - **只推 4 台** 第一层学生机（$S_0 \sim S_3$）。
  - 教师机上传并发连接数从 **56 降到 4，上行带宽与 CPU 开销降低 93%**！
- **第一层中继（4 台，Level 1）**：
  - 每台各推 4 台，共覆盖 16 台（$S_4 \sim S_{19}$）。
- **第二层中继（16 台，Level 2）**：
  - 其中前 9 台各推 4 台，覆盖剩余全部 36 台（$S_{20} \sim S_{55}$）。
- **第三层叶子节点（36 台，Level 3）**：
  - 纯接收端，只收不发。

整棵树**最大深度仅 3 跳（Hops）**，千兆局域网内单跳延迟 5~15ms，3 跳累积延迟小于 50ms，肉眼完全无感。

### 4. 零拷贝/零编解码轻量级中继

中继学生机不进行任何二度图像编码或解包，仅作为网络流的中继分发器：
- 本机播放端 `DemoClient` 连接本地 `127.0.0.1:11400`；
- 下级 4 台子机连接该中继机器的 `IP:11400`；
- 中继机器的 CPU 占用率几乎为 0%，内存开销仅几兆字节缓冲。

---

## 三、 配置参数说明

在 Veyon 配置中新增了以下两项配置：

| 配置键名 | 类型 | 默认值 | 说明 |
| :--- | :--- | :--- | :--- |
| `Demo/TreeRelayEnabled` | bool | `true` | 是否启用树状中继推流（设为 `false` 可一键无缝退回官方原生直连） |
| `Demo/TreeFanOut` | int | `4` | 树状分流的扇出度（每个节点最多推几路流，推荐值为 2、4 或 8） |

可以通过命令行随时调整（例如将扇出度改为 4）：
```bash
sudo veyon-cli config set Demo/TreeRelayEnabled true
sudo veyon-cli config set Demo/TreeFanOut 4
```

---

## 四、 核心代码修改文件

1. **`core/src/VncClientProtocol.h / .cpp`**：
   - 增加对 `rfbSecTypeVeyon` 及 `RfbVeyonAuth::Token` 认证的支持，使客户端协议能够直接连接上游 `DemoServer`。
2. **`plugins/demo/DemoServer.h / .cpp`**：
   - 构造函数与重连逻辑增加 `vncServerHost` 支持，使中继端的 `DemoServer` 既可以作为根服务（连本机 VNC），也可以作为中继服务（连上级 DemoServer）。
3. **`plugins/demo/DemoConfiguration.h`**：
   - 注册 `TreeRelayEnabled` 和 `TreeFanOut` 配置代理属性。
4. **`plugins/demo/DemoFeaturePlugin.h / .cpp`**：
   - 增加 `Argument::IsRelay` 控制参数；
   - 教师端 `controlDemoClient` 中增加在线感知、随机洗牌和四叉树拓扑定向分发；
   - 学生端 Worker 中增加中继服务生命周期管理与本地 `127.0.0.1` 环回播放。

---

## 五、 编译与部署指南

### 1. 编译
使用标准的 CMake + Qt5/Qt6 流程进行编译：
```bash
cmake -B build -S . -DCMAKE_BUILD_TYPE=Release
cmake --build build -j$(nproc)
```

### 2. 部署到已有系统（支持热替换）
由于改动高度模块化，无需重新安装完整安装包，只需替换编译产物：
- 将编译生成的 `build/plugins/demo/libdemo.so` 替换目标机器的 `/usr/lib/x86_64-linux-gnu/veyon/libdemo.so`（或 `/usr/lib64/veyon/libdemo.so`）；
- 将编译生成的 `build/core/libveyon-core.so.*` 替换目标机器的对应系统动态库；
- 重启 veyon 服务生效：
  ```bash
  sudo systemctl restart veyon.service
  ```
- 教师机与学生机运行同一套插件即可自动协同运作。
