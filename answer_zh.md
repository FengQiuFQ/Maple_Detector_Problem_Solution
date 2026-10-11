# Maple检测项解决方案（跟进 1.3.0 版本）中文版

> 整理：Maple ｜ 最后更新：2026-10 ｜ 核对版本：1.3.0（检测项总数：日常 114 / 游戏 131 / Ultra 日常 211 / Ultra 游戏 332）
> 致谢名单 / Credits：[致谢清单](thanks.md) ｜ 仅供参考，具体结果因设备/系统/内核环境而异。
> 标有「?」的内容未经确认，仅供排查参考。
> 文档链接：[github](https://github.com/FengQiuFQ/Maple_Detector_Problem_Solution/)

---

<details><summary><b>声明</b></summary>

**1. 合规与用途约束**

本文档及 Maple 检测器提供的环境检测与处置方案，**仅面向 Android 技术爱好者**，专用于**技术学习、环境研究与探讨**。**严禁**将本文档中的任何方案用于绕过应用反作弊、规避风控机制、游戏作弊等一切违法违规场景。因违规使用而产生的任何后果与责任，均由使用者自行承担。

**2. 风险提示**

文档中记录的所有操作流程、自动化脚本、模块配置均仅供技术参考。涉及修改系统镜像、替换硬件密钥、嵌入内核模块（KPM）、执行底层 Root 指令等操作，均存在**极高的不可逆风险**，可能导致设备变砖、无限重启或用户数据丢失。所有操作风险均由使用者本人承担。

**3. 结果说明**

**界面上看不到某一条，说明本次没有取到可比对的数据，不代表你已经通过该项。**

**4. 误报说明**

受 ROM、内核、Root 管理器、模块组合影响，任何检测项都存在概率性误报。如果你确认某条是误报，请按文末方式提交检测依据与环境信息。

</details>

---

## 说明与反馈

<details><summary><b>自行尝试但仍然无法通过的检测</b></summary>

请提交 Issue 并提供：

1. 命中的**检测项完整详情**（点开条目后的「检测依据」，不要只发颜色或标题）；
2. 系统版本、内核版本（`uname -a`）、安全补丁日期；
3. Root 管理器与版本、模块列表（`ls /data/adb/modules`）；
4. Xposed / LSPosed 模块列表与作用域；
5. 是否使用改机 / PIF / TEE 模拟 / 应用隐藏类模块。

不要只发「检测到第三方」或「检测到 Root」。没有检测依据无法判断是哪一条来源异常。

</details>

<details><summary><b>通用排查方法</b></summary>

1. **先重启再复测**：瞬时进程、动态 maps、开机挂载尚未收敛、时间同步都会影响结果。刚开机请等 20 秒～5 分钟。
2. **最小集复测**：只保留 Root 管理器与必需的 Zygisk 实现模块，重启后扫描，确认命中是否还在。
3. **二分定位**：每次只启用一个模块 → 重启 → 复扫，逐步收敛到具体触发项（每次只改一个变量）。
4. **挂载类条目优先处理**：元模块、Zygisk 实现模块的排除策略（「仅还原挂载」）、SusFS / PathMask 类隐藏。
5. **密钥 / TEE 类条目**：修改 `keybox.xml`、`target.txt`、安全补丁同步之后**必须重启**再复测。
6. **不要叠加多个隐藏方案**：多套隐藏互相冲突会制造新的矛盾项，反而更容易被判定。
7. **少数条目属侧信道 / 不稳定检测**：同一环境多次扫描结果可能不一致，先排除偶发再定位。

⚠️ **安全提示**：任何「更换 keybox / 更换 RKP 密钥 / 改回锁状态 / 改安全补丁同步」的操作都会改变设备的密钥与认证（attestation）状态，**不一定可逆**；操作前请评估风险，必要时先备份相关目录。

</details>

---

## 目录

- [序章](#序章)
- [用语说明](#用语说明)
- [一、Root 权限与 SELinux 检测](#一root-权限与-selinux-检测)
- [二、TEE 与密钥证明检测](#二tee-与密钥证明检测)
- [三、挂载与命名空间检测](#三挂载与命名空间检测)
- [四、注入、Hook 与运行时篡改检测](#四注入hook-与运行时篡改检测)
- [五、环境、进程与文件检测](#五环境进程与文件检测)
- [六、设备身份、属性与内核特征检测](#六设备身份属性与内核特征检测)
- [七、侧信道与视图一致性检测（新增）](#七侧信道与视图一致性检测新增)
- [八、硬件与传感器交叉验证](#八硬件与传感器交叉验证)
- [内建清单与属性](#内建清单与属性)
- [最后排查顺序](#最后排查顺序)

---

## 序章

<details><summary><b>最小完美隐藏环境构建指南</b></summary>

- 真解锁设备：**密钥模块 + Zygisk 实现模块 + 应用隐藏模块**
- 假回锁 / 免解 / 自签设备：**Zygisk 实现模块 + 应用隐藏模块**

对 **APatch / FolkPatch** 用户，另外建议加载 **[NoHello.kpm](https://t.me/welikeandroid)** 以防侧信道检测；新版管理器可能内置 SELinux hook 功能（需手动开启），旧版可再加 **[SELinux_Hook.kpm](https://t.me/APatch_nightly)**。

对 **KernelSU / SukiSU** 用户，请更新管理器到最新 CI 后重新修补镜像，并开启管理器里的「隐藏 SELinux 修改 / SELinux Hide」；AVC 日志类检测还需要管理器侧的日志伪装或内核审计补丁。

> **少即是多**：一个稳定的隐藏方案 + 干净的属性，比叠三层隐藏更容易通过。

</details>

<details><summary><b>相关模块推荐（排名不分先后）</b></summary>

#### 密钥模块（TEE 证明）

- [TEESimulator-RS](https://github.com/Enginex0/TEESimulator-RS)：继 TrickyStore 之后最知名的密钥模块，更新较勤。
- [OhMyKeymint](https://github.com/qwq233/OhMyKeymint)：行为更接近 AOSP，IO 开销可能更低。
- [Tricky-addon-Enhanced](https://github.com/Enginex0/tricky-addon-enhanced)：TS / TEES-RS 可用的 WebUI 拓展模块。
- 原版 TrickyStore 更新较慢，功能已落后于上述模块，不建议继续使用。

#### Zygisk 实现模块

- [Zygisk-Next](https://github.com/Dr-TSNG/ZygiskNext)：使用最广泛的独立 Zygisk 实现。
- 开启「使用 Zygisk 连接器」与「使用匿名内存」；排除策略优先「仅还原挂载」。

#### 应用隐藏模块

- [HMA-OSS](https://github.com/frknkrc44/HMA-OSS)：Zygisk 版 / Xposed 版双版本可选，用以隐藏 Root 管理器、Xposed 模块与风险应用。

#### 元模块 / 挂载隐藏

- [Hybrid-Mount](https://github.com/Hybrid-Mount/meta-hybrid_mount)：较广泛使用的第三方元模块。
- [NoMount](https://github.com/maxsteeel/nomount)：面向 VFS 的路径重定向框架。
- [LKM-PathMask](https://github.com/Andrea-lyz/LKM-PathMask)：按路径隐藏（**并非专注环境隐藏，请慎用**）。

#### SELinux 策略隐藏

- [DirtySepolicy](https://github.com/LSPosed/DirtySepolicy)：SELinux 策略 oracle 侧的实现参考。
- [selinux_maf_fork](https://github.com/741afb7/selinux_maf_fork)：适配 Magisk 系的 access filter。
- [selinux_hook](https://github.com/Admirepowered/selinux_hook)：KernelSU / APatch 系常用。

#### 其它工具

- [Inode-Hijacker](https://github.com/YiJieqwq/Inode-Hijacker)：修正 `/data/local/tmp` 的 inode 异常。
- [ADB-Trace-Cleaner](https://github.com/YiJieqwq/ADB-Trace-Cleaner)：清理 USB 调试痕迹。
- [FuseFixer](https://github.com/5ec1cff/FuseFixer)：修复共享存储读取受限导致的风险应用扫描盲区。

</details>

<details><summary><b>模块正确配置</b></summary>

#### 密钥模块

a. 把 Maple 加入目标应用列表（`/data/adb/tricky_store/target.txt`，或使用 WebUI 配置，可写 `com.maple.detect!` 表示强制处理）；
b. 正确配置安全补丁日期（或直接删除 `security_patch.txt`）；
c. 正确配置 boot hash（正常情况下会自动设置）。

#### Zygisk 实现模块

开启「使用 Zygisk 连接器」与「使用匿名内存」；「仅还原挂载」与 Root 管理器的「内核级卸载模块」二选一，不要同时开。

#### 应用隐藏模块

- **黑名单模式**：被开启此模式的应用，看不到黑名单模板内的应用；
- **白名单模式**：被开启此模式的应用，只能看到白名单模板内的应用。

对 Maple 生效前请先确认隐藏范围**只包含需要隐藏的应用**，不要把系统组件（输入法、无障碍服务、应用商店）一起隐藏 —— 那会直接触发 Maple 的可见性交叉类条目。

</details>

---

## 用语说明

<details><summary><b>用语介绍与规范</b></summary>

#### 真解锁设备

Bootloader（ABL）解锁标志被真实置位，系统放行未经签名校验的底层镜像。解锁状态会如实反映在系统底层属性与 KeyMint 硬件级凭据（Attestation）中。

#### 假回锁设备

在真解锁设备上，于启动链早期植入并执行自定义 payload，改写内存中的解锁状态并上报，使设备对外呈现「已锁定」，而实际仍加载被篡改的 boot / init_boot 镜像。

#### 免解设备

ABL 保持真实锁定状态下，通过 Android / 用户空间提权漏洞（常伴随 SELinux 被置为 permissive）取得 Root。此类 Root 通常不持久、需每次启动重新利用。

#### 自签设备

设备启动链信任的签名密钥可被第三方获得或复现，用户可用该密钥自行签名被篡改的镜像，在 ABL 保持锁定状态下通过校验运行。

#### Root 管理器

完整的权限管理组件集合：面向用户的 App、用户态守护服务，以及部署在内核或 ramdisk 中的持久化钩子或内存补丁。

#### 元模块

模块管理器类型的顶层模块，本身不提供伪装功能，核心职责是管控子模块与提供挂载功能；一台设备同时只能安装一个元模块。

#### 密钥模块

在 keystore2 内拦截 / 替换 KeyMint 的 attestation 路径，用 keybox 生成「设备已锁定」证明的模块。

#### Zygisk 实现模块

提供 Zygisk runtime 的模块，负责把代码注入 Zygote / app 进程，并为其它 Zygisk 模块提供运行环境。

#### 应用隐藏模块

以「包可见性」为操作对象，在目标进程或系统进程里拦截包查询链路，按配置隐藏选中应用可见性的模块。

</details>

---

## 一、Root 权限与 SELinux 检测

<details><summary><b>检测到Root权限/Selinux修改（日常基础）</b></summary>

#### 检测方式

在 **app_zygote 域**内读自身 SELinux context，并以哨兵 context 建立正控制 / 负控制 / 文件控制三组对照，走 raw `selinuxfs` 写入 `/sys/fs/selinux/access` 并校验内核回填结果；同时探测 `shell -> su` 转换、`magisk` / `ksu` / `apatch` 域与 `ksu_file` / `lsposed_file` / `magisk_file` 读取是否仍被 live policy 接受。

#### 解决办法

- **KernelSU / SukiSU**：更新管理器与内核 / LKM 到最新 CI，开启「隐藏 SELinux 修改」，重新修补并刷入后重启。
- **APatch / FolkPatch**：加载 SELinux 隐藏类 KPM（[NoHello.kpm](https://t.me/welikeandroid)、[selinux_hook](https://github.com/Admirepowered/selinux_hook)、[selinux_maf_fork](https://github.com/741afb7/selinux_maf_fork)）。
- **Magisk**：Magisk 系需内核带审计 / 策略伪装支持，或用 KPM 方案；仅换管理器无效。

#### 关系

与 `SELinux状态检测`、`SELinux策略检测`、`SELinux上下文检测`、`设备取得Root权限/免解` 属于同一判据族，常同时命中。

#### 备注

普通应用读不到 `/sys/fs/selinux/enforce` 是**正常**的，不会因读不到而命中；只有策略视图确实可以被本进程读到时才会命中。

</details>

<details><summary><b>检测到KernelSU特征</b></summary>

#### 检测方式

在应用预加载阶段，对本进程自身的运行时状态做多路交叉核对：分别比对各路观测在同一条件下的取值是否自洽、同一份输入的两种解析路径是否得到一致结果、进程基线是否仍与父进程保持派生关系，以及存储与策略视图是否出现互相矛盾的表现。任一路出现不一致即判定命中。

各路子判据彼此独立，命中哪一路就显示哪一路；读取不到一律按**未知**处理，不作为"未发现"。

#### 解决办法

- **隐藏应用**：本检测项的各路观测都以本应用进程自身的运行时状态为观测点，把 Maple 加入隐藏列表（隐藏应用列表 / HMA 一类工具）可直接阻断观测面。
- **更新最新 CI 版**：把 Root 管理器与内核 / LKM 一并更新至最新 CI 版并重启。

#### 关系

与 `KernelSU检测 / KernelSU原生接口检测 / KernelSU活动LKM / KernelSU内核与SELinux标记`、`检测到Root权限/Selinux修改`、`设备取得Root权限/免解` 属于同一判据族，常同时命中。

#### 备注

- 本项内置 6 个子判据（侧信道运行时特征、策略视图一致性特征、策略状态特征、运行环境特征、进程基线特征、存储布局特征），**任一命中即为严重**。
- 更新至最新 CI 版是必要的：相关修正先进 CI 构建、后进正式发行版，停留在旧发行版号上比对会仍然命中。
- 修正这一条不代表其他时延、挂载或策略类检测同时消失；**卸载管理器**或**不授予本应用 Root 权限**均不会改变这一判据。

</details>

<details><summary><b>KernelSU检测 / KernelSU原生接口检测 / KernelSU活动LKM / KernelSU内核与SELinux标记</b></summary>

#### 检测方式

读取 KernelSU 的版本 / 特性接口、`ksu_file` 策略标记、活动 LKM 与内核模块残留、SELinux 策略中的 KernelSU 专有类型，并与管理器包、守护进程、`/data/adb/ksu` 结构交叉。

#### 解决办法

- 更新 KSU 管理器到最新 CI 并重新修补（LKM 模式）或重新集成（GKI / Non-GKI 模式）。
- 开启管理器内的隐藏功能，并把 Maple 加入排除列表 / 应用隐藏模块黑名单模板。

#### 关系

同族判据现统一归并到 `检测到KernelSU特征`，命中时以该项上报。

#### 备注

管理器 App 本身可被应用隐藏模块隐藏；但策略标记、内核模块残留属于内核侧，隐藏模块 App 不会改变这些证据。

</details>

<details><summary><b>KernelSU免解设备/越狱模式（综合，游戏模式显著）</b></summary>

#### 检测方式

汇总 KernelSU Late-load、活动 LKM、SELinux 标记、持久化工件、Metamodule、Magica 事务窗口与越狱能力组件，并核对管理器签名与 Magica 组件是否同时存在。

#### 解决办法

- 不建议使用免解（越狱）模式。
- KSU / SukiSU：更新管理器与内核 / LKM 到最新 CI 并重新修补后开启 `selinux_hide`。
- 关闭 Late-load、移除 Metamodule 与持久化脚本，恢复未集成 KernelSU 的官方内核。

#### 关系

与 `KernelSU Late-load脚本环境`、`KernelSU越狱持久化工件`、`KernelSU Metamodule挂载`、`Magica启动事务窗口`、`Magica瞬时进程`、`免解机型Prop交叉检测` 同族；其中免解加载环境相关的判据已并入 `检测到KernelSU特征`。

</details>

<details><summary><b>KernelSU Late-load脚本环境 / KernelSU越狱持久化工件 / KernelSU Metamodule挂载 / Magica启动事务窗口 / Magica瞬时进程</b></summary>

#### 检测方式

Native 侧扫描越狱脚本位、`late-load` / `--magica` 命令、`/data/adb` 下的持久化工件、Metamodule 的 source=KSU 挂载来源，以及 Magica 属性窗口 + 5555 端点的瞬时组合。

#### 解决办法

关闭 Late-load / 越狱模式，删除 `/data/adb` 下对应残留与启动脚本，卸载 Metamodule 与 Magica 组件，恢复官方内核与启动镜像后重启。

#### 关系

免解加载环境与存储栈日志相关的判据已并入 `检测到KernelSU特征`，命中时以该项上报。

#### 备注

Magica 相关条目是**瞬时**的：命中与否取决于扫描时窗口是否打开，多次扫描结果可能不同。

</details>

<details><summary><b>su二进制检测 / 运行中su进程 / 运行中Root/外挂进程 / Root进程检测 / 可见内核外挂运行态</b></summary>

#### 检测方式

在常见路径下检测 su / 同类二进制；对可见进程做身份四视图交叉（comm / argv[0] / exe / status Name，至少两路一致才算 su），并结合 `attr/current` 是否落在 Root 域、内核任务名与外挂守护进程特征。

#### 解决办法

- 移除不需要的 su 二进制与其执行位（注意 iQOO / vivo 的 `/apex/com.android.virt/bin/su` 属误报点，见备注）。
- 停用外挂 / 脚本工具与常驻守护进程。
- 通用原则：**不要给检测器 Root 权限**。

#### 备注

新机型上普通应用通常只能看到自己 UID 的进程，「可见进程里没有 Root 进程」属正常结果，不代表已经干净；这类条目在多数新机型上不会有结论。

</details>

<details><summary><b>隐藏路径可见性检测（1.3.0 新增）</b></summary>

#### 检测方式

对 su 类只读系统路径同时做 `lstat`、`open` 与父目录列举三种观测，**任一结果互相矛盾**即命中：元数据存在但打开返回「不存在」、或元数据存在但目录列表里没有它 —— 这是路径隐藏方案（PathMask / SusFS path hide）的典型特征。

#### 解决办法

- 真实存在：卸载对应工具并清理残留。
- 出现矛盾：停用路径隐藏 / PathMask / SusFS 路径伪装与文件 API Hook 后重启复测。

#### 关系

与 `路径存在性三路核对`、`路径证据互斥`、`Java与Native路径视图分歧` 同族。

</details>

<details><summary><b>SELinux状态检测 / SELinux策略检测 / SELinux上下文检测 / Policy序列号检测 / SELinux脏策略检测</b></summary>

#### 检测方式

SELinux 文件系统状态、enforce 与 policy 文件可读性、策略中是否残留 magisk / ksu / apatch / zygisk 关键字与专有类型、当前进程是否进入 Root 域、策略序列号是否被替换。

#### 解决办法

- 恢复 SELinux Enforcing（卸载把 SELinux 置为 permissive 的内核模块）。
- 恢复未注入 Root 策略的官方内核 / boot 与 SELinux 策略。
- 停止对 Maple 生效的上下文修改与注入。

#### 备注

「enforce 文件存在但不可读」是普通应用的正常状态，属于未知情况，不作为命中依据。

</details>

<details><summary><b>内核setresuid行为探针</b></summary>

#### 检测方式

以子进程实测内核在 `setresuid` 等敏感路径上的行为（子进程是否被杀、权限是否异常），用于识别对内核行为做了改写的 SusFS / Root 内核模块。

#### 解决办法

卸载修改内核行为的 SusFS / Root 内核模块，恢复官方内核。

#### 备注

该条为强证据但依赖内核实现细节，不同内核版本表现不同；未取到预期行为时不会有结论。

</details>

---

## 二、TEE 与密钥证明检测

<details><summary><b>TEE密钥有效性检测（1.3.0 新增）</b></summary>

#### 检测方式

生成一次性硬件密钥并**实测签名**，再用证书链叶证书验签，核对挑战绑定、证书链签名、受信任根与密钥 / 证明安全级别。只要设备声明了硬件密钥能力，而密钥生成失败、叶证书与密钥不匹配、链不可解析、未绑定本次挑战、或密钥落到软件实现，都会命出。

#### 解决办法

- 恢复正常 TEE：恢复官方固件与可信启动链，确保 keystore2 / KeyMint 未被拦截。
- 使用密钥模块的用户：更新密钥模块（[TEESimulator-RS](https://github.com/Enginex0/TEESimulator-RS) / [OhMyKeymint](https://github.com/qwq233/OhMyKeymint)）、正确配置 keybox、安全补丁与 boot hash 后重启。
- 设备真处于「爆 TEE」状态（无模块也会命中）：这是硬件 / 固件层面的状态，只能通过恢复官方固件或送修解决。

#### 关系

与 `Android Key Attestation可用性`、`Attestation链结构一致性`、`证明链根与实现级别`、`TEE授权标签对照` 同族。

#### 备注

密钥 / 证明状态在一次开机内不会变化；同一环境下多次扫描结果应一致。若每次结果不同，优先排查 Keystore Hook 类模块。

</details>

<details><summary><b>Android Key Attestation可用性 / Attestation链结构一致性 / 证明链根与实现级别 / 密钥绑定与随机挑战</b></summary>

#### 检测方式

生成带随机挑战的密钥并解析 KeyDescription：链长度、链签名、是否锚定正式根、公钥是否与叶证书一致、挑战是否回填、证明与密钥安全级别是否自洽。

#### 解决办法

停用 TrickyStore / TEE / KeyMint 模拟或拦截模块，恢复官方 Keystore / KeyMint 与可信启动链；使用密钥模块时按「模块正确配置」完成 target 列表、安全补丁与 boot hash。

#### 关系

与 `TEE密钥有效性检测`、`发现TrickyStore/类似模块`、`TEE 伪造`、`Tee伪造(2)` 同族。

</details>

<details><summary><b>TEE授权标签对照 / TEE设备标识标签 / KeyUsage与KeyPurpose一致性 / 签名算法与签发钥一致性 / KeyMint占位层级不一致 / KeyMint回滚抵抗授权</b></summary>

#### 检测方式

对证明证书链做多类标签一致性校验：叶证书 KeyUsage 与扩展内 KeyPurpose 是否矛盾、叶证书签名算法与签发钥算法是否一致、软件专属标签是否出现在 teeEnforced、厂商占位 KeyMint 标签是否仍能签发、回滚抵抗声明是否异常、敏感设备标识标签是否未被拒绝。

#### 解决办法

更换 / 更新密钥模块并重启复测；仍命中说明模块实现与当前系统 KeyMint 版本不匹配，尝试 [TEESimulator-RS](https://github.com/Enginex0/TEESimulator-RS) 或 [OhMyKeymint](https://github.com/qwq233/OhMyKeymint) 互换。

#### 备注

部分标签差异在小版本更新后会自然消失（模块跟随系统更新）；这类条目属「证明实现一致性」，不等价于 Root。

</details>

<details><summary><b>TEE RootOfTrust启动状态 / Bootloader状态检测 / OEM解锁检测 / Bootloader属性扫描 / 启动状态异常</b></summary>

#### 检测方式

读取 bootloader 锁定状态相关属性（`ro.boot.flash.locked`、`ro.boot.verifiedbootstate`、`ro.boot.vbmeta.device_state`、`ro.secureboot.lockstate` 等）并与可信证明报告中的 RootOfTrust（deviceLocked / verifiedBootState）交叉。

#### 解决办法

- **真解锁设备**：使用密钥模块对检测器隐藏解锁状态，并把 Maple 加入 target 列表；或恢复完整官方固件后回锁。
- **假回锁 / 免解 / 自签设备**：对外本就是「已锁定」，通常不会命中此项。
- 属性被改乱（例如 `ro.secureboot.lockstate=unlocked` 残留）：`resetprop -n ro.secureboot.lockstate locked` 或卸载改机 / 属性伪装模块后重启。

#### 备注

恢复原厂并回锁 BL 属于**不可逆**操作，请先确认所有分区原厂且设备支持，避免变砖。

</details>

<details><summary><b>Attested安全补丁级别 / Vendor补丁级别交叉验证 / 安全补丁交叉验证 / 安全补丁年限 / TEE密钥创建时间</b></summary>

#### 检测方式

把证明证书里的 OS / Vendor / Boot 补丁级别与运行时真实属性交叉，并检查安全补丁年限与密钥创建时间是否自洽。

#### 解决办法

- 使用密钥模块：在 `/data/adb/tricky_store/security_patch.txt` 写入与系统一致的补丁日期，或直接删除该文件让模块自动取值。
- 恢复真实系统属性，确保系统与 vendor / boot 分区来自同一固件版本。
- 部分厂商（含国内定制 ROM）构建时间与补丁时间本身不一致，未解锁设备也可能出现差异，可先重测确认。

#### 关系

与 `构建完整性检测`、`安全补丁年限` 同族，属「属性与证明的时间线一致性」。

</details>

<details><summary><b>Boot哈希一致性</b></summary>

#### 检测方式

比对 boot / vbmeta 的 hash（如 `ro.boot.vbmeta.digest`）与证明中的 VerifiedBootHash。

#### 解决办法

使用密钥模块的插件（TA）配置正确 hash；真解锁设备的 hash 常变为全 0，需从可信设备取得正确值后写入。

#### 备注

hash 值随厂商固件版本变化，不要跨版本套用。

</details>

<details><summary><b>Attestation证书吊销状态（CRL）</b></summary>

#### 检测方式

取证明证书链中叶证书的序列号，与 **Google 官方证明状态列表**（`android.googleapis.com/attestation/status`）比对，命中已吊销证书即命中。离线 / 超时 / 被拦截时该项不会显示，也不会把「取不到列表」当成未吊销。

#### 解决办法

- 更换为合法且未吊销的设备证明环境（keybox）；公开渠道流传的 keybox 很可能已在吊销列表中。
- 恢复厂商官方方案：完整官方固件 + 未解锁启动链，此时证明由厂商 / Google 正式根签发。
- 网络受限环境（无 GMS / 内网）下该项不会显示，属正常。

#### 关系

与 `AOSP密钥`、`证明链根与实现级别`、`TEE密钥有效性检测` 同族。

</details>

<details><summary><b>证明链根与实现级别（AOSP密钥）</b></summary>

#### 检测方式

核对证明证书链的根是否使用 **AOSP 测试根 / 软件证明根**（而非厂商或 Google 正式根），并交叉证明安全级别是否为软件实现。非 GMS 机型的证明根本就不是 Google 根，属于正常情况。

#### 解决办法

- 换用基于正式根的密钥模块并正确配置 keybox，重启后复测。
- 恢复完整官方固件与可信启动链。
- 若设备本身就是软件证明（无 TEE，或 TEE 已损坏）：属硬件 / 固件状态，模块无法解决。

#### 备注

该条与 `TEE密钥有效性检测` 常同时命中：一个是根不对，一个是密钥 / 证明跑不到硬件。

</details>

<details><summary><b>Widevine设备ID读取 / Widevine与TEE交叉验证 / StrongBox能力声明 / Knox Warranty检测</b></summary>

#### 检测方式

读取 MediaDrm 的设备唯一 ID、安全级别（L1 / L3）与 SystemId，并与 TEE 证明结果交叉；核对系统是否声明 StrongBox、以及三星设备的 Knox warranty 位。

#### 解决办法

- 恢复官方 DRM、TEE 与完整固件。
- 声明支持 StrongBox 但结果矛盾：更新 TEE 相关模块，或恢复官方 KeyMint。
- Knox warranty 位为历史 e-fuse 状态，**通常不可逆**。

#### 备注

Widevine 为 L3 本身不是异常；只有与声明 / TEE 结果矛盾时才判定。

</details>

<details><summary><b>TEE环境不可信（Soter）</b></summary>

#### 检测方式

交叉核对 Tencent Soter 服务程序是否存在与其服务属性状态。

#### 解决办法

使用 SusFS / [LKM-PathMask](https://github.com/Andrea-lyz/LKM-PathMask) 隐藏 Soter 相关服务路径，并用应用隐藏模块对 Maple 隐藏 `com.tencent.soter.soterserver`。

#### 备注

多数设备上不会产出结论：该条只在能同时取到 Soter 服务程序与其服务属性的设备上才会判定。

</details>

---

## 三、挂载与命名空间检测

<details><summary><b>Root挂载检测 / Bind Mount专项检测 / Overlay挂载检测 / Cacerts挂载检测 / 系统分区RW检测 / 隐藏Bind-Mounts / 隐藏系统挂载</b></summary>

#### 检测方式

扫描 `mounts` / `mountinfo` 中的 magisk / `/data/adb` / KernelSU / APatch 来源、系统分区 overlay 与 bind 改写、证书目录重挂载、系统分区是否可写，以及 CPU / SoC 身份文件（`/proc/cpuinfo`、`/sys/devices/soc0/serial_number`）是否被独立覆盖挂载。

#### 解决办法

- 卸载产生挂载的模块，恢复被覆盖的系统文件，重启后复测。
- 使用 Zygisk 实现模块的排除策略「仅还原挂载」，或更换元模块（[Hybrid-Mount](https://github.com/Hybrid-Mount/meta-hybrid_mount) / [NoMount](https://github.com/maxsteeel/nomount)）。
- 恢复官方证书目录与分区可写属性。

#### 备注

普通 OEM Overlay、APEX、EROFS 与 dm 设备**本身不作为 Root 证据**（原厂设备上这些挂载大量存在）。

</details>

<details><summary><b>挂载设备一致性检测 / 挂载ID一致性检测 / 文件描述符挂载可见性 / 映射设备可见性检测（后三项 1.3.0 新增）</b></summary>

#### 检测方式

同一事实多路观测：①系统路径的 `st_dev` 设备号是否与覆盖它的挂载记录一致；②打开系统路径后 `/proc/self/fdinfo` 的 `mnt_id` 是否与挂载表覆盖记录一致；③当前进程 fd 指向的挂载是否能在挂载表中追溯到（仅在「启动时存在、现在消失」时才判定）；④内存映射文件的设备号是否能在挂载表中追溯到。

#### 解决办法

- 出现矛盾说明有挂载被隐藏 / 卸载而不一致：停用挂载隐藏（SusFS / PathMask / 元模块隐藏挂载功能）后重启。
- 若确定是某模块导致：按详情里的路径定位并卸载该模块。

#### 备注

APEX 自身用 loop 设备挂载、overlay 覆盖路径的设备号可能是下层分区、伪路径（`/dmabuf:` 等）不对应任何挂载点 —— 这些都属于正常现象，不会命中。

</details>

<details><summary><b>挂载表漂移检测 / 运行时挂载变化 / 挂载命名空间漂移检测（1.3.0 新增）</b></summary>

#### 检测方式

运行期间挂载表与挂载命名空间是否发生变化：**挂载表**看新增 / 消失 / 属性变更，**命名空间**看 `/proc/self/ns/mnt` 是否被换掉（运行中被切到另一个挂载命名空间是隐藏挂载的典型手法）；挂载ID空洞只作参考，不单独判定。

#### 解决办法

运行期挂载发生变化通常是「模块在应用运行中重新挂载 / 卸载」：把模块的挂载时机改到启动早期（`post-fs-data`），或使用「仅还原挂载」策略；若为系统本身在开机阶段挂载（如外置存储、投屏），开机后等待 20 秒～5 分钟再测。

#### 关系

与 `挂载设备一致性检测`、`挂载ID空洞检测` 同族；`挂载ID空洞` 单独**不作为命中**（原厂设备的空洞来自命名空间克隆期的并发分配，噪声很大）。

</details>

<details><summary><b>Magic Mount / Mount loophole / Mount命名空间 / PID命名空间嵌套 / 用户命名空间映射 / 根目录重定向</b></summary>

#### 检测方式

模块对系统分区的 magic mount 痕迹、`/proc/self/ns/*` 与 `NSpid` / `uid_map` 视图、`/proc/self/root` 是否被重定向，判断进程是否被 chroot / 容器化。

#### 解决办法

- 使用 Zygisk 实现模块「仅还原挂载」或更换元模块。
- 退出双开 / 云手机 / 虚拟容器，在系统正常安装路径直接运行 Maple。

#### 备注

`mount_namespace` 一条在多数设备上只给出信息，不直接判定（普通应用读不到 init 命名空间属正常）。

</details>

---

## 四、注入、Hook 与运行时篡改检测

<details><summary><b>Frida检测 / Frida文件检测 / Frida进程检测 / Frida端口扫描 / Frida候选端口主动探测 / Frida兼容协议探针</b></summary>

#### 检测方式

文件与进程名、线程名、maps 中的 frida-agent / gadget、Socket 与监听端点、以及主动回环连接 + D-Bus 兼容协议响应。

#### 解决办法

停止 `frida-server`，移除 Gadget 与注入脚本，取消对 Maple 的注入后重启。

#### 备注

`frida-server` 常驻会同时触发多条；只开调试端口但未注入时，只会命中端口类条目。

</details>

<details><summary><b>Xposed/LSPosed检测 / Xposed Class加载检测 / XposedBridge字段检测 / LSPosed Class加载检测 / LSPosed作用域检测 / LSPosed数据目录 / LSPosed运行时残留 / OAT/ODEX工件检测 / Xposed模块元数据扫描</b></summary>

#### 检测方式

Xposed 类与字段、调用栈、ClassLoader 链、已安装模块的元数据（`META-INF/xposed`）、LSPosed 数据目录与 fd / maps 残留、编译工件（OAT / ODEX）与运行时映射。

#### 解决办法

- 从 LSPosed / Xposed 作用域中取消 Maple，停用相关模块并重启。
- 若 OAT / ODEX 工件残留：卸载重装 Maple 后复测。
- 需要保留模块时，用应用隐藏模块把模块信息对 Maple 隐藏。

#### 关系

与 `框架方法本地化`、`自身方法本地化`、`ART桥接归属`、`引导类加载归属` 同族。

</details>

<details><summary><b>Zygisk/Riru检测 / Zygisk环境检测 / Zygisk模块注入检测 / Zygisk线程检测 / Zygisk进程检测 / Zygisk守护进程 / Zygisk Namespace检测 / Zygisk SELinux上下文</b></summary>

#### 检测方式

环境变量、`magisk.process`、maps 中模块路径、线程名、受限路径 SO、TracerPid、守护进程、命名空间与 SELinux 上下文异常。

#### 解决办法

关闭 Zygisk / Riru 或取消 Maple 作用域，停用对应模块并重启；或升级 [Zygisk-Next](https://github.com/Dr-TSNG/ZygiskNext) 到最新版。

</details>

<details><summary><b>Substrate检测 / ShadowHook/Dobby检测 / Hook库枚举检测 / Inline Hook检测 / SMAPS内联钩子检测 / Hook回调检测 / ECA H3</b></summary>

#### 检测方式

Hook 库 / 类名、maps 与 `dl_iterate_phdr`、函数序言被改写（inline hook opcode 对比）、smaps 中的可执行匿名段与回调表异常。

#### 解决办法

停用对 Maple 生效的 Hook 模块（SandHook / Dobby / ShadowHook / Pine / YAHFA 等）后重启复测。

#### 关系

与 `Native注入映像归因`、`匿名执行代码检测`、`签名算法与签发钥一致性` 属下无关但常同时出现。

</details>

<details><summary><b>Native注入映像归因 / 匿名执行代码检测 / 匿名RWX段检测 / 内存注入结论 / Memfd/JIT检测 / Deleted映射检测 / ART JIT deleted memfd / 隔离进程ART差分 / ART代码页磁盘校验 / 关键Native代码页完整性</b></summary>

#### 检测方式

把 maps 中的具名 / 隐藏 ELF、JNI 表 Hook、非 JIT 的可执行匿名段、可疑 memfd 与磁盘代码页做对照；主进程与隔离进程的 ART 运行时视图互相差分。

#### 解决办法

- 停止所有针对 Maple 的注入、Hook、变速与调试模块，重启后复测。
- 若为系统级代码页被修改：恢复官方 ROM；若为 Maple 自身工件：卸载重装官方 APK。

#### 备注

普通 JIT 缓存（`/memfd:jit-cache`）与 `(deleted)` 的 APK 映射属于正常现象；非 JIT 的可执行匿名段与未知 memfd 才算异常。

</details>

<details><summary><b>信号处理器完整性 / 游戏信号处理器完整性 / 异常处理器归属 / 反调试/信号检测 / 调试器检测 / Java调试器状态 / TracerPid检测</b></summary>

#### 检测方式

关键信号（SIGTRAP / SIGSEGV 等）的处理函数是否指向匿名执行区或注入模块、未捕获异常处理器是否被替换、`TracerPid` 与调试器连接状态。

#### 解决办法

退出调试器（Android Studio / LLDB / ptrace / Frida）与崩溃拦截、Hook 模块后重启。

#### 备注

若在系统日志里看到与异常处理相关的字样，不影响扫描结果与判定。

</details>

<details><summary><b>路径存在性三路核对 / 路径证据互斥 / Java与Native路径视图分歧 / proc读取双路核对</b></summary>

#### 检测方式

对 Root 框架敏感路径用三条独立内核观测（statx / openat / 目录项遍历）取证，再叠加 Java 层视角；任一路径存在、另一条确定不存在即判矛盾。`proc读取双路核对` 比较 libc 与直连系统调用读取同一 `/proc` 文件的结果。

#### 解决办法

- 真实路径：卸载对应工具并清理残留。
- 多路矛盾：停用文件隐藏、PathMask、SusFS 路径伪装或文件 API Hook 后复测。

#### 关系

与 `隐藏路径可见性检测`、`包可见性双路核对`、`Binder调用路径一致性` 同属「视图分歧」族。

</details>

<details><summary><b>Binder调用路径一致性</b></summary>

#### 检测方式

比较经 libc 与直接系统调用访问 Binder 协议版本是否连续稳定分歧（PLT Hook 残留）。

#### 解决办法

停用 Keystore / TEE 的 Binder 拦截、ioctl Hook 与服务代理模块，恢复官方服务后重启。

</details>

<details><summary><b>运行时环境变量 / 动态库路径检测 / 动态链接器检测 / 环境变量标记 / Linker符号检测 / Linker桥接符号</b></summary>

#### 检测方式

`LD_PRELOAD`、`LD_LIBRARY_PATH`、`CLASSPATH`、`PATH`、`TMPDIR` 是否含 Root / Hook 注入路径，以及 linker 命名空间与桥接符号异常。

#### 解决办法

移除注入启动器、终端包装脚本与环境变量修改模块，从系统桌面正常启动 Maple。

</details>

<details><summary><b>应用签名自校验 / 应用可调试标志 / 应用安装来源 / APK安装路径核对 / 自身APK映射一致性 / 自身native库来源 / 宿主可执行文件</b></summary>

#### 检测方式

多签名者检测、`FLAG_DEBUGGABLE`、安装来源（installer）、APK 与自身 so 的映射路径是否落在系统允许位置、宿主可执行文件是否为系统 `app_process`。

#### 解决办法

- 卸载修改版，从可信渠道安装正式 Release APK。
- 退出容器 / 分身，停用注入后重装。
- 若使用自行编译的 debug 版：`FLAG_DEBUGGABLE` 会被置位，属预期，请使用 Release 版复测。

#### 备注

该条目只显示单签名摘要供人工核对，不做在线校验。

</details>

---

## 五、环境、进程与文件检测

<details><summary><b>风险应用检测 / 隐藏工具检测 / Root管理器检测 / Hook框架应用检测 / 虚拟框架检测 / 三方环境工具 / 隐藏Manager检测</b></summary>

#### 检测方式

全量读取已安装包（含版本），与内置的风险 / 隐藏 / Root 管理器 / Hook 框架 / 虚拟框架 / 工具名单精确比对（名单见附录）。

#### 解决办法

- 只保留确实需要的工具，其余卸载。
- 需要保留时用应用隐藏模块（[HMA-OSS](https://github.com/frknkrc44/HMA-OSS)）把对应应用对 Maple 隐藏。
- 注意隐藏范围不要包含系统组件（输入法 / 无障碍服务 / 应用商店），否则会触发可见性交叉类条目。

#### 关系

与 `系统组件可见性交叉`、`包可见性双路核对`、`风险应用/黑名单包名目录检测` 同族。

</details>

<details><summary><b>系统组件可见性交叉（1.3.0 新增）</b></summary>

#### 检测方式

取当前输入法（`Settings.Secure` 的 default_input_method）与所有已启用无障碍服务所属包，与 PackageManager 的查询结果交叉：**系统认为它正在用 / 已启用，但包查询看不到它**，即命中。

#### 解决办法

在应用隐藏模块中取消对「输入法 / 无障碍服务」预设的隐藏，或把 Maple 从隐藏作用域移除后重启。

#### 备注

这条是应用隐藏模块配置过宽最直接的表现，通常与应用商店、输入法一起被隐藏时出现。

</details>

<details><summary><b>包可见性双路核对 / PM交叉方法检测</b></summary>

#### 检测方式

`getInstalledPackages`、`getPackageInfo` 与 `createPackageContext` 等不同查询路径对已知 Root 管理器包的结果是否一致。

#### 解决办法

停用 HMA、包列表过滤、虚拟容器与 PackageManager Hook 后重启；三路结果一致时按详情决定是否卸载。

</details>

<details><summary><b>无障碍服务检测 / 开发者选项检测 / ADB调试检测 / ADB端点主动探测</b></summary>

#### 检测方式

已启用无障碍服务的数量与自动化关键词（autojs / autox / 脚本 / 连点器 / 宏）、开发者选项与 USB 调试开关、ADB over WiFi，以及回环 5037 / 5555 端点是否可连接。

#### 解决办法

- 在系统无障碍设置中关闭自动化 / 连点器 / 脚本工具服务。
- 不使用 ADB 时关闭 USB 调试（`settings put global adb_enabled 0`），并关闭「无线调试」。
- 需要长期保持开启：可用 [ADB-Trace-Cleaner](https://github.com/YiJieqwq/ADB-Trace-Cleaner) 清理调试痕迹。

#### 备注

USB 调试 / 开发者选项属**用户配置**信号，按低风险单独标注，不与其他 Root 证据等权。

</details>

<details><summary><b>临时目录权限检测 / 临时目录inode检测 / 临时目录归属检测 / 临时目录时间戳检测</b></summary>

#### 检测方式

`/data/local/tmp` 的权限（默认 0771）、inode 是否偏高（阈值 10000）、属主 / 属组是否为 shell(2000)、修改时间是否晚于当前时间或状态变更时间明显晚于修改时间。

#### 解决办法

- 权限：`su -c chmod 771 /data/local/tmp`
- 属主 / 属组：`su -c chown shell:shell /data/local/tmp`
- inode 偏高（目录被删过 / 重建过）：可用 [Inode-Hijacker](https://github.com/YiJieqwq/Inode-Hijacker) 或恢复出厂；inode 在 f2fs 上天然可能偏高，属参考信号。
- 时间戳异常：删除被改过时间戳的残留后重启；不要用脚本回改时间（会引入新的状态变更差异）。

#### 备注

「目录不可访问」属未知情况，不能当作命中依据。

</details>

<details><summary><b>外挂/Root工具路径清单 / 异常文件 / Mt管理器（MT2文件夹）/异常文件 / 其他残留检测 / Root残留检测 / Root残留(扩展) / 数据残留</b></summary>

#### 检测方式

对内置的高危路径清单（见附录）逐条 `lstat`，命中即列出具体路径。

#### 解决办法

按详情里给出的**具体路径**逐一确认并删除不再需要的残留；MT2 文件夹可在 MT 管理器设置里改路径后删除旧目录。

#### 关系

与 `异常sh文件快速扫描`、`外置存储脚本快速扫描`、`调度/温控模块检测` 同族。

</details>

<details><summary><b>异常sh文件快速扫描 / 外置存储脚本快速扫描 / 零宽扫盘应用检测 / 黑名单包名目录检测 / 黑名单目录隐藏应用检测</b></summary>

#### 检测方式

在应用可访问的共享存储范围内有界扫描（深度 4、目录 256、条目 4000、1.5 秒）`.sh` 脚本、黑名单包名目录与被隐藏应用目录。

#### 解决办法

按详情查看具体路径，删除确认不再需要的脚本与目录；`/storage/emulated/0/Android/data/<包名>` 类目录异常通常来自外挂 / 多开工具的残留。

#### 备注

`.sh` 扩展名本身不证明恶意行为，命中只表示「存在脚本文件」。

</details>

<details><summary><b>系统镜像残留检测（Ultra）</b></summary>

#### 检测方式

在应用可访问的共享存储范围内有界扫描 `boot.img` / `init_boot.img` / `vbmeta.img` 等启动镜像、刷机备份，以及共享存储根目录下的异常 `.xml` 文件（Ultra 模式专属条目）。

#### 解决办法

按详情给出的路径确认来源后删除；这些镜像通常来自刷机 / 修补 boot 时的备份，异常 `.xml` 多为改机 / 模块写入的配置文件，保留在共享存储会同时带来「改机工具残留」类命中。

#### 备注

命中只表示「共享存储存在启动镜像文件」，不等于当前设备已被 Root；但刷机备份长期放在共享存储既占空间也容易被判定。

</details>

<details><summary><b>系统异常可执行文件 / 调度/温控模块检测 / 内核外挂驱动检测 / 内核模块检测 / 内核模块残留</b></summary>

#### 检测方式

`/system/bin/adb`、`/system/bin/fastboot` 等异常系统可执行文件；`/dev/cpuset/AppOpt`、`/data/encore/*`、`encore_profiler`、`yshell` 等调度 / 温控模块特征；已加载内核模块与外挂驱动名。

#### 解决办法

卸载对应调度 / 温控 / 外挂模块，恢复官方系统分区文件；官改包常见 `adb`/`fastboot`，刷回官方包或使用 [LKM-PathMask](https://github.com/Andrea-lyz/LKM-PathMask) 隐藏。

#### 备注

`/proc/modules` 不可读属正常（普通应用无权限），此时该项不会给出结论。

</details>

<details><summary><b>终端环境痕迹检测 / PTY节点归属（1.3.0 新增）</b></summary>

#### 检测方式

读取 `/dev/pts/*` 的属主与 SELinux 标签，出现 `ksu_file` / `magisk_file` / `apatch_file` 等 Root 框架标签即命中；标签不可读时不会有结论。

#### 解决办法

关闭常驻的 Root 终端 / 调试会话，停止在 Root 域里开的交互式 shell 后重启。

#### 备注

普通终端（包括应用自身开的 pty）属于正常情况；只有标签落在 Root 框架域才算异常。

</details>

<details><summary><b>进程Cgroup上下文 / 检测运行环境可疑 / 容器 / 多开 / 进程身份四元组 / 数据目录归属一致性</b></summary>

#### 检测方式

`/proc/self/cgroup` 中是否含 magisk / zygisk / ksu / apatch / lsposed / frida 标记；`Uid` / `Gid` 四元组是否全等；`dataDir` / `sourceDir` 是否符合系统规定位置；是否处于嵌套命名空间。

#### 解决办法

退出双开 / 云手机 / 虚拟容器，卸载当前副本，从可信渠道重新安装正式 APK。

#### 关系

与 `PID命名空间嵌套`、`用户命名空间映射`、`根目录重定向` 同族。

</details>

<details><summary><b>Vold隔离属性检测 / 日志缓冲区属性检测 / 核心平台API策略检测 / 隐藏方案属性检测（1.3.0 新增）</b></summary>

#### 检测方式

读取 `persist.sys.vold_app_data_isolation_enabled`、`persist.logd.size*`（非空即命中）、`persist.debug.dalvik.vm.core_platform_api_policy`，以及针对检测器的路径隐藏方案属性。

#### 解决办法

- Vold 隔离：关闭 HMA / 应用隐藏模块的「Vold app data 隔离」；残留时 `resetprop -p --delete persist.sys.vold_app_data_isolation_enabled` 后重启。
- 日志缓冲区属性：删除写入这些属性的脚本 / 模块。
- 隐藏方案属性：卸载对应的路径隐藏脚本（这类属性说明设备上跑过针对检测器的隐藏方案）。

#### 备注

`persist.logd.size*` 属性存在但值为空属于正常情况。

</details>

---

## 六、设备身份、属性与内核特征检测

<details><summary><b>模拟器属性检测 / 模拟器路径检测 / GPU模拟检测 / CPU模拟器特征 / 虚拟框架检测</b></summary>

#### 检测方式

QEMU / goldfish / ranchu / VirtualBox / VMware / Genymotion / 夜神 / 雷电等运行时证据（设备节点、属性、驱动、进程名），并结合电量 / 充电与 SIM 状态等参考面。

#### 解决办法

- 真机：停用硬件伪装、云手机 / 容器模块，恢复官方属性、内核与驱动。
- 模拟器本身无法通过改属性变成真机；在模拟器上出现此项属预期。

#### 备注

未插 SIM 卡 + 满电 + 充电状态下测试可能出现与模拟器相似的特征组合，建议换真实状态复测。

</details>

<details><summary><b>构建指纹完整性检测（1.3.0 新增）/ 分区指纹一致性检测 / Fingerprint交叉验证 / Fingerprint分量一致性检测 / Fingerprint分量交叉验证</b></summary>

#### 检测方式

①构建指纹必须满足标准分段结构（`brand/product/device:release/id/incremental:type/tags`）；②分区指纹之间只比较**构建姿态段**（release / id / incremental / type / tags），品牌与代号段不参与；③同一分区内 fingerprint 分量与独立属性交叉。

#### 解决办法

- 卸载改机 / PIF / 属性伪装模块，恢复官方属性后重启。
- 属性残留（例如指纹被改成只有品牌名）：清除对应 `resetprop` 脚本或模块的 `system.prop`。

#### 备注

小米机型的 vendor 段固定为 `mivendor_<soc>_cn` 这类通用基座，与本机代号**合法不同**，属于正常布局。

</details>

<details><summary><b>机型伪装检测 / 免解机型Prop交叉检测 / OemPropertyConsistency / 属性一致性检查 / 属性上下文目录枚举</b></summary>

#### 检测方式

OPPO / vivo 等厂商专属属性与设备标识是否自洽（该有时没有 / 不该有却有）、关键属性是否为空或来源异常、属性上下文目录是否被额外挂载。

#### 解决办法

移除属性过滤与改机模块，恢复真实属性；第三方 ROM 自身属性缺失只能等 ROM 维护者修复或刷回官方系统。

</details>

<details><summary><b>8项设备属性双源核对 / Build.TAGS交叉 / 安全补丁交叉验证 / 构建完整性检测 / 启动参数原始值交叉验证 / 内核启动安全上下文</b></summary>

#### 检测方式

Java / Build 常量 / `getprop` / Native 属性区四个来源读取同一属性的结果是否一致；`cmdline` / `bootconfig` 与 `ro.boot.*` 是否一致，是否含解锁、permissive、verity 关闭、`/data/adb`、KernelSU / APatch / SUSFS 标记。

#### 解决办法

停用改机 / PIF / resetprop 模块，恢复官方属性与启动分区后重启。

#### 关系

与 `Play Integrity Fix残留`、`属性一致性检查` 同族。

</details>

<details><summary><b>Play Integrity Fix残留</b></summary>

#### 检测方式

Native 读取处于启用状态的 PIF / PIHooks / PixelProps / 属性伪装类属性（`persist.sys.pihooks*`、`persist.sys.pixelprops*`、`persist.sys.spoof.gms` 等）。

#### 解决办法

停用并卸载对应模块，清理模块实际残留（含 `system.prop` 与 `service.d` 脚本）后重启。

</details>

<details><summary><b>属性区上下文检测（1.3.0 新增）/ 属性区可读性核对</b></summary>

#### 检测方式

从进程自身映射中枚举属性区上下文名，命中伪装 / 隐藏类标记（spoof / pif / pixelprop / hide / ksu / magisk / apatch / lsposed / xposed / tricky 等）即命中；并核对属性区文件是否可读、序列号与信息文件是否自洽。

#### 解决办法

卸载创建了自定义属性区的模块（这类上下文通常由属性伪装模块写入）后重启。

#### 备注

原厂存在的 `vendor_socket_hook_prop` 等正常上下文不会命中。

</details>

<details><summary><b>自定义ROM检测 / 内核版本检测 / 内核构建特征检测 / 三方内核</b></summary>

#### 检测方式

系统 / 机型特征与内置第三方 ROM 与内核关键字对照（LineageOS、自编译内核后缀 `-dirty`、第三方内核名等）。

#### 解决办法

- 恢复官方 ROM 与内核；或使用 SUSFS 等内核级方案隐藏内核名称（注意这类隐藏本身可能触发其它条目）。
- 内核版本带 `-dirty`：重新编译为干净版本或还原官方 boot。

#### 备注

第三方 ROM 自身特征无法通过模块完全消除，属「环境固有差异」。

</details>

---

## 七、侧信道与视图一致性检测（新增）

<details><summary><b>Netlink查询策略检测（1.3.0 新增）</b></summary>

#### 检测方式

以普通应用身份向内核 `sock_diag` 发起一次查询：正常策略下应被拒绝；若能拿到响应（尤其是**其它 UID 的 socket 记录**），说明内核 netlink 查询策略被放宽。

#### 解决办法

恢复官方内核 / SELinux 策略；若是某个隐藏方案放开了该接口，停用后重启。

#### 备注

原厂系统本身就会拒绝普通应用的该查询，「被拒绝」是正常结果。

</details>

<details><summary><b>进程防护视图一致性（1.3.0 新增）</b></summary>

#### 检测方式

`/proc/self/status` 的 Seccomp / NoNewPrivs 与 `prctl` 直连查询、以及 libc 读取三路交叉；任两路结果不一致即命中（典型是 `/proc` 读取或系统调用被拦截）。

#### 解决办法

停用 `/proc` 过滤、文件 API Hook 与针对 Maple 的隐藏模块后重启。

#### 关系

与 `proc读取双路核对`、`Binder调用路径一致性` 同属「多路观测分歧」。

</details>

<details><summary><b>Zygote补充GID检测（1.3.0 新增）</b></summary>

#### 检测方式

核对进程补充 GID 中是否保留**应用专属组**（`20000+appId`、`50000+appId`）——这是 Zygote 派生进程的必然特征；缺失说明补充组被限制（常见于「限制 zygote 权限」类隐藏配置）。

#### 解决办法

在应用隐藏模块中取消对 Maple 的「限制 zygote 权限 / SHARED USER GID / APP ZYGOTE GID」类开关后重启。

#### 备注

Android 14+ 上应用本来就没有 GID 3009（readproc），这属于正常情况。

</details>

<details><summary><b>OBB路径视图一致性（1.3.0 新增）</b></summary>

#### 检测方式

用「文件是否存在」「stat」「lstat」「父目录列举」四种方式读应用自身的 OBB 路径，结果互相矛盾即命中（说明有模块在拦截该路径的读取）。

#### 解决办法

停用针对扫描 / 文件读取的隐藏模块；必要时对 Maple 关闭「限制 zygote 权限」以外的路径拦截策略。

</details>

<details><summary><b>时钟源一致性 / 内核时钟交叉验证 / 开机时刻一致性 / 进程启动时刻一致性</b></summary>

#### 检测方式

多个时钟源（monotonic / boottime / uptime / 内核 btime / 进程 starttime）在稳定采样窗口内是否持续背离。

#### 解决办法

关闭变速、时间 Hook 与虚拟容器，恢复系统自动日期与时间后重启；持续命中再排查时钟类内核模块。

</details>

---

## 八、硬件与传感器交叉验证

<details><summary><b>CPU/GPU/Platform三向验证 / CPU+GPU交叉验证 / 硬件交叉验证报告 / 芯片身份识别 / CPU特性校验 / sysfs交叉验证 / Vulkan物理GPU直枚举 / GPU Driver验证</b></summary>

#### 检测方式

`cpuinfo`、系统属性、驱动节点、Vulkan 物理设备、sysfs、CPU 拓扑与设备节点之间的硬件身份是否互相矛盾（含 ISO 来源、频率表、核心数、SoC 序列号）。

#### 解决办法

停用改机、KPM 文件覆盖与硬件伪装模块，恢复官方内核、驱动和系统属性。

#### 备注

同一 SoC 的不同机型频率表、传感器布局可能不同，单点差异属于正常情况。

</details>

<details><summary><b>CPU频率 / CPU拓扑 / 内存总量检测 / 存储空间检测 / 温度传感器指纹 / 热区传感器计数 / 热区温度读取 / 电源检测 / 输入设备检测</b></summary>

#### 检测方式

频率表与拓扑是否自洽、内存与存储容量是否与机型匹配、热区传感器数量 / 类型 / 读数是否异常、输入设备是否为虚拟设备。

#### 解决办法

- 硬件伪装 / 改机模块：卸载后恢复官方属性。
- 温控 / 调度模块改写热区节点：卸载对应模块。
- 虚拟输入设备（自动点击 / 映射工具）：关闭对应工具的无障碍或输入注入服务。

#### 备注

部分机型（尤其欧洲 / 印度版）传感器布局与国内版不同，单点差异属于正常情况。

</details>

---

## 内建清单与属性

<details><summary><b>附录 A：Maple 读取的关键系统属性（部分）</b></summary>

```text
ro.boot.flash.locked / ro.boot.verifiedbootstate / ro.boot.vbmeta.device_state
ro.boot.vbmeta.digest / ro.boot.vbmeta.avb_version / ro.boot.avb_version
ro.secureboot.lockstate / ro.boot.veritymode / ro.boot.selinux
ro.build.fingerprint / ro.vendor.build.fingerprint / ro.system.build.fingerprint
ro.odm.build.fingerprint / ro.product.build.fingerprint / ro.bootimage.build.fingerprint
ro.build.version.security_patch / ro.build.date.utc / ro.build.type / ro.build.tags
persist.logd.size / persist.logd.size.crash / persist.logd.size.main / persist.logd.size.system
persist.debug.dalvik.vm.core_platform_api_policy
persist.sys.vold_app_data_isolation_enabled
persist.sys.pihooks.* / persist.sys.pixelprops.* / persist.sys.spoof.gms
persist.chunqiu.path_hide
dalvik.vm.dex2oat-flags
```

</details>

<details><summary><b>附录 B：Maple 关注的高危文件与目录（部分）</b></summary>

```text
/data/local/stryker
/data/system/AppRetention
/data/local/tmp/luckys
/data/local/tmp/input_devices
/data/local/tmp/HyperCeiler
/data/local/tmp/simpleHook
/data/local/tmp/DisabledAllGoogleServices
/data/local/MIO
/data/DNA
/data/local/tmp/cleaner_starter
/data/local/tmp/byyang
/data/local/tmp/mount_mask
/data/local/tmp/mount_mark
/data/local/tmp/scriptTMP
/data/local/luckys
/data/local/tmp/horae_control.log
/data/gpu_freq_table.conf
/storage/emulated/0/Download/advanced
/storage/emulated/0/Documents/advanced
/data/system/NoActive
/data/system/Freezer
/storage/emulated/0/Android/naki
/data/swap_config.conf
/data/local/tmp/resetprop
/dev/cpuset/AppOpt
/data/encore/default_cpu_gov
/data/encore/custom_default_cpu_gov
/system/bin/encore_profiler
/data/local/tmp/encore_logo.png
/data/local/tmp/yshell
/system/bin/adb
/system/bin/fastboot
/storage/emulated/0/MT2
```

> `/proc`、`/sys`、`/dev`、`/system` 等系统路径是检测器**读取**的对象，**不要删除**。

</details>

<details><summary><b>附录 C：风险应用 / 隐藏工具 / Root 管理器包名（部分）</b></summary>

```text
# Root 管理器
com.topjohnwu.magisk / io.github.vvb2060.magisk / me.weishu.kernelsu
com.rifsxd.ksunext / com.sukisu.ultra / me.bmax.apatch / io.github.a13e300.ksuwebui

# Hook / 注入框架
de.robv.android.xposed.installer / org.lsposed.manager / org.lsposed.lspatch
org.lsposed.lspd / me.simpleHook / lin.xposed / top.hookvip.pro
cn.kwaiching.hook / vn.kwaiching.tao / com.padi.hook.hookqq / io.github.qauxv

# 隐藏 / 冻结工具
com.tsng.hidemyapplist / com.tsng.pzyhrx.hma / cn.geektang.privacyspace
github.tornaco.android.thanos / nep.timeline.freezer / cn.myflv.noactive
com.hchen.appretention / me.gm.cleaner

# 虚拟框架 / 多开
com.lody.virtual / io.va.exposed / com.vphonegaga.titan / com.vmos.pro

# 其它风险工具
com.termux / com.omarea.vtools / com.byyoung.setting / bin.mt.plus
com.sevtinge.hyperceiler / com.luckyzyx.luckytool / com.lerist.fakelocation
com.silverlab.app.deviceidchanger.free / com.xayah.databackup.foss
```

> 名单会随版本更新；完整名单以应用内详情为准（命中的包名与版本会直接列在检测依据里）。

</details>

---

## 最后排查顺序

<details><summary><b>推荐顺序</b></summary>

1. **重启并重新检测**，排除瞬时进程、动态 maps 与开机挂载尚未收敛的影响（刚开机请等 20 秒～5 分钟）。
2. **按详情停用对应模块**，一次只改一个变量；不要一次叠加多个隐藏方案。
3. **取消 Maple 的 Xposed / Zygisk / Frida / 变速作用域**并重启。
4. **卸载明确命中的工具**，清理确认属于它的残留（路径、属性、脚本）。
5. **属性 / 挂载 / 内核 / TEE 仍异常**时，恢复同版本官方完整固件。
6. **确认所有分区原厂且设备支持后再回锁 BL**，避免变砖。
7. **仍然无法通过**时，提交完整检测依据与环境信息，不要只发结果颜色。

</details>
