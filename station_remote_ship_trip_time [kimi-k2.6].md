# 《戴森球计划》星际物流运输船飞行剩余时间分析

## 范围

- 反编译工具：`ilspycmd 10.1.0.8386`。
- 主模块：`/mnt/d/Steam/steamapps/common/Dyson Sphere Program/DSPGAME_Data/Managed/Assembly-CSharp.dll`。
- 分析对象：`StationComponent.InternalTickRemote`。
- 术语参考：`Locale/2052` 与 `Prototypes`，采用「星际物流运输船」「空间翘曲器」「曲速飞行」「运输船引擎」等中文名称。
- 执行顺序：先隔离分析 `InternalTickRemote`，未读取 `CalcRemoteSingleTripTime` 与 `CalcArrivalRemainingTime`；完成独立分析后，再读取两者并对比。

下文 `tick` 均为 1/60 秒。速度按「宇宙坐标单位 / 秒」理解，单 tick 位移为 `speed / 60`。

## 科研数据与速度

`InternalTickRemote` 接收已经计算好的速度：

- `shipSailSpeed`：星际物流运输船非曲速最大航行速度。
- `shipWarpSpeed`：星际物流运输船曲速速度。

`GameHistoryData` 中对应字段为：

```text
logisticShipSailSpeedModified = logisticShipSailSpeed * logisticShipSpeedScale
logisticShipWarpSpeedModified = logisticShipWarpSpeed * logisticShipSpeedScale
```

`运输船引擎` 科技会增加 `logisticShipSpeedScale`，第 4 级设置 `logisticShipWarpDrive = true`，允许星际物流运输船消耗 `空间翘曲器` 进行曲速飞行。`空间翘曲器` 的物品 ID 为 `1210`。

闭式估算可先换算：

```text
v_s = logisticShipSailSpeed * logisticShipSpeedScale
v_w = logisticShipWarpSpeed * logisticShipSpeedScale
can_warp_tech = logisticShipWarpDrive && v_w > v_s + 1
```

如果调用方已经给出 `shipSailSpeed` 与 `shipWarpSpeed`，则不必再按科技树反推。

## 飞行阶段

`ShipData.stage` 和 `ShipData.direction` 共同决定星际物流运输船所处阶段。

- `direction > 0`：从起始星际物流运输站飞向目标星际物流运输站。
- `direction < 0`：返航。

| stage | 含义 | 运动模型 | 闭式性 |
| --- | --- | --- | --- |
| `< -1` | 起始站停泊或返航入库收尾 | 船固定在起始泊位，`t` 线性变化 | 有 |
| `-1` | 起始站出库 / 入库插值 | `smoothstep(t)=t^2(3-2t)` 平滑插值，出库外移 25 米 | 有 |
| `0` | 太空航行主阶段 | 速度、曲速状态、朝向、避障逐 tick 更新 | 只能做近似闭式 |
| `1` | 目标站进港 / 离港插值 | `smoothstep` 与姿态插值 | 有 |
| `> 1`，通常 `2` | 目标站停靠、卸货、装货或等待返航 | 船固定在目标泊位，`t` 线性变化 | 有 |

出入库速率为：

```text
p = (v_s / 600)^0.4
lift_rate = p * 0.006 + 0.00001
```

`stage == -1` 或 `stage == 1` 的插值都以 `lift_rate` 为核心。`stage == 2` 的停靠计时约为 30 tick。

## 太空航行主阶段

`stage == 0` 是唯一复杂阶段。

### 目标点

飞船并不飞向星球中心，而是飞向运输站泊位外侧约 25 米：

- 去程目标：目标站 `shipDockPos + 25 * normalized(shipDockPos)`。
- 返航目标：起始站 `shipDiskPos[shipIndex] + 25 * normalized(shipDiskPos[shipIndex])`。

到目标点距离小于阈值时结束主航行：

```text
r = v_s / 600
p = r^0.4
shape = p <= 1 ? p : ln(p) + 1
arrival_threshold = 6 * shape
```

### 非曲速速度控制

非曲速状态下，代码先计算目标速度：

```text
tau = d_target / (uSpeed + 0.1) * 0.382
v_target = uSpeed * tau * shape + 6 * p + 0.15 * min(v_s / 600, 500)
v_target = min(v_target, v_s)
```

接近目的地时可近似为：

```text
v_target(d) = k * d + c
k = 0.382 * shape
c = 6 * p + 0.15 * min(v_s / 600, 500)
```

加速度分近起始天体与远离起始天体两档：

```text
a_near = 0.03 * v_s
a_far = 0.12 * v_s * shape
```

每 tick 用 `a/60` 推近 `v_target`。若速度过高，则使用较大的 `brake_step = 0.4 * v_s * min(v_s / 600, 500)` 拉回。这使非曲速主航行近似为「加速、匀速、按 `v=k*d+c` 进场」三段，但完整离散逻辑不是单一二次闭式。

### 曲速逻辑

曲速启动条件为：

```text
v_w > v_s + 1
warpState <= 0
距起始天体距离平方 > 25,000,000
到目标点距离 > warpEnableDist / 2
uSpeed >= v_s
有空间翘曲器，或演示模式免费
```

启动时消耗 1 个空间翘曲器，并令 `warpState += 1/60`。

曲速附加速度为：

```text
warp_cap = min(v_w, 2 * 两星球当前距离)
warp_extra(w) = warp_cap * (1001^w - 1) / 1000
uSpeed = v_s + warp_extra(w)
```

退出判定使用安全距离：

```text
safe_exit(w) = 0.0449 * warp_extra(w) + 5000 + 0.25 * v_s
if d_target < safe_exit(w): warpState -= 1/15
else:                       warpState += 1/60
```

曲速段理论上可对指数速度做积分，但游戏实际还包括离散 tick、一步限速、目标移动、朝向转动和避障。因此，完整 `InternalTickRemote` 不适合写成一个严格闭式表达式。

### 无法由给定输入闭式恢复的信息

题目给定输入不足以恢复以下状态：

- 当前方向向量 `uVel` 与姿态 `uRot`。
- 起始和目标星系内其他天体的位置、半径、下一 tick 位置。
- 避障造成的绕行距离。
- 目标塔泊位朝向、船位 `shipIndex` 的泊位偏移。
- `warpEnableDist` 等运输站参数，除非额外传入。
- 当前是去程还是返航，以及当前 `stage` 和插值参数 `t`，除非额外传入。

抛弃这些信息后，对长距离曲速航行影响通常较小，因为大部分时间由曲速巡航决定；对短距离、近行星绕行、刚起飞和即将进港的情况影响较大。

## 推荐的非模拟闭式剩余时间算法

### 输入

```text
v_current        当前 uSpeed
warp_state       当前 warpState，范围 [0,1]
has_warper       船上是否有空间翘曲器
d_start          到起始塔或起始天体参考点距离
d_target         到目标泊位外侧目标点距离
history          GameHistory 速度与曲速科技字段
warp_enable_dist 运输站曲速启用距离，建议额外传入
include_docking  是否加进港插值时间
```

### 常量

```text
v_s = history.logisticShipSailSpeed * history.logisticShipSpeedScale
v_w = history.logisticShipWarpSpeed * history.logisticShipSpeedScale
can_warp = history.logisticShipWarpDrive && has_warper && v_w > v_s + 1

r = v_s / 600
p = r^0.4
shape = p <= 1 ? p : ln(p) + 1
r_cap = min(r, 500)
a = 0.12 * v_s * shape
k = 0.382 * shape
c = 6 * p + 0.15 * r_cap
arrival = 6 * shape
lift_rate = p * 0.006 + 0.00001
```

### 非曲速估算

先计算进场段距离：

```text
d_approach = max((v_s - c) / k, 0)
d_acc = max((v_s^2 - v_current^2) / (2 * a), 0)
```

若存在加速和匀速段：

```text
if d_target > d_acc + d_approach:
    t = max((v_s - v_current) / a, 0)
      + (d_target - d_acc - d_approach) / v_s
      + ln((k * d_approach + c) / (k * arrival + c)) / k
```

若无匀速段，求峰值速度：

```text
A = 1 / (2*a)
B = 1 / k
C = -(d_target + v_current^2/(2*a) + c/k)
v_peak = (-B + sqrt(B^2 - 4*A*C)) / (2*A)

t = max((v_peak - v_current) / a, 0)
  + ln(max(v_peak, c) / (k * arrival + c)) / k
```

若已经非常接近目标，可直接用：

```text
t = ln((k * d_target + c) / (k * arrival + c)) / k
```

最后：

```text
ticks = ceil(60 * max(t, 0))
if include_docking:
    ticks += ceil(1.5 / lift_rate) + 30
```

### 曲速估算

保留主要常数：

```text
warp_cap = min(v_w, 2 * max(d_start + d_target, d_target))
safe_full = 0.0449 * warp_cap + 5000 + 0.25 * v_s
```

若 `warp_state == 0` 且不能满足启动条件，则退化为非曲速估算。启动条件建议用：

```text
can_warp && d_start > 5000 && d_target > warp_enable_dist / 2 && v_current >= v_s
```

曲速加速段中 `warpState` 每秒增加 1。速度积分为：

```text
S_up(w0,w1) = (v_s - warp_cap/1000) * (w1-w0)
            + warp_cap/1000 * (1001^w1 - 1001^w0) / ln(1001)
```

曲速退出段中 `warpState` 每秒减少 4。从 `w0` 降到 `w1` 的距离为：

```text
S_down(w0,w1) = 1/4 * [ (v_s - warp_cap/1000) * (w0-w1)
              + warp_cap/1000 * (1001^w0 - 1001^w1) / ln(1001) ]
```

推荐计算步骤：

1. 如果 `v_current < v_s`，先用常加速度估计加速到 `v_s`，扣除位移。
2. 预留 `safe_full` 给退出曲速和非曲速进场。
3. 对剩余距离应用 `S_up`。若距离不足以升到 `warpState = 1`，用二分或牛顿法求 `w_end`，时间为 `w_end - warp_state`。这是解指数方程，不是逐 tick 模拟。
4. 若升到 `warpState = 1` 后仍有距离，则中间巡航速度近似取 `v_s + warp_cap`。
5. 到 `safe_full` 后按曲速下降时间约 `warp_state / 4` 秒，再接非曲速进场估算。

这套算法计算量很小。对远距离曲速航线，主要误差来自 `warp_cap` 中「两星球当前距离」的近似、离散 tick 的一步限速以及星体运动。对短距离非曲速航线，误差主要来自避障和 `v_target` 离散更新。

## 与游戏内两个估算函数的对比

完成上述独立分析后，读取了 `CalcRemoteSingleTripTime` 与 `CalcArrivalRemainingTime`。

### `CalcRemoteSingleTripTime`

该函数计算一次完整单程的估计 tick。它使用的结构与独立闭式分析高度一致：

- 使用 `p=(v_s/600)^0.4`、`shape`、`lift_rate`。
- 把起飞、太空航行、进港和停靠加成合并。
- 非曲速分为长距离、短距离和极短距离三类。
- 曲速分为是否能曲速、曲速进入、巡航和退出。

但它假设「从运输站重新出发」，不是任意当前状态。因此它适合调度前预估，不适合已经在半路上的飞船。

### `CalcArrivalRemainingTime`

该函数针对已经存在的 `ShipData` 估算到达剩余 tick。它读取当前：

- `stage`、`direction`、`t`。
- `uSpeed`、`warpState`。
- 当前位置 `uPos`。
- 起始与目标天体、泊位目标点。
- `warperCnt`。

它在 `stage != 0` 时按阶段扣减或补算；在 `stage == 0` 时按当前距离、当前速度和当前曲速状态进入多分支闭式估算。曲速部分还考虑了 `warpState` 已经处于上升、满速或下降的不同情况。

### 谁更准

1. 与真实 `InternalTickRemote` 逐 tick 结果相比，最准确的是直接模拟 `InternalTickRemote`，因为只有它保留朝向、避障、天体运动补偿和离散限速。
2. 在不模拟的估算中，`CalcArrivalRemainingTime` 通常比本文推荐算法更准，因为它能读取完整 `ShipData` 与 `AstroData`；本文算法按题目给定输入主动舍弃了方向、姿态、目标天体细节和避障状态。
3. 本文算法比 `CalcRemoteSingleTripTime` 更适合「半路上的飞船」，因为 `CalcRemoteSingleTripTime` 只估完整单程，不能表达当前速度和当前 `warpState`；但如果飞船尚未出发，`CalcRemoteSingleTripTime` 更贴近游戏内部常数和分支。

因此，若运行环境能访问游戏对象，应优先调用或复刻 `CalcArrivalRemainingTime`。若只能使用题目给定的少量标量输入，本文闭式算法是更可移植的近似方案，但不会比游戏内 `CalcArrivalRemainingTime` 更准。
