# BBR LFN 补丁与主流 AQM 的兼容性深度分析

本文档详细分析了 BBR LFN 补丁与 `fq`, `fq_codel`, `fq_pie`, `cake` 等常见队列规则的搭配情况，并结合实际生产环境中的带宽波动（如甲骨文云）提供架构选型指南。

### 核心原理解析：补丁的安全守卫（Guard）

LFN 补丁在高丢包跨境链路上维持高吞吐的核心机制，依赖于**四条件安全守卫门限**。其中最具决定性的两个微观条件为：

 1. rtt_us < 1.2 × min_rtt_us （确保链路上没有发生由于自身发包过猛导致的系统级队列堆积）
 2. inflight_latest <= 1.15 × BDP （确保当前在途数据包未超出网络最大容量屋顶，非自探溢出）

底层 Qdisc 的行为模式，直接决定了当丢包发生时，这两个条件在内核网络栈中是否依然成立。

---

## ⚖️ Qdisc 核心兼容性全景矩阵

| qdisc | 默认参数<br>+ LFN 补丁 | 调参后<br>+ LFN 补丁 | 核心控制机理与对决表现 | 最佳生产场景 |
|:---:|:---:|:---:|:---:|:---:|
| **fq** | ✅ 四条件恒真<br>豁免最充分 | 无需调整<br>（原生驱动级 EDT） | 零污染时延采样，<br>自带稀疏流绝对插队保护，开销极低。 | **弹性/廉价大带宽**<br>跨境多业务通用首选 |
| **fq_codel** | ⚠️ target=5ms<br>对长 RTT 极度敏感 | target → 15ms<br>interval → 300ms | 容易引发非限速状态下的<br>定时器脉冲毛刺，误杀补丁。 | 数据中心内部<br>或挂载 HTB 硬限速 |
| **fq_pie** | ✅ target=15m<br>天生相对平滑 | target → 20ms<br>tupdate → 30ms | 比例积分控制较为温和，<br>但无 EDT 支持，高并发下仍有毛刺。 | 追求 PIE 平滑性的<br>中轻载长肥网络 |
| **cake(shaped)** | ⚠️ 默认 internet<br>配置容易打架 | 显式指定<br>→ oceanic / satellite | 本地强行构筑大坝，大 Target 容忍微观抖动，流隔离极强。 | **高价/优质固定线路**<br>边界网关多业务控制 |
| **cake (unshaped)** | ❌ 彻底自废武功<br>产生微观毛刺 | **工业级严禁使用**<br>（不带带宽参数） | 失去了 AQM 控速能力，<br>丢掉 EDT 触发软件定时器抖动，熔断补丁。 | 任何长肥网络<br>与高波动跨境链路 |

---

## fq (Fair Queue) —— 完美搭配

**默认参数**: `pacing on`, `limit 10000p`, `flow_limit 100p`, 无 AQM/Drop。

### 🔬 交互分析

`fq` 是 BBR 的原生搭档。它通过 **EDT (Earliest Delivery Time)** 机制配合 BBR 的 pacing，在网卡驱动层精准控制发包，自身几乎不缓存数据包（将压力退回至 Socket 本身）。

- **RTT 条件**: 由于 `fq` 不制造系统级队列堆积，本地 Qdisc Bufferbloat 永远为 0，`rtt_us` 几乎总是等于线路纯净传播延迟，因此 `rtt < 1.2×min_rtt` **恒成立**。
- **Inflight 条件**: `fq` 的 `flow_limit` 仅作为防恶意流量的上限，BBR 自身的 `cwnd` 通常远低于此值，因此 `inflight_latest <= 1.15×BDP` **恒成立**。

### 结论

在 `fq` 下，四条件 Guard 几乎总是放行，补丁豁免最为充分。无需任何参数调整，CPU 算力开销最低。

---

## fq_codel —— 需警惕默认参数

**默认参数**: `target 5ms`, `interval 100ms`, `ecn on`。

### 🔬 交互分析
CoDel（受控延迟）算法的核心逻辑是：当数据包在队列中的驻留时间连续超过 target` 门限时，即判定发生拥塞，开始强制执行主动丢包或打上 ECN 标记。
默认的 `target=5ms` 是基于本地数据中心或超低延迟局域网设计的。对于跨境长肥网络（LFN，RTT 通常在 150ms~300ms 之间）而言，这个门限过于苛刻（仅占总时延的 2.5%）。

- **RTT 条件**: CoDel 的特点是“无队列时不丢包，一丢包队列即清空”。这意味着丢包瞬间 `rtt_us` 可能并未显著上升（队列已空），导致 `rtt < 1.2×min_rtt` 依然成立。**四条件中的 RTT 门限无法有效拦截 CoDel 的主动丢包**。
- **Inflight 条件**: 当本地因为混合业务（如大文件上传）引发极短暂的排队时，fq_codel 会瞬间被 5ms 门限激惹，发起密集的强制主动丢包（Drop）。这会瞬间冲爆 LFN 补丁的 inflight <= 1.15 × BDP 上限屋顶，补丁误判定其为“真性严重拥塞”，从而强行关闭高丢包豁免，逼迫 BBRv3 陷入恐慌减窗。

### 🛠️ 调优建议

若必须在 LFN 环境下部署 fq_codel ，必须大幅度放宽其时延观测窗口与容忍度：

```bash
tc qdisc add dev eth0 root fq_codel \
    target 15ms \
    interval 300ms \
    limit 20000 \
    ecn

```

调整后，CoDel 触发频率降低，补丁的四条件判断将更加准确。

---

## fq_pie —— 相对温和，适合中轻载长肥

**默认参数**: target 15ms, tupdate 15ms, ecn_prob 10%, ecn off。

### 🔬 交互分析

PIE（比例积分控制器）通过周期性更新丢包概率来平滑控制队列长度。相比于 CoDel，其天生自带的 target=15ms 对跨境长肥网络表现得更为宽容。控制环流表现更平滑，微观队列震荡较小。
其与补丁的碰撞逻辑与 fq_codel 类似，由于其队列控制更具预测性，补丁主要依靠 1.15 × BDP 条件来过滤 PIE 的主动概率丢包。在中轻载环境下能够良好相处，但在极端重载多业务抢占时，依然会因为缺乏 EDT 原生整流而导致时延毛刺。

- **RTT 条件**: 相较于 CoDel，PIE 的控制环更平滑，队列震荡较小，但仍存在与 CoDel 类似的“丢包不涨 RTT”现象。
- **Inflight 条件**: 同 fq_codel，依靠 `1.15×BDP` 屋顶来防止误判。

### 🛠️ 调优建议

若必须在 LFN 环境下部署 fq_pie ，必须放宽其时延观测窗口与容忍度：

```bash
tc qdisc add dev eth0 root fq_pie \
    target 20ms \
    tupdate 30ms \
    ecn ecn_prob 10

```

---

## Cake —— 复杂的双面刃

Cake 是目前 Linux 社区功能最集成的 Qdisc 之一，但在配合 BBR LFN 补丁时，其配置方式对性能有决定性影响。

### 1. 限速模式 (Shaped Profile) —— 推荐特定场景

当显式指定 bandwidth 参数时，Cake 能够感知链路长度并内置了 Profiles：

 * **Internet Profile (默认)**: target 约为 5ms。物理抖动极易触发 COBALT 主动丢包，导致 LFN 补丁的 RTT 门限钝化互斥。
 * **Satellite/Oceanic Profile**: 当显式设置为 satellite 或 oceanic 时，Cake 会自动将 target 放大到 30-50ms 量级，并将 interval 拉长到秒级。这完美消除了之前的架构冲突，Cake 容忍微观抖动，BBR 补丁豁免海缆物理随机丢包，二者达成互补。

```bash
# 跨洋/跨境稳定专线推荐配置
tc qdisc add dev eth0 root cake bandwidth 95mbit oceanic ecn triple-isolate

```

### 2. 致命盲区：不限速模式 (Unshaped Cake) —— 严禁使用
在很多高波动线路上，由于无法确定准确的带宽上限，运维人员常倾向于**不设置 bandwidth 参数**让 Cake 跑在线速模式下。这种做法在 LFN 环境下会引发“双重暴击”：

 1. **AQM 机制彻底瘫痪**：CAKE 的 COBALT 算法生效的前提是**本地队列产生积压**。当不设限速时，数据包以网卡线速（如万兆）瞬间出网。夜间高峰期真正的瓶颈在国际出口骨干网，本地队列空空如也，时延计算直接归零，AQM 完全无法在本地缓解 Bufferbloat。
 2. **丢失 EDT 支持，背刺 LFN 补丁**：CAKE 原生不支持 EDT 离线脱落整流。不限速时，BBRv3 必须退回到 TCP 内核栈调用高精度定时器（hrtimer）进行软件 Pacing。在多业务混杂、CPU 软中断抖动时，发包会产生微观上的“突发脉冲”。这些毛刺包在出网前重叠放大延迟，**瞬间误触发 LFN 补丁的 1.2 倍 RTT 熔断红线**，导致抗丢包能力直接被关闭，引发 BBR 恐慌性减窗。
 3. **流隔离效率低下**：不限速时，CAKE 为了维护成百上千个流的绝对公平轮询（DRR），在内核三层解析上空耗大量 CPU 算力。而在外部网络瓶颈面前，这种本地公平轮询根本无法阻止外网节点的无差别物理丢包。

---

## 实际生产场景与服务器带宽模型选型指南

### 1. 服务器带宽波动模型对比
根据市场常见服务器的网络特性，我们可以将其抽象为以下两种极端的数学模型：

#### 模型 A：弹性/廉价大带宽模型（以甲骨文云 Oracle Cloud、普通公网国际混合链路为代表）

 - **网络特征**：可用带宽全天候表现出极其恐怖的潮汐剧烈震荡。例如白天由于公网空闲，VPS 物理吞吐可跑满 **4 Gbps**；夜间晚高峰（21:00 - 23:30）受到骨干网（如中国电信 163 国际出口网 AS4134）的严重挤压，可用带宽骤降至 **50 Mbps**。波动跨度高达 80 倍，且夜晚伴随着骨干网暴力丢包（5%~10%）。

 - **架构选型生死劫**：
   - **❌ 强行部署 Cake (无论限速与否)**：若设固定限速，设 4G 则夜晚失效，设 50M 则白天沦为自残；若不设限速，则触发软件定时器毛刺，在夜间直接熔断 LFN 补丁，让 BBR 彻底失速。
   - **✅ 唯一正确解：选择 fq**：不需声明任何带宽。白天 BBRv3 探测到 4G 管道，fq 以 EDT 纳米级高精度狂飙；夜晚公网堵塞，BBRv3 自动将带宽评估（bw）动态收缩至 50M。此时，**LFN 补丁开始强力续航**，稳定豁免夜间的物理随机丢包，榨干夜间残余管道的全部价值。

#### 模型 B：高价/优质固定带宽模型（以精品 CN2 GIA、联通 9929、商业专线为代表）

 - **网络特征**：带宽小（如 50M~100M）但价格高昂。网络质量极优，全天候带宽不缩水，物理硬上限极其明确。

 - **选型结论**：**推荐使用 cake (oceanic/satellite)**。

   - **✅ 最佳选择：cake (oceanic/satellite)**：由于总上限雷打不动，在本地 tc 中将限速硬性限制在物理上限的 95%（例如 bandwidth 95mbit）极具工程价值。Cake 能够在服务器系统边界牢牢控住流量，防止突发流量漫过 VPS 网卡引发机房层面的暴力惩罚性丢包。

   - **⚠️ 选用 fq**：同样优秀，但无法在网络层提供硬性的总带宽整形，需要依赖应用层（如 Nginx）去限制突发总流量。

#### 模型 C：FQ-AQM 混合模型（以系统自带默认 fq_codel / fq_pie 为代表）

 - **网络特征**：常见于 Linux 发行版（如 Ubuntu）的默认系统配置，属于未限速（Unshaped）状态。

 - **架构选型生死劫**：

   - **❌ 大波动线路下的内耗**：在甲骨文云 4G → 50M 的大波动下，夜间由于瓶颈在远端公网，其本地 AQM 同样因无积压而瘫痪。同时，由于缺乏原生驱动级 EDT Pacing，高并发混合业务下的 CPU 软中断抖动同样会产生时延毛刺，**误杀 LFN 补丁的 1.2 倍 RTT 守卫**。你虽然通过流隔离获得了短暂好看的 Ping 值，但实际长连接的整体大文件吞吐带宽（如视频流、下载）会发生断崖式下跌。

### 2. 业务混合场景组合矩阵

#### 场景一：纯代理转发节点（如 Xray / Sing-box / 跨境隧道）

 - **业务特点**：海量、碎片化、高并发的加密小包，对 RTT 极其敏感。

 - **最佳拍档**：BBRv3 + LFN 补丁 + fq

 - **深层逻辑**：代理转发最忌引入额外微观时延。fq 的 EDT 机制零污染，直接在 Socket 层拦截积压，为 LFN 补丁提供了最纯净的 RTT 采样环境，确保补丁能够完美识别出什么是真正的公网物理随机丢包。

#### 场景二：多业务混杂节点（代理转发 + Nextcloud 大文件同步 + Typecho 博客）

 - **业务特点**：典型“家贼难防”场景。Nextcloud 随时爆发的大吞吐下载极易引发 Bufferbloat，从而将同一台机器上的网页小包（如Typecho 网页首字节响应（TTFB））和代理流量挤死。

 - **选型策略**：

   - **若带宽波动大（如甲骨文云）**：仍选 **fq**。利用 fq 内部高阶的 **稀疏流（Sparse Flows）绝对优先保护机制**。当 Nextcloud 疯狂塞满队列时，fq 会自动识别出 Typecho 的网页请求、代理控制信号属于稀疏流，无条件赋予其**免排队插队特权**。在不限速的情况下完美解决多业务抢占。宏观带宽限制建议在 Nginx 层（应用层）单独针对 Nextcloud 虚拟主机进行 limit_rate 限制（把宏观控速留给应用层，把微观抗丢包留给内核层）。

   - **若带宽极固定（如精品专线）**：可选 **cake oceanic**。利用其三重哈希隔离机制，在本地大坝前让大文件同步与 Web 应用众生平等。

### 3. 生产环境落地决策树
```
                   [ 生产服务器部署 BBRv3 + LFN 补丁 ]
                                    |
                           ( 核心网络线路类型? )
                               /         \
            [ 弹性/廉价大带宽/波动剧烈 ]   [ 高价优质/固定小带宽 ]
                            /               \
              ( 推荐: fq )                     ( 业务组合类型? )
        * 拒绝 cake/fq_codel unshaped            /          \
        避免软件定时器毛刺误杀补丁          [ 纯代理/高并发 ]    [ 混合业务/需硬限速 ]
                                            /                      \
                                      ( 推荐: fq )         ( 推荐: cake oceanic )
                                   享受 EDT 纯净时延       必须搭配大 Target 配置文件

```

## 配置脚本示例

### 选项 A：甲骨文云 / 廉价大带宽波动线路（绝对通用首选）

彻底拨乱反正，切回高性能 fq，完全释放 BBRv3 LFN 补丁抗丢包能力：

```bash
# 1. 强制将根 Qdisc 切换为高性能 fq pacing
sudo tc qdisc replace dev eth0 root fq pacing

# 2. 针对大波动骨干网优化内核 sysctl 参数
sudo tee -a /etc/sysctl.conf << 'EOF'
# 锁定 BBR 拥塞控制
net.ipv4.tcp_congestion_control = bbr

# 考虑到夜间国际骨干网丢包狠烈，将 LFN 补丁的物理丢包豁免阈值放宽至 8%
net.ipv4.tcp_bbr_lfn_loss_thresh_pct = 8

# 跨境 LFN 链路长，将 Min_RTT 的刷新生存周期拉长到 15 秒，避免在拥塞期频繁探测导致降速
net.ipv4.tcp_bbr_lfn_min_rtt_fresh_ms = 15000
EOF

# 3. 刷新内核参数生效
sudo sysctl -p

```

### 选项 B：CN2 GIA / 精品专线等固定小带宽线路 + 多业务

利用 CAKE 的大时延容忍 Profile 保护多业务边界：

```bash
# 1. 限制总带宽为物理上限的 95%（以 100M 线路为例），显式启用 oceanic 容忍度与三重隔离
sudo tc qdisc replace dev eth0 root cake bandwidth 95mbit oceanic triple-isolate ecn

# 2. 写入与之配套的平滑内核参数
sudo tee -a /etc/sysctl.conf << 'EOF'
net.ipv4.tcp_congestion_control = bbr
# 精品专线物理丢包极低，将豁免阈值收紧至 4% 即可
net.ipv4.tcp_bbr_lfn_loss_thresh_pct = 4
net.ipv4.tcp_bbr_lfn_min_rtt_fresh_ms = 10000
EOF

sudo sysctl -p

```

### 选项 C：系统原生默认 fq_codel 的高风险长肥链路紧急避险

若因特定老旧内核兼容性限制，**绝对无法**切换到 fq，必须对系统默认的 fq_codel 进行“大手术”改参调优，防止其默认的 5ms target 强行掐断补丁：

```bash
# 强行放宽检测窗口与时延门限，将 target 宽限至 15ms，将 interval 拉长到 300ms
sudo tc qdisc replace dev eth0 root fq_codel target 15ms interval 300ms flows 1024 packets 10000 ecn

# 同步微调内核 sysctl，调高补丁的丢包容忍阈值，以对冲由于缺乏 EDT 带来的软件高精度定时器时延毛刺
sudo tee -a /etc/sysctl.conf << 'EOF'
net.ipv4.tcp_congestion_control = bbr
net.ipv4.tcp_bbr_lfn_loss_thresh_pct = 10
net.ipv4.tcp_bbr_lfn_min_rtt_fresh_ms = 15000
EOF

sudo sysctl -p

```

---

*撰写参考: [tc-fq(8)][1], [tc-fq_codel(8)][2], [tc-fq_pie(8)][2], [tc-cake(8)][4]*


[1]: <https://man.archlinux.org/man/tc-fq.8.en>
[2]: <https://man.archlinux.org/man/tc-fq_codel.8.en>
[3]: <https://man.archlinux.org/man/tc-fq_pie.8.en>
[4]: <https://man.archlinux.org/man/tc-cake.8.en>

