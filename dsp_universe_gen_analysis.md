# 戴森球计划 — 宇宙生成逻辑完整分析

> 基于 `Assembly-CSharp.dll` 反编译结果，入口方法 `UniverseGen.CreateGalaxy(GameDesc)`。
> 所有公式皆来自对反编译代码的逆向推导，保留了所有特殊分支和边界条件。

---

## 1. 总览流程

```
CreateGalaxy(gameDesc)
  1. 校验算法版本号 galaxyAlgo ∈ [20200101, 20591231]
  2. 稀有资源模式 → PlanetGen.gasCoef = 0.8，否则 1.0
  3. 创建 RNG = DotNet35Random(galaxySeed)
  4. GenerateTempPoses(RNG.Next(), starCount, 4, 2.0, 2.3, 3.5, 0.18)
     → 返回实际位置数（可能少于目标），starCount 被修正
  5. 创建 GalaxyData(seed, starCount)
  6. 生成 4 个全局随机数，用于确定各类型恒星占比
  7. 计算各类型恒星的数量分界线
  8. 循环生成每颗恒星 (i = 0 .. starCount-1)
     ├─ i=0   → CreateBirthStar
     ├─ i=3   → 强制目标光谱 M（如果仍是主序星）
     ├─ i=num11-1 → 强制目标光谱 O
     └─ 其余 → 根据编号在分界中的归属决定类型
  9. 初始化所有 AstroData（旋转四元数 w=1）
 10. 对每颗恒星调用 CreateStarPlanets 生成行星系统
 11. galaxyData.UpdatePoses(0.0) 计算行星初始轨道位置
 12. 搜索出生行星（恒星 0 中寻找 Theme.Distribute == Birth 的主题）
 13. CreateGalaxyStarGraph 构造星图连接和航线
 14. 重置 PlanetGen.gasCoef = 1.0，返回 galaxyData
```

**边界情况**：算法版本非法直接抛异常；如果位置生成导致 starCount <= 0，返回只有基础字段的空 GalaxyData。

### 核心数据结构

| 类型 | 作用 |
|------|------|
| `GalaxyData` | 全局数据：种子、恒星数组、星图节点、天体数据（最多 25700） |
| `StarData` | 恒星：质量、温度、光谱、光度、行星数组、安全系数、黑雾太空巢穴参数 |
| `PlanetData` | 行星：轨道参数、类型、主题、地貌算法、资源 |
| `AstroData` | 天体运行时数据，astroId = starId×100 + planetIndex |
| `StarGraphNode` | 星图节点：坐标、连接列表(conn)、航线列表(lines) |

---

## 2. 随机数引擎

所有宇宙生成使用 `DotNet35Random`，Knuth 减法随机数生成器：

- 种子数组长度 56，初始值用常数 161803398 和 |seed| 派生
- `Sample()` 推进 inext/inextp，两槽位之差乘 4.656612875245797×10⁻¹⁰ → [0,1)
- `Next()` 等价于 `Sample() × 2147483647`

整个生成过程是**确定性种子链**：`galaxySeed → 各子RNG`，相同输入必然产生完全相同的宇宙。

`RandNormal(mean, std, r1, r2)` 使用 Box-Muller 变换：
```
mean + std × sqrt(-2×ln(1-r1)) × sin(2π×r2)
```

---

## 3. 恒星位置生成

调用参数固定：
```
GenerateTempPoses(seed, targetCount, iterCount=4,
                  minDist=2.0, minStepLen=2.3,
                  maxStepLen=3.5, flatten=0.18)
```

### 3.1 过采样与抽稀

1. `RandomPoses` 先尝试生成 `targetCount × iterCount` 个候选点。
2. 从尾到头删除所有 `index % iterCount != 0` 的点。
3. 删除过程中如果剩余数量已经 <= targetCount，则停止。
4. 返回剩余点数，CreateGalaxy 用它覆盖 starCount。

`iterCount` 被限制在 [1,16]。

### 3.2 随机游走算法

初始包含原点 `(0,0,0)`。随后先生成 6~8 个“游走种子点”，再以这些点为活动点扩展。

随机方向：
```
x = rand[-1,1]
y = rand[-1,1] × flatten       // flatten=0.18，星系盘扁平化
z = rand[-1,1]
len² = x² + y² + z²
仅接受 1e-8 <= len² <= 1
step = (rand×(maxStepLen-minStepLen) + minDist) / sqrt(len²)
pos = base + dir × step
```

碰撞检测：若任意已有点到新点的平方距离 `< minDist²`，则拒绝。

初始阶段每个种子点最多尝试 256 次。扩展阶段最多 256 轮，每轮遍历活动点：70% 概率跳过，30% 概率尝试移动，每个活动点最多 256 次尝试。成功时更新活动点并加入新位置。

`flatten=0.18` 是银河系呈扁盘状的核心原因。

---

## 4. 恒星类型分配

位置生成后抽 4 个随机数 r1..r4：

```
黑洞数       bh = ceil(0.01  × starCount + r1 × 0.3)
中子星数     ns = ceil(0.01  × starCount + r2 × 0.3)
白矮星数     wd = ceil(0.016 × starCount + r3 × 0.4)
巨星密度因子 g  = ceil(0.013 × starCount + r4 × 1.4)

blackHoleStart  = starCount - bh
neutronStart    = blackHoleStart - ns
whiteDwarfStart = neutronStart - wd
giantStep       = (whiteDwarfStart - 1) / g
giantOffset     = giantStep / 2
```

逐星规则：

| 条件 | 类型 |
|---|---|
| i=0 | 出生星 |
| i >= blackHoleStart | 黑洞 |
| i >= neutronStart | 中子星 |
| i >= whiteDwarfStart | 白矮星 |
| i % giantStep == giantOffset | 巨星 |
| 其他 | 主序星 |

还有两个强制光谱点：

- i=3 → 目标光谱 M。
- i=whiteDwarfStart-1 → 目标光谱 O。

这些只影响光谱参数，不会覆盖黑洞/中子星/白矮星/巨星的类型判断。特殊类型优先级：黑洞 > 中子星 > 白矮星 > 巨星 > 主序星。

---

## 5. 出生星 CreateBirthStar

出生星固定在原点：index=0，id=1，position=(0,0,0)，resourceCoef=0.6，level=0。

质量：
```
x = RandNormal(0, 0.08)
x = clamp(x, -0.2, 0.2)
mass = 2^x
```
若 `StarGen.specifyBirthStarMass > 0.1`，覆盖质量；若 `specifyBirthStarAge > 1e-5`，覆盖年龄。

寿命与年龄：
```
d = 2.0 + 0.4×(1.0 - mass)
lifetime = 10000 × 0.1^(log10(mass×0.5)/log10(d) + 1) × random[0.9,1.1]
age = rand×0.4 + 0.3        // 默认 0.3~0.7
```

出生星安全参数：
```
hivePatternLevel = 0
safetyFactor = 0.847 + rand×0.026
```

出生星不会自然变为巨星或死亡恒星。其初始黑雾太空巢穴数量比普通星保守：若 `initialColonize < 0.015` 则为 0；否则以 `0.6 × initialColonize × maxHiveCount` 为均值做正态采样，并裁剪到合法范围。

---

## 6. 普通恒星 CreateStar

### 6.1 等级与资源系数

```
level = index / (starCount - 1)       // 单星系时为 0
dist = |position|
x = dist / 32
if x > 1: 连续 5 次 x = ln(x) + 1    // 五层对数压缩
resourceCoef = 7^x × 0.6
```

越远离中心，资源系数越高。五层对数压缩使极端距离不至于过高。

### 6.2 质量与光谱浮点

```
mean = lerp(-0.98, 0.88, level)
if mean < 0: mean -= 0.65     // 非对称调整
else:        mean += 0.65
stddev = 0.33

若 needtype==GiantStar:
    if randomOffset > -0.08: mean = -1.5   // 冷巨星
    else: mean = 1.6                         // 热巨星
    stddev = 0.3

spectrFloat = RandNormal(mean, stddev)
若 needSpectr==M: spectrFloat = -3
若 needSpectr==O: spectrFloat = 3
若 spectrFloat > 0: spectrFloat ×= 2      // 热端拉伸
spectrFloat = clamp(spectrFloat, -2.4, 4.65) + smallDrift + 1
```

质量公式：

| 类型 | 公式 |
|---|---|
| 主序星/巨星 | mass = 2^spectrFloat |
| 黑洞 | mass = 18 + r1×r2×30，范围 [18, 48] |
| 中子星 | mass = 7 + r1×11，范围 [7, 18] |
| 白矮星 | mass = 1 + r2×5，范围 [1, 6] |

### 6.3 寿命与年龄

```
base = 5.0
if mass < 2: base = 2.0 + 0.4×(1-mass)     // 低质量星寿命更长

lifetime = 10000 × 0.1^(log10(mass×coef)/log10(base) + 1) × ageFactor
// 巨星用 coef=0.58，其他用 coef=0.5
```

年龄分支：

| 条件 | 年龄 |
|---|---|
| 巨星 | rand×0.04 + 0.96 |
| 白矮星/中子星/黑洞 | rand×0.4 + 1.0 |
| 主序星 mass<0.5 | rand×0.12 + 0.02 |
| 主序星 mass<0.8 | rand×0.4 + 0.1 |
| 其他主序星 | rand×0.7 + 0.2 |

死亡恒星寿命修正：白矮星 +10000，中子星 +1000。

寿命上限压缩：
```
t = lifetime × age
if t > 5000: t = (ln(t/5000)+1) × 5000
if t > 8000: t = (ln(ln(ln(t/8000)+1)+1)+1) × 8000
lifetime = t / age
```

### 6.4 有效质量、温度、光谱

```
effectiveMass = (1 - clamp01(age)^20 × 0.5) × mass
// age^20 使接近 1 时有效质量急剧下降

temperature = effectiveMass^(0.56 + 0.14/(log10(effectiveMass+4)/log10(5))) × 4450 + 1300

classFactor = log10((temperature-1300)/4500) / log10(2.6) - 0.5
if classFactor < 0: classFactor ×= 4       // 冷端放大 4 倍
classFactor = clamp(classFactor, -4, 2)

spectr = round(classFactor + 4) → M(0) K(1) G(2) F(3) A(4) B(5) O(6) X(7)
color = clamp01((classFactor+3.5) × 0.2)
luminosity = effectiveMass^0.7
radius = mass^0.4 × 2^randomOffset[-0.2, 0.2]
```

### 6.5 宜居带与轨道缩放

```
p = classFactor + 2
habitableRadius = 1.7^p + 0.25×min(1, orbitScaler)
lightBalanceRadius = 1.7^p
orbitScaler = 1.35^p
if orbitScaler < 1: orbitScaler = lerp(orbitScaler, 1, 0.6)
```

> 注意：源码中 habitableRadius 使用的是尚未更新的 orbitScaler 值。

戴森球半径：
```
dysonRadius = orbitScaler × 0.28
if dysonRadius × 40000 < physicsRadius × 1.5:
    dysonRadius = physicsRadius × 1.5 / 40000
```

其中 physicsRadius = radius × 1200，uPosition = position × 2400000。

---

## 7. 恒星演化 SetStarAge

在 CreateStar 物理量计算之后调用，根据 age 值修改恒星参数。

### 7.1 age ≥ 1.0 — 死亡恒星

**黑洞 (mass ≥ 18 时演化前)**
```
type=BlackHole, spectr=X
mass ×= 2.5 × random[0.8, 1.2]        // 吸积盘增长
radius 不变
acdiskRadius = radius × 5              // 吸积盘
temperature = 0
luminosity ×= 0.001 × random[0.95, 1.05]
habitableRadius = 0
lightBalanceRadius ×= 0.4 × random[0.95, 1.05]
color = 1
```

**中子星 (7 ≤ mass < 18)**
```
type=NeutronStar, spectr=X
mass ×= 0.2 × random[0.95, 1.05]
radius ×= 0.15                        // 极致密
acdiskRadius = radius × 9
temperature = random[1,10] × 10,000,000 K
luminosity ×= 0.1 × random[0.95, 1.05]
habitableRadius = 0
lightBalanceRadius ×= 3 × random[0.95, 1.05]
orbitScaler ×= 1.5 × random[0.95, 1.05]
color = 1
```

**白矮星 (mass < 7)**
```
type=WhiteDwarf, spectr=X
mass ×= 0.2 × random[0.95, 1.05]
radius ×= 0.2
acdiskRadius = 0
temperature = random[0.8,1.2] × 150,000 K
luminosity ×= 0.04 × random[0.8,1.2]
habitableRadius ×= 0.15 × random[0.8,1.2]
lightBalanceRadius ×= 0.2 × random[0.95, 1.05]
color = 0.7
```

### 7.2 0.96 ≤ age < 1.0 — 巨星

先用主序星物理量计算，再在此处改写：

```
giantRadius = 5^|log10(mass)-0.7| × 5 × randomFactor
if giantRadius > 10: giantRadius = (ln(giantRadius×0.1)+1) × 10  // 对数压缩
survivalFactor = 1 - age^30 × 0.5

type = GiantStar
mass = survivalFactor × mass            // 质量流失
radius = giantRadius × random[0.8,1.2]  // 膨胀几十倍
acdiskRadius = 0
temperature = survivalFactor × temperature
luminosity = 1.6 × luminosity
habitableRadius = 9 × habitableRadius
lightBalanceRadius = 3 × habitableRadius
orbitScaler = 3.3 × orbitScaler
```

---

## 8. 行星系统生成 CreateStarPlanets

### 8.1 特殊恒星的行星数量

| 恒星类型 | 行量 | 规则 |
|---|---|---|
| 黑洞 | 1 | 固定为最近轨道（orbitIndex=3），且为陆地行星 |
| 中子星 | 1 | 同上 |
| 白矮星 | 1~2 | 70% 概率 1 颗；30% 概率 2 颗（30% 为气态+卫星） |
| 巨星 | 1~3 | 30%→1颗 50%→2颗 20%→3颗，类型和轨道由随机确定 |

### 8.2 主序星行量数量（按光谱）

| 光谱 | 行量 | 概率分布 | pGas 数组（索引0..5） |
|---|---|---|---|
| **出生星** | 4 | 固定 | pGas[0-2]=0（前3颗都不是气态） |
| **M** | 1~4 | 10%/20%/50%/20% | ≤3 颗: pGas[0,1]=0.2; 4 颗: pGas[1]=0.2, pGas[2]=0.3 |
| **K** | 1~5 | 10%/10%/50%/25%/5% | ≤3: pGas[0,1]=0.18; 4+: pGas[1]=0.18, pGas[2,3]=0.28 |
| **G** | 3~5 | 40%/50%/10% | ≤3: pGas[0,1]=0.18; 4+: pGas[1]=0.2, pGas[2,3]=0.3 |
| **F** | 3~5 | 35%/45%/20% | ≤3: pGas[0,1]=0.2; 4+: pGas[1]=0.22, pGas[2,3]=0.31 |
| **A** | 3~5 | 30%/45%/25% | ≤3: pGas[0,1]=0.2; 4+: pGas[0]=0.1, pGas[1]=0.28, pGas[2]=0.3, pGas[3]=0.35 |
| **B** | 4~6 | 30%/45%/25% | ≤3: pGas[0,1]=0.2; 4+: pGas[0]=0.1, pGas[1]=0.22, pGas[2]=0.28, pGas[3]=0.35, pGas[4]=0.35 |
| **O** | 5~6 | 50%/50% | pGas[0-5]=0.1, 0.2, 0.25, 0.3, 0.32, 0.35 |
| **其他** | 1 | 固定 | — |

### 8.3 行星轨道分配算法

逐颗行星分配轨道：
```
num14 = 0  // 绕主星轨道序号
num15 = 0  // 子卫星序号
num17 = 1  // 当前轨道索引（StarGen.orbitRadius 数组索引）

对每颗行星 i:
  if num16 == 0 (绕主星):
    num14++
    如果 i < 行星数-1 且 random < pGas[i]，该行星为气态巨星
    调整轨道索引 num17 确保剩余轨道足够放完所有行星：
      剩余行量 = 行量总数 - i
      剩余轨道 = 9 - num17
      如果剩余轨道 > 剩余行量:
        尝试增加 num17（用随机数加权）
    num17++

  else (绕气态巨星运行的卫星):
    num15++
    该行星非气态
    如果 num15 >= 1 且 random < 0.8: num16=0（恢复到绕主星）
```

### 8.4 小行星带

在行星分配完成后计算两个小行星带轨道索引：

**第一小行星带**：在第一个气态巨星轨道之内的缺失轨道中（rand < 0.2+轨道距离×0.2）。

**第二小行星带**：基于最后一个主轨道索引：
```
if rand < 0.2: index = lastOrbit + 3
elif rand < 0.4: index = lastOrbit + 2
elif rand < 0.8: index = lastOrbit + 1
else: 无小行星带
if index < 5: index = 5     // 至少轨道 5
```

小行星带半径 = orbitRadius[轨道索引] × randomFactor × star.orbitScaler

---

## 9. 单颗行星生成 PlanetGen.CreatePlanet

### 9.1 轨道半径

**绕主星**：
```
perturbation = 1.2^(random1 × (random2 - 0.5) × 0.5)
orbitRadius = StarGen.orbitRadius[orbitIndex] × star.orbitScaler
orbitRadius ×= (perturbation - 1) / max(1, orbitRadius) + 1   // 远轨微扰
```

**绕气态巨星**：
```
orbitRadius = ((1600×orbitIndex + 200) × star.orbitScaler^0.3 × lerp(perturbation,1,0.5)
              + parentPlanet.realRadius) / 40000.0
```

### 9.2 轨道倾角与升交点经度

```
orbitInclination = rand × 16 - 8  (度)
if 绕卫星: orbitInclination ×= 2.2         // 卫星轨道倾角更大
if 中子星/黑洞: orbitInclination ± 3        // 略微增大

orbitLongitude = rand × 360
```

### 9.3 轨道周期（开普勒第三定律）

**绕主星**：
```
T = sqrt(4π² × r³ / (G × M))
  = sqrt(39.4784 × r³ / (1.353855e-6 × mass))
```

**绕气态巨星**（引力常数不同）：
```
T = sqrt(39.4784 × r³ / 1.083084e-8)     // 卫星运动更快
```

### 9.4 自转轴倾斜（倾斜角）

```
if rand < 0.04:
    obliquity = rand × (rand-0.5) × 39.9
    if obliquity < 0: obliquity -= 70     // 横躺自转
    else: obliquity += 70
    singularity |= LaySide

elif rand < 0.10:
    obliquity = rand × (rand-0.5) × 80
    if obliquity < 0: obliquity -= 30
    else: obliquity += 30

else:
    obliquity = rand × (rand-0.5) × 60    // 普通倾斜
```

### 9.5 自转周期

```
rotationPeriod = (rand1 × rand2 × 1000 + 400) × orbitRadius^0.25 × (gasGiant?0.2:1)

// 死亡恒星周围行星自转加速
if !gasGiant:
    if 白矮星: rotationPeriod ×= 0.5
    if 中子星: rotationPeriod ×= 0.2
    if 黑洞:   rotationPeriod ×= 0.15

// 潮汐锁定修正
rotationPeriod = 1 / (1/orbitalPeriod + 1/rotationPeriod)
```

### 9.6 潮汐锁定与特殊自转

仅对**近轨道**（orbitIndex ≤ 4）、**非气态**行星：

| 条件 | 效果 |
|---|---|
| rand > 0.96 | 潮汐锁定：obliquity×0.01，rotation=orbital，TidalLocked |
| 0.93 < rand ≤ 0.96 | 轨道共振1:2：obliquity×0.1，rotation=orbital×0.5，TidalLocked2 |
| 0.90 < rand ≤ 0.93 | 轨道共振1:4：obliquity×0.2，rotation=orbital×0.25，TidalLocked4 |

对所有行星，如果 0.85 < rand ≤ 0.90：反向自转 ClockwiseRotate。

### 9.7 行星类型判定

**气态/冰巨星**：
```
type = Gas, radius = 80, scale = 10, habitableBias = 100
```

**陆地行星** — 基于 `ratio = sunDistance / habitableRadius`：

首先计算宜居概率：
```
habitableCountTarget = ceil(starCount × 0.29)，至少 11
remaining = habitableCountTarget - galaxy.habitableCount
remainingStars = starCount - star.index
adjustRatio = clamp(lerp(remaining/remainingStars, 0.35, 0.5), 0.08, 0.8)

habitableBias = |ln(ratio)| × (sqrt(habitableRadius) - 0.04)
habitableChance = (habitableBias / adjustRatio) ^ (adjustRatio × 10)
```

类型判定流程：

```
if (random > habitableChance 且 star.index > 0) 或
   (是出生星的第一颗卫星 orbitIndex==1):
    → Ocean (海洋行星，galaxy.habitableCount++)

elif ratio < 0.833:     // 太近
    if random < max(0.15, ratio×2.5-0.85):  → Desert
    else:                                    → 熔岩(Vocano)

elif ratio < 1.2:       // 中等距离
    → Desert

else:                   // 太远
    if random < 0.9/ratio - 0.1:  → Desert
    else:                          → Ice
```

> **出生星特殊**：出生星的第 1 颗卫星（orbitAround>0 且 orbitIndex==1）强制为海洋行星。

### 9.8 行星精度

```
if 陆地行星: precision=200, segment=5
if 气态/None: precision=64, segment=2
```

### 9.9 行星光度

```
luminosity = (star.lightBalanceRadius / (sunDistance+0.01))^0.6
if luminosity > 1:
    luminosity = ln(ln(ln(luminosity)+1)+1)+1    // 三重对数压缩
luminosity = round(luminosity × 100) / 100
```

---

## 10. 行星主题与资源

### 10.1 主题筛选

三级回退策略：

**第一轮**：匹配行星类型 + 温度偏移 + 分配规则
```
对每个 themeId:
  if 出生星海洋行星:
    仅匹配 Distribute==Birth
  else:
    要求 themeProto.PlanetType == planet.type
    要求 themeProto.Temperature × temperatureBias >= -0.1
    // Desert 类型在 |Temperature|<0.5 时额外限制温度偏移
    if 出生星: Distribute==Default
    else: Distribute==Default 或 Interstellar
  同一颗星内不能有重复主题
```

**第二轮**（首轮无结果）：仅匹配 Desert 类型 + 无重复。

**第三轮**（仍无结果）：匹配所有 Desert 类型（允许重复）。

### 10.2 主题确定后

```
algoId = 从 themeProto.Algos 中随机选择
mod_x = themeProto.ModX.x + rand × (ModX.y - ModX.x)
mod_y = themeProto.ModY.x + rand × (ModY.y - ModY.x)
style = theme_seed % 60
```

同时从主题读取：ionHeight, windStrength, waterHeight, waterItemId, iceFlag, levelized 等。

### 10.3 气态/冰巨星资源

```
对每种气体:
  speed = themeProto.GasSpeeds[i] × rand[0.909, 1.0] × gasCoef × star.resourceCoef^0.3
  heatValue = LDB.items.Select(gasItem).HeatValue
gasTotalHeat = Σ(heatValue × speed)
```

### 10.4 行星主题一览表（ThemeProto）

游戏通过 `ThemeProtoSet` 定义所有行星主题。主题的 `Distribute` 字段决定了该主题在哪些恒星系中出现：

| Distribute 值 | 含义 | 说明 |
|---|---|---|
| 0 Default | 默认 | 任何恒星系都可出现 |
| 1 Birth | 初始星球 | 仅出生星的海洋行星 |
| 2 Interstellar | 星际 | 仅非出生星的恒星系 |
| 3 Rare | 稀有 | 特殊条件触发 |

**完整主题表**：

| ID | 中文名 | 程序Name | PlanetType | Distribute | Temperature |
|---:|---|---|---|---|---:|
| 1 | 地中海 | Ocean 1 | 海洋(2) | Birth 初始星球 | 0.0 |
| 2 | 气态巨星 | Gas 1 | 巨星(5) | Default | 2.0 |
| 3 | 气态巨星 | Gas 2 | 巨星(5) | Default | 1.0 |
| 4 | 冰巨星 | Gas 3 | 巨星(5) | Default | -1.0 |
| 5 | 冰巨星 | Gas 4 | 巨星(5) | Default | -2.0 |
| 6 | 干旱荒漠 | Desert 1 | 荒漠(3) | Default | 2.0 |
| 7 | 灰烬冻土 | Desert 2 | 荒漠(3) | Default | -1.0 |
| 8 | 海洋丛林 | Ocean 2 | 海洋(2) | Interstellar | 0.0 |
| 9 | 熔岩 | Lava 1 | 熔岩(1) | Default | 5.0 |
| 10 | 冰原冻土 | Ice 1 | 冰原(4) | Default | -5.0 |
| 11 | 贫瘠荒漠 | Desert 3 | 荒漠(3) | Default | -2.0 |
| 12 | 戈壁 | Desert 4 | 荒漠(3) | Default | 1.0 |
| 13 | 火山灰 | Volcanic 1 | 熔岩(1) | Interstellar | 4.0 |
| 14 | 红石 | Ocean 3 | 海洋(2) | Interstellar | 0.0 |
| 15 | 草原 | Ocean 4 | 海洋(2) | Interstellar | 0.0 |
| 16 | 水世界 | Ocean 5 | 海洋(2) | Interstellar | 0.0 |
| 17 | 黑石盐滩 | Desert 5 | 荒漠(3) | Default | 1.0 |
| 18 | 樱林海 | Ocean 6 | 海洋(2) | Interstellar | 0.0 |
| 19 | 飓风石林 | Desert 6 | 荒漠(3) | Interstellar | 1.0 |
| 20 | 猩红冰湖 | Desert 7 | 荒漠(3) | Default | -2.0 |
| 21 | 气态巨星 | Gas 5 | 巨星(5) | Interstellar | 1.0 |
| 22 | 热带草原 | Desert 8 | 海洋(2) | Interstellar | 0.0 |
| 23 | 橙晶荒漠 | Desert 9 | 荒漠(3) | Interstellar | 0.08 |
| 24 | 极寒冻土 | Desert 10 | 冰原(4) | Default | -4.0 |
| 25 | 潘多拉沼泽 | Desert 11 | 海洋(2) | Interstellar | 0.0 |

> **注意**：代码中 `EPlanetType` 的 1 号类型拼写为 `Vocano`（缺少字母 l），对应 ThemeProto 的 `Lava 1` 和 `Volcanic 1`。
> Gas 类主题的 `PlanetType` 均为 5（Gas），但中文名区分为`气态巨星`（高温）和`冰巨星`（低温）。
> Desert 2 `灰烬冻土` 和 Desert 10 `极寒冻土` 的 PlanetType 是荒漠(3) 而非冰原(4)，这是因为代码中的行星类型判定逻辑基于 `sunDistance/habitableRadius` 比值，而非直接从主题读取。

### 10.5 矿脉原型（VeinProto）

| ID | 中文名 | SID | 采集物 ItemID |
|---:|---|---|---:|
| 1 | 铁矿脉 | 铁 | 1001 |
| 2 | 铜矿脉 | 铜 | 1002 |
| 3 | 硅矿脉 | 硅 | 1003 |
| 4 | 钛矿脉 | 钛 | 1004 |
| 5 | 石矿脉 | 石 | 1005 |
| 6 | 煤矿脉 | 煤 | 1006 |
| 7 | 原油涌泉 | 油 | 1007 |
| 8 | 可燃冰矿 | 燃 | 1011 |
| 9 | 金伯利矿 | 金 | 1012 |
| 10 | 分形硅矿 | 分 | 1013 |
| 11 | 有机晶体矿 | 胶 | 1117 |
| 12 | 光栅石矿 | 光 | 1014 |
| 13 | 刺笋矿脉 | 笋 | 1015 |
| 14 | 单极磁矿 | 磁 | 1016 |

> 矿脉 ID 1~7 为普通矿脉，8~14 为稀有矿脉。稀有矿脉的生成概率受 `resourceMultiplier`（资源倍率）影响。

---

## 11. 星图航线图 CreateGalaxyStarGraph

### 11.1 星图节点连接

逐颗添加星图节点，每添加一颗新节点时检查与所有已有节点的距离：

```
如果 distance² < 64 (即距离 < 8 光年):
    双向加入 conn 列表（按 index 有序插入）
```

### 11.2 航线筛选

对每个节点的每对连接，判断是否应该建立航线。核心算法是对每对连接 (A, B) 加上新节点 N 构成三角形，检查三角形的边是否满足三角不等式：

```
对节点 N 的每对连接 A, B:
  如果 A 和 B 也互连（形成三角形）:
    a = |N-A|², b = |N-B|², c = |A-B|²
    mid = (a+b+c - max - min) × 1.001     // 中间边（容忍 0.1% 误差）
    min_edge × 1.01                        // 最短边（容忍 1% 误差）

    // 判断 N-A 边
    if a ≤ mid 或 a ≤ min_edge: 保留航线 N-A
    else: 删除航线 N-A

    // 判断 N-B 边
    if b ≤ mid 或 b ≤ min_edge: 保留航线 N-B
    else: 删除航线 N-B

    // 判断 A-B 边
    if c > mid 且 c > min_edge: 删除航线 A-B

  如果 A, B 不互连（不是三角形）:
    默认保留 N-A 航线
```

> 航线筛选的目的是去除三角形中的最长边（类似 Delaunay 三角化），保留最短路径。容忍系数（1.001 和 1.01）防止近似等边情况下过度删除。

---

## 12. 黑雾太空巢穴轨道与战斗参数

### 12.1 黑雾太空巢穴安全系数 safetyFactor

```
colorFactor = pow(star.color, 1.3)
distFactor = clamp((dist-2)/20, 0, 2.5)
if distFactor > 1:
    distFactor = ln(ln(distFactor)+1)+1
distFactor /= 1.4

// 特殊类型覆盖
黑洞:   colorFactor = 5
中子星: colorFactor = 1.7
白矮星: colorFactor = 1.2
巨星:   colorFactor = max(0.6, colorFactor)
O型星:  colorFactor += 0.05

adjusted = colorFactor × 0.9 + 0.07
safetyFactor = clamp01(1 - adjusted^0.73 × distFactor^0.27 + rand×0.08 - 0.04)
```

### 12.2 黑雾太空巢穴模式等级

```
if safetyFactor >= 0.7: hivePatternLevel = 0   // 安全
elif safetyFactor >= 0.3: hivePatternLevel = 1  // 中等
else: hivePatternLevel = 2                       // 危险
```

### 12.3 黑雾太空巢穴最大数量

```
baseCount = combatSettings.maxDensity × (epicHive?2:1) × 1000 + rand[0,1000)
maxHiveCount = baseCount / 1000
```
epicHive 在中子星和黑洞时为 true。

### 12.4 黑雾太空巢穴初始数量

当 `initialColonize < 0.015` 时为 0。否则：

```
safetyMod = clamp01(safetyFactor × 1.05 - 0.15)^0.82
base = clamp01(1 - safetyMod - (maxHiveCount-1)×0.05) × (1.1 - maxHiveCount×0.1)

// initialColonize 缩放
if initialColonize ≤ 1: base ×= initialColonize
else: base = lerp(base, 1+(initialColonize-1)×0.2, (initialColonize-1)×0.5)

// 特殊类型加成
巨星: ×1.2  白矮星: ×1.4  中子星: ×1.6  黑洞: ×1.8  O型星: ×1.1

mean = min(base × maxHiveCount, maxHiveCount + 0.75)
stddev = 根据mean大小自适应 (sqrt(mean)×0.29+0.21 或 0.3+0.2×mean)
initialHiveCount = RandNormal(mean, stddev) + 0.5 取整
重试最多64次确保在 [0, maxHiveCount] 范围内
```

**黑洞特殊保底**：如果初始数量 < maxDensity 对应的基础数量，则补齐到基础数量，且至少为 1。

### 12.5 黑雾太空巢穴轨道分配

每颗恒星有 8 个黑雾太空巢穴轨道槽位。首先标记被行星占用的轨道为不可用，然后从可用轨道中按权重随机选择：

```
可用轨道数 = hiveOrbitCondition 中仍为 true 的数量
选择上限 = min(可用数, star.index==0 ? 2 : 5)   // 出生星最多2个

对每个轨道槽位:
    选择半径时使用 "平方选择法"：index = rand²，偏好较前（近轨）的轨道
    orbitRadius = hiveOrbitRadius[selectedIndex] × star.orbitScaler
    orbitalPeriod = sqrt(4π² × r³ / (G_eff × mass))   // 不同版本 G_eff 不同
```

行星占用轨道的标记规则 `SetHiveOrbitConditionFalse`：
```
主轨道索引 → planet2HiveOrbitTable[orbitIndex] → 标记对应 hive 轨道为 false
如果是子卫星: 额外检查相邻 hive 轨道是否距离过近，也标记为 false
```

---

## 13. 恒星命名规则

### 13.1 出生星

使用 `NameGen.RandomName(seed)` 生成一个随机名字（音节组合），然后再用 `RandomStarName` 覆盖。

### 13.2 普通恒星

```
if 巨星:
    60%: RandomGiantStarNameFromRawNames   // 从巨星专用名称表
    30%: RandomGiantStarNameWithConstellationAlpha  // 如 "15B Andromedae"
    10%: RandomGiantStarNameWithFormat     // 格式化名称如 "RX J1234-56"
elif 中子星:
    RandomNeutronStarNameWithFormat        // 如 "PSR J1234+5678"
elif 黑洞:
    RandomBlackHoleNameWithFormat          // 如 "Sgr A* J1234+5678"
else 主序星:
    60%: RandomStarNameFromRawNames        // 从 425 个真实恒星名
    33%: RandomStarNameWithConstellationAlpha // 如 "Alpha Centauri"
    7%:  RandomStarNameWithConstellationNumber // 如 "42 Orionis"
```

---

## 14. 恒星轨道参考表

`StarGen.orbitRadius[]`（主星轨道半径基准，单位 AU）：

| 索引 | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 | 13 | 14 | 15 | 16 |
|------|---|---|---|---|---|---|---|---|---|---|----|----|----|----|----|----|-----|
| 半径 | 0 | 0.4 | 0.7 | 1.0 | 1.4 | 1.9 | 2.5 | 3.3 | 4.3 | 5.5 | 6.9 | 8.4 | 10 | 11.7 | 13.5 | 15.4 | 17.5 |

`StarGen.hiveOrbitRadius[]`（黑雾太空巢穴轨道半径基准）：

| 索引 | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 | 13 | 14 | 15 | 16 | 17 |
|------|---|---|---|---|---|---|---|---|---|---|----|----|----|----|----|----|----|-----|
| 半径 | 0.4 | 0.55 | 0.7 | 0.83 | 1.0 | 1.2 | 1.4 | 1.58 | 1.72 | 1.9 | 2.11 | 2.29 | 2.5 | 2.78 | 3.02 | 3.3 | 3.6 | 3.9 |

`planet2HiveOrbitTable[]`（行星轨道 → 黑雾太空巢穴轨道映射）：

| 行星轨道索引 | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
|-------------|---|---|---|---|---|---|---|---|
| 黑雾太空巢穴轨道索引 | 0 | 0 | 2 | 4 | 6 | 9 | 12 | 15 |

---

## 15. 关键公式速查表

| 物理量 | 公式 |
|--------|------|
| 资源系数 | `7^(5层ln压缩距离/32) × 0.6` |
| 主序星质量 | `2^(正态分布采样 + 非对称加权 + drift + 1)` |
| 恒星寿命 | `10000 × 0.1^(log10(mass×coef)/log10(base)+1) × factor` |
| 有效质量 | `(1 - age^20 × 0.5) × mass` |
| 温度 | `effectiveMass^(0.56+0.14/(log10(eff+4)/log10(5))) × 4450 + 1300` |
| 光谱 | `round(log10((T-1300)/4500)/log10(2.6) - 0.5 + 4)` |
| 光度 | `effectiveMass^0.7` |
| 半径 | `mass^0.4 × 2^random` |
| 宜居带 | `1.7^(classFactor+2) + 0.25×min(1,orbitScaler)` |
| 轨道缩放 | `1.35^(classFactor+2)` |
| 轨道周期 | `sqrt(4π²×r³ / (G×M))` |
| 自转周期 | `(rand×1000+400) × orbitRadius^0.25` |
| 宜居概率 | `(habitableBias / adjustRatio)^(adjustRatio×10)` |

---

## 16. 枚举值参考

### EStarType

| 值 | 游戏内中文名 | 说明 |
|----|-------------|------|
| 0 | `{光谱}型恒星`（如 G型恒星、M型恒星） | 主序星 |
| 1 | `巨星`（红巨星/黄巨星/白巨星/蓝巨星） | 按光谱分 |
| 2 | `白矮星` | — |
| 3 | `中子星` | — |
| 4 | `黑洞` | — |

### ESpectrType

| 值 | 含义 |
|----|------|
| 0 | M 型（红矮星） |
| 1 | K 型（橙矮星） |
| 2 | G 型（黄矮星/类太阳） |
| 3 | F 型（黄白色） |
| 4 | A 型（白色） |
| 5 | B 型（蓝白色） |
| 6 | O 型（蓝色） |
| 7 | X（非主序星） |

### EPlanetType（行星类型）

| 值 | 游戏内中文名 | 说明 |
|----|-------------|------|
| 0 | — | None（无类型） |
| 1 | `熔岩` | Vocano（**注意代码拼写为 Vocano 而非 Volcano**） |
| 2 | `海洋` | Ocean（宜居类，如地中海、海洋丛林、草原等） |
| 3 | `荒漠` | Desert（如干旱荒漠、戈壁、灰烬冻土等） |
| 4 | `冰原` | Ice（如冰原冻土、极寒冻土等） |
| 5 | `巨星` | Gas（气态巨星/冰巨星，游戏中统一归类为"巨星"） |

> **注**：代码中 `EPlanetType` 将 Gas 归类为"巨星"（巨行星），包括气态巨星和冰巨星。
> 代码里拼写有误：1 号类型拼写为 `Vocano`（缺少 l），但官方翻译为`熔岩`。

### EPlanetSingularity（行星奇异性，Flags）

| 值 | 游戏内中文名 | 说明 |
|----|-------------|------|
| 1 | `潮汐锁定 永昼永夜` | TidalLocked，自转周期=公转周期 |
| 2 | `轨道共振1:2` | TidalLocked2，自转周期=公转周期×0.5 |
| 4 | `轨道共振1:4` | TidalLocked4，自转周期=公转周期×0.25 |
| 8 | `横躺自转` | LaySide，自转轴倾斜超过 70° |
| 16 | `反向自转` | ClockwiseRotate，自转方向与公转方向相反 |
| 32 | `多卫星` | MultipleSatellites，气态巨星有多颗卫星 |

---

## 17. 常数参考

| 常数 | 值 | 说明 |
|------|-----|------|
| GRAVITY | 1.353855e-6 | 引力常数（游戏单位） |
| AU | 40000 | 1 天文单位 = 40000 游戏单位 |
| LY | 2400000 | 1 光年 = 2400000 游戏单位 |
| MAX_ASTRO_COUNT | 25700 | 最大天体数量 |
| kPhysicsRadiusRatio | 1200 | 恒星物理半径比 |
| kViewRadiusRatio | 800 | 恒星可视半径比 |
| algoVersion | 20200403 | 算法版本号 |
