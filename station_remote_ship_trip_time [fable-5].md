# 《戴森球计划》星际物流运输船运动逻辑分析

> 分析对象: `DSPGAME_Data/Managed/Assembly-CSharp.dll` (2026-05 版本, 7.8 MB)
> 反编译工具: ILSpy (`ilspycmd` / ICSharpCode.Decompiler 10.1.0.8386)
> 分析方法: 反编译 `StationComponent.InternalTickRemote`(本文行号引用基于反编译输出 `StationComponent.cs`,该方法位于 2024–2950 行),并用 Python 逐 tick 复刻其动力学作为 ground truth 验证闭式解。
> 术语对照: 全部采用 Locale `2052`(简体中文)官方词条 —— 星际物流运输船(item 5002)、星际物流运输站(item 2104)、空间翘曲器(item 1210)、运输船引擎(科技 3403–3407)、曲速启用路程(`warpEnableDist`)、翘曲器必备(`warperNecessary`)。
>
> **注意**: 按要求,本文主体分析阶段未读取 `CalcRemoteSingleTripTime` 与 `CalcArrivalRemainingTime`;最后一章为分析完成后的对照补充。

---

## 1. 总体框架

`InternalTickRemote` 由 `PlanetTransport.GameTick` 每逻辑帧(60 tick/s)调用一次,对该站全部在途飞船(`workShipDatas[0..workShipCount)`)逐一推进。传入的速度参数直接来自 GameHistory:

```csharp
// PlanetTransport.GameTick (反编译)
float logisticShipSailSpeedModified = history.logisticShipSailSpeedModified;
float shipWarpSpeed = history.logisticShipWarpDrive
        ? history.logisticShipWarpSpeedModified : logisticShipSailSpeedModified;
stationComponent.InternalTickRemote(factory, num2, logisticShipSailSpeedModified,
        shipWarpSpeed, logisticShipCarries, ...);
```

GameHistory 侧(均为科研可升级量):

| 字段 | 含义 | 值 |
|---|---|---|
| `logisticShipSailSpeed` | 航行速度基础值 | **600 m/s**(从 `resources.assets` 中 `ModeConfig` 实例提取) |
| `logisticShipWarpSpeed` | 曲速速度基础值 | **120 000 m/s = 3 AU/s**(1 AU = 40 000 m) |
| `logisticShipSpeedScale` | 速度倍率 | 初始 1.0;科技解锁函数 16 每级 `+0.5` |
| `logisticShipWarpDrive` | 曲速解锁 | 科技解锁函数 17 置 true |
| `logisticShipSailSpeedModified` | **传入的 Vs** | `600 × scale` |
| `logisticShipWarpSpeedModified` | **传入的 Vw** | `120000 × scale` |

对应科技为「运输船引擎」: Lv3(3403)起每级 `+0.5` 倍率,**Lv4(3404)解锁曲速航行**,Lv7(3407)起无限研究(每级仍 `+0.5`)。即 `scale = 1 + 0.5 × max(0, 运输船引擎等级 − 2)`。

方法开头为整帧公共量(行号为反编译文件内行号):

```csharp
bool flag = shipWarpSpeed > shipSailSpeed + 1f;     // 曲速可用(已解锁)
float num8 = shipSailSpeed / 600f;                  // 速度比 r8(基准恰为基础值600;后续上限500)
float num9 = Mathf.Pow(num8, 0.4f);                 // r9 = r8^0.4
float num10 = num9 > 1 ? Mathf.Log(num9) + 1 : num9;// r10(对数压缩)
float num11 = shipSailSpeed * 0.03f;                // 近行星加速度 [m/s²]
float num12 = shipSailSpeed * 0.12f * num10;        // 太空加速度  [m/s²]
float num13 = shipSailSpeed * 0.4f  * num8;         // 减速步长(每转向tick,准瞬时)
float num14 = num9 * 0.006f + 1E-05f;               // 起降动画速率 [1/tick]
```

每艘飞船的状态机由 `ShipData.stage ∈ {-2,-1,0,1,2}` 与 `direction ∈ {+1(去程), -1(回程)}` 驱动。另外每 tick 开头还做翘曲器自动装填(从站内库存把 item 1210 计入 `warperCount`)。

## 2. 飞行阶段与运动类型

一次完整往返按时间顺序经历(`DispatchSupplyShip` 把新船初始化为 `stage=-2, direction=1, t=0`):

| # | stage | direction | 阶段名称 | 运动类型 | 时长(tick) | 闭式解 |
|---|---|---|---|---|---|---|
| 1 | −2 | +1 | 本站停靠位待发 | 静止于停机坪(随行星转动),计时器 `t += 0.03335` | **30** | 常数 |
| 2 | −1 | +1 | 从本站起飞 | 沿停机坪法向升至 +25 m,smoothstep `(3−2t)t²` 插值,`uSpeed=0` | **⌈1/num14⌉ ≈ 167/r9** | 常数(只依赖 Vs) |
| 3 | 0 | +1 | **航行 A→B** | 真实运动学(下节) | 距离相关 | **是(分段)** |
| 4 | 1 | +1 | 降落到对方站 | 对接位插值(smoothstep),`t −= num14·⅔` | **≈ 250/r9** | 常数 |
| 5 | 2 | +1 | 对方站停靠(卸/装货) | 静止,`t −= 0.0334`;结束时取货、必要时从对方站补 1 个空间翘曲器,`direction=−1` | **30** | 常数 |
| 6 | 2 | −1 | 对方站待发 | 静止,`t += 0.0334` | **30** | 常数 |
| 7 | 1 | −1 | 从对方站起飞 | 对接位插值,`t += num14` | **≈ 167/r9** | 常数 |
| 8 | 0 | −1 | **航行 B→A** | 同 3,目标换成本站自己的停机坪点 | 距离相关 | **是(分段)** |
| 9 | −1 | −1 | 降落回本站 | 停机坪插值,`t −= num14·⅔` | **≈ 250/r9** | 常数 |
| 10 | −2 | −1 | 本站卸货入库 | 静止,`t −= 0.03335`;结束后转入闲置 | **30** | 常数 |

起降阶段速率 `num14 = 0.006·r9 + 10⁻⁵`,起飞 `1/num14` tick、降落 `1.5/num14` tick(降落速率是起飞的 2/3)。这些阶段全部是**纯动画插值**(位置由 smoothstep 给出、`uSpeed` 强制为 0),时长只依赖 Vs,**严格闭式**。

### 2.1 stage 0(航行段)的运动学

每 tick 结构(伪代码,行号见 `InternalTickRemote` 内):

```
d  = |到达点 − uPos|            // num21,到达点 = 对方dock位(去程)/本站停机坪位(回程) + 径向25m
dep²= |出发行星中心 − uPos|²     // num22(去程为A星球、回程为B星球)
if d < 6·r10: stage ← direction  // 到达判定(先于一切运动更新)
〈曲速块,每 tick 执行〉
〈转向/速度块,每 num23 tick 执行一次〉   // num23∈{1,10,30},远距降频(性能优化)
uPos += uVel · uSpeed/60 + 行星跟随位移   // 每 tick 积分;uVel 是单位朝向矢量
```

`num23` 降频条件: 距出发行星 > 1000 km 且 > 70 s 航程,且距目标 > 10⁶ m(+曲速余量);只影响转向/速度刷新频率,不影响位置积分,对总时间影响 < ±0.5 s。

**(a) 普通航行(非曲速)的速度控制** —— 核心是一个"目标速度包络":

```csharp
num30 = d / (uSpeed + 0.1) * 0.382;                    // ≈0.382×剩余时间估计
num32 = min(uSpeed·num30·r10 + 6·r9 + 0.15·r8, Vs);    // 目标速度
num33 = (近行星 ? 0.03·Vs : 0.12·Vs·r10) / 60;          // 每tick加速量
if (uSpeed < num32 − num33)      uSpeed += num33;       // 线性加速
else if (uSpeed > num32 + num13) uSpeed −= num13;       // num13 巨大 ⇒ 准瞬时减速
else                             uSpeed = num32;        // 锁定包络
```

由于 `uSpeed·num30 = 0.382·d·uSpeed/(uSpeed+0.1) ≈ 0.382·d`,包络实际为

> **v\*(d) = k·d + c,其中 k = 0.382·r10 [1/s],c = 6·r9 + 0.15·r8 [m/s],上限 Vs。**

于是普通航行分为三个解析子段:

1. **线性加速**: 距出发行星中心 ≤ 1.5R(R≈200 m,代码恒用本站行星半径)内 a₁ = 0.03·Vs,出区后 a₂ = 0.12·Vs·r10,直至 v = Vs。匀加速运动,闭式。
2. **匀速巡航**: v = Vs,直到剩余距离降至刹车点 **d_brake = (Vs − c)/k**。闭式。
3. **指数刹车**: v 锁定包络后,离散递推 `d_{n+1} = d_n(1 − k/60) − c/60`,即
   **d(n) = (d₀ + c/k)·(1 − k/60)ⁿ − c/k**,到达半径 d_arr = 6·r10 时结束:
   **n = ln((d_arr + c/k)/(d₀ + c/k)) / ln(1 − k/60)**(tick)。严格闭式(等比数列)。
   速度与剩余距离成正比 ⇒ 剩余距离指数衰减,这就是肉眼可见的"进站前长长的减速尾巴",其时长只依赖 Vs(对 scale=1 约 11.1 s,scale=3.5 约 8.5 s)。

**(b) 曲速(翘曲)航行** —— 曲速块每 tick 运行:

- **进入条件**(`warpState ≤ 0` 时,2270 行): 距出发行星中心 > 5 000 m(`num22 > 25·10⁶`) 且 剩余 d > `warpEnableDist/2`(曲速启用路程之半) 且 **v 已达 Vs** 且 船上有空间翘曲器 ⇒ 消耗 1 个,`warpState += 1/60`。
- **有效曲速上限**: `num26 = min(Vw, 2·D_AB)`,D_AB 为两行星实时距离 —— 短途跳跃曲速被压到两倍行程。
- **曲速加速**: `warpState` 每 tick `+1/60`(1 秒满),附加速度
  **num24 = num26 · (1001^ws − 1)/1000**(指数上升,ws=1 时恰为 num26),实际速度 `uSpeed = Vs + num24`。
  斜坡段行进距离闭式(等比求和): `S_ramp = Vs·1s + num26·(Σq^i − 60)/60000,q = 1001^(1/60)` ⇒ **S_ramp ≈ Vs + 0.1523·num26**。
- **曲速巡航**: v = Vs + num26,直到剩余距离 < **出口半径 num28 = 0.0449·num24 + 5000 + 0.25·Vs**(满曲速时 d_exit = 0.0449·num26 + 5000 + 0.25·Vs)。
- **曲速退出**: d < num28 后 `warpState −= 1/15`,且 num24 被钳制为 `min(num24, 60.6·(d−num28))` 防止冲过出口。实测动态是飞船"贴着收缩的出口半径面滑下来": 从 d_exit 滑到渐近出口 **d_exit0 = 5000 + 0.25·Vs** 处,耗时 ≈ **33 tick(0.55 s)**,出口速度恰为 Vs。此段是钳制与衰减交织的振荡过程,无严格闭式,但时长近似常数(0.5–0.6 s),距离端点闭式已知 —— 按常数处理即可。
- 退出后转入普通航行的"巡航 + 指数刹车"(d_exit0 > d_brake,故还有一小段 Vs 巡航)。

**(c) 转向与避障**(不进入时间闭式): 角速度 PD 控制把机头转向目标方向(`uVel` 即机头朝向,位置沿机头积分);扫描两星系全部 ≤10 个天体做引力规避(径向抬升 + 横向推离,星体半径 ×2.5 判定恒星);近行星时位置叠加"行星跟随位移" `vectorLF6`(随行星公转/自转动参考系),保证起降相对行星静止。

## 3. 闭式剩余时间算法(不模拟)

### 3.1 输入

| 输入 | 对应游戏量 |
|---|---|
| `v` | `ShipData.uSpeed`(当前速度) |
| `ws` | `ShipData.warpState`(曲速阶段 0~1) |
| `has_warper` | `ShipData.warperCnt > 0`(是否有空间翘曲器) |
| `d_target` | 到目标塔(到达点)的距离 = num21 |
| `d_origin` | 到起始塔的距离 ≈ 距出发行星中心距离(差一个行星半径 ~200 m,影响 <0.3 s) |
| GameHistory | `logisticShipSailSpeedModified`(Vs)、`logisticShipWarpSpeedModified`(Vw)、`logisticShipWarpDrive` |
| 站点设置 | `warpEnableDist`(曲速启用路程,默认 480 000 m = 12 AU) |

派生常数: `r8 = min(Vs/600, 500)`、`r9 = r8^0.4`、`r10 = r9≤1 ? r9 : ln r9 + 1`、`k = 0.382·r10`、`c = 6r9 + 0.15r8`、`d_arr = 6·r10`、`a₁ = 0.03Vs`、`a₂ = 0.12·Vs·r10`、`d_brake = (Vs−c)/k`、`Vw_eff = min(Vw, 2(d_origin+d_target))`、`d_exit = 0.0449·Vw_eff + 5000 + 0.25Vs`、`d_exit0 = 5000 + 0.25Vs`。

### 3.2 算法(航行段;若需含降落把 §2 表中 4、5 两项常数加上)

```
T_brake(d_from, d_to=d_arr) = ln((d_to+c/k)/(d_from+c/k)) / ln(1−k/60) / 60      ── 指数刹车
T_sail(d, v, dep):                                            ── 无曲速余程
    if v ≥ min(k·d+c, Vs):                       # 已在包络上(典型: v=Vs)
        return max(0, d−d_brake)/Vs + T_brake(min(d, d_brake))
    (t_a, s_a) = 两区匀加速 v→Vs (a₁ 限 dep<1.5R 内, 其余 a₂)
    if d − s_a > d_brake:                        # 能加满速
        return t_a + (d−s_a−d_brake)/Vs + T_brake(d_brake)
    else:                                        # 短途: 加速中撞上下降包络
        解 (k·a/2)t² + (a+k·v)t + (v−k·d−c) = 0 取正根 t_m, v_m = v+a·t_m
        return t_m + T_brake((v_m−c)/k)

remaining(d, v, ws, has_warper):
    if ws > 0:                                                ── 已在曲速
        if d > d_exit:
            n = (1−ws)·60                        # 剩余斜坡tick
            S_ramp = n·Vs/60 + Vw_eff·(等比和−n)/60000
            T = n/60 + (d − S_ramp − d_exit)/(Vs+Vw_eff)      # 斜坡+曲速巡航
                (若 d−S_ramp ≤ d_exit: 逐tick走完≤60步斜坡微表,仍 O(1))
        else: T = 0                              # 已在出口内
        return T + 0.55·min(ws,1) + T_sail(min(d, d_exit0), Vs, ∞)
    if 已解锁曲速 且 has_warper 且 d > warpEnableDist/2:        ── 将进入曲速
        (t_a, s_a) = 匀加速 v→Vs;  d_eng = d − s_a
        if d_origin + s_a < 5000: 补巡航 (5000−d_origin−s_a)/Vs 并扣 d_eng
        if d_eng > warpEnableDist/2:
            return t_a + [1s 满斜坡 + (d_eng − S_ramp − d_exit)/(Vs+Vw_eff)]
                   + 0.55 + T_sail(d_exit0, Vs, ∞)
    return T_sail(d, v, d_origin)                              ── 纯航行
```

完整可运行实现见 `closed_form.py`(单函数 `remaining_time`,常数次代数运算,无逐帧模拟;唯一的"微表"是短途曲速时 ≤60 步的斜坡步进,上界固定仍是 O(1))。

### 3.3 验证(对照逐 tick 精确复刻模拟器 `sim.py`)

全程误差(出发→到达半径,部分摘录;`err = 闭式 − 模拟`):

| 配置 | 模式 | 模拟 (s) | 闭式 (s) | 误差 (s) |
|---|---|---|---|---|
| Vs=600, 5 AU | 纯航行 | 347.75 | 347.73 | −0.02 |
| Vs=600, 40 AU | 纯航行 | 2681.08 | 2681.06 | −0.02 |
| Vs=600, 1 ly | 曲速 | 52.48 | 52.48 | −0.00 |
| Vs=600, 20 ly | 曲速 | 430.60 | 430.59 | −0.01 |
| Vs=2100, 5 ly | 曲速 | 46.00 | 45.99 | −0.01 |
| Vs=2100, 40 AU | 纯航行 | 772.62 | 772.60 | −0.01 |
| Vs=6000, 0.2 ly | 曲速 | 14.05 | 14.07 | +0.02 |
| Vs=6000, 20 ly | 曲速 | 53.37 | 53.38 | +0.01 |

中途任意状态(Vs=2100、5 ly 曲速航线,快照后比较剩余时间): 加速段、曲速斜坡、满曲速巡航、出口、刹车尾全部 |err| ≤ 0.01 s;55 个全组合场景最大绝对误差 **0.02 s**。

> 结论: **stage 0 在一维理想化(直线、无避障、行星静止)下完全闭式可解**,精度达到单 tick 量级。

## 4. 无法闭式化的因素与取舍评估

| 因素 | 机制 | 丢弃后的误差量级 | 评估 |
|---|---|---|---|
| 姿态转向(起飞后掉头、进站对准) | 角速度 PD + 四元数插值,位置沿机头积分,路径有弧线 | 弧线发生在低速段,额外路程数十米,**< 1 s** | 安全丢弃 |
| 引力规避绕行 | 沿途天体(半径+5000+v 判定圈)触发抬升/推离转向 | 视线通畅时为 0;掠过行星/恒星时几秒;**目标在行星背面**时最差十几秒 | 丢弃;输入中没有天体几何,本质不可闭式 |
| 行星跟随位移 + 目标移动 | 到达点随行星公转,游戏逐 tick 追踪;闭式解用快照距离 | 误差 ≈ 行星位移/船速。曲速航段(秒级)可忽略;**慢速长途行星间航行**最差可达百分之几~百分之十 | 部分丢弃;可用 `uPosNext` 外推一阶修正,但收益有限 |
| `num23` 转向降频与 `gene` 错峰 | 远距时速度每 10/30 tick 才刷新 | **± 0.5 s** | 安全丢弃 |
| 曲速出口滑行细节 | 钳制+衰减振荡,无严格闭式 | 用常数 0.55 s 拟合,**± 0.2 s** | 安全丢弃 |
| 曲速二次进入 | 出口后若剩余仍 > warpEnableDist/2 且还有翘曲器,会再次消耗翘曲器重新进曲速(玩家把曲速启用路程调得过小时的"乒乓"现象) | 正常配置不发生;发生时低估数秒/多耗翘曲器 | 丢弃,文档注明边界 |
| 到塔距离 vs 到行星中心/到达点 | 输入是"到塔距离",代码用行星中心(曲速判定)与 dock+25 m(到达判定) | **< 0.3 s** | 安全丢弃 |

总体: 对曲速航线,以上全部丢弃后的误差通常 **< 1 s(总程几十秒)**;对慢速行星间纯航行,主要误差源是行星运动(目标动了),量级为 `行星速度/船速` 的相对误差。**这两类都不依赖逐帧模拟即可拿到秒级准确度,取舍完全值得。**

附带说明: 站点电力**不影响**飞行速度 —— `InternalTickRemote` 根本不接收功率参数;能量只在派遣时一次性扣除(`DetermineDispatch` → `CalcTripEnergyCost`,行程 ≥ 曲速启用路程时整程 +1 亿 J),且「翘曲器必备」勾选时库存 < 2 个翘曲器直接不发船,飞船起飞时最多带 2 个翘曲器(每程 1 个),回程翘曲器可在对方站补领 1 个(2680 行)。

---

## 5. 【分析完成后的对照】与官方 `CalcRemoteSingleTripTime` / `CalcArrivalRemainingTime` 的对比

主体分析完成后,读取了此前禁读的两个方法(反编译文件 2962–3045 / 3047–3351 行)。

### 5.1 它们是什么

两者都是**游戏自带的闭式估算器**,服务于「物流控制面板」UI:

- `CalcRemoteSingleTripTime`: 单程总时间(含两端停靠 60 tick、起飞、降落),被 `UIControlPanelStationRouteEntry` 调用(路线条目显示去程+回程总时长),也被 `CalcArrivalRemainingTime` 复用。
- `CalcArrivalRemainingTime`: 指定在途飞船的剩余到达时间,被 `UIControlPanelStationTransportPanel` 调用(运输船倒计时,`/60+1` 显示为秒)。`stage ≠ 0` 时直接用起降/停靠插值进度折算(`stage×direction < 0` 取整程估计减去已耗时,`> 0` 只剩降落+停靠常数);`stage = 0` 时进入与本文 §3 同构的分段闭式计算。

### 5.2 结构对照 —— 两者与本文 §3 的分解完全同构

| 物理段 | 本文 §3 | 官方实现 | 差异 |
|---|---|---|---|
| 近行星加速 | 两区匀加速,近区行程 `1.5R − d_origin`(≈75 m),a₁=0.03Vs | 行程直接取 **R(去程)/ R/4(回程)**;中途版用 `v₁=√(v²+8(或2)·(1.5R−dep)·a₁)`(系数 8 非物理,为拟合) | 官方近区行程偏大 ~2.7×,起步段高估 1–2 s;且近区行程未从总程中扣除 |
| 太空加速 | 匀加速 a₂=0.12·Vs·r10,闭式 | 同公式(`(Vs+v₁)(Vs−v₁)/a₂·0.5` 距离 + Δv/a 时间) | 一致 |
| 加速中撞包络(短途) | v-空间二次方程 | `num15=√(2a₂(d+c/k)+v₁²+(a₂/k)²)−a₂/k` | **同一个根,推导等价** |
| 巡航 | `(d−d_brake)/Vs` | 同 | 一致 |
| 指数刹车 | 离散精确解 `ln(…)/ln(1−k/60)` | 连续 ODE 解 `ln(k·d/c+1)/k`(同比值) | 官方少 ~0.3%(刹车段 ~0.04 s) |
| 曲速进入 | v=Vs 且距行星 5 km,`warpEnableDist/2` 门槛 | 同(中途版 3139 行同样校验未来进入点 `num8 < warpEnableDist·0.5 + num15`);整程版把"是否带翘曲器"交给调用方 `canWarp` | 一致 |
| 有效曲速 | `min(Vw, 2×行星间距)` | `min(Vw, 2×塔间距)` | 差 ≤ 2R+塔高,可忽略 |
| 斜坡+出口 | 精确等比和(60 tick,0.15225·Vw_eff+Vs)+ 实测出口滑行 33 tick 至 `d_exit0` | 合并拟合: 等效距离 `0.18776·Vw_eff + 1.3333·Vs`,等效时间 **80 tick**;短途用 `6400/(79+1/x)` 平滑插值(`Math.Pow(x,0)` 为遗留死代码);中途斜坡进度用 `Vw·ws⁷/7 + Vs·ws` 拟合已走过的斜坡距离,出口衰减剩余 `ws·20` tick | 官方出口时长少算 ~13 tick(−0.2 s);斜坡距离拟合误差 ±0.2 s |
| 取整 | 浮点 | 每段 `(int)(…+1)`/`+0.5` 截断 | 官方逐段累积 +数 tick |

> 值得肯定: 官方两个函数证实了本文对 `InternalTickRemote` 的全部解读 —— 包络刹车系数 `0.382·r10`、刹车截距 `6r9+0.15r8`、到达半径 `6·r10`、出口半径 `5000+0.25Vs(+0.0449·num24)`、5 km 进入曲速距离、`min(Vw, 2D)` 上限等所有常数与本文推导一一对应。

### 5.3 谁算得更准(以逐 tick 精确模拟为基准,1-D 直线理想化)

**整程(静止出发 → 到达半径,50 个场景: Vs∈{600…6000}, 2.5 AU–20 ly, 曲速/纯航行):**

| 估算器 | 平均误差 | 最大绝对误差 |
|---|---|---|
| 本文闭式解 | −0.01 s | **0.02 s** |
| 官方 `CalcRemoteSingleTripTime` | +0.93 s | **1.73 s**(Vs=600 时;Vs=6000 时降至 ~0.37 s) |

官方为**系统性高估**,主因: 起步近区行程取 R=200 m(实际 ~75 m)、近区行程未从航程中扣除、每段取整向上、出口滑行常数偏小。误差与航程无关(常数偏置),随 Vs 升高而减小。

**中途剩余时间(21 个快照,覆盖加速/斜坡/满曲速/出口/刹车尾):**

| 估算器 | 平均误差 | 最大绝对误差 |
|---|---|---|
| 本文闭式解 | −0.01 s | **0.03 s** |
| 官方 `CalcArrivalRemainingTime` | +0.07 s | **1.43 s**(起飞后 1 s 内的近区快照;脱离近区后仅 ±0.2 s) |

### 5.4 结论

1. **在两者共同采用的一维直线模型下,本文闭式解更准**: 最大误差 0.02–0.03 s,比官方(0.4–1.7 s)小约 1–2 个数量级。差距来源不是模型结构(完全同构),而是官方在起步段、斜坡/出口段使用了粗粒度拟合常数并逐段取整 —— 对 UI 显示(整秒)而言这些误差本就无关紧要,官方的取舍完全合理。
2. **对真实 3-D 游戏的精度,两者差距会被共同盲区稀释**: 转向弧线、引力规避绕行、行星公转位移、`num23` 错峰这些非闭式因素(§4)双方都未建模。在最坏情形(慢速长途、目标行星位移大、路径掠过恒星)下,真实误差由这些共同盲区主导(秒级到百分之几),此时两者实际表现同量级。
3. 官方中途估算在脱离起步区后(±0.2 s)已相当精确,其 `ws⁷/7` 斜坡积分、`6400/(79+1/x)` 平滑等拟合手法颇为讲究;若只需 UI 级精度,直接移植官方函数即可,若需要 tick 级精度(例如做物流吞吐规划器),用本文 §3 算法。

### 附: 文件清单

| 文件 | 内容 |
|---|---|
| `StationComponent.cs` | ILSpy 反编译输出(本文行号引用基准) |
| `InternalTickRemote.cs` | 切割出的逐帧逻辑(分析期间隔离禁读方法) |
| `sim.py` | 逐 tick 精确复刻模拟器(ground truth) |
| `closed_form.py` | 本文 §3 闭式剩余时间算法实现 |
| `validate.py` | 闭式解 vs 模拟器验证(§3.3 数据来源) |
| `compare_official.py` | 官方两函数逐行移植 + 三方对比(§5.3 数据来源) |
