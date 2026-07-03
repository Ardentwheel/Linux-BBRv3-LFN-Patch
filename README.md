# BBRv3 LFN Loss Tolerance Patch

> 为 Linux 6.13+ 的 BBRv3 添加长肥管道（LFN）丢包容忍能力，解决跨境/卫星链路物理丢包导致的吞吐受限问题

⚠️ **重要警告**

- 本补丁仅适用于**跨境/卫星等非拥塞物理丢包场景**，请勿在数据中心、局域网等低延迟环境开启
- 生产环境请先灰度测试，避免因链路特征不符导致性能下降
- 默认关闭（`pct=0`），不会对原版 BBRv3 行为产生任何影响

---

## 📌 项目简介

BBRv3 默认将丢包率上限锁定为 2%，在跨境、卫星等长肥管道（LFN）场景中，链路常存在 2-5% 的非拥塞物理丢包，
导致 BBRv3 反应过度：
  - 冻结 `inflight_hi`（长期天花板）
  - 把 `bw_lo` / `inflight_lo` 按 (1-beta) 砍一刀
  - STARTUP 阶段因 `loss>2% + ≥6 discontiguous loss` 提前切 `DRAIN`，`max_bw` 没探到顶

本补丁在 **4 层严格安全约束**下动态放宽 2% 限制，对 `upper` / `lower` / `STARTUP` 三处做对称豁免，
完全保留 BBRv3 的 AQM 友好性、公平性优势，仅针对物理丢包场景做精准优化，100% 兼容原版内核逻辑。

---

## 📎 实现溯源

本补丁基于 Google 官方 BBRv3 仓库（对应 Linux 6.13+ 主线内核中的 BBRv3 代码）[tcp_bbr.c][1]。

核心修改围绕原版 `bbr_loss_thresh=2%` / `bbr_ecn_thresh=50%` 展开，所有扩展逻辑均严格遵循
BBRv3 的拥塞控制设计原则，确保兼容性和行为一致性。


---

## ✅ 核心安全机制

补丁通过 `bbr_lfn_eligible()` 统一判断，三处豁免（upper / lower / STARTUP）共用同一套 gate：

| # | 条件 | 目的 |
|:---:|:---:|:---:|
| 1 | `mode == BBR_PROBE_BW && cycle_idx != BBR_BW_PROBE_UP` <br>**或** `mode == BBR_STARTUP` | 避开 PROBE_UP 爬坡（公平性锚点）；<br>STARTUP 也纳入 |
| 2 | `rtt_us < 1.2 × min_rtt_us` | 无队列堆积，丢包不像是 AQM |
| 3 | `min_rtt_stamp` 新鲜 ≤ `tcp_bbr_lfn_min_rtt_fresh_ms` | 防止路由切换后 min_rtt 污染 |
| 4 | `inflight_latest ≤ 1.15 × BDP` | 防止把 fq_pie/fq_codel/cake 主动丢包误判成物理丢包 |

PROBE_UP 阶段**故意不参与豁免**——1.25× 爬坡的 abort 是 BBRv3 与 Reno/CUBIC 公平性的锚点，不能动。

---

## 🚀 功能特性

- ✅ **三处对称豁免**：
  - **`bbr_is_inflight_too_high()`**：loss/ECN 阈值替换 pct（loss 2%→pct, ECN 50%→pct）
  - **`bbr_adapt_upper_bounds()`**：eligible 时直接 return，**连 inflight_hi 冻结都免**（v2 只松阈值）
  - **`bbr_loss_lower_bounds()`**：eligible 时跳过 (1-beta) 砍刀
  - **STARTUP loss exit**：通过 `bbr_is_inflight_too_high()` 传导 pct，≥6 discontiguous loss 保留
- ✅ **零行为变更**：默认 pct=0，与原版 BBRv3 100% 兼容
- ✅ **全编译器兼容**：内核标准 jiffies 处理，GCC/Clang 通吃
- ✅ **配置简单**：全局 sysctl，无需 per-netns
- ✅ **AQM 友好**：1.15×BDP 屋顶 + 四条件 guard，fq/fq_pie/fq_codel/cake 均安全
- ✅ **多架构**：ARM64/x86_64，Clang LTO 验证通过
- ✅ **符合内核规范**：代码风格、符号导出对齐原生 TCP sysctl

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

### 1. 获取内核源码

```bash
git clone -b v3 https://github.com/google/bbr.git --depth=1
cd bbr
```

### 2. 应用补丁

将补丁保存为 `tcp_bbr_lfn_tolerance.patch`，进入内核源码目录执行：

```bash
wget -O bbr-lfn.patch \
https://github.com/Ardentwheel/Linux-BBRv3-LFN-Patch/raw/main/patches/0001-tcp_bbr-add-optional-LFN-loss-tolerance-via-sysctl-d.patch
git apply bbr-lfn.patch
```

#### ⚠️ 修补 CVE-2026-53235：Linux 内核 GRO 数据拉取漏洞

 > 此问题并非本 LFN 补丁引入，而是 Linux 6.13.y 主线内核自带缺陷，<br>
 > 在 Oracle Cloud / AWS / 任何 virtio_net + GRO 的 KVM 虚拟机上均可触发，<br>
 > 表现为 **服务器周期性 Kernel Panic 需要强制重启。**

```bash
wget -O gro-fix-pskb_may_pull.patch \
https://github.com/Ardentwheel/Linux-BBRv3-LFN-Patch/raw/main/patches/0001-net-add-pskb_may_pull-to-skb_gro_receive_list.patch
git apply gro-fix-pskb_may_pull.patch
```

> [CVE-2026-53235][2] 是 Linux 内核 skb_gro_receive_list() 函数中的一个数据处理缺陷。<br>
> 当某个特定序列的 TCP 段到达时，skb_gro_offset(skb) 大于 skb->len，skb_pull 中的 BUG_ON 触发，内核在 softirq 上下文崩溃。

### 3. 配置内核

复用当前内核配置，自动补齐新参数：
```bash
# 拷贝当前系统配置
cp /boot/config-$(uname -r) .config

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
```

#### 增量编译（已编译过同版本内核）

```bash
# 仅清理受影响的编译产物，大幅缩短编译时间
rm -f net/core/gro.o net/ipv4/sysctl_net_ipv4.o net/ipv4/tcp_bbr.o net/ipv4/tcp_bbr.mod*
# make...
```

> 💡 **编译参数说明**：
> - `LLVM=1 LLVM_IAS=1`：使用 Clang 编译器和集成汇编器，提升编译速度和代码质量
> - `KCFLAGS="-mcpu=native"`：针对当前 CPU 架构自动优化（ARM64/x86_64 通用）
> - `REBUILD_DEBUG=0`：不生成调试符号，减小内核体积
> - 如需指定特定 CPU（如 Neoverse-N1），可替换为 `-mcpu=neoverse-n1`

### 4. 安装并启用

```bash
# 安装内核包
 sudo dpkg -i ../linux-image-*.deb && sudo reboot

# 启用 LFN 优化（跨境链路示例）
sudo tee /etc/sysctl.d/99-bbr-lfn.conf << EOF
# BBR LFN: 仅在 PROBE_BW 阶段、无队列堆积、inflight 不超过 1.15xBDP 时生效
# 跨境链路推荐值：5-7，卫星链路推荐值：7-10
net.ipv4.tcp_bbr_lfn_loss_thresh_pct = 7
net.ipv4.tcp_bbr_lfn_min_rtt_fresh_ms = 15000
EOF
sudo sysctl --system
```

---

## ⚙️ 参数说明

| 参数 | 默认值 | 取值范围 | 说明 |
|------|--------|----------|------|
| `net.ipv4.tcp_bbr_lfn_loss_thresh_pct` | 0 | 0-20 | 放宽后的最大丢包百分比；0=关闭；ECN ce_thresh 同步替换 |
| `net.ipv4.tcp_bbr_lfn_min_rtt_fresh_ms` | 15000 | 0-60000 | min_rtt 最大有效年龄；0=不校验新鲜度；跨境建议 15000 |

### 推荐取值

| 场景 | 推荐值 |
|------|--------|
| 默认/数据中心/局域网 | 0（关闭） |
| 跨境链路（中美/中欧） | 5-7 |
| 卫星/高 RTT 链路 | 7-10 |
| 实验环境 | 2-20 |

- **`pct` 取值**: 建议设置为 `物理丢包率 + 2%` 的余量。例如跨境物理丢包 3%，建议 `pct=5`。
- **`min_rtt_fresh_ms`**: 跨境链路建议保持默认值 `15000` (15秒)，防止因路由抖动导致 min_rtt 过期。

> ⚠️ pct 只决定阈值上限，**四条件任一不满足都会回退原版逻辑**（2% loss / 50% CE）。
> 所以"开大了也不会炸 AQM"，但建议仍按链路实测物理丢包率取略高一点即可。

---

## 📶 Qdisc 兼容性指南

本补丁通过严格的四条件 Guard（`rtt<1.2×min_rtt` + `inflight≤1.15×BDP`）避免误判 AQM 主动丢包。以下是针对常见队列规则的部署建议：

| Qdisc | 默认参数安全性 | 推荐调优 | 备注 |
| :---: | :---: | :---: | :---: |
| **fq** | ⭐⭐⭐⭐⭐ | 无需调整 | 理论最佳，无 AQM 干扰 |
| **fq_codel** | ⭐⭐⭐ | `target 15ms` | 需调参以避免敏感丢包 |
| **fq_pie** | ⭐⭐⭐ | `target 20ms` | 比 codel 温和 |
| **cake** | ⭐⭐⭐ | `oceanic` | 默认对长 RTT 偏激进 |

### 首选：fq
- **状态**: ✅ 完美兼容
- **理由**: `fq` 的 pacing 机制与 BBR 原生协作，几乎不引入队列延迟，补丁豁免最充分。
- **配置**: 无需特殊调整。

> 💡 **深度解析**: 如需了解补丁与各类 AQM 交互的技术细节（如为何 CoDel 下 RTT 门限可能失效），请参阅项目文档 [doc/AQM_Compatibility.md](doc/Qdisc_Compatibility.md)。

---

## 🔍 验证方法
1. **确认参数生效**
   ```bash
   # 验证 BBR 已启用
   sysctl net.ipv4.tcp_congestion_control  # 应输出 bbr
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
   # inflight_hi 不被过早冻、bw_lo 不被 beta 砍，STARTUP 也不因 3% 提前 DRAIN
   ss -ti | grep bbr

   ```

---

## ❓ 常见问题

### Q：为什么不直接用 BBRv1？
A：BBRv1 无 2% 丢包限制，但 AQM 不友好（易导致 bufferbloat），且公平性较差。本补丁保留 BBRv3 的全部优势，仅修补 LFN 场景缺陷。

### Q: 为什么 PROBE_UP 不豁免？
A: PROBE_UP 的 1.25× 爬坡 + abort 是 BBRv3 与 Reno/CUBIC 公平性的锚点，动这里 upstream 必拒。
物理丢包在 PROBE_UP 阶段导致的 abort 属于可接受的代价。

### Q: STARTUP 的 ≥6 discontiguous loss 没动，为什么？
A: 6 这个数是 BBRv3 与 CUBIC 公平性论证的一部分，v3 只把 2%→pct 替换，6 保留，
这样即使 LFN 开 pct=7，只要 discontiguous loss <6 就不会 exit——更稳。

### Q：必须用 Clang 编译吗？
A: 不必，GCC/Clang 都行。Clang LTO 是推荐不是必须。仅 Clang 编译时需要添加 `LLVM=1 LLVM_IAS=1` 参数。

### Q：开启后性能没有提升？
A：本补丁仅针对**非拥塞物理丢包**场景生效，如果是拥塞导致的丢包，开启后反而可能降低性能。可先用 `tc` 模拟丢包测试验证。

### Q：打了补丁后 sysctl 参数看不到？
A：确认已加载新编译的内核，且 `net/ipv4/sysctl_net_ipv4.o` 已重新编译。可重新执行增量编译步骤解决。

### Q：如何卸载补丁？
A：切换回原版内核，删除 `/etc/sysctl.d/99-bbr-lfn.conf` 即可，无需额外操作。

### Q：支持 OpenWrt 吗？
A: 内核 ≥6.13 的 OpenWrt 可试，但 `sysctl_net_ipv4.c` 上下文可能已被 OpenWrt 自家补丁改过，
需要手动调 hunk offset。

---

## 📝 版本更新日志

### v3 (Current)
- **ECN 对称放松**：修复 v2 中 ECN 分支误用 `eff_loss` 变量的致命 bug，同时对 loss 和 ECN 阈值进行对称放松。
- **Upper Bound 豁免**：在 `bbr_adapt_upper_bounds()` 中，符合 LFN 条件时完全跳过 `inflight_hi` 冻结（v2 仅放松阈值），实现上下界对称豁免。
- **Startup 优化**：Startup 阶段的丢包退出逻辑继承 pct 阈值，避免跨境链路因物理丢包过早切 DRAIN。
- **参数调整**：`tcp_bbr_lfn_min_rtt_fresh_ms` 默认值从 5000ms 提升至 15000ms，适应跨境 RTT 抖动。
- **代码重构**：统一使用 `bbr_lfn_eligible()` 进行条件判断，提升可维护性。

### v2 (Historical)
- 仅放松 `bbr_is_inflight_too_high()` 中的 loss_thresh 阈值。
- 在 `bbr_loss_lower_bounds()` 中跳过 (1-beta) 削减。
- 未处理 upper bound 冻结问题，Startup 阶段未放松阈值。

---

## 📜 许可
GPLv2，与 Linux 内核协议一致。

---

## 🙏 致谢
- Google BBR 团队的原生 BBRv3 实现
- Linux 内核网络子系统维护者的长期工作

💡 小贴士：如果发现任何问题或有改进建议，欢迎提交 Issue 或 PR！

[1]: <https://github.com/google/bbr/blob/90210de4b779d40496dee0b89081780eeddf2a60/net/ipv4/tcp_bbr.c>
[2]: <https://www.sentinelone.com/vulnerability-database/cve-2026-53235/>
