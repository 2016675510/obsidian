# 打印机色彩管理深度专题笔记：RIP · ICC · 色域映射 · 线性化 · 灰平衡 · G7

> **定位**：本文是《打印机与热升华色彩管理详细知识笔记》（37 章版）的**深化补充层**。不重复其基础内容，而是针对六个专题做深度展开，并补充 2022–2026 年的最新规范与行业动态（含 ICC.1:2022 / iccMAX v5 / G7 Verifier / VSDT 可变墨滴 / PDF Print Engine 等）。
>
> **全文主线**：
> ```plain
> 校准层（线性化 → 墨量 → 灰平衡 → G7）
>     ↓ 让设备"稳、准、中性"
> 描述层（ICC Profile：设备+材料+工艺的事实描述）
>     ↓ 让颜色"可翻译"
> 决策层（色域映射 / 渲染意图：域外颜色怎么办）
>     ↓ 让转换"有策略"
> 执行层（RIP：分色、加网、墨滴、喷头数据）
>     ↓ 让一切"落到纸上"
> 验证 → 反馈（闭环）
> ```
>
> 六个专题的关系一句话：**RIP 是落地执行层，ICC 是描述/转换层，色域映射是转换中的决策层，线性化/灰平衡/G7 是校准层。**

---

## 导览：与主笔记章节的对应关系

| 本笔记专题 | 对应主笔记章节 | 本笔记新增/深化点 |
|---|---|---|
| 第一篇 RIP | 第 13 章 | 三层流水、加网深挖、VSDT 灰度打印头与 PRN 数据、APPE/Mercury 架构、RIP 校准四步向导、RIP 法与密度曲线法对比 |
| 第二篇 ICC | 第 12 章 | 规范演进时间线（v4.4.0.0 / 2025 AGC / iccMAX v5）、v4 渲染语义、BPC 标准 ISO 18619:2015、色适应模型 |
| 第三篇 色域映射 | 第 12.7 章 | GBD 概念、裁剪/压缩/明度重标定/色相映射/空间 GMA 五类算法、HPMinDE/GCUSP/SGCK、工程选择表 |
| 第四篇 线性化 | 第 8 章 | "ICC 无意义于未线性化设备"、度量选择深挖、RIP 曲线注入机制、向导四步、喷墨特殊性 |
| 第五篇 灰平衡 | 第 10 章 | ISO 12647-1 三种"灰"定义、密度法 vs 色度法、灰平衡曲线反查修正（GrayFinder/P2P） |
| 第六篇 G7 | 第 11 章 | G7 与 TVI 校准的本质区别、P2P25/P2P51/G7 Verifier 目标、NPDC 验证细节、四曲线生成、认证体系现状 |

---

# 第一篇 RIP 专题（深化第 13 章）

## 1.1 RIP 是什么：三层流水

RIP（Raster Image Processor，光栅图像处理器）把"页面描述"变成"喷头能执行的点阵数据"。业界通用的抽象是**三个阶段**（可能与商业 RIP 的实际流水交错执行，但职责不变）：

```plain
① 解释（Interpretation）
   解析页面描述语言（PostScript / PDF / EPS / TIFF / JPEG 等）
   → 得到内部页面表示（对象、矢量、文字、位图、透明混合）
        ↓
② 渲染（Rendering）
   把页面表示转成"设备像素网格上的连续调位图"（RGB 或 CMYK 连续调）
   包含：拼版、缩放、镜像、套准、白墨版、透明合成
        ↓
③ 加网 / 半色调（Screening / Halftoning）
   把连续调位图降为"打印引擎支持的有限墨态"（喷头只有"喷/不喷"，
   或 VSDT 下的"几种墨滴大小"）
   → 输出 PRN / 喷头位图数据
```

**一句话：RIP = 解释器 + 渲染器 + 加网器 三合一。**

## 1.2 RIP 的完整职责域

| 职责域 | 具体事项 | 与色彩管理的关系 |
|---|---|---|
| ① 文件解释 | PDF/PS/EPS/TIFF 解析、字体、透明度混合 | 源数据质量决定后续一切 |
| ② 色彩管理 | 源 ICC → PCS → 目标 ICC、渲染意图、专色转 CMYK/多通道、TAC 强制 | **本文重点** |
| ③ 分色 | RGB/CMYK → 多通道（+Lc/Lm/Or/Gr/Vi/W）、GCR/NGC、墨量分配 | 通道越多越依赖 RIP 内置算法 |
| ④ 校准曲线注入 | 线性化曲线、墨量曲线、灰平衡/G7 曲线在转换后、加网前应用 | 校准层与执行层的接缝 |
| ⑤ 加网/半色调 | AM/FM/误差扩散、线数 lpi、网点形状/角度、每通道独立参数 | 决定阶调还原与细节 |
| ⑥ 墨滴/打印控制 | pass 数、单双向、羽化、叠印顺序、介质走纸、温度/电压参数 | 直接影响实测 Lab（见主笔记 13.2） |
| ⑦ 输出 | 喷头位图（firing data）、PRN 数据、分辨率适配、拼版 | 最终交付物 |

**关键认知：RIP 里的色彩管理是"转换执行"，ICC 的"生成"在色彩管理软件侧完成（见主笔记 13.3 责任边界）。**

## 1.3 RIP 的形态与最新架构

| 形态 | 说明 | 典型代表 |
|---|---|---|
| 软件 RIP | 纯软件解释渲染，运行在 PC/服务器，灵活、可升级 | 多数喷墨厂家的桌面 RIP |
| 硬件 RIP | 专用板卡/盒子，速度快、稳定，配置固化 | 老式大幅面打印机的选件 |
| 嵌入式 PDF 引擎 | 直接原生渲染 PDF，避免 PS 解释误差，支持透明与 VDP | **Adobe PDF Print Engine (APPE)** |
| RIP 集群 | 多实例并行，多 CPU 核/刀片/网络节点组成 RIP farm，可同时驱动多台设备 | **Adobe Mercury RIP Architecture** |

**最新动态（2025）：** Adobe PDF Print Engine 7（2025 年发布）把印前/印后智能直接嵌入 RIP 渲染管线，减少人工干预点，面向包装、标签、工业印刷的高吞吐自动化；Mercury 架构通过并行调度多个 APPE 实例来驱动多台印刷机，专为可变数据印刷（VDP）与可变产品印刷（VPP）优化。

**工程含义（自研/选型）：** 热升华与大幅面行业的主流选择仍是软件 RIP（Caldera / Onyx / SAi / PrintFactory / Wasatch / EFI 等）；是否支持"原生 PDF 渲染 + 透明 + 多通道 + 曲线注入"决定了色彩软件的集成深度。

## 1.4 加网（Halftoning）深挖

### 1.4.1 为什么必须加网

喷头在喷嘴层面是**二元设备**——每个喷嘴要么喷要么不喷；连续调图像与平滑渐变必须被转成"点阵图案"，人眼在正常观看距离上把网点做空间平均，从而"看出"中间调。加网算法决定每通道网点的**形状、频率（线数）、角度**。

### 1.4.2 三种主流方法

| 方法 | 原理 | 优点 | 缺点/注意 |
|---|---|---|---|
| **AM 调幅** | 网点间距固定、面积随阶调变化 | 稳定、易测 TVI、墨量可控 | 有玫瑰花结（rosette）与摩尔纹风险；角度组合（如 0/15/45/75°）需精心设计 |
| **FM 调频** | 网点大小固定、疏密随阶调变化 | 无花结、细节好、无角度问题 | 小网点易扩散、对喷头与纸要求高；密度分辨率有限 |
| **误差扩散 / Dither** | 逐像素把量化误差扩散到邻域（或按阈值矩阵抖动） | 平滑、适合照片与喷墨 | 依赖算法质量；某些 RIP 的"Dither"与"Error Diffusion"是两套混合半色调实现，参数不同效果不同 |

> 说明：在 Roland VersaWorks 等 RIP 中，用户常看到"Dither（伪混合/抖动）"与"Error Diffusion（误差扩散）"两个选项，二者都是半色调成形技术，但算法路径与适用内容不同（照片/渐变 vs 图形/线条）。选择标准：**图像内容越连续、越照片化，误差扩散类越有利；线条文字越硬、越图形化，AM/常规加网越稳。**

### 1.4.3 喷墨加网的特殊性：VSDT 与灰度打印头

现代工业喷头（Ricoh、Kyocera、Epson 等）支持**灰度打印（grayscale printing）**，即通过 **VSDT（Variable Size Dot Technology，可变墨滴技术）** 在单次喷射中控制墨滴大小（如小/中/大三档），而不是简单的"喷/不喷"：

```plain
二元打印头：  每像素 1 bit（喷/不喷）→ 只有 2 级
灰度打印头：  每像素 2–4 bit（如 4 级墨滴）→ 每通道多级
              → 颗粒感降低、渐变更平滑、细节更多、墨量控制更细
```

**色彩管理含义：**

1. **线性化的粒度提高**：灰度头在"同一输入阶调"上有了多个物理墨滴档位，RIP 的线性化曲线要作用于"多级墨滴的组合"，而不再是一条简单二值曲线；
2. **RIP 输出的是多 bit 位图（PRN 数据）**：加网后生成的是按墨滴档位编码的喷头数据，RIP 的加网参数与墨滴映射表（drop map）必须与喷头波形（waveform）匹配；
3. **"网点"概念的弱化**：喷墨本质上没有胶印式网点扩大，行业更倾向直接用密度/L\* 曲线监控（与主笔记 3.4 结论一致）。

## 1.5 RIP 内的色彩转换执行链

```plain
源图像（RGB / CMYK / Lab / 专色）
   ↓ ① 源 ICC（A2B：device → PCS）
   ↓ ② 渲染意图（色域映射决策，见第三篇）
   ↓ ③ 目标 ICC（B2A：PCS → device）
   ↓ ④ 分色 / GCR / 墨量分配（RGB→CMYK 或多通道）
   ↓ ⑤ 墨量限制（TAC/单通道上限）
   ↓ ⑥ 线性化 / 校准曲线（1D LUT 逐通道注入）
   ↓ ⑦ 加网（AM/FM/误差扩散 + 墨滴映射）
   ↓ ⑧ 喷头位图（PRN）→ 打印
```

**两个容易搞混的点：**

- **曲线注入顺序**：线性化曲线在"分色之后、加网之前"应用（作用于设备数值），而不是在 ICC 转换之前——否则 ICC 描述的设备特性与实际的输入输出关系脱节；
- **ICC 与曲线的分工**：ICC 负责"颜色对应"，曲线负责"数值→实际输出"的修正。**ICC 建立在已线性化的设备上**（主笔记 7.6 前置条件），RIP 只是按顺序把它们串起来。

## 1.6 RIP 校准向导的四个步骤（行业通行）

主流 RIP（Caldera / Wasatch / Onyx 等）的"色彩校准向导"高度一致，是四步流水：

```plain
Step 1  Droplet（墨滴）
        定义喷头基础墨滴配置（墨滴大小、波形、pass/方向、dpi）
Step 2  Linearization（线性化）
        打印单通道梯尺 → 分光测量 → 逐通道生成 1D 校正曲线
Step 3  Ink Limit（墨量上限）
        确定 TAC 与单通道上限（饱和点 + 物理失效点取小者）
Step 4  ICC Profile（特性文件）
        在以上三者锁定后，打印特性化测试版 → 建模 → 生成 .icc
```

**纪律：四步顺序不能颠倒。** 未完成线性化与墨量限制就做 ICC，得到的 Profile 会"继承"设备的非线性误差——这是"看似稳定、实则偏色"的根源。RIP 向导把这条纪律固化成 UI 流程，是它比裸 ICC 工具更不容易出错的原因。

## 1.7 RIP 方法 vs 密度曲线法（热转移印花专项研究）

在热转移印花领域，国内期刊报道过三种出墨控制方式的对比（2024 年前后研究）：

```plain
① RIP 法：   按 RIP 要求打印各通道色块，分光仪扫描测量 → 建曲线
② 最大墨量法：直接取各通道最大墨量输出
③ 密度曲线法：在 RIP 校准向导中做线性化后生成密度曲线
```

报道结论（期刊口径，具体数值以原文为准）：

| 指标 | 密度曲线法表现 |
|---|---|
| 1mm 线条线宽变化率 | 最低 3.64%，相比 RIP 法与最大墨量法分别降低约 17.65% 与 33.82% |
| 同浓度色块表观深度（K/S） | 优于其他两种方法 |
| Pantone TSX 涤纶色卡整体平均色差 | 相比常用 RIP 法降低约 43.24% |
| 生成的 ICC 特性文件色域 | 优于其他两种方法 |

**工程启示：** 在热升华/热转移场景，"如何定义每条通道的输入→输出关系"（密度曲线 vs RIP 内建曲线 vs 直接打满墨）会显著影响线条清晰度、K/S 值与最终色差。**先用密度曲线/线性化把单通道关系做对，再做 ICC——这条纪律在研究中被再次验证。**

## 1.8 RIP 常见色彩陷阱与排障

| 现象 | 常见根因 | 处置 |
|---|---|---|
| 双向打印出现周期性横纹 | 双向 banding（"卡布奇诺纹"） | 固定单双向；或做双向补偿 |
| 渐变出现台阶/断层 | 加网量化或线性化未平滑 | 检查曲线平滑、灰度头墨滴档位 |
| 同色块不同 pass 深浅不一 | pass 数/方向变更未重做线性化 | 回到标准 PrintMode 重做 |
| 相邻色块互相污染 | 热升华气相串色（转印阶段） | 加大色块间距、缓冲带（主笔记 26.1） |
| 白墨/专色层错位 | 分色时白墨通道参与 ICC 转换 | 白墨单独线性化，不并入彩色 ICC（主笔记 22.2） |
| 线条发虚、细节糊 | 加网参数与喷头波形不匹配 / 热扩散 | 检查墨滴映射与转印工艺 |

---

# 第二篇 ICC 深化（深化第 12 章）

## 2.1 ICC 规范演进时间线（2022 年之后的最新状态）

| 版本/文件 | 状态 | 关键内容 |
|---|---|---|
| **ICC.1:2022（Profile v4.4.0.0）** | 现行 v4 规范 | 在 ICC.1:2010 基础上并入新增标签与勘误；新增 **CICP 标签**（视频信号识别，支持 HDR 处理）与元数据标签 |
| **ICC.1:2022 + 2025 年 4 月修正案** | 已批准 | 新增 **Adaptive Gain Curve（AGC，自适应增益曲线）标签**（SMPTE 正在同步推进视频端 AGC） |
| **ICC.2:2023（iccMAX，v5）** | 已批准（2023-10-12） | 超越 D50 色度的扩展架构；上一版 ICC.2:2019 已作为 **ISO 20677-1** 发布 |
| **ISO 15076-1:2025** | 现行 | "Image technology colour management — Part 1: Based on ICC.1:2022"，把 ICC.1:2022 纳入 ISO 体系 |

**工程含义：**

1. **日常打印/热升华用的仍是 v4（4.3/4.4）Profile**，v5 是面向光谱、多光谱与新兴应用（HDR、工业检测、材料可视化）的扩展，不是 v4 的简单升级；
2. **CICP 与 AGC 表明 ICC 正在进入 HDR 视频/显示领域**：未来显示器 Profile、宽色域 HDR 工作流会用到这些新标签；
3. 自研软件**至少应兼容 v4 读写**，并留出 v5/iccMAX 的扩展接口。

## 2.2 v4 的核心语义：Perceptual vs Media-Relative

v4 与 v3 的最大区别之一是**把"匹配颜色"与"讨好颜色"分成两套独立的变换**（来源：ICC v4 模型文档）：

```plain
v4 输入 Profile 提供两套独立变换：
  ① Media-Relative Colorimetric（媒体相对色度）
     = 可靠的"测量/匹配"路径：域内颜色精确对应，白点映射到目标介质白
  ② Perceptual（感知）
     = "讨好"路径：允许整体压缩/重塑，让照片"看起来自然"
```

关键规则：

- v4 中 **Media-Relative、Perceptual、Saturation 三种渲染意图的 PCS 白点都被规定为"实际介质白点"**（即媒体相对），只有 Absolute 保留源白点——这解决了 v3 时代白点语义混乱的问题；
- 因此 v4 下做"打样仿真某台机的偏黄纸"必须显式使用 **Absolute**，而不是靠 Profile 内部猜测。

## 2.3 iccMAX（v5）：到底扩展了什么

iccMAX 是 ICC 面向未来的架构扩展，核心是**打破"D50 色度 PCS"这个固定连接空间**：

```plain
v4：   设备 ↔ D50 XYZ/Lab（固定色度 PCS）↔ 设备
iccMAX：设备 ↔ 任意 PCS（可含光谱、BRDF、多光源）
       支持：
       · 光谱处理（spectral processing）——保存/转换光谱而非仅 Lab
       · 材料识别与可视化（material identification & visualization）
       · BRDF（双向反射分布函数）——随观察几何变化的外观
       · 新数据类型、改进的色域边界描述符（GBD）
       · 任意/可编程变换（arbitrary & programmable transforms）
Profile 头中版本号显示为 v5
```

**为什么和打印机有关：** 热升华/纺织品的荧光、金属光泽、珠光外观，本质上需要"光谱 + 几何外观"描述——这正是 v4 的 Lab 框架做不到、iccMAX 正在补的位置。短期内自研项目不必实现 iccMAX，但应理解它是"外观管理"（appearance management）的演进方向。

## 2.4 渲染意图与白点映射（v4 语义下的精确表述）

| 意图 | PCS 白点 | 域内色 | 域外色 | 典型用途 |
|---|---|---|---|---|
| Perceptual | 目标介质白 | 允许整体移动（保关系） | 整体压缩 | 照片、色域差大 |
| Relative Colorimetric | 目标介质白 | **精确匹配** | 裁剪（clip） | 校样、专色近似 |
| Absolute Colorimetric | 源介质白 | 精确匹配（含纸白差） | 裁剪 | 打样仿真偏黄纸 |
| Saturation | 目标介质白 | 允许移动 | 保鲜艳裁剪/压缩 | 图表、商务 |

**官方语义补充（ICC.1:2022）：** Perceptual 意图通常包含**色调缩放（tone scale adjustments）**——把一个介质的动态范围映射到另一个介质的动态范围——以及**色域重塑与映射（gamut reshaping and mapping）**；Saturation 意图"牺牲色相保鲜艳"是历史用途，实际上"以牺牲色相保纯色鲜艳度"的表述只适用于图表类内容。

## 2.5 色适应（Chromatic Adaptation）：转换的"白点翻译器"

设备间转换不可能只用一套 XYZ，因为源与目标的观察白点不同。色适应变换把"在一个白点下测得的 XYZ"映射到"另一个白点下"：

```plain
主流模型：
  von Kries    —— 对角缩放锥体响应（最基础）
  Bradford     —— von Kries 扩展，含交叉项，ICC/Adobe 生态最常用
  CAT02        —— CIECAM02 的色适应变换，用于 CAM 工作流
  chAD 标签     —— ICC Profile 里记录"建 Profile 时的色适应矩阵"
```

**工程要点：**

1. Profile 内部已把设备测量值色适应到 D50 PCS，转换时 CMM 负责源/目标两侧的白点一致性；
2. 若自研引擎：**用 Bradford + D50 锚定**即可与 Adobe/lcms2 对齐；CAT02 用于需要 CIECAM02 色貌模型的高级场景；
3. 热升华场景特别注意：**基材白点（可能含 OBA 荧光）必须与看样光源一致**，否则"色适应对了、目视还是不对"——这是荧光与色适应耦合的典型坑（主笔记 2.4/5.2）。

## 2.6 黑点补偿（BPC）：有正式标准的"暗调救星"

**标准：ISO 18619:2015《Image technology colour management — Black point compensation》**（ICC 与 ISO/TC 130 联合制定）。

```plain
目的：调整源→目标色彩变换，使暗调细节被保留、目标设备可用的
     黑位被充分利用——尤其当目标黑位"不够黑"时（喷墨/热升华常见）。
做法：CMM 分别确定源与目标 Profile 的黑点，然后做 XYZ 缩放，
     把源黑点映射/归一化到目标黑点。
触发：取决于渲染意图与源/目标 Profile，与图像内容无关。
```

**何时必须开 BPC（经验判据）：**

| 场景 | 是否开启 |
|---|---|
| 源是 sRGB/显示器（黑位很深），目标是喷墨/热升华（黑位浅） | **必须开**，否则暗调全糊 |
| 打样仿真（Absolute 意图） | 按打样规范处理，通常不开（要保留纸黑差异） |
| 目标黑位与源接近（如胶印对标胶印） | 可开可不开，影响小 |
| 热升华暗调（主笔记 21.4：最暗 L\* 只能到 12–18） | **必须开**，且配合 GCR/暗调压缩 |

**注意：BPC 不是 Perceptual 的替代品。** "媒体相对色度 + BPC"是 ICC 官方认可的最简单感知再现（简单白黑缩放）；完整 Perceptual 还会做色调缩放与色域重塑。

## 2.7 引擎与工程要点（更新主笔记 12.5 的清单）

| 引擎 | 现状 | 用途 |
|---|---|---|
| **Little CMS (lcms2)** | 开源跨平台，v2/v4 支持、BPC、多通道 | **自研首选**，运行时转换 |
| **ArgyllCMS** | 开源，含建模（colprof）、测量、求逆 | 自研建模流程参考 |
| **Apple ColorSync / Adobe ACE** | 系统级 / Adobe 内核 | 兼容性目标 |
| **Windows WCS** | 微软色管，含 Gamut Map Model（见第三篇） | 了解色域映射的另一种官方实现 |

**自研 ICC 模块的工程要点（补充主笔记 12.6）：**

```plain
① 序列化：至少支持 v4 读写（Header + Tag Table + A2B/B2A CLUT + wtpt/kpt/ptbl）
② 求逆：用"正算造网格 → 插值反查"而非直接解方程组（防墨量爆炸）
③ 元数据：desc / tech / rMDAT 写入创建条件（材料、工艺、仪器、M 条件）
④ 引擎：lcms2 处理运行时转换，BPC 用 ISO 18619 语义实现
⑤ 兼容：v3 Profile 仍大量存在，读取端需兼容 v2/v3/v4
```

---

# 第三篇 色域映射（Gamut Mapping）专题

## 3.1 色域与 GBD（色域边界描述）

```plain
设备色域： 某"墨水+纸+工艺"能产生的全部 Lab/XYZ 点的集合
共同色域： 源与目标色域的交集（相对色度意图下"能被保留"的区域）
不可再现色：目标色域之外的颜色（必须靠映射策略处理）

GBD（Gamut Boundary Description）：
  描述色域边界的数学形式（凸包、分区分段、角度网格、三角网格），
  是几乎所有色域映射算法的输入。
  iccMAX 提供了改进的 GBD 数据类型（见 2.3）。
```

## 3.2 为什么需要映射

源（sRGB/Adobe RGB/打样空间）与目标（打印机 CMYK/热升华）色域几乎不可能重合：

```plain
源色域 ⊄ 目标色域（目标缺高饱和橙/深蓝等）
源色域 ⊋ 目标色域（目标小得多，必须整体压缩）
白点不同（纸白 vs 屏幕白）
黑位不同（暗调深度差异）
```

**色域映射（GMA）= 在"域外颜色怎么处理"与"域内颜色要不要动"之间做决策。** 这是 ICC 渲染意图背后的算法层。

## 3.3 四大渲染意图 = 四类映射策略（官方语义）

| 意图 | 映射策略本质 | 域内颜色 | 域外颜色 |
|---|---|---|---|
| Perceptual | **整体压缩（压缩式 GMA）**：保平滑渐变与整体色调关系 | 允许失真（略微去饱和/移位） | 平滑地移入域内 |
| Relative | **裁剪式 GMA**：域内精确、域外切到边界最近点 | 不动 | 裁剪 |
| Absolute | 裁剪式 + 保留源白点 | 不动（含纸白差异） | 裁剪 |
| Saturation | 保鲜艳（压明度/色相优先） | 可能移动 | 向最鲜艳边界映射 |

**ICC 官方要点（White Paper #2）：** 感知意图适合"再现对象是场景照片、目标是获得在该介质上最讨喜的结果"的场合；若目标介质色域比源大，Perceptual 还允许**提升彩度**以更悦目；色度类意图则追求"数字匹配"而非"视觉讨喜"。

## 3.4 色域映射算法族（GMA 的核心算法）

### 3.4.1 裁剪类（Clipping）

```plain
HPMinDE（Hue-Preserving Minimum Color Difference）
  对每个域外颜色：在固定色相平面上，
  找目标色域边界上"加权色差最小"的点作为映射结果。
  权重包含明度差、彩度差、色相角差（ΔE 公式可选 ΔE76/ΔE00）。
  特征：域内颜色完全不动，映射"最短、最保守"。
  → 相当于"智能版 Relative Colorimetric"，
    常被用于精度优先的专色/品牌色近似。
```

### 3.4.2 压缩类（Compression）

```plain
GCUSP（Gamut Compression toward Cusp，向色域尖峰压缩）
  明度：按彩度加权做重标定（chroma-weighted lightness rescaling）
  彩度：向色域尖峰（cusp，即该色相最饱和点）做线性压缩
  色相：保持（metric hue）
  特征：整体移动、保关系、保平滑——Perceptual 的经典骨架。

SGCK（Sigmoidal Gamut Mapping with Chroma-dependent Lightness Rescaling）
  GCUSP 的演进（Morovic & Luo 等研究确立）：
  ① 明度用 S 形（sigmoidal）重标定——暗部/亮部保护更强
  ② 重标定程度随彩度变化（高彩度少压明度、多压彩度）
  ③ 彩度向 cusp 线性压缩
  → 现代多数高质量 Perceptual 意图的事实标准思路
    （各家厂商的具体曲线参数不同，算法细节不公开）
```

### 3.4.3 明度重标定（Tone Scale Adjustment）

```plain
线性压缩：  L' = a·L + b（简单、老）
折线压缩：  亮部/暗部分段映射（保两端）
S 形映射： 中间调对比度提升、两端压缩（"漂亮但略失真"）
   ICC.1:2022 明确：Perceptual 意图通常包含 tone scale adjustment
   ——把一个介质的动态范围映射到另一个介质的动态范围。
```

### 3.4.4 色相映射（Hue Mapping）

```plain
WCS 色相轮（Microsoft WCS Gamut Map Model）：
  ① 先做"色相轮"：找出源/目标设备一次色与二次色的色相，
     建立 6–12 个关键色相的对应表
  ② 输入色相按相邻关键色相插值调整（Hue wheels 映射）
  ③ 再对明度+彩度做 shearing（剪切）映射
  ④ 最后裁剪保证在域内
  特征：能纠正"源色相轴与目标色相轴不一致"（如青偏绿）的问题；
  常用于把源空间"转成"目标空间形状的系统级映射。
```

### 3.4.5 空间色域映射（Spatial GMA，新一代）

```plain
思路：逐像素映射会破坏图像的空间结构（边缘、对比度），
     空间 GMA 把邻域信息纳入映射：
  ① 域外颜色投影到目标色域边界
  ② 对映射场做保边平滑滤波（edge-preserving smoothing）
  ③ 按滤波后的映射场向色域中心压缩
  特征：边缘锐度保持更好，但计算量大、依赖图像内容，
  主要用于 HDR/高质量照片，打印 RIP 中少用。
```

## 3.5 感知意图的两种工程实现

```plain
实现 A：Profile 内嵌 Perceptual LUT（v4 正规做法）
  源 Profile 的 A2B0 + 目标 Profile 的 B2A0 组合成变换，
  映射决策在生成 Profile 时固化（厂商用 GCUSP/SGCK 类算法）。

实现 B：CMM 运行时算法（"简单感知"）
  "媒体相对色度 + 黑点补偿（BPC）"即 ICC 官方认可的简单感知：
  白点缩放 + 黑点缩放（XYZ scaling），不做色域重塑。
  优点：无需额外 LUT，速度快；缺点：色域差大时仍会生硬裁剪。

工程选择：
  色域差小、暗调是主要问题 → 实现 B（Relative + BPC）足够
  色域差大、照片主导         → 用目标 Profile 的 Perceptual（实现 A）
```

## 3.6 色域映射的工程选择表

| 内容类型 | 推荐意图 | 推荐映射思路 | 备注 |
|---|---|---|---|
| 照片、人像、风景 | Perceptual | 压缩式（SGCK 类） | 保关系优先 |
| 品牌色、专色、logo | Relative（或 HPMinDE 类专用裁剪） | 裁剪式 | 精度优先，域外色"能多近就多近" |
| 打样仿真（模拟偏黄纸） | Absolute | 裁剪式 + 保留源白点 | 纸白差必须"印出来" |
| 图表、演示、矢量色块 | Saturation | 保鲜艳 | 明度/色相可让步 |
| 暗调丰富的夜景/深色内容 | Relative + BPC（或 Perceptual） | 压缩式 + 黑点补偿 | 先解决黑位再谈映射 |
| 热升华高饱和广告画面 | Perceptual（基于最终基材色域） | 压缩式 | 基材色域小，必须整体压缩 |

**通用规则：** 内容决定意图，意图决定算法；同一个 Profile 在不同意图下是"不同的翻译器"。验证时务必同时标注**意图**，否则 ΔE 报告没有意义。

## 3.7 色域可视化与验证

```plain
① 色域体积：Lab 点 3D 凸包 / voxel 计数；LCh 网格积分
② 覆盖率对比：把源/目标 GBD 按 L* 分层叠在 a*b* 平面
   → 定位"哪个明度层、哪个色相区丢得多"（如"红丢在暗调"）
③ 映射质量验证：
   - 域内色是否被误动（Perceptual 允许，Relative 不允许）
   - 平滑度：沿色相扫描相邻色的 ΔE（应连续，不应跳变）
   - 梯度保持：渐变图映射前后是否出现断层
④ 报告：色域体积比、覆盖率%、按色相/明度分区的最大 ΔE
```

---

# 第四篇 线性化深化（深化第 8 章）

## 4.1 为什么线性化是"一切的前提"

行业共识（也是 RIP 厂商文档反复强调的）：

> **"ICC 色彩管理在基线线性化缺失时毫无意义。"**
> 线性化告诉 RIP："你要求 50% 时实际出了 65%，下次请反推输出 35% 的物理墨量，让视觉上得到 50%。"

主笔记已讲清"线性化 = 求逆函数使 F∘L⁻¹ = identity"。这里补充其在整个链条中的位置：

```plain
线性化不正确 → 打出来的梯尺是弯的
    → ICC 建模样数据"继承"弯曲 → Profile 输出整体偏深/偏浅
    → 灰平衡/G7 校准在错误曲线上迭代 → 永远无法收敛
∴ 线性化是校准链的第一块地基（主笔记 7.6 的 ③ 号前置条件）
```

## 4.2 度量选择深挖：用什么"尺子"看阶调

| 度量 | 定义 | 适用 | 注意 |
|---|---|---|---|
| 网点面积率（TVI） | Murray-Davies / Yule-Nielsen 反推 | 有明确加网的胶印/柔印 | 喷墨无传统网点，慎用 |
| **视觉密度 D_v** | Status E/T/A 或全光谱视觉密度 | 通用、机台快捷 | 不反映偏色 |
| **L\*** | 感知明度 | **喷墨/热升华/无网点场景最合适** | 与人眼阶调单调对应，不受滤色片状态影响 |
| 视觉密度和（HR） | 若干亮调小点密度之和 | G7 亮调控制 | 对亮调细节敏感 |

**实践规则（更新主笔记 8.2 的判据）：**

```plain
单色墨（尤其 K）→ 密度线性化即可（快、车间习惯）
含荧光/强扩散体系 → 以 L* 为目标（更贴近人眼）
G7 流程 → 用视觉密度（NPDC 的纵轴本来就是中性视觉密度）
多通道浅色墨（Lc/Lm）→ 建议 L*，因为浅色区密度动态范围太小
```

## 4.3 RIP 中的实现机制：曲线注入

```plain
RIP 校准曲线的本质：逐通道 1D LUT（常见 256 点/通道，或按 RIP 限制）
注入位置：分色之后、加网之前（见 1.5 执行链）
三种曲线常被同时管理（别混用）：
  ① 墨量曲线（Ink Limit Curve）——限制物理墨量
  ② 线性化曲线（Linearization Curve）——输入→输出的阶调校正
  ③ ICC 内部 LUT——颜色对应
RIP 向导通常把 ②③ 分开保存：换介质只需重做 ② + 验证 ③ 是否仍成立
（Wasatch/Caldera 均按"校准曲线 + ICC"双轨管理）
```

**工程要点：**

- **曲线要平滑且单调**：不平滑 → 渐变断层（banding）；非单调 → 加网后出现"回跳"；
- **边界锚定**：0→0、100→100（或允许暗调略升），限制最大校正量防过矫正；
- **迭代收敛**：RIP 不能直接吃 LUT 时，用"打→测→修正→重打"阻尼迭代（如每轮修正量 ×0.7），2–4 轮收敛；
- **版本化**：每条曲线带版本号，与灰平衡/G7/ICC 版本一起可回滚（主笔记 30.1）。

## 4.4 标准向导工作流（可当作业指导书）

```plain
① 固定条件：PrintMode（pass/方向/dpi）、介质温度、墨水批次
② 打印单通道梯尺：每通道 0–100%，≥11 阶，含亮调特殊阶
   （1/2/3/5/8/12/16/25% 等小点阶）
③ 稳定/冷却（热升华：转印+冷却后再测最终，纸面梯可单独测）
④ 分光测量：固定 M 条件与几何，存光谱
⑤ 建 输入→实测 曲线（密度或 L*）
⑥ 单调平滑 → 数值求逆 → 边界锚定 → 限幅
⑦ 生成 1D LUT 注入 RIP
⑧ 重打验证：每阶误差 ≤ ±1–2%（面积）或 ΔL* ≤ 0.8
⑨ 记录版本号，与墨量/G7/ICC 版本绑定
```

## 4.5 喷墨与热升华的线性化特殊性

```plain
喷墨：
  · VSDT 灰度打印头 → 曲线作用于"多级墨滴组合"，粒度更细
  · 双向/pass 变更 = 需要重做线性化（带宽补偿问题）
  · 介质温度影响墨水扩散 → 固定介质加热设置再测

热升华（回顾主笔记 23 章）：
  · 纸面线性化（Print Linearization）≠ 最终线性化
  · 三通道转移率 η_C ≠ η_M ≠ η_Y → 纸面线性≠最终线性
  · 推荐：纸面快速迭代 + Transfer LUT 预估 + 关键节点真实转印复核
```

## 4.6 失效触发与日常监控

```plain
需重做线性化的触发（同主笔记 8.6，此处再强调）：
  换墨水/纸批次、换 PrintMode、喷头清洗/更换/波形调整、
  环境温湿度大幅变化（±5℃/±15%RH 即可出带）、长停机后开机

日常监控：control strip 法
  每次开机打一条控制梯（含各通道 10/25/50/75/100% 与灰块）
  对比基准：ΔL* 或 ΔE00 超阈值 → 触发重做
  这是"稳定性控制"与"校准"的分工，也是 QC 模块的输入（主笔记 15/30）
```

---

# 第五篇 灰平衡深化（深化第 10 章）

## 5.1 "灰"的三种定义（ISO 12647-1:2013）

ISO 12647-1 对印刷中的"灰"给出了三种可操作定义——这是灰平衡测量的法律依据：

```plain
定义 A：与承印物（纸张/基材）具有相同 CIELAB a*、b* 的颜色
定义 B：与"用黑墨印刷的、L* 相近的半色调块"具有相同 a*、b* 的颜色
定义 C：A 与 B 的函数（线性或非线性）组合
```

**工程含义：**

- 定义 A = "以纸白为中性的灰"（绝对中性），常用于灰平衡曲线的参考系；
- 定义 B = "以 K 灰为中性的灰"（印刷灰），更贴近实际观感（因为人眼适应的是印品的灰）；
- 两种参考系的**中性轴可能不同**（尤其纸白带荧光/偏色时）→ 灰平衡判据必须写清楚"对谁为中性"，否则 A 法合格 B 法偏色，验收吵架。

## 5.2 灰平衡的两种测量方法

### 方法一：密度法（传统车间最广）

```plain
① 打印灰平衡导表（不同 CMY 组合的灰色块，通常围绕 50% 网点附近）
② 用分光密度计分别测青/品/黄分色密度（C/M/Y 三滤色片状态）
③ 依据 Murray-Davies 或 Yule-Nielsen 公式反推各通道网点面积率
④ 找"CMY 三通道网点面积率最均等"的色块 → 其 CMY 组合即为灰平衡值
优点：快、适合机台巡检
缺点：密度不反映色偏，只能粗略定位；对荧光/扩散体系误差大
```

### 方法二：色度法（现代标准做法）

```plain
① 打印候选灰块梯（覆盖各 L* 层）
② 分光测量 → Lab
③ 判据：a* ≈ a*_reference、b* ≈ b*_reference
   （reference 按 5.1 选定义 A 或 B）
④ 每 L* 层取偏差最小的 CMY 组合 → 拟合灰平衡曲线 C(L*), M(L*), Y(L*)
优点：直接以"颜色"为准，可量化、可跨设备比较
缺点：需要分光仪、色块数量多
```

**两种方法的关系：** 密度法是色度法的"廉价代理"，适合日常巡检；正式建曲线、建 ICC、做 G7 必须用色度法（或光谱法）。主笔记 10.4 的三种求解算法（单变量比例搜索 / 数据库回归 / NPDC 反解）即色度法的软件实现。

## 5.3 灰平衡曲线获取与 P2P 修正（G7 灰平衡的实操细节）

国内 G7 灰平衡研究论文给出的标准修正流程（与 Idealliance 官方 P2P 流程一致）：

```plain
① 灰平衡曲线获取：
   按理论灰平衡数据（由密度法/模型计算得到各 L* 的 CMY 值），
   在 GrayFinder（G7 官方灰平衡查找工具）上找出各灰平衡点的 CMY，
   分别画出 C、M、Y 三条灰平衡曲线（横轴=灰阶 L*，纵轴=网点面积率）
② P2P 色表数据修正：
   P2P 色表中第 4 列是"CMY 灰"色块列；
   若该列未达到灰平衡，则把每个色块的 CMY 值代入灰平衡曲线反向修正。
   示例（M 通道）：以 M=50% 为例，从纵轴 50% 处画水平线交曲线于一点，
   再向下画垂直线交横轴，交点对应的网点面积率即为修正后的 M 值。
   （C、Y 通道同理）
③ 将修正后的 CMY 写入 P2P 灰列 → 重新校准 → 验证
```

**要点：** 灰平衡曲线是"逐通道 1D"的（C/M/Y 各一条），修正也是逐通道的——这与"调一个整体灰偏"的直觉相反，但正是印刷色彩管理的标准做法。

## 5.4 CMY 灰 vs K 灰，以及与 GCR 的关系

```plain
CMY 灰：纯三色叠印 → 依赖油墨纯度、对批次最敏感、暗调墨量高
K 灰：  黑墨为主（可加少量 CMY 修正色偏）→ 稳、省墨、干燥快

工程结论（主笔记 10.3 + 本节补充）：
  ① CMY 灰与 K 灰必须在同一 L* 上"对齐"
     （若同明度下两者偏色方向不同 → 染料热/光行为不一致）
  ② 灰平衡曲线与 GCR 曲线是"互相约束"的两个参数集：
     提高 GCR 深度（用 K 替代灰成分）会同时改变暗调灰平衡，
     所以"改 GCR → 重测灰平衡 → 重做线性化 → 重打 Profile"顺序固定
  ③ 热升华：K 通道热稳定性通常优于 CMY（黑染料迁移窗口不同），
     适度提高 GCR 可改善暗调稳定性，但牺牲暗调彩度（主笔记 4.2）
```

## 5.5 灰平衡的容差与工程要点

```plain
常用判据（经验值，须以合同/标准为准）：
  中性灰：|Δa*| ≤ 1.0–1.5，|Δb*| ≤ 1.0–2.0（暗调可放宽至 ±2.5）
  G7：按 Idealliance Pass/Fail 表的分层容差（见第六篇）
监控：灰平衡曲线比值 M/C、Y/C 是"油墨指纹"——
     比值随批次漂移比单块 ΔE 更早暴露油墨问题（主笔记 10.5）
```

---

# 第六篇 G7 深化（深化第 11 章）

## 6.1 G7 与 TVI 校准的本质区别

```plain
传统 TVI 校准： 分别瞄准 C/M/Y/K 各自的"输入→输出"单通道曲线
               （各通道独立达标）
G7：           瞄准"合成印刷品的观感"——
               · NPDC（中性印刷密度曲线）：K 梯与 CMY 灰梯的密度阶调
               · 灰平衡：CMY 灰梯每阶 a*、b* 在中性带内
               · HR/IT（亮调范围）：亮调小点的密度和
               → G7 用 4 条 1D 曲线（K、C、M、Y）同时达成
                 "灰平衡正确 + 中性调性正确"
```

**关键认知（ChromaChecker 等工具的表述）：** G7 针对的是**感知中性的灰平衡**（perceptually neutral gray balance across the full tonal scale），而不是单通道 TVI。所以 G7 校准后，单通道曲线可能不完全"线性"——这是设计取舍，不是 bug（主笔记 11.3 已强调）。

## 6.2 G7 测试目标：P2P25 / P2P51 / G7 Verifier

| 目标 | 内容 | 用途 |
|---|---|---|
| **P2P51** | 51 色块（K 梯 + CMY 灰梯 + 七色 + HR 亮调块等），G7 完整校准目标 | 做 G7 校准曲线（灰平衡计算必需） |
| **P2P25** | 25 色块简化版 | 快速校准/验证 |
| **G7 Verifier**（新一代） | 评估 G7 Grayscale 与 G7 Targeted 合规的最小色块集 | **合规验证**专用，不替代 P2P25/51 做校准；有 i1Pro 手扫版与 i1iSis 机扫版 |

**重要提醒（Kodak ColorFlow 文档明确）：** 简单的 Tint Ramp（单色阶调版）**不能**用于 G7 灰平衡校准——测量版必须包含 G7 规范要求的全部灰阶色块（即 P2P 类目标），才能计算灰平衡。

## 6.3 NPDC 验证细节（官方 ADS 程序）

Idealliance G7 System Certification 的 NPDC 验证程序（ADS 文档口径）：

```plain
① K-only 梯 与 CMY-only 梯 分别用密度计或分光仪测量
② 每个色块记录"相对中性密度（relative neutral density, ND）"
   取值通道："K" 通道 或 "Visual" 通道
③ 获得相对 ND 的方法：测量仪在承印物上归零，
   或用白块 ND 值对每个色块做减法（即减掉纸白密度）
④ 把 K 梯与 CMY 梯的 ND 曲线与目标 NPDC 比较 → 得偏差曲线
⑤ 偏差用于生成 4 条 1D 校正曲线
```

**工程要点：** 相对密度（relative ND）意味着**纸白被归零**——基材白度只影响曲线的"起点"而非"形状"，这使 NPDC 具有跨基材可比性（前提是基材中性近似）。

## 6.4 灰平衡校正与四条曲线的生成

```plain
校正数据流：
  测量 P2P（或 G7 Verifier）→ Lab + 相对 ND
     → ① NPDC 偏差：K 梯/CMY 梯 vs 目标曲线
     → ② 灰平衡偏差：CMY 灰梯每阶 Δa*/Δb*
     → ③ 生成 4 条 1D 曲线：
          K  曲线 ← K 梯 NPDC 匹配
          C/M/Y 曲线 ← 灰平衡再分配（保持 L* 的 CMY 调整，见 5.3）
     应用 → 重打 → 重测 → 迭代 2–4 轮收敛
```

主流 RIP（如 Kodak ColorFlow）的曲线生成器提供 **"Gray Balance" 校准方法**：选择该方法的曲线生成选项后，软件自动用灰梯数据生成灰平衡校准曲线——这即是 G7 曲线生成的工程化实现。

## 6.5 G7 认证体系现状（2024–2026 口径）

```plain
体系：Idealliance G7 System Certification Program
分级（2018 年起三级制，取代旧两级）：
  · G7 Grayscale  —— 灰调性（NPDC + 灰平衡）达标
  · G7 ColorSpace —— + 七色块落在所选色彩空间容差带内
  · G7 Master     —— + 全部所选色彩空间色块达标（最高级）
认证载体：G7 System Certification Application Data Sheet（ADS）——
  由软件/系统厂商向 Idealliance 提交，评估其"能否用 4 条 1D 曲线
  在容差内把设备校准到 G7 灰度定义"。
```

**对自研软件的意义：** 如果你的色彩管理软件想宣称"支持 G7"，对标点就是"生成 4 条 1D 曲线 + 满足 G7 灰度容差"；想获得官方认可，需走 Idealliance G7 System 认证程序（ADS 流程），并配备 P2P/G7 Verifier 测量支持。

## 6.6 与标准生态的关系（更新主笔记 11.6）

```plain
ISO 15339-1/-2  —— 参考打印条件/中性基线（NPDC 目标来源之一）
ISO 10128        —— "如何用特性化数据把设备对准参考条件"的方法学
ISO 12647-2/-7   —— 胶印/打样过程控制与容差（NPDC 常对齐其曲线）
ISO 19303/19306  —— 数码印刷/大幅面色度要求（热升华最接近）
Idealliance P2P  —— Print-to-Proof 跨设备同貌认证（G7 达标后的下一步）
```

## 6.7 热升华 G7 要点（回顾 + 更新）

```plain
适用性：完全适用且价值大（把"颜色不对"拆成"调性/中性"两个可独立
       验证的子问题——主笔记 24.4）
铁律：必须声明完整条件（Printer+Ink+Paper+Substrate+HeatProfile+M 条件）
     ——NPDC 与灰平衡是在"最终基材"上定义的
目标 NPDC 选择：推荐"自定义设备最优 NPDC"（选项 3），
    不硬套胶印目标（热升华暗调受转移率/TAC 限制）
新增要点（本篇补充）：
  ① 测量相对 ND 时在"最终基材"上归零（而非转印纸）
  ② 灰平衡修正作用于"纸面输入"（预偏置）——主笔记 23.3
  ③ 基材荧光/OBA 会干扰 NPDC 的相对密度 → 固定 M1 条件
```

---

# 第七篇 综合：端到端工程案例（六专题串联）

**场景：** 一台热升华打印机 + 指定墨水/转印纸/涤纶布 + 平板压烫机，目标：按 G7 灰平衡 + ICC 实现稳定交付，并接入 RIP 生产。

```plain
阶段 1 稳定性（主笔记 SOP 步 1–3）
   喷嘴检查、PrintMode 锁定、板温均匀性实测（满幅色块法）
   → 通过判据：同位 ΔE00 ≤ 1.0–1.5 才继续

阶段 2 线性化（本篇第四篇 + 主笔记第 8 章）
   纸面单通道梯 → 分光测量 → 密度/L* 双度量曲线 → 求逆平滑 → 注入 RIP
   → 判据：纸面每阶 ΔL* ≤ 0.8

阶段 3 墨量（主笔记第 9 章）
   饱和点 + 物理失效点实验 → TAC/单通道上限锁定

阶段 4 G7 灰平衡（本篇第六篇 + 主笔记 23/24 章）
   转印到最终基材 → 测 P2P51（或 G7 Verifier）
   → NPDC（相对 ND，基材归零）+ 灰平衡（Δa*/Δb*）
   → 生成 K/C/M/Y 四条曲线（作用于纸面输入 = 预偏置）
   → 迭代 2–4 轮收敛 → 锁定 G7 版本

阶段 5 ICC（本篇第二篇 + 主笔记第 12 章）
   在 G7 达标基础上打特性化测试版（转印后测最终 Lab）
   → 建模（device→PCS）→ 求逆（B2A）→ BPTC 开启（暗调）
   → 元数据写全（材料/工艺/仪器/M1）→ 发布 .icc

阶段 6 RIP 输出（本篇第一篇）
   源 ICC + Perceptual（照片）/Relative（品牌色）+ 目标 ICC
   → 分色/GCR → 墨量 → 线性化曲线注入 → 加网（VSDT 墨滴映射）→ PRN

阶段 7 验证闭环（主笔记第 14/28 章）
   纸面 control strip（过程级）+ 最终基材验证版（交付级）
   → ΔE00 均值/最大/95%、灰梯 Δa*Δb*、NPDC 进带
   → 超差 → 按决策树回退（先查工艺层，再动 ICC）
```

**每条链路的"输入 → 输出"检查点（验收速查）：**

| 环节 | 关键输入 | 关键输出 | 判据 |
|---|---|---|---|
| 线性化 | 单通道梯 + 测量 | 1D 曲线 ×N | 每阶 ΔL* ≤ 0.8 |
| 墨量 | 墨量阶梯实验 | TAC/上限 | 饱和点与失效点取小者 |
| G7 | P2P51/Verifier 最终测量 | 4 条曲线 | NPDC 进带 + 灰 Δa*b* 达标 |
| ICC | 特性化版 + 独立验证集 | .icc + 报告 | 均值 ΔE00 < 1.5–2 |
| RIP | 源/目标 ICC + 曲线 | PRN 数据 | 输出参数快照可回溯 |
| 验证 | control strip + 成品样 | ΔE00 三件套 | 均值/最大/95% 达标 |

---

# 附 A：新增术语速查

| 术语 | 含义 |
|---|---|
| RIP | Raster Image Processor，光栅图像处理器（解释+渲染+加网） |
| VSDT | Variable Size Dot Technology，可变墨滴技术（灰度打印头） |
| PRN | 喷头点阵数据（RIP 最终输出） |
| APPE | Adobe PDF Print Engine，PDF 原生渲染引擎 |
| Mercury | Adobe 多实例并行 RIP 架构（RIP farm） |
| GBD | Gamut Boundary Description，色域边界描述 |
| GMA | Gamut Mapping Algorithm，色域映射算法 |
| HPMinDE | Hue-Preserving Minimum Color Difference（色相保持最小色差裁剪） |
| GCUSP / SGCK | 两类压缩式色域映射算法（Perceptual 常用骨架） |
| cusp | 色域"尖峰"（某色相最饱和点） |
| CICP | Coding Independent Code Points，视频信号识别标签（HDR） |
| AGC | Adaptive Gain Curve，自适应增益曲线标签（2025 修正案） |
| iccMAX | ICC.2:2023（v5），超越 D50 色度的扩展架构 |
| BPC | Black Point Compensation，黑点补偿（ISO 18619:2015） |
| chAD | Profile 中的色适应矩阵标签 |
| ND | Neutral Density，中性密度（G7 NPDC 用相对 ND） |
| P2P25/51 | G7 校准/验证色表（灰梯必需） |
| G7 Verifier | G7 合规验证最小色表（新一代） |
| GrayFinder | Idealliance 灰平衡查找工具（G7 工作流） |

# 附 B：最新资料与来源清单（本笔记网络检索核对）

**规范与官方（ICC / ISO / Idealliance）：**

- [ICC Specifications（ICC.1:2022 v4.4.0.0，2025-04 AGC 修正案）](https://www.color.org/icc_specs2/)
- [ICC.1:2022 规范全文 PDF](https://archive.color.org/specification/ICC.1-2022-05.pdf)
- [iccMAX Profile Specification（ICC.2:2023）](https://www.color.org/iccmax/)
- [Introducing iccMAX（官方介绍）](https://www.color.org/iccmax.xalter)
- [Introducing iccMAX – New Frontiers in Color Management（TAGA 论文）](https://library-files.taga.org/2015-proceedings/T20150256.pdf)
- [ICC White Paper #2：Perceptual Rendering Intent Use Case Issues](http://www.color.org/ICC_white_paper_2_Perceptual_rendering_use_cases.pdf)
- [Design & Use of the Perceptual Rendering Intent for v4 Profiles](https://color.org/Design_and_use_of_v4_PI.pdf)
- [ICC Version 4 Model（v4 语义：Perceptual vs Media-Relative）](https://www.color.org/CMSystemsandICCV4.pdf)
- [ISO 15076-1:2025（基于 ICC.1:2022 的 ISO 版）](https://www.iso.org/obp/ui/ru/#!iso:std:86740:en)
- [ISO 18619:2015（Black point compensation）](https://www.iso.org/obp/ui/fr/#!iso:std:63033:en)
- [ISO 12647-1:2013（灰的三种定义）](https://www.iso.org/obp/ui/es/#!iso:std:57816:en)
- [Idealliance G7 System Certification ADS 程序文档](https://www.idealliance.org/wp-content/uploads/2017/10/G7_System_ADS_pressSIGN8-2.pdf)
- [G7 System ADS（NPDC 验证方法：相对 ND）](https://www.printing.org/docs/default-source/certifications/idealliance-g7-system_chromachecker_ads.pdf?sfvrsn=fad7238a_1)
- [Idealliance GBA China：G7 Verifier 目标说明](https://idealliancechina.org/download-g7/)

**RIP / 加网 / 打印技术：**

- [Understanding RIP for Print Production（PDFPress）](https://pdfpress.app/blog/raster-image-processor-basics)
- [What Is a RIP in Printing?（Printing TLDR）](https://printingtldr.com/what-is-a-rip-in-printing/)
- [Raster Image Processor（PrinterArchive）](https://printerarchive.net/tools/raster-image-processor)
- [Halftoning（PrinterArchive）](https://printerarchive.net/tools/halftoning)
- [The Ultimate Guide to RIP Software: CMYK Curves, ICC, and PRN Dot Data（MYJET，VSDT/灰度打印头/线性化）](https://www.myjetprint.com/news/blog/Tech---Guides/-ultimate-guide-rip-software-cmyk-icc-prn-data.html)
- [Adobe PDF Print Engine FAQ（Mercury RIP 架构）](https://www.adobe.com/gt/products/pdfprintengine/faq.html)
- [Adobe PDF Print Engine 7 What's New（2025）](https://www.adobe.com/th_en/products/pdfprintengine/whats-new.html)
- [Caldera Printer Calibration Wizard（四步：Droplet/Linearization/Ink Limit/ICC）](https://helpdesk.caldera.com/hc/en-us/articles/15789364258321-How-to-manage-colors-in-a-preset-Printer-Calibration-Wizard)
- [Wasatch Color Calibration（线性化 + ICC 双轨）](https://www.wasatch.com/htmlHelp/en/wwrip/color/wwripColorCalibration.html)
- [Murakami：Linearization of Halftones（丝印 RIP 内线性化）](https://www.murakamiscreen.com/technicalinfo/linearization-of-halftones-)

**色域映射：**

- [About ICC profiles and Gamut Mapping（ArgyllCMS）](https://argyllcms.com/doc/iccgamutmapping.html)
- [Color gamut and gamut mapping（PrinterArchive）](https://printerarchive.net/guides/gamut-mapping)
- [HPMINDE 色域裁剪实现（含数学背景）](https://shankhya.github.io/musings/hpminde.pdf)
- [General-Purpose GMA：Contrast-Preserving Rescaling（Fairchild，GCUSP）](http://markfairchild.org/PDFs/PRO08.pdf)
- [WCS Gamut Map Model Profile Schema and Algorithms（Microsoft，色相轮/shearing）](https://learn.microsoft.com/hi-in/windows/win32/wcs/wcs-gamut-map-model-profile-schema-and-algorithms)
- [Spatial Color Gamut Mapping（保边空间 GMA 综述）](https://pdfs.semanticscholar.org/f4d6/a34432a8a10e6e9334aac206c16d28d07228.pdf)

**G7 灰平衡 / 灰平衡方法：**

- [基于 G7 灰平衡控制的色彩修正方法（《出版与印刷》论文：GrayFinder + P2P 灰列修正）](https://cbyys.sppc.edu.cn/cn/article/pdf/preview/b49404e0-de0d-4ee2-bb95-bbd2401627f7.pdf)
- [印刷品灰平衡检测分析（北检：密度法/Murray-Davies/Yule-Nielsen）](https://www.beijiance.com/qt/qt/183861/)
- [如何在印刷工艺中定义正确的灰平衡（东上中拓：密度法与色度法）](http://www.tungshung.com/News-Details.aspx?Id=479)
- [Kodak ColorFlow：Create a print curve to align a press to G7 gray balance（P2P51 必需）](https://workflowhelp.kodak.com/display/CFT/Activity+5:+Create+a+print+curve+to+align+a+press+to+G7+gray+balance)
- [ChromaChecker G7 Curve Inspector（G7 与灰平衡：NPDC/HR）](https://chromachecker.com/insp-g7-curve-inspector)

**热升华 / 热转印专项：**

- [基于密度曲线模型提升热转移印花印品质量（维普期刊：RIP 法 vs 密度曲线法对比）](http://lib.cqvip.com/Qikan/Article/Detail?id=7108958639)
- [Textile color calibration guide（RIP/ICC/CIELAB，线性化→墨量→ICC 顺序）](https://www.ripservice.com.ar/guia-calibracion-textil)
- [Sublimation ICC Profile setup（驱动管理 vs RIP 管理双工作流）](https://sublimationguides.com/sublimation-icc-profile/)
- [DTF 色彩管理：ICC 配置文件（Textek，显示器校准/工作流）](https://www.textek.cn/zh-CN/news/how-to-get-perfect-colors-in-dtf-icc-profiles-and-color-management.html)

> ⚠️ 与主笔记一致的原则：本文引用的行业数值（G7 容差、TAC、ΔE 门槛等）是通行经验或公开资料口径，**落地生产必须以正式标准文本、Idealliance 当期 Pass/Fail 文档或合同规范为准**；涉及"换什么必须重做"等工程规则以主笔记第 25/29 章为基准。

*本笔记最后更新：2026-10-24，作为《打印机与热升华色彩管理详细知识笔记》的深化补充层。*
