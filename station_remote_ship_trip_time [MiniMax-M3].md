# `StationComponent.InternalTickRemote` 中星际物流运输船运动与剩余时间估算

## 来源与约束

- 反编译工具：`ilspycmd 10.1.0.8386`。
- 目标模块：`/mnt/d/Steam/steamapps/common/Dyson Sphere Program/DSPGAME_Data/Managed/Assembly-CSharp.dll`。
- 术语参考：`/mnt/d/Steam/steamapps/common/Dyson Sphere Program/Locale/2052`。
- 第一阶段只读取 `InternalTickRemote`，未读取 `CalcRemoteSingleTripTime` 和 `CalcArrivalRemainingTime`。
- 第二阶段读取上述两个方法，并在文末对比。

文中「星际物流运输船」对应代码中的 `ShipData`，「空间翘曲器」对应物品 `1210`。

## 关键状态

| 字段 | 含义 |
| --- | --- |
| `stage` | 停靠、离港、航行、进港阶段。 |
| `direction` | `1` 为去目标塔，`-1` 为返回本塔。 |
| `uPos` | 宇宙坐标位置。 |
| `uVel` | 单位方向向量。 |
| `uSpeed` | 当前标量速度。 |
| `warpState` | 曲速阶段，`0` 为普通航行，`0..1` 为启动或退出，`1` 为满曲速。 |
| `warperCnt` | 船上空间翘曲器数量。 |
| `t` | 离港、进港、停靠动画进度。 |

`GameHistory` 直接影响飞行时间的字段为：

```text
shipSailSpeed = logisticShipSailSpeed * logisticShipSpeedScale
shipWarpSpeed = logisticShipWarpDrive ? logisticShipWarpSpeed * logisticShipSpeedScale : shipSailSpeed
```

「运输船引擎」通过效果类型 `16` 增加 `logisticShipSpeedScale`，通过效果类型 `17` 解锁星际物流运输船曲速飞行。「运输机舱扩容」只影响运载量，不直接影响飞行时间。

## 派生量

```text
S = shipSailSpeed
W = shipWarpSpeed
r0 = S / 600
p = r0^0.4
k = p > 1 ? ln(p) + 1 : p
r = min(r0, 500)
aNear = 0.03S
aFar = 0.12Sk
brakeA = 0.4Sr
animRate = 0.006p + 0.00001
warpCap = min(W, 2D)
```

`D` 是起点天体与目标天体的当前距离。`warpEnableDist` 默认为 `480000`，可由物流塔设置修改。

## 飞行阶段

### `stage < -1`：本塔停靠或离泊前置

- `direction > 0`：`t += 0.03335`，超过 `1` 后进入 `stage = -1`。
- `direction < 0`：`t -= 0.03335`，小于 `0` 后交付并回到空闲船。
- 位置固定在本塔船坞圆盘。
- 闭式解：有，约 30 帧量级。

### `stage == -1`：本塔出入港插值

- 去程：`t += animRate`，超过 `1` 后进入 `stage = 0`。
- 回程：`t -= 2/3 * animRate`，小于 `0` 后进入 `stage = -2`。
- 位置使用 `smooth(t) = (3 - 2t)t^2` 插值。
- 标量速度清零。
- 闭式解：有，由 `t` 和 `animRate` 直接求剩余帧数。

### `stage == 0`：宇宙航行主阶段

目标点不是塔心，而是船坞法线外推 25 米后的点：

- 去程：目标塔 `shipDockPos + normalized(shipDockPos) * 25`。
- 回程：本塔 `shipDiskPos[shipIndex] + normalized(shipDiskPos[shipIndex]) * 25`。

当到目标点距离 `targetDist < 6k` 时，进入 `stage = direction`，并保存进港插值使用的相对位置和姿态。

#### 普通航行

每次方向更新时先算目标速度：

```text
tau = targetDist / (uSpeed + 0.1) * 0.382
vTarget = min(uSpeed * tau * k + 6p + 0.15r, S)
```

随后按区域加速或减速：

- 离起点天体较近：`uSpeed += aNear / 60`。
- 远离起点后：`uSpeed += aFar / 60`。
- 若速度明显高于目标速度：`uSpeed -= brakeA`。

这里 `vTarget` 可化为近似关系：

```text
vTarget ≈ 0.382k * targetDist + 6p + 0.15r
```

因此普通航行可在一维直线模型下写出分段闭式；严格闭式不存在，因为真实路径受朝向、角速度、避让、天体运动补偿影响。

#### 曲速航行

曲速启动条件：

```text
W > S + 1
warperCnt > 0 或演示模式免费
uSpeed >= S
到起点天体距离 > 5000
targetDist > warpEnableDist * 0.5
```

曲速状态更新：

```text
启动或巡航：warpState += 1/60
接近目标退出：warpState -= 1/15
warpState clamp 到 [0, 1]
```

曲速附加速度：

```text
warpExtra = warpCap * ((1001^warpState - 1) / 1000)
uSpeed = S + warpExtra
```

退出距离：

```text
exitDist = warpExtra * 0.0449 + 5000 + 0.25S
```

曲速状态函数本身有闭式积分：

```text
∫ warpCap * ((1001^x - 1) / 1000) dx
= warpCap / 1000 * (1001^x / ln(1001) - x) + C
```

但游戏按 60 帧离散更新，且每帧根据 `exitDist` 切换启动、巡航、退出，所以连续闭式只能近似。若需要精度，应对曲速状态做一维离散求和，不需要模拟三维轨迹。

#### 避让和天体运动补偿

主航行还会：

- 在起点星系和目标星系内搜索近距离天体。
- 按天体半径、相对距离、相对速度调整方向。
- 使用天体 `uPosNext` 做位置补偿。
- 曲速时强制朝向目标。

这些会改变路径长度，所以只给距离时无法严格复现真实到达时间。

### `stage == 1`：目标塔出入港插值

- 去程：`t -= 2/3 * animRate`，结束后进入 `stage = 2`。
- 回程：`t += animRate`，结束后回到 `stage = 0`。
- 位置是三次平滑插值。
- 闭式解：有。

### `stage > 1`：目标塔停靠、卸货、装货和返航准备

- 去程：`t -= 0.0334`。结束后卸货、尝试从目标塔补充空间翘曲器，并设置 `direction = -1`。
- 回程：`t += 0.0334`。结束后进入 `stage = 1`。
- 位置固定。
- 闭式解：有，约 30 帧量级。

## 不模拟三维轨迹的剩余时间计算方法

### 输入

```text
v              当前 uSpeed
warpState      当前 warpState
hasWarper      船上是否有空间翘曲器
fromStartDist  到起始天体或起始塔参考点距离
toTargetDist   到目标塔参考点距离
S0             GameHistory.logisticShipSailSpeed
W0             GameHistory.logisticShipWarpSpeed
speedScale     GameHistory.logisticShipSpeedScale
warpDrive      GameHistory.logisticShipWarpDrive
warpEnableDist 当前塔的 warpEnableDist，缺省 480000
startRadius    起始天体半径，缺省可近似取 200
tripDistance   本程起止天体距离；若没有，可近似取 fromStartDist + toTargetDist
```

### 普通航行闭式

设：

```text
lambda = 0.382k
vFloor = 6p + 0.15r
dEnd = 6k
brakeTriggerDist(v) = max(0, (v - vFloor) / lambda)
```

若已经进入减速段：

```text
T = ln((lambda*d + vFloor) / (lambda*dEnd + vFloor)) / lambda
```

若需要先加速到 `S`：

```text
a = fromStartDist < 1.5 * startRadius ? aNear : aFar
dAccel = max(0, (S^2 - v^2) / (2a))
dBrakeS = brakeTriggerDist(S)
```

若 `d > dAccel + dBrakeS`：

```text
T = (S - v)/a + (d - dAccel - dBrakeS)/S + ln(S / (lambda*dEnd + vFloor)) / lambda
```

否则求峰值速度 `vp`：

```text
d = (vp^2 - v^2)/(2a) + (vp - vFloor)/lambda
T = (vp - v)/a + ln(vp / (lambda*dEnd + vFloor)) / lambda
```

`vp` 是一元二次方程正根。以上 `T` 单位为秒，乘以 60 得到帧数。

### 曲速一维离散求和

```text
function remainingTime(d, v, warpState):
    S = S0 * speedScale
    W = warpDrive ? W0 * speedScale : S
    canWarp = warpDrive && hasWarper && W > S + 1
    warpCap = min(W, 2 * tripDistance)
    frames = 0

    while canWarp and (warpState > 0 or (v >= S and fromStartDist > 5000 and d > warpEnableDist * 0.5)):
        extra = warpCap * ((1001^warpState - 1) / 1000)
        exitDist = extra * 0.0449 + 5000 + 0.25S
        if warpState > 0 and d < exitDist:
            warpState = max(0, warpState - 1/15)
        else:
            warpState = min(1, max(warpState, 0) + 1/60)

        extra = warpCap * ((1001^warpState - 1) / 1000)
        exitDist = extra * 0.0449 + 5000 + 0.25S
        extra = min(extra, max(0, d - exitDist) * 60 * 1.01)
        v = S + extra
        d -= v / 60
        frames += 1

        if warpState <= 0:
            break

    return frames + sailTimeClosedForm(d, min(v, S)) * 60
```

这个算法不更新姿态、不做天体避让，只在剩余距离上推进曲速状态。计算量很小，通常几十到数百次循环。

### 纯闭式远距离近似

用于排序或估算，不建议用于精确显示：

```text
D_exit ≈ 5000 + 0.25S + 0.0449 * warpCap
D_accel ≈ S + warpCap / 1000 * ((1001 - 1) / ln(1001) - 1)
T_warp ≈ 1 + max(0, d - D_accel - D_exit) / (S + warpCap)
```

再加普通航行的离港和进港段。这个近似忽略曲速离散启动、退出、单帧防越界和天体运动。

## 取舍与误差

无法仅凭给定输入严格闭式得到的内容：

1. 天体当前坐标、下一帧坐标和半径。
2. 飞船朝向、速度方向和角速度。
3. 起点与目标星系内其他天体造成的避让。
4. 船坞位置、船坞法线和 `shipIndex`。
5. `warpEnableDist` 若未输入，只能用默认值。

影响评估：

- 远距离且使用曲速时，误差主要来自曲速退出距离和天体运动补偿，通常相对误差较小。
- 近距离不曲速时，普通航行的转向、避让、进出港动画占比更高，闭式近似误差更明显。
- 若当前已经在曲速中，使用一维离散求和通常比纯闭式近似可靠。

## 与官方计算方法对比

第二阶段读取 `CalcRemoteSingleTripTime` 和 `CalcArrivalRemainingTime` 后，结论如下。

### `CalcRemoteSingleTripTime`

官方单程估算会：

- 使用起点和目标船坞外推 25 米后的实际宇宙距离。
- 加入离港、进港和停靠常数项。
- 使用与主逻辑相同的 `p`、`k`、`animRate`、近起点加速度和远距离加速度。
- 将普通航行拆为加速、巡航、接近目标减速三段。
- 对曲速使用经验式距离：`warpCap * 0.18775714286 + S * 1.33333333333`，并加入约 80 帧启动段。

它不是逐帧模拟，但明显是为游戏逻辑调过的解析近似。

### `CalcArrivalRemainingTime`

官方剩余时间估算比单程估算更接近实时状态：

- 若不在主航行阶段，会根据 `stage`、`direction` 和 `t` 扣除或只计算剩余动画时间。
- 若在 `stage == 0`，会读取当前 `uSpeed`、`warpState`、当前位置到起点和目标的距离。
- 普通航行分支考虑当前是否还在起点半径 `1.5 * uRadius` 内。
- 曲速分支区分未进曲速、正在启动、满曲速、正在退出。
- 正在曲速时会用 `warpState` 修正剩余启动或退出时间。

### 谁更准

结论：官方 `CalcArrivalRemainingTime` 最准，其次是本文「一维曲速离散求和 + 普通航行闭式」，最后是纯闭式远距离近似。

原因：

1. `CalcArrivalRemainingTime` 可访问完整 `ShipData`、当前天体数据、船坞坐标和 `warpEnableDist`，输入信息比本文限定输入更多。
2. 它对 `stage`、`direction`、`t`、`uSpeed`、`warpState` 做了专门分支，能处理离港、返航、停靠等状态。
3. 它仍不是严格逐帧模拟，不包含完整姿态、避让和天体补偿积分，因此在复杂近距离路径上仍可能有误差。
4. 本文方法在限定输入下放弃三维朝向和避让，换取低计算量；远距离曲速场景误差较小，近距离普通航行误差较大。

若能直接调用游戏对象，优先使用 `CalcArrivalRemainingTime`。若只能使用题设输入，建议使用本文的一维曲速离散求和方案；若只做远距离排序，可用纯闭式近似。
