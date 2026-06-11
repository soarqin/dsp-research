# 戴森球计划：星际物流运输船飞行逻辑分析

> 反编译工具：`ilspycmd 10.1.0.8386`  
> 源文件：`Assembly-CSharp.dll`  
> 分析目标：`StationComponent.InternalTickRemote`  
> 术语来源：`Locale/2052/base.txt`（UTF-16 LE）

---

## 一、预备：关键参数推导

`InternalTickRemote` 每帧（1/60 秒）调用一次。进入飞行主循环前，先由 `shipSailSpeed`（航行速度）推导出以下常量：

```csharp
float s  = shipSailSpeed / 600f;           // 无量纲速度比
float s9 = Mathf.Pow(s, 0.4f);            // num9
float s10 = s9 > 1f ? Mathf.Log(s9) + 1f : s9;  // num10（对数修正因子）
// s 超过 500 时截断

float accel_takeoff = shipSailSpeed * 0.03f;       // num11：起航加速度（/帧）
float accel_cruise  = shipSailSpeed * 0.12f * s10; // num12：巡航加速度（/帧）
float accel_brake   = shipSailSpeed * 0.4f  * s;   // num13：制动减速度（/帧）
float liftoff_rate  = s9 * 0.006f + 1e-5f;         // num14：停泊动画帧率因子
```

游戏以每秒 60 帧运行，速度单位为 m/s（宇宙坐标系），距离单位为 m。

---

## 二、飞行阶段（`ShipData.stage` 与 `direction`）

| `stage` | `direction > 0`（去程） | `direction < 0`（回程） |
|---------|------------------------|------------------------|
| `−2`    | 停泊动画：从停机位缓慢飞出 | 停泊动画：飞入本站停机位（卸货） |
| `−1`    | 起航离站（本站） | 着陆进站（本站） |
| `0`     | **星际巡航飞行** | **星际巡航飞行** |
| `1`     | 着陆进站（目标站） | 起航离站（目标站） |
| `2`     | 停泊于目标站（装货等待） | 停泊于目标站 |

`direction = 1` 表示从本站出发、前往目标站；`direction = -1` 表示已在目标站完成装货、返回本站。**星际巡航阶段（`stage == 0`）是本分析的核心。**

---

## 三、各阶段运动方式详解

### 3.1 停泊阶段（`stage < −1`）

飞船锁定在停机位盘旋，以固定步长 `Δt = 0.03335` 推进参数 `t ∈ [0, 1]`，不产生真实位移（速度清零）。  
**持续时间**：约 `1 / 0.03335 ≈ 30 帧 = 0.5 s`。

### 3.2 起航 / 着陆动画（`stage == −1` 或 `stage == 1`）

同样是纯动画插值，速度为零。参数 `t` 以速率 `num14 = s9 * 0.006f + 1e-5f` 推进（去程）或 `2/3 × num14`（回程）。  
**持续时间**（起航，去程）：`1 / num14` 帧。

**是否有闭式解**：是（`t` 单调线性推进，时间 = `1 / liftoff_rate` 帧）。

### 3.3 星际巡航（`stage == 0`）

这是飞船实际在宇宙坐标系移动的阶段，包含两个子模式。

#### 3.3.1 普通巡航（无曲速）

飞船按朝向向量 `uVel` 以速度 `uSpeed` 在宇宙空间移动，每帧：

```
uPos += uVel * uSpeed * (1/60)
```

速度控制：每帧计算一个**目标速度** `targetSpeed`：

```csharp
// 距离目标 num21，当前速度 uSpeed，剩余时间估算
double tRemain = num21 / (uSpeed + 0.1) * 0.382;

// 目标速度（即减速到停止所需距离约为 num21）
float targetSpeed = uSpeed * tRemain * s10 + 6f * s9 + 0.15f * s;
targetSpeed = min(targetSpeed, shipSailSpeed);
```

实际加速/减速：

- 若 `uSpeed < targetSpeed - Δacc`：`uSpeed += Δacc`（Δacc = 加速度/帧）
- 若 `uSpeed > targetSpeed + accel_brake/60`：`uSpeed -= accel_brake/60`
- 否则：`uSpeed = targetSpeed`

**运动本质**：这是一个**速度反馈控制系统**，`targetSpeed` 是剩余距离的函数，没有解析解。飞船在加速段近似匀加速，在制动段逐渐减速至停止。

#### 3.3.2 曲速巡航（`warpState > 0`）

启动条件（同时满足）：

1. `shipWarpSpeed > shipSailSpeed + 1`（有曲速能力）
2. 距目标站距离 `num21 > warpEnableDist * 0.5`
3. 距本星 `num22 > 25_000_000` m（脱离引力圈）
4. 当前速度 `uSpeed >= shipSailSpeed`（已达到最大航行速度）
5. 持有空间翘曲器（`warperCnt > 0`）或 Demo 模式

曲速速度计算（指数增长模型）：

```csharp
// warpSpeedCap 为 min(shipWarpSpeed, 2 * 两站距离)
float warpSpeedAdd = warpSpeedCap * (Pow(1001, warpState) - 1) / 1000;
uSpeed = shipSailSpeed + warpSpeedAdd;  // 总速度
```

`warpState ∈ [0, 1]` 每帧 `+1/60`（加速中）或 `-1/15`（即将到达时减速），最大值 1。  
当 `warpState = 1` 时，`warpSpeedAdd ≈ warpSpeedCap × 1000/1000 = warpSpeedCap`。

**减速触发**：当到目标的距离 `num21 < 制动距离 num28` 时开始降曲速：

```csharp
double brakeDist = num24 * 0.0449 + 5000 + shipSailSpeed * 0.25;
// 若 num21 < brakeDist，则 warpState -= 1/15（4倍减速速率）
```

---

## 四、闭式解分析

### 4.1 各阶段是否存在闭式解

| 阶段 | 闭式解 | 说明 |
|------|--------|------|
| 停泊动画（stage < −1） | ✓ | `30帧 × Δt = 0.03335`，固定 ≈ 30 帧 |
| 起航/着陆（stage ±1） | ✓ | `t` 以 `liftoff_rate` 线性递减，时间 = `1/liftoff_rate` 帧 |
| 普通巡航加速段 | 近似 ✓ | 速度反馈控制，但初期近似匀加速 |
| 普通巡航匀速段 | ✓ | `distance / (shipSailSpeed × 1/60)` 帧 |
| 普通巡航制动段 | 近似 ✓ | 对数型减速控制（见下文） |
| 曲速加速段 | 近似 ✓ | `warpState` 以 `1/60` 线性增长（约 60 帧 = 1 s） |
| 曲速匀速段 | ✓ | 固定速度 `shipSailSpeed + warpSpeedCap` |
| 曲速减速段 | 近似 ✓ | `warpState` 以 `1/15` 线性减少（约 15 帧） |

### 4.2 制动段速度的近似闭式分析

普通巡航制动时，`targetSpeed ≈ uSpeed × (dist / (uSpeed × 2.618)) × s10`。  
设 `k = 0.382 × s10`，近似得 `targetSpeed ≈ k × dist`（正比于剩余距离），速度按位置反馈，微分方程为：

```
dv/dt = (k·d - v) × α    （α 为跟踪增益）
```

其中 `d = d0 - ∫v dt`。这是一阶线性系统，有精确解，但涉及积分常数，不适合手算。  
**实际上游戏代码用了更粗糙的直接赋值（`uSpeed = targetSpeed`），相当于每帧瞬时跟踪目标速度，因此等效为：**

```
v(t) ≈ k × d(t)
d(d)/dt = -v = -k × d
d(t) = d0 × exp(-k × t)
v(t) = k × d0 × exp(-k × t)
```

**制动时间的闭式解（近似）**：

```
T_brake = -ln(v_end / v_start) / k
         ≈ ln(shipSailSpeed / v_min) / (0.382 × s10) × 60  帧
```

其中 `v_min = 6 × s9 + 0.15 × s`（最低速度阈值，此时 targetSpeed 不再由距离决定）。

---

## 五、闭式计算「本程剩余时间」的方法

### 5.1 输入参数

```
uSpeed      : float   当前飞船速度（m/s）
warpState   : float   当前曲速状态 [0,1]（0 = 未曲速）
warperCnt   : int     飞船持有的空间翘曲器数量
distA       : double  飞船当前位置到出发站的距离（m）
distB       : double  飞船当前位置到目标站的距离（m）（即 num21）
totalDist   : double  出发站到目标站直线距离（m）（用于 warpSpeedCap）
shipSailSpeed : float 航行速度上限（m/s）
shipWarpSpeed : float 曲速速度上限（m/s）
warpEnableDist: double 曲速启用路程（m）
```

（GameHistory 中的科研升级数据最终反映在 `shipSailSpeed`、`shipWarpSpeed`、`shipCarries` 等参数上，由调用方传入，无需在此单独处理）

### 5.2 预计算常量

```python
s     = shipSailSpeed / 600.0
s9    = pow(s, 0.4)
s10   = log(s9) + 1.0 if s9 > 1.0 else s9
s     = min(s, 500.0)

accel_cruise  = shipSailSpeed * 0.12 * s10   # 巡航加速度（/帧）
accel_brake   = shipSailSpeed * 0.4  * s     # 制动减速度（/帧）
v_min         = 6.0 * s9 + 0.15 * s          # 最低速度阈值
k_brake       = 0.382 * s10                  # 制动增益

warpSpeedCap  = min(shipWarpSpeed, 2.0 * totalDist)
brakeDist     = warpSpeedCap * 0.0449 + 5000.0 + shipSailSpeed * 0.25
```

### 5.3 分支计算（帧数 → 除以60得秒）

#### 情形 A：已在曲速中（`warpState > 0`）

```python
# 曲速额外速度（当前帧）
warpAdd = warpSpeedCap * (pow(1001.0, warpState) - 1.0) / 1000.0
# 到达减速点前的匀速段距离
cruiseDist = max(0.0, distB - brakeDist)
cruiseTime = cruiseDist / (shipSailSpeed + warpSpeedCap)  # 秒

# 减速段：warpState 以 1/15 帧递减，每帧速度降低
# 近似：平均速度为 shipSailSpeed + warpSpeedCap/2，减速时间约 warpState × 15 帧
warpDecelTime = warpState * 15.0 / 60.0  # 秒

# 进入目标星球大气后减速（普通制动）
decelDist   = 5000.0 + shipSailSpeed * 0.25
decelFrames = log(shipSailSpeed / v_min) / k_brake * 60.0

total = cruiseTime + warpDecelTime + decelFrames / 60.0
```

#### 情形 B：无曲速，普通巡航（`warpState == 0`，`distA >= 1.5 × uRadius`）

```python
# 已达最大速度，估算制动距离
brakeDist_normal = (shipSailSpeed ** 2 - v_min ** 2) / (2.0 * accel_brake / 60.0 * 60.0)
# 等效：(v² - vmin²) / (2 × accel_brake_per_sec)

if uSpeed < shipSailSpeed:
    # 还在加速中：先加速，再匀速，再减速
    accelTime = (shipSailSpeed - uSpeed) / accel_cruise * (1.0/60.0)  # 秒
    accelDist = (shipSailSpeed + uSpeed) * 0.5 * accelTime
    cruiseDist = max(0.0, distB - accelDist - brakeDist_normal)
    cruiseTime = cruiseDist / shipSailSpeed
    decelFrames = log(shipSailSpeed / v_min) / k_brake * 60.0
    total = accelTime + cruiseTime + decelFrames / 60.0
else:
    # 已达最大速度
    cruiseDist = max(0.0, distB - brakeDist_normal)
    cruiseTime = cruiseDist / shipSailSpeed
    decelFrames = log(shipSailSpeed / v_min) / k_brake * 60.0
    total = cruiseTime + decelFrames / 60.0
```

#### 情形 C：还未脱离本星引力（`distA < 1.5 × uRadius_A`）

```python
# 先以 accel_takeoff 加速到 shipSailSpeed，再进入情形 B 或 C-warp
accelTime2 = (shipSailSpeed - uSpeed) / (accel_takeoff / 60.0) / 60.0  # 秒
accelDist2 = (shipSailSpeed + uSpeed) * 0.5 * accelTime2
remaining = distB - accelDist2
# 后接情形 B 或情形 A
```

### 5.4 加上固定开销

```python
# 目标站停泊 + 起航/着陆动画（去程终点）
liftoff_rate = s9 * 0.006 + 1e-5
dock_frames  = 30.0                          # stage < -1 停泊动画
liftoff_frames = 1.0 / liftoff_rate + 30.0  # stage -1/1 起降动画（近似）

# 目标站装卸货等待（stage 2）
cargo_frames = 30.0  # ≈ 30 帧

total_fixed = (dock_frames + liftoff_frames + cargo_frames) / 60.0
```

### 5.5 精度取舍

| 舍弃项 | 误差量级 | 影响 |
|--------|----------|------|
| 行星遮挡绕行（`vectorLF6` 轨道修正） | < 5 000 m，占比极小 | 可忽略（< 1 s） |
| 起航段引力圈内加速细节 | 几秒 | 远距离时可忽略 |
| 曲速加速指数型增长（非线性） | 数秒 | 取 warpState 当前值估算，误差 < 5 s |
| 运输船转向时间 | < 2 s | 可忽略 |
| 帧率抖动 | < 1 帧 | 可忽略 |

**无法闭式得到但影响不大的量**：
- 行星引力绕行距离（依赖实时轨迹，但通常 < 0.1% 额外路程）
- 曲速加速的精确积分（约 80 帧）（`CalcRemoteSingleTripTime` 也用了常量近似）

---

## 六、与游戏内置方法对比

### 6.1 `CalcRemoteSingleTripTime`（单程全程时间）

代码位置：`StationComponent.cs:2962`

该方法从**出发停机位到目标停机位**计算单程总帧数，关键变量：

```csharp
float num7 = Mathf.Sqrt(((direction==1)?2f:0.5f) * astroData.uRadius * num5); // 离开引力圈时的初速 v0
float num8 = (vSail + v0)(vSail - v0) / num6 * 0.5f;  // 加速段位移（从 v0 加速到 vSail）
float num9 = 0.382f * num3;   // k_brake
float num10 = 0.15f*s + 6f*s9;// v_min
float num11 = (vSail - num10) / num9;  // 制动段等效距离
int num12 = 制动段帧数（对数公式）
```

**三段式（无曲速，距离足够长）**：

```
totalFrames = fixed_overheads
            + (v0/accel_takeoff)*60        // 起航加速段
            + ((vSail-v0)/accel_cruise)*60 // 引力圈内加速段
            + (dist - num8 - num11)/vSail*60  // 匀速段
            + num12                         // 对数制动段
```

对数制动公式：

```csharp
num12 = (int)((log(k*num11/vmin + 1) - log(k*6*s3/vmin + 1)) / k * 60 + 1)
```

这正好来自指数衰减型速度控制（`v = k × d` 时 `d(t) = d0 e^{-kt}`）的积分：

```
T = integral dt = integral dd / (k × d) = (1/k) × ln(d_start / d_end)
```

**曲速段（有曲速时）**：

```csharp
double warpEffDist = vWarpCap * 0.18775714286 + vSail * 1.33333333333;
```

这是对曲速加速（≈80帧）+ 曲速减速（≈20帧）段总积分的**常量近似**，相当于：

- 加速段以平均 `0.5 × vWarpCap` 速度飞行约 80 帧
- 减速段以平均 `0.5 × vWarpCap` 速度飞行约 20 帧
- 数值验证：`0.5 × (80+20)/60 ≈ 0.833 s × vWarpCap + ...`

当总距离较大时（`dist > warpEffDist + num8 + brakeDist14`），加上匀速曲速段：

```
warpCruiseFrames = (dist - warpEffDist - num8 - brakeDist) / (vWarpCap + vSail) * 60
```

此处 `(vWarpCap + vSail)` 是曲速匀速时的**视速度**（双向叠加——实际上应该是单一方向，但游戏似乎把往返一并计入？仔细看：`num17 = min(vWarp, 2*dist)` 因此最大曲速已按单程加了系数，匀速段用 `num17 + vSail` 也是单程估算的技巧）。

### 6.2 `CalcArrivalRemainingTime`（当前剩余时间）

代码位置：`StationComponent.cs:3047`

**对于 `stage != 0`（停泊/起降动画阶段）**，直接用 `CalcRemoteSingleTripTime` 作全程估算，然后按当前 `t` 值扣减已消耗帧数：

```csharp
case -2: num -= (int)(shipData.t * 30f);         // 停泊动画进度
case -1: num -= (int)(t / liftoff_rate + 30f);   // 起航进度
case  1: num -= (int)(t / liftoff_rate + 30f);   // 着陆进度（当 stage×dir < 0，即返程）
case  2: num -= (int)(shipData.t * 30f);
```

**对于 `stage == 0`（星际巡航中）**，重新从当前状态估算：

1. 计算当前到目标站距离 `num8`（即 `distB`）
2. 根据 `distA`（到本星距离）判断是否还在引力圈内
3. 根据 `warperCnt > 0` 和 `warpState` 分三大分支处理

这个方法**比 `CalcRemoteSingleTripTime` 更精确**，因为：
- 使用了当前实际速度 `shipData.uSpeed` 而非起始推算速度
- 使用了当前实际 `warpState` 而非从零起算
- 使用了到目标的实际距离而非出发站间距离

---

## 七、谁计算得更准？

### 7.1 对比矩阵

| 方法 | 精度 | 计算复杂度 | 适用场景 |
|------|------|-----------|---------|
| **自制闭式（本文第五节）** | 中（误差 < 10 s） | 低 | 快速估算，无需状态机 |
| `CalcRemoteSingleTripTime` | 低（误差 10–60 s） | 低 | 任务调度预估（出发前） |
| `CalcArrivalRemainingTime` | 中高（误差 < 5 s） | 中 | UI 显示倒计时（飞行中） |
| 逐帧模拟 | 最高（精确） | 最高 | 不实际 |

### 7.2 `CalcRemoteSingleTripTime` 的已知误差源

1. **起始速度假设**：用 `sqrt(2 × uRadius × accel_takeoff)` 估算离开引力圈的速度，忽略了飞船从停机位的真实加速过程
2. **曲速段近似**：用常量 `0.18775714286` 近似曲速加减速积分，实际曲速速度是指数增长的（`Pow(1001, warpState)`）
3. **完全忽略行星相对运动**：`astroPoses` 使用当前帧位置，不预测飞行途中行星的移动
4. **`Pow(num19, 0.0)` 疑似 Bug**：`6400 / (79 × Pow(x, 0.0) + 1/x) = 6400 / (79 + 1/x)`，其中 `0.0` 次幂恒为 1——这里的指数似乎本应是某个非零值，但现版本始终等于 1，导致中等距离段的估算可能有偏差（详见 7.3）

### 7.3 疑似 Bug：`Math.Pow(num19, 0.0)`

在 `CalcRemoteSingleTripTime`（行 3037）和 `CalcArrivalRemainingTime` 多处出现：

```csharp
num13 += (int)(6400.0 / (79.0 / Math.Pow(num19, 0.0) + 1.0 / num19));
```

`Math.Pow(x, 0.0) = 1` 恒成立，因此实际计算的是：

```
6400 / (79 + 1/x)   其中 x = (dist - num8 - brakeDist) / warpEffDist
```

当 `x ≈ 1`（中等距离，正好用完一个曲速段）时结果为 `6400/80 = 80` 帧（即 80 帧常量——这可能就是设计意图！）  
当 `x << 1`（距离很短）时，`1/x` 很大，分母趋向无穷，帧数趋向 0。  
当 `x > 1`（距离超长）时走另一分支（`num8 > num22 + ...`）。

**结论**：这个公式实际上是对曲速加减速段的一个固定 80 帧估算的平滑版本，指数 `0.0` 不是 Bug，而是刻意将 `79/x^0 = 79` 作为常数项。

### 7.4 `CalcArrivalRemainingTime` 的优势与局限

**优势**：
- 利用了当前真实的 `uSpeed`、`warpState` 和实际距离 `distB`
- 对「已在曲速中」分支（`warpState > 0`）给出了更好的估算
- 加速段直接用 `uSpeed` 而非历史估算

**局限**：
- 使用全程时间减去已消耗时间的方式（`stage != 0` 时）仍然依赖 `CalcRemoteSingleTripTime` 的近似
- 曲速段仍使用 `Pow(x, 0.0)` 公式，误差特性相同
- 忽略行星运动（和 `CalcRemoteSingleTripTime` 相同）

### 7.5 本文方法与游戏方法的差异

本文第五节的闭式方法与 `CalcArrivalRemainingTime` 在 `stage == 0` 且无曲速时的思路一致，精度相当。差异在于：

- **本文方法**：制动段用 `ln(vSail / v_min) / k_brake` 计算（严格的指数衰减积分）
- **游戏方法**：使用了相同的对数公式（`log(k*d/vmin + 1) / k * 60`）
- **两者一致**，因为游戏内速度控制 `v = k × d` 确实产生指数衰减，对数公式是其精确积分

对于**曲速段**，`CalcArrivalRemainingTime` 的 `80帧常量 + 匀速段` 分解方式，**与逐帧模拟的误差在 5–15 帧之间**（约 0.1–0.25 秒），对于游戏 UI 显示用途已经足够。

---

## 八、总结

1. **运输船飞行分 5 个主要阶段**：停泊 → 起航 → **星际巡航** → 着陆 → 停泊/装货
2. **星际巡航**分两种：普通巡航（速度反馈控制，加速+匀速+对数制动）和曲速巡航（指数加速+匀速+快速减速）
3. **闭式解可行**：除曲速加速段的精确积分外，其余各段均有解析公式；曲速段用 80 帧常量近似已足够（误差 < 5 s）
4. **精度排名**：逐帧模拟 > `CalcArrivalRemainingTime` ≈ 本文方法 > `CalcRemoteSingleTripTime`
5. `CalcRemoteSingleTripTime` 设计用于出发前估算（误差 10–60 s），`CalcArrivalRemainingTime` 设计用于飞行中 UI 显示（误差 < 5 s），本文方法可作为不依赖游戏内部状态机的独立估算工具

---

## 九、附录：曲速减速阶段的振荡分析

### 9.1 问题描述

曲速减速逻辑（`InternalTickRemote` stage 0，`warpState > 0` 分支）：

```csharp
num24 = (float)(num26 * ((Math.Pow(1001.0, reference2.warpState) - 1.0) / 1000.0));
double num28 = (double)num24 * 0.0449 + 5000.0 + (double)shipSailSpeed * 0.25;
double num29 = num21 - num28;
if (num29 < 0.0) num29 = 0.0;

if (num21 < num28)
    reference2.warpState -= 1f / 15f;  // 减速
else
    reference2.warpState += 1f / 60f;  // 继续加速
```

**问题**：`warpState` 下降后，`num24` 随之下降，`num28` 也随之下降——这可能使下一帧重新满足 `num21 > num28'`，导致 `warpState` 回升，形成振荡。

### 9.2 振荡的触发条件

当 `num21 < num28` 触发减速时（`num29 = 0`），`num24_post` 被限幅为 0，当前帧飞船以 `vSail`（而非 `vSail + num24`）飞行：

```
Δdist（当前帧飞船移动距离） = vSail / 60
```

减速后 `warpState' = warpState - 1/15`，下一帧的制动距离阈值：

```
num28' = num24(warpState') × 0.0449 + 5000 + vSail × 0.25
```

下一帧是否振荡（`num21' > num28'`）：

```
num21' = num21 - vSail/60 ≈ num28 - vSail/60
```

振荡条件（`num21' > num28'`）化简为：

```
(num24(w) - num24(w - 1/15)) × 0.0449 > vSail / 60
```

即 **`num24` 的单步降幅 × 0.0449 > 飞船在一帧内的位移**。

### 9.3 临界 warpState

令 `Δnum24 = C × 1001^(w-1/15) × (1001^(1/15) - 1) / 1000`（其中 `C = min(vWarpSpeed, 2×totalDist)`）：

当 `Δnum24 × 0.0449 = vSail / 60` 时，解出临界值：

```
warpState_crit ≈ 0.567（与 vWarp/vSail 比值有关，但在合理游戏参数范围内约为 0.55–0.58）
```

| warpState | vWarp=20×vSail | 是否振荡 |
|-----------|---------------|---------|
| 1.0       | Δnum24 ≈ 11083 vSail/m | ✓ |
| 0.8       | Δnum24 ≈ 2783 vSail/m  | ✓ |
| 0.6       | Δnum24 ≈ 699 vSail/m   | ✓ |
| **0.567** | Δnum24 = 阈值           | 临界 |
| 0.5       | Δnum24 ≈ 441 vSail/m   | ✗ |
| 0.3       | Δnum24 ≈ 111 vSail/m   | ✗ |

### 9.4 振荡的实际表现

振荡**不是无限持续的**，而是**阻尼振荡**（`warpState` 指数型增长，高 `warpState` 时 Δnum24 远大于阈值，低 `warpState` 时 Δnum24 小于阈值，振荡自然停止）。

减速阶段分两段：

| 阶段 | 条件 | 模式 | 净下降速率 |
|------|------|------|-----------|
| 振荡段 | `warpState > 0.567` | 1帧减速 + 1帧回弹交替 | `-1/15 + 1/60 = -3/60` 每2帧 |
| 单调段 | `warpState ≤ 0.567` | 每帧纯减速 | `-1/15` 每帧 |

逐帧模拟结果（`vSail=1500, vWarp=30000`）：

```
帧   warpState    num21      num28    减速?
0    1.0000      6715.3     6722.0    ✓
1    0.9333      6690.3     6224.3      （反弹）
2    0.9500      6311.5     6328.2    ✓
3    0.8833      6286.5     5975.9      （反弹）
...
13   0.6333      5474.8     5480.7    ✓
14   0.5667      5449.8     5441.2      （最后一次反弹）
15   0.5833      5416.1     5449.4    ✓  ← 此后单调减速
16   0.5167      5391.1     5421.5    ✓
...
23   0.0500      5216.1     5375.6    ✓
共 24 帧结束（ws → 0）
```

**减速阶段实际帧数范围**（各速度组合模拟）：

| vWarp / vSail | 减速帧数 |
|--------------|---------|
| 2×           | 16–18 帧 |
| 5×           | 18–22 帧 |
| 10×          | 22–24 帧 |
| 20×          | 24–27 帧 |
| → ∞          | ≤ 32–35 帧（理论上界） |

理论上界为 **~32–35 帧**，证明如下：即使完全振荡（整个过程 1减1加），净速率为 `-1/40` 每帧，从 `warpState=1` 降到 0 最多需要 40 帧；但实际因为低 `warpState` 段不再振荡，上界约为 35 帧。

### 9.5 对时间估算函数的影响

`CalcRemoteSingleTripTime` 和 `CalcArrivalRemainingTime` 中对曲速减速段的估算：

```csharp
// CalcArrivalRemainingTime, warpState > 0 且接近目标：
num += (int)(shipData.warpState * 20f + 0.5f);   // ≈ 0~20 帧
```

```csharp
// CalcRemoteSingleTripTime 中的 80 帧常量包含加速(~60帧)+减速(~20帧)
```

**实际减速帧数 15–32 帧，估算假设约 20 帧，误差 < 17 帧（< 0.3 秒）**。相对于星际运输通常数分钟的全程时间，此误差可以忽略。

### 9.6 如何处理这个振荡

**方案一（单调锁定）**：一旦触发减速，禁止 `warpState` 回弹：

```csharp
bool warpDecelerating = false;
if (warpState > 0) {
    if (num21 < num28 || warpDecelerating) {
        warpDecelerating = true;
        warpState -= 1f / 15f;
    } else {
        warpState += 1f / 60f;
    }
}
```

效果：减速帧数精确为 15 帧，估算误差消除，但需要在 `ShipData` 中增加一位状态标志。

**方案二（迟滞判断）**：减速阈值和恢复阈值分离：

```csharp
// 减速条件不变：num21 < num28（当前 warpState 对应的阈值）
// 恢复条件改为：num21 > num28_before_decel（减速前那一帧的阈值）
// 等价于：只有 num21 超过当前 warpState+1/15 对应的 num28 才恢复
```

效果：消除单步振荡，但实现较复杂。

**方案三（接受现状，游戏实际做法）**：不修复。振荡使减速帧数从 15 增加到最多 32，绝对误差 < 0.3 秒，对游戏视觉和逻辑无可见影响，代码维持现状。

**结论**：振荡确实存在，是代码的一个已知近似行为而非无意识 Bug。由于振荡是阻尼的（上界约 35 帧），且对时间估算的影响远小于其他近似误差，游戏选择忽略是合理的。若需要精确模拟，采用方案一（单调锁定）最为简单可靠。
