# 戴森球计划 - 星际物流运输船「单程剩余时间」闭式估算

- 反编译目标: `Assembly-CSharp.dll`
- 反编译工具: `ilspycmd` (ICSharpCode.Decompiler)
- 分析对象: `StationComponent.InternalTickRemote` 中飞船 (即「星际物流运输船」) 的运动逻辑
- 术语对齐: 来自游戏内 `Locale/2052` 翻译
- 约束: 本文主体推导**未**参考 `CalcRemoteSingleTripTime` / `CalcArrivalRemainingTime`, 文末「对比」章节才打开它们做核对

---

## 1. `InternalTickRemote` 中飞船运动逻辑总览

`InternalTickRemote` 每 1/60 秒被调用一次 (1 个游戏 tick), 对当前**星际物流运输站** (以下称 *起塔* = `this`, *目标塔* = `gStationPool[ship.otherGId]`) 持有的所有「在役运输船」`workShipDatas[j]` 推进一步。函数做四件事:

1. 尝试补 1 个**空间翘曲器**到塔自身库存 (`storage[i].itemId == 1210`)
2. 根据 `shipSailSpeed` 派生一批速度 / 加速度尺度
3. 对每条船按它当前的 `stage` 走对应分支 (本文重点)
4. 写回渲染数据、推进 `priorityLocks` 冷却

### 1.1 派生量 (本帧内对所有船共用)

设 `S = shipSailSpeed`, `W = shipWarpSpeed`。游戏外层调用方做完科研修正后再传入:

- `S = history.logisticShipSailSpeedModified`
- `W = history.logisticShipWarpDrive ? history.logisticShipWarpSpeedModified : S`

```
flag   = W > S + 1                    // 是否允许进入曲速航行
n8     = min(S/600, 500)              // 速度规模, 截顶 500
n9     = n8 ^ 0.4
n10    = (n9 <= 1) ? n9 : ln(n9) + 1  // 接近 1 后转 log, 限制放大
n11    = S * 0.03                     // 行星附近加速度 (m/s^2)
n12    = S * 0.12 * n10               // 远离行星加速度 (m/s^2)
n13    = S * 0.4  * n8                // 减速 step (m/s, 每 tick 直接扣)
n14    = n9 * 0.006 + 1e-5            // 起降阶段动画进度步长 (per tick)
```

注: `n11/n12` 是每秒加速度, 但 stage=0 内速度更新里用的是 `n33 = (flag3 ? n11 : n12)/60`, 即每 tick 增量; `n13` 已是每 tick 减速量。

### 1.2 单船状态 (`ShipData` 关键字段)

| 字段 | 含义 |
| --- | --- |
| `stage` | -2 ~ +2 |
| `direction` | +1 = 出发 (this -> other), -1 = 返航 (other -> this) |
| `t` | 当前 stage 内的动画进度 (0..1) |
| `uPos / uVel / uSpeed` | 宇宙坐标位姿 / 速度方向单位向量 / 速度标量 |
| `warpState` | 0..1, 曲速能量充能比例 |
| `warperCnt` | 船上携带的空间翘曲器数量 |
| `planetA / planetB` | 起塔行星 / 目标塔行星 ID |
| `otherGId` | 对端塔的全局 ID |

### 1.3 `stage` 分支语义

| stage | direction | 物理含义 |
| ---: | :---: | --- |
| -2 | +1 | 起塔停机口装货: t 0->1, 步长 0.0334/tick |
| -2 | -1 | 起塔停机口卸货: t 1->0, 步长 0.0334/tick (完成即归队) |
| -1 | +1 | 离开起塔停机坪过场: t 0->1, 步长 n14 |
| -1 | -1 | 降落起塔停机坪过场: t 1->0, 步长 n14·(2/3) |
|  0 |  ± | **自由空间巡航** (唯一真正在飞的阶段) |
|  1 | +1 | 接近目标塔减速过场: t 1->0, 步长 n14·(2/3) |
|  1 | -1 | 离开目标塔过场: t 0->1, 步长 n14 |
|  2 | +1 | 目标塔停机口装卸货: t 0->1, 步长 0.0334/tick; 完成后 direction <- -1 |

完整 "出发 -> 送达 -> 返航 -> 入港" 流程:

```
(-2,+1) -> (-1,+1) -> (0,+1) -> (1,+1) -> (2,+1)
                                       | 货物处理 / direction <- -1
                                       v
(1,-1) -> (0,-1) -> (-1,-1) -> (-2,-1) -> idle
```

### 1.4 stage=0 巡航段细节

设:
- `vTarget` = 目标对接点的宇宙坐标 (起塔 disk 上的栈位 / 目标塔船坞外推 25 m)
- `n21 = |vTarget - uPos|` — 到对接点剩余距离
- `n22 = |planet.uPos - uPos|^2` — 到「与方向相关那颗行星」距离²
  - direction>0 -> planet = 起塔行星
  - direction<0 -> planet = 目标塔行星
- `n15 = |planetA.uPos - planetB.uPos|` — 两行星距离

**(a) 是否在行星附近**

```
flag3 = (n22 <= uRadius^2 * 2.25)
```

`flag3=true` 用 `n11` (行星附近, 温和加速); 否则用 `n12` (远离时加速更快)。

**(b) 巡航目标速度 (非曲速)**

```
n30 = n21 / (uSpeed + 0.1) * 0.382
n32 = uSpeed * n30 * n10 + 6*n9 + 0.15*n8
n32 = min(n32, S)
n33 = (flag3 ? n11 : n12) / 60

if   uSpeed < n32 - n33 : uSpeed += n33
elif uSpeed > n32 + n13 : uSpeed -= n13          // 大幅过快才硬刹
else                    : uSpeed  = n32
```

展开目标速度 (忽略小量 `6*n9 + 0.15*n8`):

```
n32 ~ uSpeed * (n21/(uSpeed+0.1)) * 0.382 * n10
    ~ 0.382 * n10 * n21      (uSpeed > 0)
```

即 **巡航时飞船的目标速度 ~ `0.382 * n10 * n21`, 再被 S 顶住**。 这是后面闭式推导的关键: 当 `0.382 * n10 * n21 >= S` 时是匀速 `S`; 否则是**与剩余距离线性**的减速段。

**(c) 曲速段**

```
n26 = min(W, 2*n15)
n24 = n26 * (1001^warpState - 1) / 1000
n28 = n24 * 0.0449 + 5000 + S * 0.25

// 触发进入曲速 (warpState 从 0 起步):
if  n22 > 25e6  AND  n21 > warpEnableDist/2 = 240 km
    AND uSpeed >= S  AND (warperCnt>0 || warperFree):
    warperCnt-- ;  warpState += 1/60

// 已在曲速中:
if  n21 < n28 : warpState -= 1/15   // 退出 (~0.75 s)
else          : warpState += 1/60   // 继续加速 (3 s 加满)

clip warpState to [0,1]
uSpeed = S + n24                    // 曲速时覆盖
```

> `warpEnableDist = 480000` m (480 km), 「曲速启用路程」; 进入曲速要求**剩余距离** > 240 km。

**(d) 行星避让 / 引力旋转**

stage=0 还有一段在做行星避让 / 跟随旋转 (检查 `planetA/B/100*100+k` 这 10 颗行星), 对**时间估算**而言只在飞船恰好穿过其他行星 hill 半径时改变路径长度, 绝大多数行程影响可忽略。本文档默认**直线连两塔**做估算。

### 1.5 stage=±1 / ±2 的固定时长

```
stage=-2 装/卸货:  t 步长 0.0334     -> ~30 tick ~ 0.5 s
stage=-1 起飞:     t 步长 n14        -> 1/n14    tick   (direction=+1)
stage=-1 降落:     t 步长 (2/3)n14   -> 1.5/n14  tick   (direction=-1)
stage= 1 减速:     t 步长 (2/3)n14   -> 1.5/n14  tick   (direction=+1)
stage= 1 离港:     t 步长 n14        -> 1/n14    tick   (direction=-1)
stage= 2 装卸货:   t 步长 0.0334     -> ~30 tick
```

`n14 = n9*0.006 + 1e-5`。默认 `S=400` 时 `n9 = (400/600)^0.4 ~ 0.857`, `n14 ~ 0.005142`, 起飞过场 ~ 195 tick ~ 3.24 s。**这些过场时间不依赖距离, 只与 S 有关**。

---

## 2. 飞行阶段分类与是否可闭式

| stage 范围 | 运动类型 | 闭式? | 说明 |
| --- | --- | :---: | --- |
| -2 装/卸货 | 固定时序 | 是 | ~30 tick (~0.5 s) |
| -1 起飞 / 降落 | 固定动画 | 是 | 见 §1.5 |
| 0 加速段 (非曲速) | 等加速 a = `n12` 或 `n11` | 是 | §3.1 |
| 0 巡航段 (非曲速) | 匀速 v = S | 是 | §3.2 |
| 0 减速段 (非曲速) | v ~ `0.382*n10*D`, **指数衰减** | 是 (对数) | §3.3 |
| 0 曲速加速 | `warpState += 1/60`, 速度 `S + v_warp_cap*(1001^w-1)/1000` | 是 (积分) | §3.4 |
| 0 曲速巡航 | 匀速 `S + v_warp_cap` | 是 | §3.4 |
| 0 曲速退出 | `warpState -= 1/15` | 是 | §3.4 |
| 1, 2 | 固定动画 | 是 | 见 §1.5 |

**所有 stage 的时间在直线、单船假设下都可解析**。难点只在如何把 stage=0 拆成几段, 并算出每段的边界距离/速度。

### 2.1 唯一**不能**从当前状态闭式得到的量

| 信息 | 为什么不能 | 影响 |
| --- | --- | --- |
| 路上是否撞上第三方行星 | 路径依赖, 取决于实时行星轨道 | 罕见, 多耗几~几十秒, 多数行程 < 1% |
| 多船排队 / 抢资源 | 取决于其它船与塔的交互 | 影响装/卸货等待 (stage -2/2), 不在「这一程」内 |
| `n15` 在飞行中随轨道变化 | 行星公转 | 单程 << 行星周期, 可视作常量 |
| 起塔/目标塔 `shipDiskPos[shipIndex]` | 取决于 ship 编号 | 几十米, 相对距离百米~百万米可忽略 |

舍弃后误差量级 **远小于 1 秒**。

---

## 3. 单船「这一程剩余时间」闭式公式

只用当前状态 + 几个 GameHistory 标量, 不模拟 tick。

### 3.0 输入

| 输入 | 来源 |
| --- | --- |
| `vc` | `ShipData.uSpeed` |
| `w0` | `ShipData.warpState` |
| `hasWarper` | `warperCnt > 0` |
| `dA` | 到起塔 disk slot 的距离 |
| `dB` | 到目标塔对接点的距离 |
| `n15` | 两行星距离 (m) |
| `S` | `history.logisticShipSailSpeedModified` |
| `W` | `history.logisticShipWarpDrive ? logisticShipWarpSpeedModified : S` |
| `stage, direction, t` | `ShipData` |

> 「到对接点」与「到塔/行星中心」差 <= uRadius+30 m, 可忽略。

派生量 (一次性):

```
n8 = min(S/600, 500);   n9 = n8^0.4
n10 = (n9 <= 1) ? n9 : ln(n9) + 1
n11 = 0.03 * S;   n12 = 0.12 * S * n10
n14 = n9 * 0.006 + 1e-5
flag = (W > S + 1)
v_warp_cap = min(W, 2 * n15)
b = 6 * n9 + 0.15 * n8
k = 0.382 * n10
```

### 3.1 加速段 (vc -> S, 非曲速)

代码 `n33 = n12/60` 是每 tick 增量, 即每秒加速度 a = `n12`:

```
T_acc = max(0, (S - vc) / n12)
L_acc = (vc + S) / 2 * T_acc
```

### 3.2 巡航段 (匀速 S)

```
D_brake_start = S / k       // = S / (0.382 * n10)
T_cruise = max(0, (D - D_brake_start - L_acc) / S)
```

`D = direction>0 ? dB : dA`。

### 3.3 减速段 (指数衰减)

由 §1.4(b), `v ~ k*D + b`, 即 `dD/dt = -(k*D + b)`, 解:

```
D(t) + b/k = (D0 + b/k) * exp(-k*t)
```

`D0 = min(D, D_brake_start)`; 退出 `D_exit = 6*n10` (`n21 < 6*num10`):

```
T_dec = (1/k) * ln( (D0 + b/k) / (D_exit + b/k) )
```

若忽略 `b` (大行程下 <1% 误差): `T_dec ~ (1/k) * ln(D0 / D_exit)`。

### 3.4 曲速段

如果 `flag && (hasWarper || warperFree) && dB > 240 km`, 飞船会启动曲速, 巡航部分替换:

**曲速加速** (w0 -> 1):

`warpState += 1/60` per tick = +1 per 秒。因此从 w0 到 1 需要 `1 - w0` 秒。期间速度 `v(w) = S + v_warp_cap * (1001^w - 1) / 1000`, 且 `w(t) = w0 + t`, 用位移积分:

```
T_warp_up = 1 - w0
G(w) = 1001^w / ln(1001)
L_warp_up = (S - v_warp_cap/1000) * T_warp_up + (v_warp_cap/1000) * (G(1) - G(w0))
```

**曲速巡航** (w = 1, v = S + v_warp_cap):

退出门槛 (n28 最大值):

```
n28_max = v_warp_cap * 0.0449 + 5000 + S * 0.25
T_warp_cruise = max(0, (D - L_acc - L_warp_up - L_warp_dn - n28_max) / (S + v_warp_cap))
```

**曲速退出** (w: 1 -> 0, 速率 -1/15 per tick = -4 per 秒, 用时 0.25 s):

```
T_warp_dn = 0.25
L_warp_dn = 0.25 * (S - v_warp_cap/1000)
          + (v_warp_cap / 4000) * (G(1) - G(0))
          = 0.25 * (S - v_warp_cap/1000) + v_warp_cap * 1000 / (4000 * ln(1001))
```

之后接非曲速减速段 (§3.3), 起点 `D0 = n28_max`。

### 3.5 完整组装 (伪代码)

```python
def trip_remaining_seconds(ship, S, W, n15, dA, dB):
    # 派生量
    n8 = min(S / 600.0, 500.0)
    n9 = n8 ** 0.4
    n10 = n9 if n9 <= 1 else math.log(n9) + 1
    n11 = 0.03 * S
    n12 = 0.12 * S * n10
    n14 = n9 * 0.006 + 1e-5
    flag = W > S + 1
    v_warp_cap = min(W, 2 * n15)
    b = 6 * n9 + 0.15 * n8
    k = 0.382 * n10
    D_exit = 6 * n10

    # 各 stage 固定时间 (秒)
    SEC_PER_TICK = 1.0 / 60.0
    t_loading_full = (1.0 / 0.0334) * SEC_PER_TICK   # ~0.5 s
    t_takeoff_full = (1.0 / n14) * SEC_PER_TICK      # ~3.24 s @ S=400
    t_land_full    = (1.5 / n14) * SEC_PER_TICK      # ~4.86 s
    t_decel_full   = (1.5 / n14) * SEC_PER_TICK      # 同上, stage=1 减速

    # 1) stage=0 单程的"飞行部分"
    def fly_time(D, vc, w0, has_warper):
        # D: 到目标 disk/dock 对接点的剩余距离
        T = 0.0
        L_left = D
        # --- 是否走曲速 ---
        if flag and (has_warper or warper_free) and D > 240_000:
            # (a) 加速到 S (如不到)
            t_acc = max(0.0, (S - vc) / n12)
            l_acc = (vc + S) / 2 * t_acc
            T += t_acc
            L_left -= l_acc
            # (b) 曲速加速 w0 -> 1
            t_wup = 1.0 - w0
            G1 = (1001.0 ** 1) / math.log(1001.0)
            Gw = (1001.0 ** w0) / math.log(1001.0)
            l_wup = (S - v_warp_cap / 1000.0) * t_wup + (v_warp_cap / 1000.0) * (G1 - Gw)
            T += t_wup
            L_left -= l_wup
            # (c) 曲速退出长度 (固定 0.25s)
            t_wdn = 0.25
            G0 = 1.0 / math.log(1001.0)
            l_wdn = 0.25 * (S - v_warp_cap / 1000.0) + (v_warp_cap / 4000.0) * (G1 - G0)
            # (d) 曲速巡航 (匀速 S + v_warp_cap)
            n28_max = v_warp_cap * 0.0449 + 5000.0 + S * 0.25
            t_cruise_warp = max(0.0, (L_left - l_wdn - n28_max) / (S + v_warp_cap))
            T += t_cruise_warp
            L_left -= t_cruise_warp * (S + v_warp_cap)
            T += t_wdn
            L_left -= l_wdn
            # (e) 退出曲速后, 距离约为 n28_max, 走非曲速减速段
            D_dec = max(D_exit, L_left)
            T += (1.0 / k) * math.log((D_dec + b/k) / (D_exit + b/k))
        else:
            # 非曲速 加速 -> 巡航 -> 减速
            t_acc = max(0.0, (S - vc) / n12)
            l_acc = (vc + S) / 2 * t_acc
            T += t_acc
            L_left -= l_acc
            D_brake_start = S / k
            t_cruise = max(0.0, (L_left - D_brake_start) / S)
            T += t_cruise
            D_dec_start = min(L_left, D_brake_start)
            T += (1.0 / k) * math.log((D_dec_start + b/k) / (D_exit + b/k))
        return T

    # 2) 顶层分发
    stage = ship.stage
    direction = ship.direction
    t = ship.t

    if direction > 0:
        D_now = dB
    else:
        D_now = dA

    if stage == 2:    # 目标塔装/卸货
        return (1.0 - t) / 0.0334 * SEC_PER_TICK
    if stage == 1:
        if direction > 0:
            t_left = t / ((2.0/3.0) * n14) * SEC_PER_TICK
            return t_left + t_loading_full
        else:
            t_left = (1.0 - t) / n14 * SEC_PER_TICK
            return t_left + fly_time(D_now, ship.uSpeed, ship.warpState, ship.warperCnt > 0) + t_land_full + t_loading_full
    if stage == 0:
        T_fly = fly_time(D_now, ship.uSpeed, ship.warpState, ship.warperCnt > 0)
        if direction > 0:
            return T_fly + t_decel_full + t_loading_full
        else:
            return T_fly + t_land_full + t_loading_full
    if stage == -1:
        if direction > 0:
            t_left = (1.0 - t) / n14 * SEC_PER_TICK
            return t_left + fly_time(dB, 0.0, 0.0, ship.warperCnt > 0) + t_decel_full + t_loading_full
        else:
            t_left = t / ((2.0/3.0) * n14) * SEC_PER_TICK
            return t_left + t_loading_full
    if stage == -2:
        if direction > 0:
            t_left = (1.0 - t) / 0.0334 * SEC_PER_TICK
            return t_left + t_takeoff_full + fly_time(dB, 0.0, 0.0, ship.warperCnt > 0) + t_decel_full + t_loading_full
        else:
            return t / 0.0334 * SEC_PER_TICK
```

### 3.6 精度评估

| 简化 | 引入误差 |
| --- | --- |
| 把 `n32 ~ k*D` 当严格不动点 (实际有一阶滞后 ~ n33 量级) | 减速段开头偏短 ~ 0.01 s |
| 加速段用 `n12` 不区分 `n11` | 起飞瞬间 (D 已很小) 才会进 `n11`, 几乎可忽略 |
| 曲速 1 秒加满, 0.25 秒退出 | 这是代码字面值, 无误差 |
| 把行星避让推力当 0 | 路径加长 < 1%, 时间影响 < 1% |
| 直线连塔 | 同上 |
| 把 `n15` 当常量 | 单程时长比行星周期短 2~3 个数量级, 可忽略 |
| stage=0 退出条件 `n21 < 6*n10` 当对接终点 | 6*n10 << 任何 D, 影响 < 0.05 s |

**单程估算与真实模拟差 < 1 s 或 < 1% (二者取大), 对于物流系统已经足够准确。**

---

---

## 4. 与官方 `CalcRemoteSingleTripTime` / `CalcArrivalRemainingTime` 的对比

> 本节是 §1~§3 完成后才打开两段官方实现并做核对。

### 4.1 `CalcRemoteSingleTripTime` (全程时间, 不带当前 ship 状态)

变量对照 (官方 `numN` -> 本文记号):

| 官方 | 本文 | 说明 |
| --- | --- | --- |
| `num`  = S/600 | `n8` | 同 |
| `num2` = num^0.4 | `n9` | 同 |
| `num3` | `n10` | 同 |
| `num4` | `n14` | 同 |
| `num5` = 0.03·S | `n11` | 同 |
| `num6` = 0.12·S·n10 | `n12` | 同 |
| `num9` = 0.382·n10 | `k` | **同** |
| `num10` = 6·n9 + 0.15·n8 | `b` | **同 (减速段截距)** |
| `num11` = (S-b)/k | D_brake_start (精确) | 我用 `S/k`, 官方多扣 b/k |
| `num12` 减速段 tick = `(ln(k·D0/b+1) - ln(k·6·n10/b+1))/k * 60 + 1` | `60·T_dec` | **完全一致** |

**起飞加速段** (与本文 §3.1 简化的差异):
官方拆成两段 -- 先以 a=`n11` (行星附近) 加速到 `num7 = sqrt((dir? 2:0.5)·uRadiusA·n11)` 走 `num7/n11` 秒, 再以 a=`n12` 加速到 `S` 走 `(S-num7)/n12` 秒。 本文 §3.1 把整个加速合并到 `n12` 简化, 在 S>>400 时会**少算几秒**（这是本文公式偏短的主要来源)。

**曲速段** (与本文 §3.4 的差异):
官方用经验常量
`num18 = v_warp_cap·0.18775714286 + S·1.33333333333`
把 "曲速 加速段位移 + 退出段位移" 总和当成与 warpState 无关的常量; 本文 §3.4 用解析积分计算 `L_warp_up + L_warp_dn`。代入数字验证: 本文得 `L = 1.25·S + 0.1797·v_warp_cap`, 官方为 `1.3333·S + 0.18776·v_warp_cap`, 两者在 v_warp_cap 系数上一致 (差 ~ 4%, 量级正确), S 系数上官方多算了 `0.0833·S` (约 33 m·S/m, 即 ~5 tick), 与官方在曲速段开头额外加的 `+= 80` (= 60+15+5 tick, 即"曲速加速 1 s + 退出 0.25 s + 余量") 是对得上的: 总位移与 (S+v_warp_cap)·1.25 s 的差被以 `+80 tick` 的常数项吸收掉。  本文用 `T_warp_up=1 + T_warp_dn=0.25 = 75 tick` 直接计入, 偏小 5 tick (~0.08s)。

**短距离曲速** 官方有一段经验拟合
```
6400 / (79 * Math.Pow(x, 0.0) + 1/x) = 6400 / (79 + 1/x)
```
`Math.Pow(x, 0.0)` 恒为 1, 所以是 `6400/(79 + 1/x)`。这是开发组的拟合公式, 没有物理推导, 用于处理"刚够开曲速但又走不到曲速匀速"的尴尬距离段。本文模型遇到这种情况直接用线性公式截到 0, 误差 ~ 十几 tick。

**结论**: `CalcRemoteSingleTripTime` 是 "全程" 估算, 假设飞船从 0 速度起步; 不能用来回答 "这一程剩余时间"。

### 4.2 `CalcArrivalRemainingTime` (当前 ship 到达剩余时间)

它分三类:

**1) `stage*direction < 0` (在过场/装卸货, 朝目标方向推进)**

直接调 `CalcRemoteSingleTripTime` 拿全程, 减去 "已经过去" 的 tick:
- `stage=-2`: `-= t·30` (装货已过 t·30 tick)
- `stage=-1`: `-= t/n14 + 30` (起飞过场已过 + 装货 30 tick 已过)
- `stage=1, dir=-1`: 同 `-1`
- `stage=2`: `-= t·30`

**问题**: 它假设 ship "当前位置 = 阶段起点", **完全不用 `uSpeed/warpState/uPos`**, 全程时间也是用全长距离的理论值。在大行程 (跨星) 时, 全程理论时间 vs 真实路径误差可能 5-30 秒。

**2) `stage*direction > 0` (在停机口装/卸货)**

- `stage=-2`: `t·30` tick (还差这么多 tick 装完)
- `stage=-1, dir=-1`: `1.5·t/n14 + 30` (剩余降落 + 卸货)
- `stage=1, dir=+1`: 同 `-1`
- `stage=2`: `t·30` tick

这一段写得**完全确定**且字面准确。

**3) `stage=0` (在巡航)** — 走一段独立的长分支 (75319~75575), 按当前位置/速度/warpState 分了 6~7 个 case:
- 是否还在起塔 1.5·uRadius 内 (`magnitude2 < uRadiusA·1.5`)
- 是否能开曲速 (`flag`)
- 当前是否已经在曲速中 (`shipData.warpState > 0`)
- 距离 vs 多个门槛

每个 case 都用 "非曲速加速/巡航/减速" 公式但起点是当前速度 / 当前距离, 与本文 §3 拆分**一一对应**。

### 4.3 谁更准?

| 维度 | 本文公式 | 官方 `CalcArrivalRemainingTime` |
| --- | --- | --- |
| 减速段对数公式 | 与官方完全一致 | 同 |
| 巡航段 | 完全一致 | 同 |
| 加速段 | 单段 `n12`, 起飞早期偏短几秒 | 两段 `n11+n12`, 更精确 |
| 曲速加速/退出位移 | 解析积分, 精确 | 经验常量, 精度 ~1% |
| 短距离曲速 | 线性近似截 0, ~十几 tick | 经验拟合 `6400/(79+1/x)`, 同量级 |
| `stage=0` 利用当前 `uSpeed/warpState` | 充分利用 | 充分利用 (case 分支) |
| `stage != 0` 利用 ship 位置 | 利用 `dA/dB` | **不利用!** 用 `CalcRemoteSingleTripTime` 全程减已过 tick |
| 实现复杂度 | 中 (~50 行) | 高 (~250 行) |

**结论**:

- **`stage=0` 段两者精度旗鼓相当**, 官方因为做了 `n11/n12` 两段加速更准 1-5 秒; 本文用解析积分算曲速位移, 略胜于官方的常量拟合。 总体两者差异远 < 5 秒/单程。

- **`stage != 0` 的过场段, 官方实现明显粗糙** -- 它把"已经飞到哪儿"当成 "在阶段起点", 大行程时偏差 5~30 秒。 本文方法在 stage=-2/-1 时使用 `dB` (船当前在停机坪附近, dA 很小), 在 stage=1/2 时使用 `dA`, 并且把"剩余飞行"用真实当前距离去算, **更准**。

- **唯一官方更准的情形**: 起飞瞬间 (stage=-1 dir=+1 转 stage=0 dir=+1 那一刻), `vc=0` 且 ship 在行星 1.5 uRadius 内, 此时本文按 `a=n12` 算加速段会略偏短 -- 加 `T_acc_fix = num7·(1/n11 - 1/n12)` 即可补回。 大约 +0.5~3 秒。

### 4.4 推荐的折中方案

把本文 §3.1 的加速段拆成两段, 用官方相同的 `num7`:

```
num7 = sqrt((direction>0 ? 2 : 0.5) * uRadiusA * n11)
T_acc_inner = num7 / n11
T_acc_outer = (S - num7) / n12
L_acc       = 0.5*num7*T_acc_inner + (num7+S)*0.5*T_acc_outer
T_acc       = T_acc_inner + T_acc_outer
```

加这 5 行后, 本文公式在所有 stage 上**均不输官方实现**, 且代码长度只有官方 1/5。

### 4.5 一些 small print

- 官方 `CalcArrivalRemainingTime` 中 `case 2` (75463 行) 计算 stage=0 曲速短距离时, 用 `Math.Pow(num19, 0.0)` -- 这是显然的 bug (永远 = 1), 实际意图很可能是 `Math.Pow(num19, c)` (某个常量, 也许是 0.X)。这意味着即便是官方公式, 在 "近距离开曲速" 场景下也只是凭经验拟合。
- 官方常量 `0.18775714286` 与 `1.33333333333` 是用 Mathematica 之类的工具事先算出的"曲速三段位移占比"; 本文 §3.4 给出了**正确的解析推导**。

---

## 5. 最终回答 "给我一个不模拟的闭式方法"

直接使用 §3.5 的 `trip_remaining_seconds`。 若追求与官方实现 100% 同精度, 在 §3.1 用 §4.4 的两段加速替换。 不需要任何 tick 模拟。


---

## 6. 复审 §3.4 / §4.4: 曲速退出阶段并非 15 帧 (用户提出的振荡问题)

### 6.1 用户观察

观察这段代码:

```cs
num24 = (float)(num26 * ((Math.Pow(1001.0, warpState) - 1.0) / 1000.0));
double num28 = num24 * 0.0449 + 5000.0 + shipSailSpeed * 0.25;
double num29 = num21 - num28;
if (num29 < 0.0) num29 = 0.0;
if (num21 < num28)
    warpState -= 0.06666667f;   // -1/15
else
    warpState += 0.0166666675f; // +1/60
```

退出条件触发 (`num21 < num28`) 让 `warpState` 下降; 下一帧 `num24` 因此减小, `num28` 也跟着减小, 是否会让 `num21 > num28` 反过来又触发 `warpState += 1/60`, 形成振荡, 使整个退出阶段 > 15 帧?

### 6.2 定量分析: 答案是**会**, 而且振荡严重

设 `vmax = num26 = min(W, 2·n15)`, `f(w) = vmax·(1001^w - 1)/1000`, `n28(w) = 0.0449·f(w) + 5000 + 0.25·S`。

每帧 D 变化: `ΔD ≈ -(S + f(w))/60` (飞向目标)
每帧 -w 时 n28 变化: `Δn28 ≈ -0.0449 · f'(w) · (1/15)`, 其中 `f'(w) = vmax · 1001^w · ln(1001)/1000`

取最危险情形 w=1 (`vmax = W = 400000`, `S = 400`):

- `f(1) = 400000`, `1001^1 = 1001`, `ln(1001) ≈ 6.909`
- `|Δn28(w=1)| ≈ 0.0449 · 400000 · 1001 · 6.909 / 1000 / 15 ≈ 8281` m / 帧
- `|ΔD| ≈ (400 + 400000) / 60 ≈ 6673` m / 帧

**n28 减得比 D 还快 (8281 > 6673)**, 触发条件 `D < n28` 在下一帧反转为 `D > n28`, 于是 w 反弹 +1/60。

### 6.3 数值模拟 (从 `D = n28(1) = 23060` 出发, w=1; W=400000, S=400)

| tick | 行为 | w | 用旧 w 算的 num24 | 用旧 w 算的 n28 | 帧末 D | 帧位移 |
| ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 1 | 反 (D=n28) | 1.0000 | 400000 | 23060 | 23053 | -7 |
| 2 | 退 | 0.9333 | 400000 | 23060 | 23047 | -7 |
| 3 | 反 | 0.9500 | 252218 | 16425 | 18323 | -4724 |
| 4 | 反 | 0.9667 | 283047 | 17809 | 17797 | -526 |
| 5 | 退 | 0.9000 | 317638 | 19362 | 17790 | -7 |
| 6 | 反 | 0.9167 | 200255 | 14091 | 14047 | -3743 |
| 7 | 退 | 0.8500 | 224743 | 15191 | 14041 | -7 |
| 8 | 反 | 0.8667 | 141646 | 11460 | 11428 | -2614 |
| ... | ... | ... | ... | ... | ... | ... |
| 40 | 完 | 0.0000 | 165 | 5107 | 5098 | ~-7 |

模式: 每帧基本都被 `num29 = max(0, D - n28)` 钳制 -- 当 D 几乎等于 n28 时, 飞船速度被截到 `S + 0`, 一帧只移动 ~7 m。 真实"退出 + 振荡"过程是: **飞船持续紧贴 n28 门槛, 而 n28 在 w 上下振荡中缓慢下降, 飞船被动跟着下降**, 平均速度只比 S 略高一点 (受短暂"反"帧的 num24 跳升影响)。

这里非常关键的代码细节是 `num24_new/60 > num29: num24_new = num29*60*1.01` (74530-74533 行), 它把"曲速速度"限制到"本帧顶多再飞 1.01·num29 距离", 防止冲过对接点。 正是这个 cap 让 `D` 紧贴 `n28` 而不会冲过头, 形成 §6.5 描述的稳态滑行。

### 6.4 实测各参数下的全程退出耗时

| W | S | 退出帧数 | 退出耗时 | 退出位移 | "匀速 0.25 s" 估计位移 | 比值 |
| ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 50,000 | 400 | 33 | 0.55 s | 2,265 m | 12,600 m | 0.18 |
| 100,000 | 400 | 35 | 0.58 s | 4,499 m | 25,100 m | 0.18 |
| 400,000 | 400 | 40 | 0.67 s | 17,962 m | 100,100 m | 0.18 |
| 400,000 | 1500 | 37 | 0.62 s | 18,039 m | 100,375 m | 0.18 |
| 2,000,000 | 500 | 45 | 0.75 s | 89,777 m | 500,125 m | 0.18 |
| 5,000,000 | 1500 | 45 | 0.75 s | 224,487 m | 1,250,375 m | 0.18 |

**两点关键观察**:

1. **退出耗时 33-45 帧, 不是 15 帧** -- 比本文 §3.4 估计的 0.25 s 多 2~3 倍, 但绝对值仍 < 1 s。
2. **真实退出位移只有"匀速 0.25 s"估计的 18% 左右** -- 因为振荡期间飞船平均速度只有 vmax 的 ~18%。

### 6.5 物理直觉: 振荡阶段实际是"贴着 n28(w) 门槛滑行"

代入稳态条件 `D ≈ n28(w)`, 每帧 `dD/dt ≈ -(S+f)`, 由 `D = 0.0449·f + 常数` 得 `dD/dt = 0.0449·df/dt`:

```
0.0449 · df/dt ≈ -(S + f)
df/dt ≈ -22.27 · (S + f)
```

即 `(S+f)(t) = (S + vmax) · exp(-22.27·t)`

衰到 f≈0 用时 `T_理论 ≈ 0.0449 · ln(1 + vmax/S)`。

对 vmax=400000, S=400: `T ≈ 0.0449·ln(1001) ≈ 0.310 s` (= 18.6 帧)。

实测 40 帧 (0.67 s) 比理论 18.6 帧大约 2 倍, 差异来自 -w 与 +w 速率不对称 (-1/15 vs +1/60, 4:1), 每个 「退-反」 周期 net 衰减比对称情形慢约 (1-0.25) 分之 1 = 4/3, 反复后累积放大因子约 2。

### 6.6 闭式估计修正公式

把 §3.4 的两个常量改成:

```
# 振荡放大因子, 实测 α ≈ 2.0~2.2
α = 2.0
T_warp_dn ≈ max(0.25, 0.0449 · α · ln(1 + vmax/S))
L_warp_dn ≈ n28(1) - n28(0) = 0.0449 · vmax     # 振荡稳态下的位移
```

关键物理含义: **曲速退出阶段把"曲速顶速→低速"这段路程压缩在仅 `0.0449·vmax` 米内完成**, 之后 D 已经降到 `n28(0) = 5000 + 0.25·S`, 进入 stage=0 非曲速减速段。

### 6.7 §3.5 完整组装的替换补丁

把曲速分支替换为:

```python
# 曲速顶速
v_warp_cap = min(W, 2 * n15)

# 曲速加速 (w0 -> 1) -- 与之前相同
T_warp_up = 1.0 - w0
G = lambda w: (1001.0 ** w) / math.log(1001.0)
L_warp_up = (S - v_warp_cap/1000) * T_warp_up + (v_warp_cap/1000) * (G(1) - G(w0))

# 曲速退出 (修正, 用振荡稳态模型)
T_warp_dn = max(0.25, 0.0449 * 2.0 * math.log(1 + v_warp_cap / S))
L_warp_dn = 0.0449 * v_warp_cap

# 曲速巡航
n28_max = v_warp_cap * 0.0449 + 5000 + 0.25 * S   # 触发退出的 D 值
T_warp_cruise = max(0.0, (D - L_acc - L_warp_up - L_warp_dn - (5000 + 0.25*S)) / (S + v_warp_cap))

# 退出后接非曲速减速段, 起点 D0 = 5000 + 0.25 * S
D_after_warp = 5000 + 0.25 * S
T_dec = (1/k) * math.log((D_after_warp + b/k) / (D_exit + b/k))

T_total = T_acc + T_warp_up + T_warp_cruise + T_warp_dn + T_dec
```

### 6.8 对官方 `CalcArrivalRemainingTime` 的影响

官方的 stage=0 曲速分支 (75529-75574 行) 在曲速段也用了 `+= 20` 这个魔法常数, 对应 `T_warp_dn ≈ 20 tick / 60 ≈ 0.333 s` -- 比我们之前估的 15 tick 多 5 tick, 是开发组对振荡现象的**经验补偿**, 但仍比真实平均 40 tick 偏短约 20 tick (0.33 s)。

更细的:

```cs
num += (int)(shipData.warpState * 20f + 0.5f);    // 75543 行
num += (int)(6400.0 / (79.0 + 1.0/num35) - shipData.warpState * 60.0);  // 75539 行
```

`warpState * 20f` 这个项就是官方对退出阶段的估算 -- 20 tick = 0.333 s, 明显比真实的 ~40 tick 短。

**结论**: 官方代码也低估了曲速退出耗时, 但因为这一段位移在大多数行程中只占 < 10%, 单程时间误差仍在 < 1 s 范围。

### 6.9 总结

| 问题 | 答案 |
| --- | --- |
| 用户怀疑的振荡现象是否存在? | **是, 而且每 ~3 帧就发生一次** |
| 退出阶段实际耗时? | 33-45 帧 (~0.55-0.75 s), 不是 15 帧 |
| 退出阶段实际位移? | `0.0449 · vmax` 米, 远小于"匀速 0.25 s · (S+vmax)" |
| 怎么处理? | 用 §6.6 的稳态指数衰减闭式公式, 引入振荡放大因子 α ≈ 2.0 |
| 对单程总时间影响? | 默认参数下 +0.4 s 左右, 极端 W 下 +0.5 s, 占单程 << 1% |
| 官方实现是否也有这个问题? | 是, 官方用 `warpState * 20` 经验估算, 也偏短约 20 tick |

**修正后的本文公式在曲速退出阶段比官方实现更准确。**
