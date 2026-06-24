# BBR LFN Loss Tolerance Patch
> 为 Linux 6.13+ 的 BBRv3 添加长肥管道（LFN）丢包容忍能力，解决跨境/卫星链路物理丢包导致的吞吐受限问题

⚠️ **重要警告**
- 本补丁仅适用于**跨境/卫星等非拥塞物理丢包场景**，请勿在数据中心、局域网等低延迟环境开启
- 生产环境请先灰度测试，避免因链路特征不符导致性能下降
- 默认关闭（`pct=0`），不会对原版 BBRv3 行为产生任何影响

---

## 📌 项目简介
BBRv3 默认将丢包率上限锁定为 2%，在跨境、卫星等长肥管道（LFN）场景中，链路常存在 2-5% 的非拥塞物理丢包，导致 BBRv3 过早冻结 `inflight_hi`，无法跑满瓶颈带宽。

本补丁在**4 层严格安全约束**下动态放宽 2% 限制，完全保留 BBRv3 的 AQM 友好性、公平性优势，仅针对物理丢包场景做精准优化，100% 兼容原版内核逻辑。

---

## 📎 实现溯源
本补丁基于 Google 官方 BBR 开发仓库的[对应提交](https://github.com/google/bbr/blob/90210de4b779d40496dee0b89081780eeddf2a60/net/ipv4/tcp_bbr.c)实现，该提交对应 BBRv3 的稳定版本，与 Linux 6.13+ 主线内核中的 BBRv3 代码完全对齐。补丁的核心修改围绕原版代码中定义的 `bbr_loss_thresh`（2% 固定丢包阈值）展开，所有扩展逻辑均严格遵循 BBRv3 的拥塞控制设计原则，确保兼容性和行为一致性。


---

## ✅ 核心安全机制
补丁仅在**全部满足以下 4 个条件**时才会放宽丢包阈值，避免误判拥塞：
1. 处于 `BBR_PROBE_BW` 模式，且不在 `PROBE_UP` 阶段（避开 RTT 滞后窗口）
2. 当前 RTT < 1.2 × 最小 RTT（无队列堆积）
3. 最小 RTT 时间戳新鲜（可通过 sysctl 配置有效期，默认 5 秒，避免路由切换后使用陈旧基准）
4. 当前飞行数据量 ≤ 1.15 × BDP（避免误判 fq_codel/cake 等 AQM 的主动丢包）

---

## 🚀 功能特性
- ✅ **全编译器兼容**：采用内核标准 jiffies 处理方式，无编译器特有语法，完美支持 GCC/Clang
- ✅ **零行为变更**：默认关闭（`pct=0`），与原版 BBRv3 100% 兼容
- ✅ **配置简单**：全局 sysctl 参数，无需复杂的 per-netns 配置
- ✅ **AQM 友好**：内置 1.15×BDP 屋顶保护，完美适配 `fq`/`fq_codel`/`cake`
- ✅ **轻量无残留**：单补丁文件，一键应用，卸载仅需切换内核即可
- ✅ **多架构支持**：ARM64/x86_64 全兼容，支持 Clang LTO 优化
- ✅ **符合内核规范**：代码风格、符号导出完全对齐内核原生 TCP sysctl 实现

---

## 📋 前置要求
- Linux 内核版本 ≥ 6.13
- 编译依赖：
  ```bash
  sudo apt install -y build-essential libncurses-dev bison flex \
                      libssl-dev libelf-dev bc git clang lld
  ```

---

## ⚡ 快速开始
### 1. 应用补丁
将补丁保存为 `tcp_bbr_lfn_tolerance.patch`，进入内核源码目录执行：
```bash
cd linux-source-dir
git apply tcp_bbr_lfn_tolerance.patch
```

### 2. 配置内核
复用当前内核配置，自动补齐新参数：
```bash
# GCC 编译（默认）
make olddefconfig

# Clang 编译（推荐，支持 LTO）
make LLVM=1 olddefconfig
```

### 3. 编译内核
#### 首次编译
```bash
# GCC 编译
make -j$(nproc) \
     KCFLAGS="-mcpu=native" \
     bindeb-pkg REBUILD_DEBUG=0

# Clang 编译（推荐，自动适配当前 CPU 架构）
make -j$(nproc) LLVM=1 LLVM_IAS=1 \
     KCFLAGS="-mcpu=native" \
     bindeb-pkg REBUILD_DEBUG=0

#### 增量编译（已编译过同版本内核）
```bash
# 仅清理受影响的编译产物，大幅缩短编译时间
rm -f net/ipv4/sysctl_net_ipv4.o net/ipv4/tcp_bbr.o
make -j$(nproc) LLVM=1 LLVM_IAS=1 KCFLAGS="-mcpu=native" bindeb-pkg REBUILD_DEBUG=0
```

> 💡 **编译参数说明**：
> - `LLVM=1 LLVM_IAS=1`：使用 Clang 编译器和集成汇编器，提升编译速度和代码质量
> - `KCFLAGS="-mcpu=native"`：针对当前 CPU 架构自动优化（ARM64/x86_64 通用）
> - `REBUILD_DEBUG=0`：不生成调试符号，减小内核体积
> - 如需指定特定 CPU（如 Neoverse-N1），可替换为 `-mcpu=neoverse-n1`

### 4. 安装并启用
```bash
# 安装内核包
sudo dpkg -i ../linux-image-*.deb
sudo reboot

# 验证 BBR 已启用
sysctl net.ipv4.tcp_congestion_control  # 应输出 bbr

# 启用 LFN 优化（跨境链路示例）
sudo tee /etc/sysctl.d/99-bbr-lfn.conf << EOF
# BBR LFN: 仅在 PROBE_BW 阶段、无队列堆积、inflight 不超过 1.15xBDP 时生效
# 跨境链路推荐值：5-7，卫星链路推荐值：7-10
net.ipv4.tcp_bbr_lfn_loss_thresh_pct = 7
net.ipv4.tcp_bbr_lfn_min_rtt_fresh_ms = 5000
EOF
sudo sysctl --system
```

---

## ⚙️ 参数说明
| 参数 | 默认值 | 取值范围 | 说明 |
|------|--------|----------|------|
| `net.ipv4.tcp_bbr_lfn_loss_thresh_pct` | 0 | 0-20 | 放宽后的最大丢包百分比，0 为关闭 |
| `net.ipv4.tcp_bbr_lfn_min_rtt_fresh_ms` | 5000 | 0-60000 | 最小 RTT 的最大有效年龄，0 为不校验新鲜度 |

### 推荐取值
| 场景 | 推荐值 |
|------|--------|
| 默认/数据中心/局域网 | 0（关闭） |
| 跨境链路（中美/中欧） | 5-7 |
| 卫星/高 RTT 链路 | 7-10 |
| 实验环境 | 2-20 |

---

## 📶 AQM 兼容性
| 队列规则 | 兼容性 | 说明 |
|----------|--------|------|
| `fq` | ✅ 完美 | 原生 pacing，建议搭配使用 |
| `fq_codel` | ✅ 良好 | 1.15×BDP 屋顶机制避免误判 CoDel 主动丢包 |
| `cake` | ✅ 良好 | 尊重 cake 的延迟目标，避免抢占其他流带宽 |

---

## 🔍 验证方法
1. **确认参数生效**
   ```bash
   sysctl net.ipv4.tcp_bbr_lfn_loss_thresh_pct  # 应返回配置的值
   ```

2. **模拟跨境链路测试**
   ```bash
   # 模拟 200ms 延迟 + 3% 物理丢包
   sudo tc qdisc add dev eth0 root netem delay 200ms loss 3%
   # 运行 iperf3 测试
   iperf3 -c <server-ip> -t 60 -i 1
   ```

3. **观察 BBR 行为**
   ```bash
   ss -ti | grep bbr
   # 若 inflight_hi 未被过早冻结，且吞吐量明显提升，说明补丁生效
   ```

---

## ❓ 常见问题
### Q：为什么不直接用 BBRv1？
A：BBRv1 无 2% 丢包限制，但 AQM 不友好（易导致 bufferbloat），且公平性较差。本补丁保留 BBRv3 的全部优势，仅修补 LFN 场景缺陷。

### Q：必须用 Clang 编译吗？
A：不需要，补丁完全兼容 GCC 和 Clang，可根据需求选择。仅 Clang 编译时需要添加 `LLVM=1 LLVM_IAS=1` 参数。

### Q：开启后性能没有提升？
A：本补丁仅针对**非拥塞物理丢包**场景生效，如果是拥塞导致的丢包，开启后反而可能降低性能。可先用 `tc` 模拟丢包测试验证。

### Q：打了补丁后 sysctl 参数看不到？
A：确认已加载新编译的内核，且 `net/ipv4/sysctl_net_ipv4.o` 已重新编译。可重新执行增量编译步骤解决。

### Q：如何卸载补丁？
A：切换回原版内核，删除 `/etc/sysctl.d/99-bbr-lfn.conf` 即可，无需额外操作。

### Q：支持 OpenWrt 吗？
A：OpenWrt 内核 ≥6.13 也可使用该补丁，需自行调整 `sysctl_net_ipv4.c` 的上下文适配 OpenWrt 的内核补丁集。

---

## 📜 许可
GPLv2，与 Linux 内核协议一致。

---

## 🙏 致谢
- Google BBR 团队的原生 BBRv3 实现
- Linux 内核网络子系统维护者的长期工作

💡 小贴士：如果发现任何问题或有改进建议，欢迎提交 Issue 或 PR！