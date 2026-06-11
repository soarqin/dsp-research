# 《戴森球计划》星际物流运输船远程飞行时间分析

## 结论摘要

`StationComponent.InternalTickRemote` 中的星际物流运输船不是按「匀速直线」运动。它由 `stage` 和 `direction` 组成状态机：起飞动画、自由飞行、到站入坞、站内装卸、返航起飞、返航自由飞行、回站入坞、回站收尾。

自由飞行阶段存在可用的近似闭式解。若只要求估算当前自由飞行段剩余时间，可以把路径视为静止目标之间的直线，把速度包络分为「普通航行」和「曲速航行」两段计算；误差主要来自转向控制、天体运动补偿、避障修正和入坞动画。

完全精确的闭式解不存在。原因是游戏每 tick 会根据当前姿态、角速度、目标方向、附近天体、天体下一帧位置、`gene + j + timeGene` 的降频更新条件、`warpState` 分段逻辑共同更新位置。只给出速度、`warpState`、是否带空间翘曲器、到起始塔和目标塔的距离，无法唯一恢复方向、当前 `stage`、动画进度 `t`、姿态和附近天体状态。

## 术语与数据来源

术语按 `Locale/2052` 校正：

- `ShipData` 对应本文的「星际物流运输船」。2052 文本中物品名为「星际物流运输船」。
- `warperCnt` 对应船上携带的「空间翘曲器」数量。
- `warpState` 对应曲速启动程度，范围大致为 `[0, 1]`。
- `shipSailSpeed` 对应「航行速度」。实际传入值为 `GameHistoryData.logisticShipSailSpeedModified`。
- `shipWarpSpeed` 对应「曲速航行速度」。启用曲速科技后传入 `GameHistoryData.logisticShipWarpSpeedModified`，否则等于航行速度。
- 相关科技名为「运输船引擎」。`TechProtoSet.json` 中该科技的第 3 级开始提升星际物流运输船航行速度，第 4 级允许星际物流运输船使用曲速航行。

`GameHistoryData` 中与本计算直接相关的字段如下：

```text
sailSpeed = logisticShipSailSpeed * logisticShipSpeedScale
warpSpeed = logisticShipWarpDrive ? logisticShipWarpSpeed * logisticShipSpeedScale : sailSpeed
```

普通模式默认值来自 `ModeConfig`：

```text
logisticShipSailSpeed = 400
logisticShipWarpSpeed = 400000
logisticShipSpeedScale = 1
logisticShipWarpDrive = false
```

## InternalTickRemote 中的阶段状态机

`ShipData.stage` 和 `ShipData.direction` 共同决定飞船阶段。

| `stage` | `direction` | 阶段含义 | 主要运动 |
| --- | --- | --- | --- |
| `< -1`，通常为 `-2` | `> 0` | 起始塔出发前收尾动画 | 固定在起始塔船位，`t += 0.03335`，约 0.5 秒后进入 `stage = -1` |
| `-1` | `> 0` | 起始塔起飞 | 从停泊圆盘外推约 25 米，平滑插值，速度置 0，完成后进入 `stage = 0` |
| `0` | `> 0` | 去目标塔自由飞行 | 姿态导引、普通航行加速、可选曲速、附近天体避让，距离目标接泊点小于 `6 * turnScale` 后进入 `stage = 1` |
| `1` | `> 0` | 目标塔入坞 | 从自由飞行末端插值到目标塔船坞附近，速度置 0，完成后进入 `stage = 2` |
| `2` 或其他正阶段 | `> 0` | 目标塔装卸前停留 | 固定在目标塔，`t -= 0.0334`，约 0.5 秒后执行卸货、补充空间翘曲器和取货，然后 `direction = -1` |
| `2` 或其他正阶段 | `< 0` | 目标塔返航前停留 | 固定在目标塔，`t += 0.0334`，约 0.5 秒后进入 `stage = 1` |
| `1` | `< 0` | 目标塔起飞 | 从目标塔船坞外推出约 39.4 米，速度置 0，完成后进入 `stage = 0` |
| `0` | `< 0` | 回起始塔自由飞行 | 与去程自由飞行同一套逻辑，目标改为起始塔接泊点，接近后进入 `stage = -1` |
| `-1` | `< 0` | 起始塔入坞 | 从自由飞行末端插值回起始塔圆盘，速度置 0，完成后进入 `stage = -2` |
| `< -1`，通常为 `-2` | `< 0` | 回站收尾 | 固定在起始塔，`t -= 0.03335`，约 0.5 秒后交货并把船放回空闲池 |

除 `stage = 0` 外，其他阶段都不使用实际飞行速度，位置是插值或固定点。这些阶段的剩余时间可以由 `stage`、`direction`、`t` 和当前航行速度直接闭式得到。

## 自由飞行阶段的运动逻辑

以下变量使用反编译代码中的等价含义：

```text
s = shipSailSpeed
r = s / 600
p = r^0.4
turnScale = p <= 1 ? p : ln(p) + 1
rCap = min(r, 500)
animStep = p * 0.006 + 0.00001
arrivalRadius = 6 * turnScale
```

### 普通航行

自由飞行中，代码每次重新导引时会计算目标速度：

```text
lookAhead = distanceToDock / (currentSpeed + 0.1) * 0.382
desiredSpeed = currentSpeed * lookAhead * turnScale + 6 * p + 0.15 * rCap
desiredSpeed = min(desiredSpeed, s)
```

当船还离目标很远时，`desiredSpeed` 会被钳制到 `s`，飞船按每秒约 `s * 0.12 * turnScale` 加速到航行速度。靠近目标时，`desiredSpeed` 近似变成距离的一次函数：

```text
desiredSpeed(distance) ≈ min(s, k * distance + c)
k = 0.382 * turnScale
c = 6 * p + 0.15 * rCap
```

因此普通航行可以近似拆成两段：

- 远距离段：常加速度到 `s`，之后匀速 `s`。
- 近距离段：按 `v(distance) = k * distance + c` 接近目标，积分后有对数形式闭式解。

近距离段从 `D1` 到 `D0` 的耗时近似为：

```text
T_near(D1 -> D0) = ln((k * D1 + c) / (k * D0 + c)) / k
```

其中 `D0 = arrivalRadius`。

### 曲速航行

只有以下条件同时满足时，`InternalTickRemote` 才会从普通航行进入曲速：

```text
shipWarpSpeed > shipSailSpeed + 1
distanceFromDeparturePlanetCenter^2 > 25,000,000
distanceToDock > warpEnableDist * 0.5
currentSpeed >= shipSailSpeed
warperCnt > 0 或 warperFree
```

默认 `warpEnableDist = 480000`，因此自由飞行中距离目标接泊点小于约 `240000` 时不会从 `warpState = 0` 新启动曲速。

曲速的附加速度为：

```text
W = min(shipWarpSpeed, 2 * planetDistance)
warpAdd(w) = W * (1001^w - 1) / 1000
speed(w) = shipSailSpeed + warpAdd(w)
```

其中 `planetDistance` 是起始星球和目标星球之间的当前距离。若只输入飞船到起始塔、目标塔的距离，可以用 `dStart + dTarget` 近似它。

`warpState` 的变化是分段的：

- 远离目标时，每 tick 增加 `1 / 60`，也就是约 1 秒从 0 增至 1。
- 进入刹车区时，每 tick 减少 `1 / 15`，也就是约 0.25 秒从 1 降至 0。
- 刹车区按当前曲速附加速度动态计算：

```text
brakeDistance(w) = warpAdd(w) * 0.0449 + 5000 + shipSailSpeed * 0.25
```

曲速加速段的距离积分有闭式形式。因为 `warpState` 远离目标时每秒增加 1，故从 `w0` 到 `w1` 的距离为：

```text
S_up(w0, w1) = (s - W / 1000) * (w1 - w0)
             + W / (1000 * ln(1001)) * (1001^w1 - 1001^w0)
```

曲速降速段每秒减少约 4，因此距离为：

```text
S_down(w1 -> w0) = S_up(w0, w1) / 4
```

这些公式只描述速度包络，不描述姿态导引和天体避障。

## 直接非逐帧估算剩余时间的方法

输入建议拆成两类：

```text
v0: 当前 ShipData.uSpeed
w0: 当前 ShipData.warpState
hasWarper: 当前 ShipData.warperCnt > 0；若 w0 > 0，可视为 true
dStart: 飞船到起始塔接泊点或塔中心的距离
dTarget: 飞船到目标塔接泊点或塔中心的距离
leg: 当前飞行目标，outbound 使用 dTarget，return 使用 dStart
history.logisticShipSailSpeed
history.logisticShipWarpSpeed
history.logisticShipSpeedScale
history.logisticShipWarpDrive
station.warpEnableDist，缺省 480000
```

若没有 `leg`，仅靠 `dStart` 和 `dTarget` 无法判断飞船正在去目标塔还是返航。调用方应返回两种候选，或从 `ShipData.direction`、货物状态、速度方向中补充判断。

### 步骤 1：计算科技修正后的速度

```text
s = logisticShipSailSpeed * logisticShipSpeedScale
warpCap = logisticShipWarpDrive ? logisticShipWarpSpeed * logisticShipSpeedScale : s
canWarpByTech = warpCap > s + 1
```

### 步骤 2：选择当前目标距离

```text
D = leg == outbound ? dTarget : dStart
routeDistance = dStart + dTarget
arrivalRadius = 6 * turnScale
```

若 `D <= arrivalRadius`，自由飞行段可视为已经结束。

### 步骤 3：普通航行闭式估算

定义：

```text
a = s * 0.12 * turnScale
k = 0.382 * turnScale
c = 6 * p + 0.15 * rCap
D_full = max(arrivalRadius, (s - c) / k)
```

普通航行剩余时间近似：

```text
nearTime(D1, D0) = ln((k * D1 + c) / (k * D0 + c)) / k

if D <= D_full:
    T_sail = nearTime(D, arrivalRadius)
else:
    far = D - D_full
    if v0 < s:
        tAcc = (s - v0) / a
        dAcc = (s*s - v0*v0) / (2*a)
        if far > dAcc:
            T_sail = tAcc + (far - dAcc) / s + nearTime(D_full, arrivalRadius)
        else:
            T_sail = (-v0 + sqrt(v0*v0 + 2*a*far)) / a + nearTime(D_full, arrivalRadius)
    else:
        T_sail = far / s + nearTime(D_full, arrivalRadius)
```

这是一个「不逐帧」的闭式估算。它用距离函数替代了游戏中的姿态控制和分帧速度钳制。

### 步骤 4：曲速航行闭式估算

若不满足以下任一条件，直接使用 `T_sail`：

```text
canWarpByTech == true
w0 > 0 或 hasWarper == true
D > warpEnableDist * 0.5
```

否则估算曲速包络：

```text
W = min(warpCap, 2 * routeDistance)
warpAdd(w) = W * (1001^w - 1) / 1000
brakeDistance(w) = warpAdd(w) * 0.0449 + 5000 + s * 0.25
S_up(w0, w1) = (s - W / 1000) * (w1 - w0)
             + W / (1000 * ln(1001)) * (1001^w1 - 1001^w0)
S_down(w1, w0) = S_up(w0, w1) / 4
```

实用算法：

```text
1. 若 v0 < s，先用普通航行的常加速度段把速度提升到 s，并扣除对应距离。
2. 令 B = brakeDistance(1)。
3. 若剩余距离小于 B + S_up(max(w0, 0), 1)，不强行估算短距离曲速，返回普通航行结果。
4. 曲速升至满状态：T += 1 - max(w0, 0)，距离扣除 S_up(max(w0, 0), 1)。
5. 满曲速巡航：T += (D_remaining - B) / (s + W)，距离扣到 B。
6. 曲速降速：T += 0.25，距离扣除 S_down(1, 0)。
7. 剩余距离用普通航行近距离公式收尾。
```

短距离刚好进入曲速但达不到满 `warpState = 1` 时，可以解方程 `S_up(w0, wPeak) = availableDistance` 得到 `wPeak`。这个方程单调，牛顿迭代 2 到 4 次即可，仍然不是逐帧模拟。若追求公式简单，可以直接放弃这段短曲速并按普通航行估算；影响通常小于几秒，因为曲速启动和降速总时长只有约 1.25 秒，但在非常短的跨星球航线中会偏慢。

### 步骤 5：加入入坞或收尾动画

若目标是「自由飞行到接泊点」，无需加入动画时间。若目标是「完成本段到站事件」，需要补上动画：

```text
animStep = p * 0.006 + 0.00001
dockInTime = 1 / (animStep * 2 / 3) / 60
stationHoldTime = 1 / 0.0334 / 60
homeHoldTime = 1 / 0.03335 / 60
```

去程完成目标塔装卸前，约增加：

```text
dockInTime + stationHoldTime
```

返程完成回站收尾，约增加：

```text
dockInTime + homeHoldTime
```

如果当前已经处于起飞、入坞或停站阶段，则必须额外输入 `stage`、`direction`、`t`。否则无法只靠距离和速度判断动画剩余时间。

## 不能从指定输入中闭式得到的内容

以下内容会影响精确时间，但不在指定输入中：

- 当前 `stage`、`direction` 和动画进度 `t`。缺失后，最多相差约一次起飞、入坞或停站动画，通常是 0.5 秒到十几秒，取决于航行速度对应的 `animStep`。
- 船的姿态 `uRot`、方向向量 `uVel`、角速度 `uAngularVel`。缺失后无法得到转向滞后，远距离直线飞行影响很小，刚起飞和接近目标时影响更明显。
- 起始星球和目标星球当前中心距离 `planetDistance`。用 `dStart + dTarget` 近似时，飞船不在线段上会造成曲速上限 `min(shipWarpSpeed, 2 * planetDistance)` 偏差。
- 当前位置到出发星球中心的距离。曲速启动条件要求该距离大于 5000，塔距离不能完全替代。
- `AstroData.uPosNext` 和附近天体半径。代码会对近天体做规避和位置补偿，无法从两段塔距恢复。
- `gene + j + timeGene` 导致的 10 tick 或 30 tick 降频导引。它会让速度和姿态更新有最多约 0.5 秒的离散误差。

对星际长距离运输而言，最大耗时通常来自满速航行或满曲速巡航；上述缺失项对总时间影响较小。对短距离、刚起飞、即将入坞、曲速刚启动或刚降速的状态，误差会明显变大。

## 初始建议

面向工具或计算器实现，建议提供两个模式：

- 快速模式：使用本文的普通航行闭式公式和满曲速分段公式。适合大批量估算，复杂度 O(1)。
- 精准模式：不逐帧模拟完整姿态，但在短曲速段求解 `wPeak`，并要求额外输入 `stage`、`direction`、`t`。适合显示单艘星际物流运输船的剩余到站时间。

在只有题述输入的情况下，最稳妥的 API 返回值应包含：

```text
timeToStartCandidate
timeToTargetCandidate
assumptions
missingStateImpact
```

调用方若能额外提供 `ShipData.direction`，即可把候选值收敛为当前这一程的单值。

## 与游戏自带估算函数的对比

第一轮分析完成后，继续读取了 `StationComponent.CalcRemoteSingleTripTime` 和 `StationComponent.CalcArrivalRemainingTime`。结论是：这两个函数也是闭式或经验闭式估算，并不调用 `InternalTickRemote` 做逐帧模拟。

### CalcRemoteSingleTripTime 的方法

`CalcRemoteSingleTripTime` 用于运输站路线面板的整段路线预估。UI 中会分别计算去程和返程，并相加显示预估耗时。

它的输入是起始站、目标站、航行速度、曲速航行速度、是否有空间翘曲器、方向和船位索引。它先取起始塔圆盘外推 25 米的位置，以及目标塔船坞外推 25 米的位置，再用两点距离作为整段路程。

普通航行分支与前文推导高度一致：

- 使用 `p = (shipSailSpeed / 600)^0.4` 和 `turnScale = p > 1 ? ln(p) + 1 : p`。
- 使用 `k = 0.382 * turnScale` 和 `c = 0.15 * min(shipSailSpeed / 600, 500) + 6 * p`。
- 使用 `ln((k * D + c) / (...)) / k` 形式估算末端接近时间。
- 根据距离拆成「近起点加速」「远距离加速到航行速度」「匀速航行」「末端接近」几段。

曲速分支不是严格积分 `warpAdd(w) = W * (1001^w - 1) / 1000`，而是使用经验常数：

```text
W = min(shipWarpSpeed, 2 * routeDistance)
warpTransitionDistance ≈ W * 0.18775714286 + shipSailSpeed * 1.33333333333
fullWarpTransitionTime = 80 ticks
fullWarpCruiseSpeed = W + shipSailSpeed
brakeBaseDistance = 5000 + shipSailSpeed * 0.25
```

如果距离不足以进入完整曲速巡航，它使用下面的经验式估算短曲速段时间：

```text
x = (remainingDistanceAfterStartAndBrake) / warpTransitionDistance
timeTicks ≈ 6400 / (79 + 1 / x)
```

源码中表达式为 `6400.0 / (79.0 * Math.Pow(x, 0.0) + 1.0 / x)`，由于 `Math.Pow(x, 0.0)` 恒为 1，所以等价于上式。

### CalcArrivalRemainingTime 的方法

`CalcArrivalRemainingTime` 用于运输站面板中的当前在途船 ETA。UI 调用路径会将结果除以 60 并向上显示为秒数。

它比 `CalcRemoteSingleTripTime` 使用更多当前状态：

- `ShipData.stage`
- `ShipData.direction`
- `ShipData.t`
- `ShipData.uSpeed`
- `ShipData.warpState`
- `ShipData.warperCnt`
- 当前飞船位置到出发星球中心的距离
- 当前飞船位置到本段目标接泊点的距离
- 起始站和目标站当前宇宙坐标

当飞船不在 `stage = 0` 时，它不会估算空间飞行，而是用 `stage * direction` 判断飞船是在本段自由飞行之前还是之后：

- 自由飞行之前：调用 `CalcRemoteSingleTripTime` 得到完整单程，再按 `stage` 和 `t` 扣除已完成的起飞或停站动画。
- 自由飞行之后：只计算剩余入坞或停站动画。

当飞船处于 `stage = 0` 时，它直接基于当前速度和当前距离估算剩余自由飞行时间。普通航行分支仍使用与前文相同的加速、匀速和末端对数公式。曲速分支分为三种情况：

- 仍在出发星球附近，尚未满足曲速启动距离。
- 已离开出发星球，但 `warpState <= 0`，可在后续启动曲速。
- `warpState > 0`，已经处于曲速启动、满曲速或降速阶段。

已经处于曲速时，它用 `warpState` 修正剩余时间，但仍是经验式。例如源码中使用 `shipData.warpState^7 / 7` 近似已经走过的曲速过渡距离，并用 `shipData.warpState * 20` 近似非常接近目标时的曲速降速剩余 tick。

### 谁更准

对「当前在途的单艘星际物流运输船」而言，`CalcArrivalRemainingTime` 通常比本文受限输入方法更准。

原因不是它做了逐帧模拟，而是它拿到了更多必要状态：`stage`、`direction`、`t`、当前坐标、到出发星球中心的距离、实际目标接泊点、`warpState` 和船上空间翘曲器数量。题述输入只有速度、曲速阶段、是否有空间翘曲器、到起始塔和目标塔距离、部分科技数据，无法唯一恢复这些状态。

对「尚未出发的一整段路线预估」而言，`CalcRemoteSingleTripTime` 通常也比本文快速方法更准。它使用真实塔位、真实星球坐标和方向相关的近起点加速模型，并包含起飞、入坞、停站动画时间。

本文方法的优势是输入少、计算式清晰、适合批量外部估算。它的曲速积分部分如果使用 `S_up` 和 `S_down` 的指数闭式公式，在速度包络上比游戏函数的 `80 ticks + 0.18775714286 * W` 经验近似更贴近 `InternalTickRemote` 的 `warpAdd(w)` 定义；但整体 ETA 仍会因为缺少姿态、阶段和天体状态而不如 `CalcArrivalRemainingTime` 稳定。

最终建议如下：

- 已在游戏内拿到 `StationComponent` 和 `ShipData` 时，优先复刻 `CalcArrivalRemainingTime`。
- 只做路线级预估时，优先复刻 `CalcRemoteSingleTripTime`。
- 只能取得题述受限输入时，使用本文的两候选闭式方法，并在结果中标注方向和阶段未知带来的误差。
- 若外部工具追求更好的曲速段精度，可以在 `CalcArrivalRemainingTime` 框架上替换曲速经验式为 `warpAdd(w)` 的指数积分，但仍应保留它对 `stage`、`direction`、`t` 和实际坐标的处理。
