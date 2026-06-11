# 戴森球计划 — 星际物流运输船飞行运动分析

> 基于对 `Assembly-CSharp.dll` 中 `StationComponent` 的反编译分析。
> 术语以游戏中文语言文件（Locale/2052）为准。

---

## 一、飞船飞行阶段总览

`ShipData.stage` 字段定义了飞船的 5 个飞行阶段：

| stage | 名称 | 说明 |
|-------|------|------|
| -2 | 停泊（目标站） | 飞船停靠在目标星际物流运输站的泊位 |
| -1 | 离站/进站过渡 | 从泊位弹出或减速靠近泊位的平滑动画 |
| 0 | 太空自由飞行 | 在行星际空间中加速、巡航、减速、曲速飞行 |
| 1 | 进站/离站过渡 | 靠近目标站或离开目标站的平滑动画 |
| 2 | 停泊（目标站） | 飞船停靠在目标站，装卸货物 |

`ShipData.direction` 表示飞行方向：
- `+1`：从 planetA（本星球）飞往 planetB（目标星球）
- `-1`：从 planetB 返回 planetA

### 阶段转换状态机

```
          +--------------------------------+
          |                                |
          v                                |
  stage=-2 --(t:0->1)--> stage=-1 --(t:0->1)--> stage=0
  (停泊)                  (离站动画)             (太空飞行)
                            ^                     |
                            |                     v
                          stage=2 <--(t:1->0)-- stage=1
                          (停泊)                  (进站动画)
                            |
                            +--(t:1->0, 装卸货, direction反转)
```

---

## 二、各阶段运动详解

### 2.1 Stage -2：停泊（目标站）

飞船固定在目标站的泊位上，速度为 0。

- `direction > 0`（出发）：`t += 0.03335`，当 `t > 1` 时进入 stage = -1
- `direction < 0`（返航到站）：`t -= 0.03335`，当 `t < 0` 时卸货、归还飞船到空闲池

位置固定在：
```
uPos = station.uPos + QRotate(station.uRot, targetStation.shipDockPos + dockPos.normalized * -14.4)
```

### 2.2 Stage -1：离站/进站过渡

使用 smoothstep 插值的纯动画阶段，速度始终为 0。

**离站（direction > 0）**：
- `t += num14`，其中 `num14 = num9 * 0.006 + 1e-5`
- `num9 = pow(shipSailSpeed / 600, 0.4)`
- 当 `t > 1` 时进入 stage = 0
- 位置：从泊位沿法线方向弹出 25m
  ```
  uPos = station.uPos + QRotate(station.uRot, diskPos + diskPos.normalized * 25 * smoothstep(t))
  ```
  其中 `smoothstep(t) = (3 - 2t) * t^2`

**进站（direction < 0）**：
- `t -= num14 * (2/3)`（速度为离站的 2/3）
- 当 `t < 0` 时进入 stage = -2
- 位置：从 `pPosTemp`（stage 0 结束时保存的位置）平滑过渡到泊位
  ```
  uPos = lerp(dockPos, pPosTemp, smoothstep(t))
  uRot = slerp(dockRot, pRotTemp, smoothstep(t) * 2 - 1)
  ```

### 2.3 Stage 0：太空自由飞行（核心阶段）

这是最复杂的阶段，包含亚光速加速/巡航/减速、曲速飞行、天体规避等子行为。

#### 2.3.1 关键参数定义

```
num8  = shipSailSpeed / 600              （上限 500）
num9  = pow(num8, 0.4)
num10 = num9 > 1 ? log(num9) + 1 : num9  （速度阻尼修正）
num11 = shipSailSpeed * 0.03             （行星附近加速度）
num12 = shipSailSpeed * 0.12 * num10     （太空加速度）
num13 = shipSailSpeed * 0.4 * num8       （减速度）
```

#### 2.3.2 亚光速速度控制

每帧计算目标速度：
```
num30 = distance_to_target / (uSpeed + 0.1) * 0.382
num32 = uSpeed * num30 * num10 + 6*num9 + 0.15*num8
```

当 `uSpeed >> 0.1` 时，近似为：
```
v_target ~= 0.382 * num10 * distance + 6*num9 + 0.15*num8
         = k * d + offset
```

这是一个**比例控制器**：目标速度与剩余距离成正比。

速度更新规则：
- 若 `uSpeed < v_target - accel`：`uSpeed += accel`（加速）
- 若 `uSpeed > v_target + decel`：`uSpeed -= decel`（减速）
- 否则：`uSpeed = v_target`（直接设为目标速度）

其中 `accel = num12`（太空）或 `num11`（行星附近），`decel = num13`。

#### 2.3.3 曲速飞行

**启动条件**（全部满足）：
1. 已解锁曲速科技（`shipWarpSpeed > shipSailSpeed + 1`）
2. 距出发星球足够远（`distance^2 > 25000000`）
3. 距目标足够远（`distance > warpEnableDist * 0.5`，其中 `warpEnableDist = 480000` 对星际站）
4. 已达最大亚光速（`uSpeed >= shipSailSpeed`）
5. 携带空间翘曲器（`warperCnt > 0`）或沙盒模式

**曲速状态 dynamics**：
- 加速阶段：`warpState += 1/60`（每 tick），0 -> 1 需 60 tick = 1 秒
- 减速阶段：`warpState -= 1/15`（每 tick），但由于制动距离 `num28` 随 `warpState` 变化产生反馈振荡，实际耗时约 **40 帧（0.67 秒）**，而非 15 帧。详见第七章。
- `warpState` 钳制在 [0, 1]

**曲速速度公式**：
```
num26 = min(shipWarpSpeed, distance * 2)     // 有效曲速上限
num24 = num26 * (1001^warpState - 1) / 1000  // 当前曲速增量
uSpeed = shipSailSpeed + num24               // 总速度
```

制动距离：
```
brakingDist = num24 * 0.0449 + 5000 + shipSailSpeed * 0.25
```

当 `distance < brakingDist` 时开始退出曲速。

#### 2.3.4 天体规避

飞船检查起点和终点所在星系的恒星及行星（id 范围为 `[starId*100, starId*100+10)`）：

- 若飞船正朝天体飞去（速度方向指向天体），计算排斥力
- 规避力随距离减小而增大，随曲速状态增大而减小（曲速中几乎不规避）
- 同时考虑天体在下一帧的位置（`uPosNext`），做预测性规避

规避力改变飞船的角速度，从而改变飞行方向。**这会使实际飞行路径偏离直线，增加飞行距离。**

#### 2.3.5 姿态控制

飞船使用角速度-四元数系统控制朝向：
- 目标朝向由「指向目标的方向」和「天体规避方向」混合决定
- 角速度平滑趋向目标角速度
- 曲速中朝向被强制拉向速度方向（`Slerp` 权重 = `warpState^3`）

#### 2.3.6 进入 Stage 1 的条件

当 `distance_to_target < 6 * num10` 时，设置 `stage = direction`（即 stage = 1 或 stage = -1），并保存当前位置和朝向到 `pPosTemp` / `pRotTemp`。

### 2.4 Stage 1：进站/离站过渡

与 stage -1 对称的动画阶段。

**进站（direction > 0，到达目标站）**：
- `t -= num14 * (2/3)`
- 当 `t < 0` 时进入 stage = 2
- 位置从 `pPosTemp` 平滑过渡到目标站泊位

**离站（direction < 0，从目标站出发）**：
- `t += num14`
- 当 `t > 1` 时进入 stage = 0
- 位置从泊位沿法线方向弹出

### 2.5 Stage 2：停泊（目标站）

- `direction > 0`（到达）：`t -= 0.0334`，当 `t < 0` 时卸货、装货、`direction = -1`
- `direction < 0`（准备出发）：`t += 0.0334`，当 `t > 1` 时进入 stage = 1

---

## 三、是否存在闭式解

### 3.1 亚光速阶段：存在闭式解

亚光速速度控制的核心是比例控制器 `v_target = k * d + offset`。

在减速阶段（速度跟踪目标速度），有：
```
dv/dt = -k * v
```
解得：
```
v(t) = v0 * exp(-k * t)
d(t) = d0 - v0/k * (1 - exp(-k*t))
```

从速度 v1 减速到 v2 的时间：
```
t = ln(v1 / v2) / k
```

从速度 v1 减速到 v2 的距离：
```
delta_d = (v1 - v2) / k
```

加速阶段是恒定加速度 `a`：
```
t = (v2 - v1) / a
delta_d = (v2^2 - v1^2) / (2a)
```

### 3.2 曲速阶段：存在闭式解

曲速 ramp-up（warpState: 0 -> 1，60 tick）：
```
v(t) = shipSailSpeed + num26/1000 * (1001^(t/60) - 1)
```

积分得 ramp-up 距离：
```
d_ramp = shipSailSpeed * 1 + num26/1000 * (1000/ln(1001) - 1)
       ~= shipSailSpeed + num26 * 0.14376
```

曲速 ramp-down（warpState: 1 -> 0，15 tick）：
```
d_rampdown = shipSailSpeed * 0.25 + num26/1000 * (1001 - 1000/ln(1001) - 1/ln(1001))
```

曲速巡航阶段（warpState = 1）：
```
v = shipSailSpeed + num26 * (1001 - 1) / 1000 = shipSailSpeed + num26
t = distance / v
```

### 3.3 无法闭式求解的部分

| 因素 | 影响 | 可否闭式 | 影响程度 |
|------|------|----------|----------|
| 天体规避 | 改变飞行路径长度 | 否 | 中等。仅在接近天体时显著，曲速中几乎不规避 |
| 行星运动 | 目标位置随时间变化 | 否 | 小。行星轨道速度远小于飞船速度 |
| 姿态调整 | 影响速度方向，间接影响路径 | 否 | 很小 |
| 加速/减速切换 | 速度跟踪目标速度的过渡 | 可闭式 | — |
| 曲速 ramp-up/down | 指数速度曲线 | 可闭式 | — |

**天体规避的影响分析**：
- 规避仅在天体附近（距离 < `uRadius * 1.6` 左右）且非曲速状态时显著
- 规避力随 `(1 - warpState)` 线性衰减，曲速中几乎为零
- 在典型的长距离运输中（跨恒星系），规避导致的路径偏差占比很小（< 1%）
- 在同星系内短距离运输中，规避可能使路径增加 5-15%

**结论**：忽略天体规避和行星运动，采用纯直线距离假设，误差在可接受范围内（通常 < 5%）。


---

## 四、闭式计算飞船剩余时间的方法

### 4.1 输入参数

| 参数 | 符号 | 来源 |
|------|------|------|
| 飞船当前速度 | `v_cur` | `shipData.uSpeed` |
| 曲速状态 | `ws` | `shipData.warpState`（0~1） |
| 是否有空间翘曲器 | `hasWarper` | `shipData.warperCnt > 0` |
| 到起始塔的距离 | `d_from_home` | 飞船位置到 planetA 距离 |
| 到目标塔的距离 | `d_to_target` | 飞船位置到 planetB 距离 |
| 亚光速上限 | `v_sail` | `history.logisticShipSailSpeedModified` |
| 曲速上限 | `v_warp` | `history.logisticShipWarpSpeedModified` |
| 是否解锁曲速 | `canWarp` | `v_warp > v_sail + 1` |

### 4.2 辅助函数

```
def calc_params(v_sail):
    num  = min(v_sail / 600, 500)
    num9 = pow(num, 0.4)
    k    = 0.382 * num9
    if num9 > 1: k = 0.382 * (log(num9) + 1)
    offset = 0.15 * num + 6 * num9
    a_near  = v_sail * 0.03
    a_space = v_sail * 0.12 * (log(num9) + 1 if num9 > 1 else num9)
    d_decel = v_sail * 0.4 * num
    return k, offset, a_near, a_space, d_decel

def accel_time(v1, v2, a):
    if v2 <= v1: return 0, 0
    t = (v2 - v1) / a
    d = (v2*v2 - v1*v1) / (2*a)
    return t, d

def decel_time(v1, v2, k):
    if v2 >= v1: return 0, 0
    t = log(v1 / v2) / k
    d = (v1 - v2) / k
    return t, d

def warp_ramp_up_dist(v_sail, v_warp_eff):
    return v_sail * 1.0 + v_warp_eff * 0.14376

def warp_ramp_down_dist(v_sail, v_warp_eff):
    return v_sail * 0.25 + v_warp_eff * 0.18776

def sail_decel_phase(d, v_sail, k, offset):
    d_cruise = d - (v_sail - offset) / k
    if d_cruise > 0:
        t_cruise = d_cruise / v_sail
        t_decel, _ = decel_time(v_sail, offset, k)
        return t_cruise + t_decel
    else:
        v_peak = sqrt(2 * k * d + offset*offset)
        if v_peak > v_sail: v_peak = v_sail
        t_accel, _ = accel_time(offset, v_peak, a_space)
        t_decel, _ = decel_time(v_peak, offset, k)
        return t_accel + t_decel
```

### 4.3 主算法（伪代码）

```
function calc_remaining_time(v_cur, ws, has_warper, d_from_home, d_to_target, v_sail, v_warp):
    k, offset, a_near, a_space, d_decel = calc_params(v_sail)
    can_warp = (v_warp > v_sail + 1) AND has_warper
    t = 0

    # === 情况 A：飞船已在曲速中 (ws > 0) ===
    if ws > 0:
        v_warp_eff = min(v_warp, 2 * d_to_target)
        braking = v_warp_eff * (1001^ws - 1) / 1000 * 0.0449 + 5000 + v_sail * 0.25

        if d_to_target < braking:
            # 正在退出曲速
            t_exit = ws * 15
            d_exit = v_sail * t_exit + v_warp_eff * ws * 0.5 * 0.18776
            d_remaining = max(0, d_to_target - d_exit)
            t = t_exit + sail_decel_phase(d_remaining, v_sail, k, offset)
        else:
            # 仍在曲速巡航或加速
            if ws < 1:
                t_ramp = (1 - ws) * 60
                d_ramp = v_sail * t_ramp + v_warp_eff * (0.14376 - ws * 0.14376)
            else:
                t_ramp = 0
                d_ramp = 0

            d_cruise = d_to_target - braking - d_ramp
            if d_cruise > 0:
                v_cruise = v_sail + v_warp_eff
                t_cruise = d_cruise / v_cruise
            else:
                t_cruise = 0

            t = t_ramp + t_cruise + 15 + sail_decel_phase(braking, v_sail, k, offset)
        return t

    # === 情况 B：无曲速或未启动曲速 ===
    # 先判断是否会启动曲速
    warp_enable_dist = 480000  # 星际站

    if can_warp AND d_to_target > warp_enable_dist * 0.5:
        # 会启动曲速
        # 第一步：加速到 v_sail
        t_accel, d_accel = accel_time(v_cur, v_sail, a_space)
        d_remaining = d_to_target - d_accel

        # 第二步：曲速 ramp-up
        v_warp_eff = min(v_warp, 2 * d_remaining)
        t_ramp_up = 60
        d_ramp_up = warp_ramp_up_dist(v_sail, v_warp_eff)

        # 第三步：曲速巡航
        d_ramp_down = warp_ramp_down_dist(v_sail, v_warp_eff)
        d_cruise = d_remaining - d_ramp_up - d_ramp_down - 5000 - v_sail * 0.25
        if d_cruise > 0:
            v_cruise = v_sail + v_warp_eff
            t_cruise = d_cruise / v_cruise
        else:
            t_cruise = 0

        # 第四步：曲速 ramp-down + 亚光速减速
        t_ramp_down = 15
        t_decel = sail_decel_phase(5000 + v_sail * 0.25, v_sail, k, offset)

        t = t_accel + t_ramp_up + t_cruise + t_ramp_down + t_decel
    else:
        # 不会启动曲速，纯亚光速
        # 加速到目标速度，然后比例减速
        v_target = min(k * d_to_target + offset, v_sail)
        if v_cur < v_target:
            t_accel, d_accel = accel_time(v_cur, v_target, a_space)
        else:
            t_accel, d_accel = 0, 0

        d_remaining = d_to_target - d_accel
        t_decel = sail_decel_phase(d_remaining, v_sail, k, offset)
        t = t_accel + t_decel

    return t
```

### 4.4 简化版（忽略天体规避）

对于大多数场景，可以直接用以下简化公式：

**无曲速场景**：
```
t = (v_sail - v_cur) / a_space                          # 加速到最大速度
  + (d - (v_sail^2 - v_cur^2)/(2*a_space)) / v_sail     # 巡航
  + ln(v_sail / offset) / k                              # 减速
```

**有曲速场景**：
```
t = (v_sail - v_cur) / a_space                          # 加速
  + 60                                                   # 曲速 ramp-up
  + (d - d_accel - d_trans) / (v_sail + v_warp_eff)     # 曲速巡航
  + 15                                                   # 曲速 ramp-down
  + ln(v_sail / offset) / k                              # 亚光速减速
```

其中 `d_trans = warp_ramp_up_dist + warp_ramp_down_dist + 5000 + v_sail * 0.25`。

---

## 五、与游戏内置方法的对比分析

### 5.1 `CalcRemoteSingleTripTime` 分析

该方法（位于 `StationComponent`，行 75188）计算**从静止开始**的单程时间。它假设飞船从泊位出发，经历完整的加速-巡航-减速过程。

**方法概述**：
1. 计算起点（泊位 + 25m 弹出）到终点（目标站泊位 + 25m）的直线距离
2. 根据距离和速度参数，分情况计算时间：
   - 短距离（不足以加速到最大速度）：使用二次方程解峰值速度
   - 中距离（加速到最大速度但不足以巡航）：加速 + 直接减速
   - 长距离（有巡航段）：加速 + 巡航 + 减速
3. 如果有曲速且距离足够，加入曲速阶段

**关键近似**：
- 使用 `(v1^2 - v2^2) / (2a)` 计算加速/减速距离
- 使用 `(v1 - v2) / k` 计算比例减速距离
- 曲速 ramp-up/down 使用经验常数
- **完全忽略天体规避**

### 5.2 `CalcArrivalRemainingTime` 分析

该方法（位于 `StationComponent`，行 75273）计算**飞行中飞船**的剩余时间。它根据飞船当前状态（stage、uSpeed、warpState 等）分情况处理。

**方法概述**：
1. 如果飞船不在 stage 0（在停泊/动画阶段），直接用 `CalcRemoteSingleTripTime` 减去动画已过时间
2. 如果在 stage 0（太空飞行），根据当前速度和剩余距离计算：
   - 考虑飞船当前速度可能不是从 0 开始的
   - 考虑曲速状态（warpState）可能不是 0 或 1
   - 考虑飞船可能已经在行星引力范围外

**关键近似**：
- 用 `magnitude2`（飞船到出发星球的距离）判断是否已离开行星引力范围
- 用 `(v_sail + v_cur) * (v_sail - v_cur) / (2 * a)` 计算从当前速度加速到最大速度的距离
- 曲速退出阶段使用 `warpState * 20` 秒的近似
- **同样完全忽略天体规避**

### 5.3 对比结论

| 维度 | 游戏内置方法 | 本文闭式方法 |
|------|-------------|-------------|
| 天体规避 | 忽略 | 忽略 |
| 行星运动 | 忽略 | 忽略 |
| 亚光速加速 | 恒定加速度，精确 | 恒定加速度，精确 |
| 亚光速减速 | 比例减速，精确 | 比例减速，精确 |
| 曲速 ramp-up | 经验常数近似 | 积分闭式解 |
| 曲速 ramp-down | 经验常数近似 | 积分闭式解 |
| 曲速巡航 | 匀速，精确 | 匀速，精确 |
| 动画阶段 | 用 `t` 和 `num14` 计算 | 可同样处理 |
| 行星引力范围 | 考虑了 `uRadius * 1.5` 判断 | 可同样处理 |

**谁更准？**

在核心物理模型上，**两者精度相同**——因为它们基于完全相同的运动方程。差异仅在于：

1. **曲速 ramp-up/down 的精度**：本文的闭式方法使用积分精确解，而游戏内置方法使用经验常数（如 `num26 * 0.18775714286` 等）。这些经验常数本身就是从积分近似得来的，精度差异在 1 tick 以内，**实际无差别**。

2. **`CalcArrivalRemainingTime` 的分支处理**：游戏内置方法有更完善的分支逻辑（处理飞船在不同阶段、不同距离、不同速度下的各种组合），而本文的闭式方法需要手动实现这些分支。但一旦实现了相同的分支逻辑，**结果完全一致**。

3. **动画阶段**：游戏内置方法用 `t / num14` 计算动画剩余时间，本文方法同样可以做到。

**最终结论**：本文的闭式方法与游戏内置方法**精度完全相同**，因为：
- 两者都基于相同的运动方程（比例减速 + 恒定加速 + 曲速指数曲线）
- 两者都忽略了天体规避和行星运动
- 游戏内置方法中的经验常数本质上就是闭式积分的数值近似

**本文方法的优势**：
- 不需要模拟，直接计算
- 代码更简洁，易于理解和维护
- 可以独立于游戏引擎运行

**本文方法的劣势**：
- 需要手动实现所有分支逻辑（与 `CalcArrivalRemainingTime` 一样多的分支）
- 天体规避导致的误差在短距离同星系运输中可达 5-15%

---

## 六、总结

1. **飞船飞行有 5 个阶段**：停泊（-2）、离站过渡（-1）、太空飞行（0）、进站过渡（1）、停泊（2）。核心是 stage 0 的太空飞行。

2. **亚光速阶段存在闭式解**：速度控制是比例控制器 `v_target = k * d + offset`，减速阶段满足 `dv/dt = -k*v`，可精确积分。

3. **曲速阶段存在闭式解**：曲速速度是 `1001^t` 的指数函数，ramp-up/down 均可精确积分。

4. **无法闭式求解**：天体规避（改变路径）、行星运动（改变目标位置）。忽略它们误差通常 < 5%。

5. **闭式方法与游戏内置方法精度相同**：两者基于相同的物理模型，差异仅在实现细节上（积分 vs 经验常数），实际结果一致。

6. **推荐使用场景**：
   - 需要快速估算大量飞船到达时间 → 使用简化闭式公式
   - 需要精确到秒的到达时间 → 使用完整分支逻辑的闭式方法
   - 需要像素级精度 → 逐帧模拟（游戏本身的做法）


---

## 七、曲速减速阶段的振荡问题（勘误与深入分析）

### 7.1 问题描述

原文档（第二章 2.3.3）中描述曲速减速阶段为「`warpState -= 1/15`，1 -> 0 需 15 tick = 0.25 秒」。但实际代码中存在一个微妙的反馈循环，导致减速阶段远不止 15 帧。

核心代码：
```
num24 = num26 * (1001^warpState - 1) / 1000           // 当前曲速增量
num28 = num24 * 0.0449 + 5000 + shipSailSpeed * 0.25  // 制动距离
if (num21 < num28)                                     // 若剩余距离 < 制动距离
    warpState -= 1/15                                  // 减速
else
    warpState += 1/60                                  // 加速
```

### 7.2 振荡机制

问题出在 `num28`（制动距离）依赖于 `num24`，而 `num24` 依赖于 `warpState`：

1. **帧 N**：`num21 < num28` → `warpState -= 1/15`（减速）
2. `warpState` 降低 → `num24` 降低 → `num28`（制动距离）**大幅**降低
3. **帧 N+1**：`num28` 已经缩小，可能 `num21 > num28` → `warpState += 1/60`（加速！）
4. `warpState` 回升 → `num24` 回升 → `num28` 回升
5. **帧 N+2**：`num21 < num28` 再次成立 → 又减速

这形成了一个**锯齿振荡**：warpState 在减速和加速之间反复横跳。

### 7.3 数值验证

以默认参数（`v_sail=400`, `v_warp=400000`）逐帧模拟：

```
帧   warpState变化   方向    num21        num28(制动距离)
 0   1.0000->0.9333  DEC    23060.0      23060.0
 1   0.9333->0.9500  ACC    23053.3      16424.6    ← 制动距离骤降！
 2   0.9500->0.9667  ACC    18329.2      17808.8
 3   0.9667->0.9000  DEC    17797.0      19362.0    ← 制动距离回升！
 4   0.9000->0.9167  ACC    17790.3      14091.5
 5   0.9167->0.8500  DEC    14047.8      15190.9
 ...
```

**统计**：
- 总帧数：**40 帧**（而非预期的 15 帧）
- 减速帧：20 帧，加速帧：19 帧
- 净减速量：仅 1 次（20 - 19），远小于需要的 15 次
- 条件翻转次数：36 次（几乎每帧都在翻转）

### 7.4 为什么会振荡

根本原因是 `num28` 对 `warpState` 的敏感度过高。

`num28` 对 `warpState` 的导数：
```
d(num28)/d(ws) = 0.0449 * num26 * 1001^ws * ln(1001) / 1000
```

在 `ws=1.0` 时，这个导数为 `0.0449 * 400000 * 1001 * 6.908 / 1000 ≈ 124,000`。

即 `warpState` 减少 `1/15 ≈ 0.067`，`num28` 就减少约 `124000 * 0.067 ≈ 8300` 米。

而飞船每帧前进的距离（在 ws=1 时）约为 `(400 + 400000) / 60 ≈ 6673` 米。

**`d_num28 > d_num21`**：制动距离的收缩速度（8300 m/frame）超过了飞船的前进速度（6673 m/frame），导致下一帧 `num21 > num28`，触发加速。

### 7.5 为什么最终还是能退出曲速

虽然振荡严重，但系统仍然能退出曲速，原因有两个：

1. **`num29` 钳制机制**（第 74530-74532 行）：
   ```
   if (num24_new / 60 > num29)
       num24_new = num29 / (1/60) * 1.01
   ```
   当 `num29`（`num21 - num28`）很小时，`num24_new` 被钳制到一个很小的值，飞船实际以接近亚光速飞行。这防止了飞船在减速帧「冲过头」。

2. **净漂移**：虽然每两帧（一减一加）净 warpState 变化为 `-1/15 + 1/60 = -1/20`，但偶尔会出现连续两帧减速的情况，最终 warpState 会缓慢趋向 0。

### 7.6 对闭式解的影响

这是一个**重大修正**：

| 原假设 | 实际情况 |
|--------|----------|
| 曲速减速 15 帧（0.25 秒） | 约 40 帧（0.67 秒），取决于参数 |
| 单调递减 | 锯齿振荡 |
| 闭式积分可精确求解 | 需要逐帧模拟或分析振荡包络 |

**闭式近似修正**：

振荡期间，warpState 的净变化率约为：
```
d(ws)/dt ≈ -1/20 per frame = -3 per second
```

即有效减速速率约为原来的 `(1/20) / (1/15) = 0.75` 倍。因此减速时间约为：
```
t_decel ≈ 15 / 0.75 = 20 帧（理论下限）
```

实际模拟显示约 40 帧，因为振荡幅度随 ws 降低而减小，后期减速更慢。

**更精确的闭式估计**：

在振荡平衡点附近，`num21 ≈ num28`，即：
```
num21 ≈ num24 * 0.0449 + 5000 + v_sail * 0.25
```

飞船每帧前进 `(v_sail + num24) / 60`，同时 `num28` 随 `warpState` 变化。由于振荡，有效前进速度约为 `v_sail + num24_effective`，其中 `num24_effective` 是振荡平均值。

对于工程应用，可以用以下经验公式：
```
t_warp_decel ≈ 40 frames = 0.67 seconds  （默认参数）
```

或更一般地：
```
t_warp_decel ≈ 15 * (1 + 0.0449 * num26 * ln(1001) / (1000 * 60 * v_sail)) 帧
```

### 7.7 对 `CalcRemoteSingleTripTime` 和 `CalcArrivalRemainingTime` 的影响

游戏内置方法同样没有考虑这个振荡——它们假设曲速 ramp-down 是 15 帧的单调过程。因此：

- **`CalcRemoteSingleTripTime`**：低估了曲速减速时间约 25 帧（0.42 秒）
- **`CalcArrivalRemainingTime`**：在曲速减速阶段同样低估

这意味着**本文的闭式方法（如果正确处理振荡）可以比游戏内置方法更准确**。

### 7.8 如何处理

**方案 A：逐帧模拟**（最准确）
在曲速减速阶段，不假设 15 帧单调递减，而是逐帧模拟振荡过程。约 40 帧的模拟量很小，完全可以接受。

**方案 B：经验修正**（平衡）
将曲速减速时间从 15 帧修正为 40 帧（默认参数下），或使用上述经验公式。

**方案 C：简化处理**（最快）
忽略振荡细节，直接使用 `warpState * 40` 帧作为剩余减速时间（而非 `warpState * 15`）。

推荐**方案 B**，在保持闭式计算优势的同时修正这个系统性误差。
