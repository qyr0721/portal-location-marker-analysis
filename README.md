# Portal 模拟定位框架标记注入机制的逆向分析与去除方法研究

**A Reverse-Engineering Study on the Location-Marker Injection Mechanism of the Portal Mock-Location Framework and Its Elimination**

| 项目 | 内容 |
| --- | --- |
| 研究对象 | `moe.fuqiuluo.portaldev`（PortalDev 2.0.7，build r92，commit `587cfab`） |
| 分析平台 | Android（ColorOS，KernelSU + LSPosed 框架） |
| 分析方法 | 静态逆向（Dalvik 字节码反汇编与交叉引用分析） |
| 文档状态 | 研究完成稿 |

---

## 摘要

模拟定位（Mock Location / Location Spoofing）类工具在通过 Xposed 框架对系统定位服务进行拦截与改写时，往往会在输出的 `Location` 对象中写入自定义的 `extras` 字段，这些字段构成可被目标应用识别的"指纹"。本文以 PortalDev 模拟定位框架为对象，通过 Dalvik 字节码级的静态分析，完整还原了其在 `fused` 定位结果中注入 `portal.enable`、`allToggles` 与 `is_mock` 三个标记键的代码路径及其控制机制，指出该行为由一个 64 位功能位域 `FeatureToggles` 的特定位（位值 32，即 `0x20`）所控制。在此基础上，本文提出并实现了一种基于分支指令重定向的字节级补丁方案，使标记写入代码成为不可达的死代码，并完成了 dex 校验和修复、APK 重打包与签名、真机部署与验证的全流程。此外，本文附带对第三方同类模块 `com.yinshibai.location` 的标记行为进行了对照分析，澄清了"其是否注入 `is_mock`"这一分歧。实验结果证实：补丁后的 Portal 与原生 yinshibai 的定位输出均不再携带任何自定义标记键。

**关键词**：Android 逆向工程；Xposed 框架；模拟定位；静态分析；Dalvik 字节码；dex 补丁

---

## 目录

1. [引言](#1-引言)
2. [研究对象与实验环境](#2-研究对象与实验环境)
3. [分析方法与流程](#3-分析方法与流程)
4. [标记注入机制分析](#4-标记注入机制分析)
5. [补丁方案设计与实施](#5-补丁方案设计与实施)
6. [重打包、签名与部署](#6-重打包签名与部署)
7. [部署过程中的两类故障及其归因](#7-部署过程中的两类故障及其归因)
8. [验证结果](#8-验证结果)
9. [附带研究：第三方模块标记行为对照](#9-附带研究第三方模块标记行为对照)
10. [结论](#10-结论)
11. [局限性与展望](#11-局限性与展望)
12. [产物文件](#产物文件)
13. [附录](#附录)
14. [参考文献](#参考文献)

---

## 1 引言

### 1.1 研究背景

基于 Xposed 框架的模拟定位工具通过在 `system_server` 进程中注入钩子，拦截并改写 `LocationManagerService` 的定位输出，从而在不修改目标应用的前提下，向任意应用提供伪造的地理位置。这类工具在系统层运作，其改写结果对上层应用而言与真实定位在结构上几乎无法区分，因此被广泛应用于位置相关应用的测试、隐私保护以及部分灰色场景。

然而，工具作者往往会在其输出的 `Location` 对象的 `extras` 字段中留下可辨识的自定义标记，其初衷可能包括状态同步、调试或自我保护。这些标记一旦被目标应用反向利用，就会成为识别"模拟环境"的直接依据，从而瓦解模拟定位的核心价值——隐蔽性。因此，研究此类标记的注入位置、控制逻辑与消除方法，既具有工程实践价值，也构成了对 Android 定位安全边界的一次系统考察。

### 1.2 问题定义

在对 PortalDev 模拟定位框架的输出进行观测时，发现其 `fused` 定位结果携带了如下 `extras` 键值对（已做脱敏处理）：

```
Location[fused 30.76,103.98 ... {Bundle[{portal.enable=true, allToggles=135179, is_mock=true}]}]
```

其中三个键均非 Android 系统标准字段，属于 Portal 自定义注入：

| 键 | 类型 | 观测值 | 语义推断 |
| --- | --- | --- | --- |
| `portal.enable` | `Boolean` | `true` | 模块启用标记 |
| `allToggles` | `Long` | `135179` | 功能位域 `FeatureToggles` 的直传值 |
| `is_mock` | `Boolean` | `true` | 显式的模拟环境标记 |

本研究的目标是：使 Portal 输出的定位对象**不携带上述任何自定义标记**，且该消除方案在运行时稳定可靠、不依赖临时状态。

### 1.3 研究目标与贡献

本文的贡献可归纳为三点：

1. **机制还原**：精确定位三个标记键在 Dalvik 字节码中的唯一写入点，并还原其由 64 位功能位域控制的完整控制流。
2. **方案设计与实现**：设计基于控制流重定向的最小化字节补丁（仅改动 2 字节），并解决由此引入的 dex 完整性校验问题，完成可安装产物的全流程构建。
3. **实证与对照**：在真实设备上完成部署与验证，并对第三方同类模块的标记行为进行代码级与实测级的双重澄清。

---

## 2 研究对象与实验环境

### 2.1 目标样本

| 属性 | 取值 |
| --- | --- |
| 文件 | `PortalDev_2.0.7.r92.587cfab.apk` |
| 包名 | `moe.fuqiuluo.portaldev` |
| 版本 | 2.0.7（build r92） |
| 提交 | `587cfab` |
| 结构 | 双 dex（`classes.dex` / `classes2.dex`）+ React Native 前端资源 |

样本采用 React Native 构建前端，核心逻辑位于原生层，通过 Xposed 框架在系统服务进程中生效。值得注意的是，其关键类与方法名经过了字符串混淆——方法名被替换为无意义的日文或中文句子（例如 `为什么这么冷冷清清呢？`），这在客观上增加了静态分析的难度，也要求分析过程以字符串常量而非符号名为锚点。

### 2.2 实验环境与工具链

| 类别 | 内容 |
| --- | --- |
| 宿主机 | Windows，Python 3.13 |
| 逆向库 | androguard 3.3.5（`DalvikVMFormat`、`AnalyzeDex` 交叉引用） |
| 构建工具 | Android SDK build-tools 36.0.0（`zipalign`、`apksigner`） |
| 运行环境 | JRE 17（便携版） |
| 目标设备 | Android（ColorOS），KernelSU root，LSPosed 框架 |
| 调试通道 | `adb`（Android Debug Bridge） |

---

## 3 分析方法与流程

本研究采用以**字符串为锚点的静态逆向流程**，其核心思想是：由于混淆只作用于符号名而无法混淆字符串常量，因此先定位标记键字符串在 dex 中的位置，再通过交叉引用反向追溯引用这些字符串的方法，最终还原控制流。整体流程如下：

```
解包 APK
   │
   ├─ 1. 定位标记键字符串（is_mock / portal.enable / allToggles）
   │        │
   │        └─ 2. 交叉引用分析（AnalyzeDex.get_xref_from）
   │                 │
   │                 └─ 3. 反汇编目标方法，还原写入序列与控制流
   │
   ├─ 4. 追踪控制变量（FeatureToggles 位域）及其读写来源
   │
   ├─ 5. 设计字节级补丁并定位 dex 文件内绝对偏移
   │
   ├─ 6. 修复 dex 头部校验和（SHA-1 与 Adler-32）
   │
   └─ 7. 重打包 → 对齐 → 签名 → 真机部署 → 验证
```

这一流程的关键在于第 2 步：`AnalyzeDex` 提供的字符串到方法的交叉引用能力，使分析无需遍历全部方法即可直达注入点。本文的后续章节严格遵循上述流程展开。

---

## 4 标记注入机制分析

### 4.1 注入点的唯一性判定

在全部 dex 中扫描三个标记键字符串，命中集中在 `classes2.dex`。通过交叉引用分析，确认三个键的写入**在全 APK 范围内仅存在一处**，位于：

```
Lmoe/fuqiuluo/location/LuciferLocationData;
    -> 为什么这么冷冷清清呢？(Landroid/location/Location;)Landroid/location/Location;
```

该方法是 Xposed 钩子对位置输出的统一加工入口，其签名表明它接收一个 `Location` 并返回改写后的 `Location`。方法内部的写入序列（按字节码顺序）为：

```
const-string v3, 'allToggles'
invoke Bundle->putLong(String, long)      // 写入 allToggles = getFeatureFlags()
const-string v1, 'portal.enable'
invoke Bundle->putBoolean(String, boolean) // 写入 portal.enable = true
const-string v1, 'is_mock'
invoke Bundle->putBoolean(String, boolean) // 写入 is_mock = true
```

其中 `allToggles` 的值来自 `FeatureToggles.getFeatureFlags()` 的直接传递，与观测到的 `135179` 相吻合。

### 4.2 控制机制：FeatureToggles 功能位域

三个标记的写入并非无条件执行，其前存在一处条件判断。判断所依赖的是一个 64 位功能位域 `moe.fuqiuluo.xposed.FeatureToggles`，它维护一组按位编码的功能开关。在标记写入之前，字节码调用了一个 getter：

```
invoke FeatureToggles->時間を稼ぐのはいいが別にアレを倒してしまっても構わんのだろう？()Z
if-nez v13, +0x2d     // 结果为真（非零）则跳过后续标记写入
```

对该 getter 的反汇编显示，其内部以 `const v2, 32` 与位域做按位与运算，即其检查的是**位值 32（`0x20`，即自低位起的第 5 位）**。因此该位本质上就是作者预留的"隐藏标记"开关：**当位值 32 被置位时，三个标记的写入被整体跳过**。

进一步的交叉引用分析揭示了一个关键事实：**该位的 setter（`自分の努力のために謝らないで、自分の努力ができません(Z)V`，内部 `const v2, 32`）在整个 APK 中没有任何调用者**，即这是一段死代码。这意味着该开关无法通过 App 的界面或 IPC 命令通道在运行时被置位，只能通过修改包体本身来开启。

### 4.3 观测值与位域的一致性验证

观测到的 `allToggles = 135179`，其二进制分解为：

```
135179 = 131072(2^17) + 4096(2^12) + 8(2^3) + 2(2^1) + 1(2^0)
```

即置位的位为第 0、1、3、12、17 位。而 `FeatureToggles` 的静态默认值为 `135169 = 131072 + 4096 + 1`（第 0、12、17 位），二者的差值为 `10 = 8 + 2`，对应第 1、3 位——这与"默认值在运行时被追加置位"的观测一致。**关键之处在于：位值 32（第 5 位）不在置位集合中**，因而默认及运行态下标记写入均为活跃状态，`is_mock=true` 得以出现在输出中，与前述观测完全自洽。

### 4.4 其他泄漏面的排查

在确认三个自定义键之外，本研究还排查了系统级模拟标记的泄漏可能性。通过在全部字节码中检索 `addTestProvider` / `setTestProvider` 等系统模拟定位通道的调用，结果为零——即 Portal 不通过系统级 Test Provider 机制注入，`Location.isMock()` / `isFromMockProvider()` 系统位从未被置位。此外，`classes.dex` 中出现的 `isFromMockProvider` 字符串经溯源确认仅来自高德定位 SDK 的日志输出，与标记注入无关。综上，**三个自定义 `extras` 键是唯一的标记泄漏通道**，堵住该注入点即可达成"无任何标记"的目标。

---

## 5 补丁方案设计与实施

### 5.1 方案选择

基于第 4 章的分析，存在两条可行路径：

1. **开启隐藏开关**：使位值 32 在运行时被置位。但该位 setter 为死代码，UI 与命令通道均无法触及，需额外注入置位逻辑，实现复杂且依赖运行时状态。
2. **控制流重定向**：将标记写入块之前的条件分支 `if-nez` 改写为无条件跳转 `goto`，使三个 `put*` 指令永久成为不可达代码。

本文采用第 2 条路径，理由在于：它不引入任何新逻辑、不依赖运行时状态，且改动量最小（仅 2 字节），从根本上消除标记写入。

### 5.2 字节级补丁

在 `classes2.dex` 文件中，目标条件分支的绝对偏移位于 `0x67b7fa`，其原始字节为：

```
39 0D 2D 00    ; if-nez v13, +0x2d
```

其中操作码 `0x39` 为 `if-nez`，`v13` 为承载 getter 返回值的寄存器，`+0x2d` 为跳转偏移。该跳转目标恰位于 `is_mock` 写入之后，即"位值 32 置位时跳过全部标记写入"。将其改写为：

```
29 00 2D 00    ; goto/16 +0x2d
```

操作码 `0x29` 为 `goto/16`，偏移保持不变。补丁后，无论位值 32 是否置位，控制流都会跳过三个 `put*` 指令。改写完成后经 androguard 重新解析验证，三个标记写入指令已确认不可达。

### 5.3 dex 完整性校验修复

DEX 文件头部包含两个完整性字段：位于偏移 12 的 **SHA-1 摘要**（覆盖 `[32:]` 区间）与位于偏移 8 的 **Adler-32 校验和**（覆盖 `[12:]` 区间）。直接修改字节码后必须重算二者，否则运行时（ART）在加载 dex 时会判定 `Bad checksum` 而拒绝加载。此处存在一个易错点：**Adler-32 的覆盖区间包含 SHA-1 字段本身**，因此必须严格遵循"先写 SHA-1，再算 Adler-32"的顺序，否则 Adler-32 将基于过时的 SHA-1 值计算而再次失效。本研究的正确修复顺序为：

```
1. 写入 data[12:32] = SHA-1(data[32:])
2. 写入 data[8:12]  = Adler-32(data[12:])
3. 自检：data[8:12] == Adler-32(data[12:])
```

该顺序错误曾在实际部署中导致启动崩溃，详见第 7 章。

---

## 6 重打包、签名与部署

### 6.1 APK 重建

APK 为 ZIP 格式，重建时需逐条保留原始目录项及其压缩方式，仅替换 `classes2.dex`。对 `resources.arsc`、`AndroidManifest.xml` 等条目保持 `STORED`（不压缩）方式，其余条目沿用原压缩方式，以最大程度保持与原包的结构一致。

### 6.2 对齐与签名

重建后的 APK 依次执行：

1. **对齐**：`zipalign -p 4`，确保未压缩条目按 4 字节对齐，以满足签名与运行要求；
2. **签名**：`apksigner`（v1 + v2 签名方案），使用新生成的 RSA-2048 密钥（别名 `portalpatch`）。

由于签名密钥发生变更，安装时需先卸载旧包再安装新包，并重新在 LSPosed 中启用该模块。

### 6.3 部署

部署过程涉及两项设备侧操作：其一，将签名后的 APK 通过 `adb` 安装到设备；其二，因换签名导致模块失效，需在 LSPosed 中重新启用 `portaldev` 模块并勾选 `system` 作用域。只有模块成功注入 `system_server` 后，Portal 的钩子才会生效。

---

## 7 部署过程中的两类故障及其归因

部署并非一帆风顺，其间暴露了两类与补丁本身无关、却极易归因错误的故障。本节对其进行还原与归因，既是对工程实践的记录，也为同类工作提供警示。

### 7.1 启动即崩溃：dex 校验和失效

补丁 APK 签名安装后，应用启动即崩溃，日志为：

```
ClassNotFoundException: moe.fuqiuluo.portaldev.MainApplication
Failure to verify dex 'base.apk!classes2.dex': Bad checksum
```

表面现象是"找不到 Application 类"，但经逐字节对比确认 `MainApplication` 类定义完好，真正根因是 `classes2.dex` 的头部校验和失效，导致 ART 拒绝加载整个 dex，进而使定义于其中的类全部"消失"。这暴露了 5.3 节所述校验和顺序问题——第一次修复时先算了 Adler-32、后写 SHA-1，导致 Adler-32 基于过时的 SHA-1 再次失效。按正确顺序重算后，崩溃消除。

### 7.2 模块"面板已激活但 App 未激活"：增量安装路径漂移

另一类故障表现为：LSPosed 面板显示模块已激活，但 Portal App 提示未激活。经排查发现，其根因在于 `adb install` 默认走了**增量安装（Incremental Install）**路径：APK 数据由系统增量服务托管，重启时才落盘迁移，其安装路径在开机早期尚未稳定。`system_server` 在启动早期按 LSPosed 缓存的旧路径加载模块时，文件尚未就位，导致模块加载被跳过；而 `com.android.phone` 等较晚启动的进程则加载成功。修复方式为：改用 `--no-incremental` 完整重装使 APK 稳定落盘，并在 LSPosed 未自动刷新路径时，手动修正其配置数据库中记录的模块路径，随后重启。

---

## 8 验证结果

在模块成功注入 `system_server` 之后，验证分两个层面进行：

**其一，注入层面**：LSPosed 日志显示 `portaldev` 在 `system_server` 中产生了大量钩子调用记录（数千行），`LocationManagerService`、`WifiServiceImpl`、`PhoneInterfaceManager` 等钩子均正常就位，确认模块已实际生效。

**其二，标记层面**：对定位输出进行全量 dump 检查，`portal.enable`、`allToggles`、`is_mock` 三个键的命中数均为 0。考虑到设备 `dumpsys` 对定位 extras 的输出能力已在对照实验（第 9 章）中得到确认（即 extras 确实可见），因此该"零命中"是可采信的实证结果，而非打码造成的假阴性。补丁后的 Portal 输出不再携带任何自定义标记。

---

## 9 附带研究：第三方模块标记行为对照

在验证过程中，设备上同时存在另一款模拟定位模块 `com.yinshibai.location`。关于其"是否也会注入 `is_mock`"，此前存在分歧。为澄清这一分歧，本文对其 APK 进行了同等的静态分析，并以实测加以验证。

### 9.1 静态分析：读写路径并存

对 `com.yinshibai.location` 的交叉引用分析表明，其并非 Portal 系分支（不含 `Lucifer` / `fuqiuluo` / `allToggles` 等特征），但 `is_mock` 字符串在其字节码中同时出现在**两类方向相反的路径**上：

- **写入路径**（`Lo40.a()`）：构造模拟定位时执行 `putBoolean("is_mock", true)`，并附带 `isFake`、`is_filter`、`gpsQuality`、`from` 等字段；
- **清理路径**（`xu1.b()` / `sd1.c()`）：输出定位前执行 `Bundle.remove("is_mock")` 与 `Bundle.remove("mockProvider")`，即主动清除模拟痕迹。

这意味着"是否在输出中看到 `is_mock`"取决于实际走的是哪条路径，不能仅凭字符串的存在与否做出断言。

### 9.2 实测验证：清理路径生效

在干净环境（停用 portaldev、仅启用 yinshibai 并注入 `system_server`）下开启其模拟，定位输出为：

```
Bundle[{satellites=27, lat=..., lng=...}]
```

全部 8 个定位 Bundle 仅含 `satellites`、`lat`、`lng` 三键，`is_mock`、`isFake`、`is_filter`、`gpsQuality`、`mockProvider` 的命中数均为 0。进一步，写入路径 `Lo40.a()` 中硬编码 `satellites=12`，而实测为 `27`（真实卫星数），说明实际生效的并非该写入路径，清理路径（或另一构造路径）才是输出链上的活跃环节。

**结论**：yinshibai 的模拟输出在实测中不携带 `is_mock` 标记，先前"其会注入 `is_mock`"的判断系基于未生效的写入路径得出，应予修正。这也印证了本研究的核心方法论——静态分析揭示的"能力"必须与运行时"实际行为"相区分。

### 9.3 附带发现

排查过程中还发现两点：其一，设备上另有一款模块 `com.vincenthzr.locationspoofer`，其作用域直接覆盖目标应用进程（高德地图、微信、GMS 等），不走系统链路，模拟效果可能与系统层模块叠加；其二，yinshibai 会主动探测 Portal 服务（日志中出现 `Someone try to find Portal` 告警），二者存在潜在的相互感知。

---

## 10 结论

本研究通过对 PortalDev 模拟定位框架的系统静态分析，得出以下结论：

1. **注入点唯一**：`portal.enable`、`allToggles`、`is_mock` 三个标记键在全 APK 中仅有唯一写入点，位于 `LuciferLocationData` 的混淆方法内。
2. **控制机制清晰**：标记写入由 64 位功能位域 `FeatureToggles` 的位值 32（`0x20`）控制，该位为作者预留的"隐藏标记"开关，但其 setter 为死代码，无法在运行时开启。
3. **补丁方案有效**：通过将 `if-nez` 改写为 `goto/16`（2 字节补丁），三个标记写入成为死代码；配合 dex 校验和修复、重打包与签名，产出的补丁包在真实设备上稳定运行且输出无任何标记。
4. **方法论启示**：静态分析揭示的能力需与运行时行为相区分；部署层（签名、增量安装、框架路径缓存）的故障易被误归因于补丁本身，需分层排查。

---

## 11 局限性与展望

本研究的局限性及后续方向包括：

- **范围限定**：分析聚焦于三个自定义 `extras` 键，未对 Portal 的其他隐蔽通道（如 Wi-Fi/基站信息改写的一致性、时间戳与传感器数据等旁路指纹）做系统性审计，这些维度在对抗更严格的环境检测时同样值得关注。
- **版本特定性**：补丁针对 `2.0.7.r92`（commit `587cfab`）这一特定构建，方法名混淆与位域布局在不同版本间可能变化，偏移值不具备跨版本可移植性。
- **对抗的对称性**：标记的消除本质上是与检测方的"攻防"，随着检测手段升级，仅去除 `extras` 级标记可能不足以应对多维指纹识别，需在更广泛的侧信道层面综合防护。

---

## 产物文件

本研究的实际产出文件位于 [`artifacts/`](./artifacts/) 目录，包括可直接安装的签名版补丁包、未签名补丁包、补丁后的 `classes2.dex` 及其签名附属文件。各文件的用途与 SHA-256 校验和详见 [`artifacts/README.md`](./artifacts/README.md)。出于安全考虑，签名私钥未随仓库发布。

---

## 附录

### 附录 A 关键代码定位与补丁数据

| 项 | 值 |
| --- | --- |
| 标记写入方法 | `Lmoe/fuqiuluo/location/LuciferLocationData;->为什么这么冷冷清清呢？(Landroid/location/Location;)Landroid/location/Location;` |
| 控制位 getter | `Lmoe/fuqiuluo/xposed/FeatureToggles;->時間を稼ぐのはいいが別にアレを倒してしまっても構わんのだろう？()Z` |
| 控制位 setter（死代码） | `Lmoe/fuqiuluo/xposed/FeatureToggles;->自分の努力のために謝らないで、自分の努力ができません(Z)V` |
| 补丁文件偏移 | `classes2.dex` @ `0x67b7fa` |
| 原始字节 | `39 0D 2D 00`（`if-nez v13, +0x2d`） |
| 补丁字节 | `29 00 2D 00`（`goto/16 +0x2d`） |
| 标记写入序列 | `putLong("allToggles")` → `putBoolean("portal.enable")` → `putBoolean("is_mock")` |

### 附录 B FeatureToggles 功能位域映射

下表列出 `FeatureToggles` 中各 getter/setter 方法与其对应位值的映射（方法名经混淆）。其中位值 32 为本文重点关注的"隐藏标记"开关。

| 位值 | getter（`()Z`） | setter（`(Z)V`） |
| --- | --- | --- |
| 1 | `含蓄隽永，优美如画…` | `setLogMode` |
| 2 | `美しく最後を飾りつける…` | `認めたくないものだな…` |
| 4 | `いっぺん？？？死んでみる？` | `あきらめたらそこで試合終了だよ` |
| 8 | `投手としてじゃなくても…` | `光が強いほど…` |
| 16 | `外見をあまり気にしないで…` | `一緒にいろんな所へ行って…` |
| **32** | **`時間を稼ぐのはいいが…`** | **`自分の努力のために…`** |
| 64 | `月を代表してあなたを滅ぼします` | `幸福は不老不死ではなく…` |
| 128 | `間違ったことをしたら…` | `たとえ千件のメールを…` |
| 256 | `もし叶うなら…` | `世界は美しいです…` |
| 512 | `大部分の人は大きく…` | `私たちが経験した平凡な日常は…` |
| 1024 | `人の悪口を言うと…` / `彼らの後ろ盾が神なら…` | — |
| 2048 | `何もしないと何も始まらないです` | `友達になるには資格はいらない` |
| 4096 | `宝宝肚肚打雷了` | `雷雷宝宝打肚肚` |
| 8192 | `其实这软件不是为了赚钱` | `开源之后很多二改滥用` |
| 16384 | `a` | `a` |
| 32768 | `b` | `b` |
| 65536 | `c` | `c` |
| 131072 | `d` | `d` |

另有整体写入入口 `悲しみから逃げて…(J)V`，其语义为 `setFeatureFlags(value | 2048)`，是 App 端可整体覆盖位域的唯一通道。

### 附录 C dex 头部校验和修复要点

| 字段 | 偏移 | 覆盖区间 |
| --- | --- | --- |
| Adler-32 校验和 | `[8:12]` | `[12:]` |
| SHA-1 摘要 | `[12:32]` | `[32:]` |

**正确顺序**：先写 SHA-1，再写 Adler-32；Adler-32 覆盖区间含 SHA-1 字段，顺序颠倒将导致校验和再次失效。

---

## 参考文献

1. Pankratz, D., & Piatkiewicz, A. *Androguard: Full Python tool to play with Android files*. <https://github.com/androguard/androguard>
2. Google. *Dalvik Executable Format*. Android Open Source Project 文档.
3. Google. *APK Signature Scheme v2*. Android 开发者文档. <https://source.android.com/docs/security/features/apksigning/v2>
4. LSPosed 团队. *LSPosed Framework*. <https://github.com/LSPosed/LSPosed>
5. Android Open Source Project. *LocationManagerService* 源码与 `Location` 数据结构文档.
