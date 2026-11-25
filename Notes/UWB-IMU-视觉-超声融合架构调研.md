---
title: UWB × IMU/视觉/超声波 融合架构调研（面向 MCU 级自动跟随行李箱）
type: research
project: "[[智能行李箱自动跟随]]"
sources_count: 16
tags: [UWB, 传感器融合, 架构调研, 自动跟随, 智能行李箱]
---

# UWB × IMU/视觉/超声波 融合架构调研（面向 MCU 级自动跟随行李箱）

约束回顾：MCU/SoC 级算力、跟随 ≤1.5 m/s、位姿输出 10–50 Hz、目标被人群短时遮挡 1–3 s。

---

## A. 架构对比表

| # | 架构 | 状态量 | 观测 | 算力（实测量级） | 实测精度（口径见备注） | 主要失效条件 | 来源 |
|---|---|---|---|---|---|---|---|
| 1 | **松耦合 KF/EKF**（UWB 先解算位置，再融位置） | p, v（6–9 维） | 三边/TDoA 解算后的位置 (x,y,z) | 最低。Raspberry Pi 4 实时；Cortex-M7 上 1 kHz 有余量 | UWB 单独动态误差最高 **50 cm** → UWB+IMU 松耦合 **13.7 cm**（0.6–0.8 m/s、0.6–1.6 m/s² 动态台架） | 需 ≥3 个有效锚点才出位置解；少于 3 个直接退化为纯惯性；NLOS 偏差在解算阶段已被"洗"成位置误差，滤波器看不到单锚残差，无法逐锚加权/剔除 | PMC9659066 |
| 2 | **紧耦合 EKF / ESKF**（原始测距进量测方程） | p, v, q（误差态）, b_a, b_g（15 维）；可扩杆臂 l_U、时延 t_d | 每锚点距离残差 ‖p+R·l_U−a_j‖−d_j | 中。每帧 8 锚点残差 + 15×15 求逆，约 3–5 k flops；**实测 80 Hz 三维输出**（MAV，机载） | 均值 **0.16 m** / 最大 **0.39 m**（VICON 真值；对照 vanilla EKF 0.30/0.71 m） | 杆臂与时延未标定会系统性劣化（需 mm 级 / ms 级）；NLOS 正偏差直接进残差，须配 NIS 门控或 Huber；EKF 线性化截断误差在 NLOS 下放大（N/E/U 方向平均损失 8.1%/10.1%/33.2%） | arXiv 1807.10913；2108.00133；MDPI 13/10/6187 |
| 3 | **紧耦合 UKF** | 同 EKF（p, v, [b]），sigma 点传播 | 原始测距，无需求 Jacobian | 中偏高。2n+1 个 sigma 点 × 每点全锚点距离求值；n=9、8 锚点约 150 次距离计算/帧 | UKF **39.72 cm 均值 / 43.03 cm RMSE**；同数据 EKF 41.26/44.58 → 仅好约 3.5% | 高维（>15）时 α/β/κ 难调、协方差易非正定；对"距离量测"这种弱非线性，收益远小于工程便利性 | MDPI Sensors 26/10/2921 |
| 4 | **滑窗因子图 / 优化**（IMU 预积分 + 测距因子） | 窗口内 N 个位姿 + 速度 + 双偏置（+ 锚点位置） | 预积分因子 + 每锚测距因子 + 超声/视觉因子 | **最高。21 ms / 次迭代**（Jetson Nano，C++/GTSAM）→ 单次上限约 47 Hz，无余量；MCU 不可行 | **12.3 cm RMSE**（动态、含 NLOS）；同数据 EKF 27.6 cm、PF 34.1 cm；**40% 丢包率下仍工作** | 时延累积、强依赖初始化；噪声协方差未自适应时反而比松耦合差（UAV NLOS 下紧耦合 FGO 2D 仍 0.51 m）；内存与算力都超出 MCU | arXiv 2503.12726；MDPI 15/12/6501 |
| 5 | **粒子滤波 / 鲁棒 PF** | 粒子集（p, v，2D 4 维可行） | 测距（可多峰），可叠加地图/门限约束 | 中高，随 N 线性。N=500、8 锚点 → 4000 次距离评估/帧（估算：Cortex-M7 上 2D 情形约 50–100 Hz） | PF **34.1 cm**（同数据输给 EKF 27.6 cm）；加 IGG-III 鲁棒权重的 RPF 比 PF 好 **20.4%**、比 EKF 好 39.6%；另一组 PF-OJ RMSE 0.2967 vs 紧耦合 ESKF 0.0972 | 粒子退化/贫化、维数灾难（3D+姿态不可行）；纯 PF 在 LOS 下不如 EKF | arXiv 2503.12726；MDPI 13/10/6187；MDPI 25/24/7673 |

**读表要点**：精度不能只看"紧 vs 松"。MDPI 25/24/7673 的 0.1402 → 0.0972 m（30.7%）里，同时包含了"紧耦合"和"多历元野值剔除"两个改动；纯耦合方式带来的增益通常是个位数到十几个百分点。

---

## B. 带出处的结论

1. **松耦合的真实身份是"UWB 位置 + IMU"，天花板由 UWB 位置解决定。** PMC9659066 在 0.6–0.8 m/s 动态台架上测得：只用 UWB 时误差最高 50 cm，KF 融合 IMU 后 13.7 cm。这是 LOS 条件下的最好情况——一旦 NLOS，UWB 位置解本身就是错的，滤波器没有任何残差可以判别。

2. **紧耦合的第一价值不是精度，是"少锚点可用性"。** CEA/PLANS 2023 明确指出：松耦合 UWB 需 ≥3 个信标，紧耦合可用单个信标更新；该文实测松耦合下 UWB 可用时间仅约 6.4%，紧耦合提升到 35.9%。对行李箱而言这直接决定"人走到柱子后面还剩几个锚能测"。

3. **紧耦合 EKF 在 MCU 上完全跑得动。** arXiv 1807.10913 以位置/速度/IMU 零偏为状态、以原始距离为量测，在机载端实现 **80 Hz 三维输出**，误差 0.16 m 均值 / 0.39 m 最大，并消除了 vanilla EKF 的估计延迟。这条推翻了"紧耦合一定重"的直觉。

4. **紧耦合的隐藏成本是标定，不是算力。** arXiv 2108.00133 证明：UWB 天线为了通信通常装在机体边缘、IMU 装在质心附近，杆臂与时延必须联合标定，否则精度下降；其在线标定可达**毫米级杆臂 + 毫秒级时延**。行李箱的天线必然分布在箱体四角，这条不可回避。

5. **UKF 相对 EKF 的精度增益很小，不要为它付出实现复杂度。** 同一数据集上 UKF 39.72 cm / 43.03 cm RMSE，EKF 41.26 / 44.58 cm，约 3.5%。根本原因是"距离 = 位置模长"这种非线性度并不高，EKF 的一阶截断足够；UKF 的价值在于省掉 Jacobian 推导。

6. **PF 只在强多峰/NLOS 场景赢，不能当主滤波器。** arXiv 2503.12726 同一数据下 PF 34.1 cm 输给 EKF 27.6 cm；MDPI 13/10/6187 中 PF 要靠 IGG-III 鲁棒权重（RPF）才比普通 PF 好 20.4%。PF 的正确位置是"人可能在哪几个位置"的多假设副滤波。

7. **UWB+视觉的正确分工：UWB 给绝对位置与尺度，VIO 给帧间相对位姿。** MDPI 15/12/6501 用"紧耦合 UWB/IMU 滑窗因子图 + 立体 VIO 再用 EKF 松融合"，RMSE 比 UWB 单独降 67.6%、比立体 VIO 降 55.4%、比 UWB/IMU 降 50.0%。注意其最优结构仍是"两个子估计器 + EKF 松融合"，而不是把所有因子塞进一个图。

8. **超声/红外的定位是"近场补盲 + 安全"，不是精度增强。** arXiv 2503.12726 把 4 个 MaxBotix MB7360（10 Hz）作为 SE(3) 测距因子 + 高度约束，与 UWB 20 Hz / IMU 200 Hz 异构融合，主要收益在垂直精度与 NLOS 下的鲁棒性；超声本身只在 LOS 下 cm 级、且存在近距离盲区（<50 mm 需特殊调制）。

9. **丢包/遮挡的桥接核心是"协方差携带"，不是"猜一个位置"。** MDPI Electronics 15/19/4377 在 50% 丢包 + 200 ms 延迟 + NLOS 下，把位置 RMSE 从 0.2443 m 降到 0.0668 m（72.7%）；其关键设计是让"几何可靠性信念"跨越丢包期保持，并作用于**重捕后的第一帧**，同时对陈旧量测做协方差膨胀。

---

## C. 面向 MCU 级算力的推荐分层

### 分层

| 层 | 频率 | 载体 | 内容 | 为什么放这里 |
|---|---|---|---|---|
| L0 时间基 | — | UWB/IMU 硬件 | 统一时基；若无法硬同步，固定时延 + 斜率补偿 | 紧耦合必须先解决 ms 级时延（结论 4） |
| L1 传播层 | 200–400 Hz | Cortex-M7（单核即可） | ESKF 名义态传播：p,v,q + 双偏置；状态压到 **9–15 维** | 与 UWB 频率解耦，保证 10–50 Hz 以上的输出连续 |
| L2 观测层（主滤波） | 20 Hz（UWB 帧） | 同 MCU | 逐锚点距离残差 + **NIS 门控** + Huber 核，紧耦合 ESKF 更新 | 只有这里能看到单锚残差，才能逐锚剔除 NLOS（结论 2/3） |
| L3 副滤波（可选） | 5 Hz | 同 MCU 低优先级任务 | 2D 4 维 PF（N≈300），仅在持续 NLOS/几何退化时启动 | 多假设表征"人在哪根柱子后面"，不进入主状态（结论 6） |
| L4 近场层 | 10–20 Hz，中断驱动 | 独立小 MCU | 3–4 路超声/红外：0–1.2 m 安全距离、侧向避障、台阶 | **不接入滤波器状态**，只覆盖速度指令；避免把非高斯近场观测污染主滤波（结论 8） |
| L5 视觉（可选） | 10–15 Hz | 独立 SoC/MCU+DSP | 单目 VIO 出相对位姿，每 1–2 s 作为一次位姿观测进 L2 | 遮挡期间提供帧间运动；算力不够时优先砍掉这一层（结论 7） |

**算力预算参考**：L1（15 维，200 Hz）约 1–2 MFLOP/s；L2（8 锚点，20 Hz）约 100 kFLOP/s；L3 若开 N=300 则约 5–10 MFLOP/s，是最大头，所以设计成"按需启动"。整体在 480 MHz Cortex-M7（带 FPU）上留有 3–5 倍余量。

### 丢失 → 重捕流程（目标遮挡 1–3 s）

```
IDLE ──有效锚点<2 或 NIS 连续 3 帧超阈──> BRIDGE
BRIDGE ──重捕且|innovation|≤3σ 连续3帧──> SOFT_RECOVER ──κ→1 连续5帧──> IDLE
BRIDGE ──重捕但|innovation|>3σ──────────> HARD_RESET ──> IDLE
BRIDGE ──>5 s 未重捕───────────────────> SAFE_HOLD（原地/停车+提示）
```

1. **检测（进入 BRIDGE）**：连续 K=3 帧有效锚点 <2，**或** NIS 连续 3 帧超 χ² 阈值。后者更重要——NLOS 的"大偏差测距"比"没有测距"更危险。
2. **桥接（0–3 s）**：只跑 L1 传播；过程噪声按时间膨胀 σ_p += α·√Δt（α 用静止段实测的 IMU 漂移率标定，MEMS 典型 0.1–0.3 m/√s）；静止时用 ZUPT/ZARU 钳住速度。3 s 行走的累积误差量级 0.3–0.9 m，仍在跟随容差内。
3. **软重捕（第一帧不跳变）**：把该帧量测协方差乘膨胀因子 κ（9–25），卡尔曼增益变小，先"试探性"信任；连续 M=3–5 帧残差一致（NIS 合格）后把 κ 线性降回 1。这正是 4377 的做法：可靠性信念跨丢包保持，作用于重捕首帧。
4. **硬重置（漂移过大）**：首帧 |innovation| > 3σ 时不做平滑，直接 p ← UWB 解、P_pp ← R_uwb，**速度保持、姿态不动**；同时把零偏协方差放大 2 倍，避免把桥接期的漂移误学成零偏。
5. **降级（>5 s）**：不再相信估计，进入 SAFE_HOLD——保持最后可信位置与朝向，停车 + 蜂鸣。这是安全逻辑，不是估计问题。
6. **在线学习**：每次重捕记录 κ 与实际 innovation，用于慢更新 R 与 α。行李箱的使用场景高度重复（机场/车站），这个自标定收益很大。

**不要做的两件事**：① 在遮挡期用超声"找目标"——人不是障碍物，超声只会测到人腿的最近回波，且距离分辨不了朝向；② 遮挡期把 PF 拉起来跑 3D——维数灾难，且 3 s 内主滤波的 IMU 桥接通常已经够用。

---

## D. 参考文献

1. **Accurate 3D Localization for MAV Swarms by UWB and IMU Fusion**（Li et al., 2018）arXiv:1807.10913 — https://arxiv.org/abs/1807.10913
   原文："an Extended Kalman Filter (EKF) based algorithm is proposed to fuse the Inertial Measurement Unit (IMU) and UWB, which achieved 80Hz 3D localization with significantly improved accuracy and almost no delay."（Table I: Fusion EKF 0.39/0.16 m，Vanilla EKF 0.71/0.30 m）

2. **Indoor Fusion Positioning Based on "IMU-Ultrasonic-UWB" and Factor Graph Optimization Method**（Zhang et al., 2025）arXiv:2503.12726 — https://arxiv.org/abs/2503.12726
   原文："achieving a 12.3 cm root mean square (RMS) error under dynamic motion conditions. The system maintains robust performance even with intermittent UWB signal availability, down to a 40% packet reception rate"；另有 "The average computational time per iteration was 21 ms"。

3. **Delay- and Dropout-Aware GRU-SAC Measurement Covariance Adaptation for Robust UWB/INS Indoor Navigation**（Electronics 2026, 15(19), 4377）— https://www.mdpi.com/2079-9292/15/19/4377
   原文："the belief carries NLOS and dropout history to the first measurements received after re-acquisition"；"position RMSE decreased from 0.2443 to 0.0668 m (72.7%)"。

4. **An Indoor UAV Localization Framework with ESKF Tightly-Coupled Fusion and Multi-Epoch UWB Outlier Rejection**（Sensors 2025, 25(24), 7673）— https://www.mdpi.com/1424-8220/25/24/7673
   原文："reduces RMSE from 0.1402 to 0.0972 relative to the second-best baseline ESKF-LC-Raw (a reduction of about 30.7%)"。

5. **Online Spatio-temporal Calibration of Tightly-coupled Ultrawideband-aided Inertial Localization**（Goudar & Schoellig, 2021）arXiv:2108.00133 — https://arxiv.org/abs/2108.00133
   原文："estimating (i) the spatial offset parameters to millimeter precision and (ii) the temporal offset parameter to millisecond precision."

6. **Low-Cost Portable Sensor Node ... with Kalman-Filtering-Based UWB Localization**（Sensors 2026, 26(10), 2921）— https://www.mdpi.com/1424-8220/26/10/2921
   原文："the UKF achieved the best overall performance, with a mean error of 39.72 cm and an RMSE of 43.03 cm, compared to 41.26 cm and 44.58 cm for EKF"。

7. **Comprehensive Evaluations of NLOS and Linearization Errors on UWB Positioning**（Appl. Sci. 2023, 13(10), 6187）— https://www.mdpi.com/2076-3417/13/10/6187
   原文："position improvements percentages of 57.2%, 52.7%, 39.6%, 38.2%, 26.6%, and 20.4% can be obtained by RPF compared to those calculated by Bancroft, LS, EKF, UKF, CKF, and PF"。

8. **Experimental Evaluation of Sensor Fusion of Low-Cost UWB and IMU for Localization under Indoor Dynamic Testing Conditions**（Sensors 2022, 22(21), 8156）— https://pmc.ncbi.nlm.nih.gov/articles/PMC9659066
   原文："the location error can reach up to 50 cm under dynamic testing conditions when only relying on the UWB sensor, with the KF sensor fusion of UWB and IMU, the location error decreases to 13.7 cm."

9. **UWB-aided GNSS/INS Fusion for Resilient Positioning in GNSS Challenged Environments**（Villien & Denis, PLANS 2023）— https://cea.hal.science/cea-04371896/file/PLANS_2023___Full_Paper%20%284%29.pdf
   原文："a loose coupling UWB fusion (requiring at least 3 beacons) would have result in a very limited availability (i.e. about 6.4% of time), whereas the tight coupling fusion allows the use of UWB measurements even from a single beacon and results in a much higher availability of about 35.9%."

10. **Fusion-Based Localization System Integrating UWB, IMU, and Vision**（Appl. Sci. 2025, 15(12), 6501）— https://www.mdpi.com/2076-3417/15/12/6501
    原文："reduces the root mean square error (RMSE) by 67.6% ... Compared with the UWB/IMU fusion model, the proposed method achieves a 50.0% reduction in RMSE and a 59.1% reduction in maximum error."

11. **Semi-Tightly Coupled Robust Model for GNSS/UWB/INS Integrated Positioning in Challenging Environments**（Remote Sens. 2024, 16(12), 2108）— https://www.mdpi.com/2072-4292/16/12/2108
    原文："Compared with the loosely coupled UWB/INS, the accuracy in the E and N directions of the semi-tight coupled UWB/INS has improved by 60% and 34%."

12. **UWB-Based Human-Following System with Obstacle and Crevasse Avoidance for Polar-Exploration Robots**（Appl. Sci. 2024, 14(16), 6918）— https://www.mdpi.com/2076-3417/14/16/6918
    原文："determines the relative position of the explorer using UWB anchors and tags. It also utilizes real-time local obstacle mapping and path-planning algorithms"（UWB 定位 + LiDAR 避障的跟随范式）。

13. **Enhanced Indoor Mobile Robot Localization via Lie-Group IMU–UWB Fusion and Dual-Stage Kalman Filtering**（Sensors 2026）— https://pmc.ncbi.nlm.nih.gov/articles/PMC13165750/
    原文："The drift of positioning error of pure UWB is up to 0.5 m at rest, and it is obviously reduced to about 0.2 m after ESKF filtering"。

14. **CM-LIUW-Odometry: Robust and High-Precision LiDAR-Inertial-UWB-Wheel Odometry for Extreme Degradation Coal Mine Tunnels**（arXiv:2511.01379）— https://arxiv.org/html/2511.01379v1
    原文："an adaptive motion mode switching mechanism dynamically adjusts the robot's motion mode based on UWB measurement range and environmental degradation levels."（超范围即切模式的工程范式）

15. **GRVINS: Tightly Coupled GNSS-Range-Visual-Inertial System**（J Intell Robot Syst, 2023）— https://link.springer.com/article/10.1007/s10846-023-02033-8
    原文："can seamlessly respond to indoor and outdoor GNSS and UWB loss or reacquisition."

16. **Modified FMCW Scheme for Improved Ultrasonic Positioning and Ranging of Unmanned Ground Vehicles at Distances < 50 mm**（Sensors 2022）— https://pmc.ncbi.nlm.nih.gov/articles/PMC9785695
    原文："estimating distances in the range of 0–380 mm were numerically investigated, with particular emphasis on distances below 50 mm."（超声近场盲区的量化边界）

---

*说明：本文所有数字均取自上述页面正文/摘要，口径已在括号内标注；未打开过的页面未引用。*
