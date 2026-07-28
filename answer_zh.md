# Maple检测项解决方案（跟进1.0.2版本）中文版
> 整理：Maple | 仅供参考，具体结果因设备/系统/内核环境而异。
>
> 标有"?"的内容未经确认，仅供排查参考。
>
> 文档链接：[github](https://github.com/mingzun09/Chunqiu-Detector-Problem-solution)

# 自行尝试但仍然无法通过的检测
> 请提交Issue并提供：检测项完整详情、系统版本、内核版本、Root管理器、模块列表、Xposed模块列表、是否使用改机/PIF/TEE模拟模块。
>
> 不要只发"检测到第三方"。没有检测依据无法判断是哪一条来源异常。

# 8项设备双源属性核对
> 通过Java/Native双源读取设备数据来识别伪装
> 尝试更换改机模块/卸载改机模块

# Attested安全补丁级别
> 在/data/adb/trick_store文件夹中修改安全补丁日期为系统补丁/使用TA配置系统补丁日期解决

# BindMount/Root挂载专项检测
> 检测到mount --bind挂载操作，来源于Magisk、KernelSU、APatch或/data/adb，以及CPU/SoC身份文件的独立覆盖挂载
> 卸载产生挂载的模块，恢复被覆盖的系统文件，重启后复测

# CPU/GPU识别
> 通过识别驱动层，启发式检测其他驱动与各类CPU与GPU特性
> 当前背景下无解决可能性，卸载相关模块解决

# KernelSU原生接口检测
> 探测KernelSU相关泄露特性
> 等待KSU管理器更新/尝试更新KSU管理器/尝试开启隐藏Selinux修改解决

# 检测到Root/检测到免解设备/Xposed/三方环境工具/Root管理器检测
> 使用HMA-OSS/隐藏应用列表对Maple检测隐藏相关软件解决

# KernelSU免解设备
> 尝试开启隐藏Selinux修改解决

> KSU使用者[更新KSU管理器](https://t.me/KernelSU_group/3234/482579)并重新修补镜像并刷入后重启再开启selinux_hide功能解决

> APatch/FolkPatch使用者[使用此kpm](https://github.com/Admirepowered/selinux_hook)

> selinux_hook模块使用说明
> 1. 内核版本4.19-6.12的设备，必须嵌入此模块才能生效，如果使用加载模式，则不会启用任何伪装方法。
> 注意：由于编译优化导致的机器指令与预期不符，6.12内核设备请慎重嵌入此模块，会有很大概率导致kernelpanic，此问题后续将会被解决。
> 2. 内核版本4.14的设备，建议嵌入模块，但由于模拟context_struct_compute_av存在风险，在嵌入模块前请备份原boot.img，以便在出现kernelpanic后可救砖；加载模式同样可生效，但会使用基于关键词过滤的备选方法，效果相对较差，如果设备的policy中包含了模块没有收录的且能被证明异常的关键词，则会发生泄露。
> 3. 内核版本4.9的设备，建议嵌入模块，且无模拟context_struct_compute_av的风险，加载模式效果同4.14。

> Magisk......尝试更换内核级管理器
>
> 在将来Magisk可能会合并保存Clean policy blob功能，如果合并此功能，Magisk将有机会通过此检测。

# TEE RootOfTrust启动状态
> 可信硬件证明报告Self-signed、Unverified、Failed或设备未锁定
> 尝试更新至TEESimulator-RS最新版本解决
>
> 恢复完整官方固件、vbmeta与可信启动链，确认分区原厂后再回锁BL

## 异常文件
- 检测路径：/dev和/data/local/tmp
  1. 重命名/删除相关目录文件
  2. 排查并删除以下高危路径：
     ```
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
     ```

# Root挂载检测/Bind Mount专项检测
> 检查mounts/mountinfo中的Magisk、KernelSU、APatch、/data/adb来源，以及CPU/SoC身份文件的独立覆盖挂载模块
> 卸载产生挂载的模块，恢复被覆盖的系统文件，重启后复测
> 普通OEM Overlay、APEX、EROFS与dm设备本身不作为Root证据

# Overlay挂载/Cacerts挂载/系统分区RW
> 检测到Root来源的Overlay、证书目录重挂载或系统分区可写
> 移除修改系统/证书的模块，恢复官方分区与证书文件后重启

# Debug/Overlay检测
> debug_ramdisk中存在调试文件，或内核支持OverlayFS
> 若使用自行刷入的调试镜像，恢复对应官方boot/vendor_boot

# 挂载组ID连续性
> 挂载ID存在空洞，可能与Root挂载模块相关
> 根据详情排查实际挂载模块。更换管理器或"元模块"不能保证解决

# build.prop与属性区交叉验证
> build type、签名标签、ro.debuggable、ro.secure等跨分区属性不一致
> 检查改机、PIF、resetprop与属性隐藏模块，恢复磁盘文件与运行时属性一致

# Build.TAGS交叉/安全补丁交叉验证/Fingerprint交叉验证
> Java、Build、getprop、Native等来源对同一属性的结果不一致
> 停用改机/PIF/resetprop模块，恢复官方属性和对应系统分区后重启

# Fingerprint分量交叉验证
> ro.build.fingerprint与brand/product/device/release/id等独立属性不匹配
> 卸载改机模块或恢复完整官方属性解决

# 属性一致性检查
> 多项关键属性为空或来源异常
> 移除属性过滤与改机模块，恢复真实属性
> 第三方ROM自身属性缺失只能等待ROM维护者修复或刷回官方系统

# Play Integrity Fix残留
> Native读取到处于启用状态的PIF/PIHooks/PixelProps等明确属性
> 停用并卸载对应模块，清理模块实际残留后重启

# 启动参数原始值交叉验证
> cmdline/bootconfig与ro.boot属性不一致
> 恢复官方boot/vendor_boot与真实启动参数

# 路径存在性三路核对/Root路径多路仲裁
> 同一路径通过多种方式核对，表明Root路径真实存在或多路结果互相矛盾
> 真实路径：卸载对应工具并清理残留
> 多路矛盾：停用文件隐藏、PathMask、SusFS路径伪装或API Hook后复测

# proc读取双路核对
> Java与Native读取proc内容不一致
> 停用maps/mounts过滤、文件API Hook与针对Maple的隐藏模块

# 包可见性双路核对
> pm、PackageManager与createPackageContext对已知Root管理器包的查询结果不一致
> 停用HMA、包列表过滤、虚拟容器与PackageManager Hook后重启
> 三路都能看到已安装包：按详情决定是否卸载

# SELinux状态检测
> SELinux处于Permissive或Disabled状态
> 恢复SELinux Enforcing，卸载修改SELinux状态的内核模块并重启

# SELinux策略检测
> 读取到magisk/ksu/apatch/zygisk相关策略内容
> 恢复未注入Root策略的官方内核/boot与SELinux策略

# SELinux上下文检测
> 当前进程进入su、magisk、kernelsu等Root域
> 停用对Maple生效的注入/上下文修改，移除对应Root框架后重启

# SELinux策略类型预言机
> 内核策略确认存在KernelSU/Magisk等专有类型
> 恢复未注入对应策略类型的官方内核和启动镜像

# 内核setresuid行为探针
> setresuid异常成功（子进程被杀或权限异常）
> 卸载修改setresuid行为的SusFS/Root内核模块，恢复官方内核

# 内核加固参数
> ASLR、kptr_restrict、dmesg_restrict等关键内核加固参数被放宽
> 恢复官方量产内核或修正自编译内核配置

# ASLR个性标志/信号处置procfs视图
> ASLR关闭，或SIGTRAP/SIGSEGV被异常屏蔽/忽略
> 退出setarch、调试器、注入器与信号Hook工具，重启应用
> 若为内核级修改，恢复官方内核

# 内核配置暴露
> /proc/config.gz可读，暴露自编译/调试内核特征
> 更换为不暴露该接口的量产内核

# 调试器检测/Java调试器状态/TracerPid检测
> 当前进程连接了Android Studio、LLDB、ptrace、Frida等调试器
> 断开调试器，停止Frida/ptrace工具并结束调试会话，重启Maple

# FD扫描/文件描述符扫描
> 当前进程打开的文件描述符指向/data/adb、Magisk、Frida等注入来源
> 停止对Maple生效的注入，卸载对应模块并重启

# 运行时环境变量/动态库路径检测
> LD_PRELOAD、LD_LIBRARY_PATH、CLASSPATH、PATH或TMPDIR含Root/Hook注入路径
> 移除注入启动器、终端包装脚本和环境变量修改模块，从系统桌面正常启动Maple

# 内核启动安全上下文
> cmdline/bootconfig含解锁、Permissive、verity关闭、/data/adb、KernelSU、APatch或SUSFS标记
> 恢复官方boot/vendor_boot和安全启动配置

# 进程Cgroup上下文
> cgroup路径中出现Magisk、Zygisk、KernelSU、APatch、LSPosed或Frida标记
> 停止对应框架并移除针对Maple的注入作用域后重启

# Frida检测/Frida文件/Frida进程
> 检测到Frida文件、进程、线程、Socket、协议响应、maps与Gadget等证据
> 停止frida-server，移除Gadget与注入脚本，取消对Maple的注入后重启

# Xposed/LSPosed检测
> 检测到Xposed类、字段、调用栈、ClassLoader、Handler、模块元数据、FD、maps与线程
> 从LSPosed/Xposed作用域中取消Maple，停用相关模块并重启
> 若OAT/ODEX工件仍存在，卸载重装Maple后复测

# Zygisk/Riru检测
> 检测到环境变量、magisk.process、maps模块路径、线程、受限路径SO、TracerPid与守护进程
> 关闭Zygisk/Riru或取消Maple作用域，停用对应模块并重启

# Substrate/ShadowHook/Dobby/Pine检测
> 检测到对应Hook库、类、maps、dl_iterate_phdr与Inline Hook痕迹
> 卸载或停用对Maple生效的Hook模块，重启后复测

# 自身方法本地化/框架方法本地化
> Java方法被改写为native
> 停用LSPosed、YAHFA、Pine、SandHook等方法Hook后重启

# 引导类加载归属
> 核心类加载器与java.lang.String实际引导加载器实例不一致
> 停用DEX影子化、类替换与虚拟容器，重启Maple

# Native注入映像/匿名执行代码/内存注入结论
> 检测到具名或隐藏ELF、DEX、JNI表Hook、非JIT RWX、匿名执行代码与可疑memfd
> 停止所有针对Maple的注入、Hook、变速和调试模块，重启后复测

# ART代码页/关键Native代码页/隔离进程ART差分
> 内存代码页与磁盘文件不一致，或主进程与隔离进程运行时差异
> 停用修改ART/linker/libc/android_runtime的模块并重启
> 若仍命中，重装官方APK；系统级代码页被修改时需恢复官方ROM

# ART私有脏页
> 与其他注入证据同时出现
> 按关联条目排查具体注入来源

# 匿名RWX段/Memfd/JIT/Inline Hook
> 检测到非标准匿名RWX、未知可执行memfd或Native代码修改
> 关闭注入器、Hook框架与变速工具，重启应用

# 信号处理器完整性
> 关键信号处理器指向匿名执行区或明确注入模块
> 停用调试器、崩溃拦截、Hook与注入模块后重启

# 当前进程Socket/Frida兼容协议探针
> 当前进程监听Frida常用端口或可疑Unix Socket
> 停止对应服务和注入进程后重启

# 跨进程可执行内存扫描
> 检测到明确INJECTION或deleted memfd强证据
> 停止目标游戏中的外挂、注入器与调试器，重启游戏和Maple

# 无障碍服务检测
> 检测到自动化/脚本工具关键词或异常多的已启用无障碍服务
> 在系统无障碍设置中关闭对应自动化、连点器、脚本工具服务后复测

# Linux能力检测/Seccomp状态检测
> Maple进程CapEff非零或seccomp被关闭
> 停止提权、容器与注入，让Maple在普通应用沙箱中运行

# 模拟器属性/GPU模拟/CPU模拟器特征
> 检测到QEMU、goldfish、ranchu、VirtualBox、VirGL、VirtIO GPU等运行时证据
> 真机：停用硬件伪装、云手机/容器模块并恢复官方属性、内核和驱动
> 模拟器本身无法通过修改属性变成真机

# CPU/GPU/Platform硬件交叉验证
> cpuinfo、属性、驱动、Vulkan、sysfs、拓扑与设备节点之间硬件身份互相矛盾
> 停用改机、KPM文件覆盖与硬件伪装，恢复官方内核、驱动和系统属性

# PID命名空间/用户命名空间/根目录重定向
> 检测到分身、虚拟容器、chroot、UID映射与嵌套命名空间
> 退出双开/云手机/虚拟容器，在系统正常安装路径直接运行Maple

# 进程身份四元组/数据目录归属一致性
> UID、包归属、sourceDir、dataDir与实际进程身份不一致
> 退出虚拟容器，卸载当前副本，从可信渠道重新安装正式APK

# 应用签名自校验/APK安装路径/应用可调试标志
> 多签名者、异常安装目录或debuggable APK
> 卸载修改版，从可信渠道安装正式Release APK
> 当前应用只显示单签名摘要供人工核对

# 宿主可执行文件/自身native库来源/自身APK映射一致性
> Maple由非系统app_process宿主启动，或APK/so映射来自异常路径
> 退出容器/分身，停用注入后卸载重装官方APK

# 时钟源一致性/内核时钟交叉验证
> 多个时钟源在稳定窗口中持续背离
> 关闭变速、时间Hook与虚拟容器，恢复自动日期和时间后重启

# 开机时刻一致性
> framework/proc推导出的开机时刻偏差过大
> 开启自动时间、重启后复测。持续命中再排查时间Hook、变速模块或ROM时钟问题

# 进程启动时刻一致性
> 多种方式推导出的Maple启动时间持续不一致
> 停用变速、时间Hook、虚拟容器后结束进程并重新打开

# Attested安全补丁级别/Vendor补丁级别
> 可信证明中的OS/Vendor/Boot补丁与运行时真实日期不一致
> 恢复真实系统属性和可信证明环境，确保系统与对应vendor/boot分区来自同一固件版本

# Android Key Attestation可用性
> 生成一次性密钥或解析证明失败，本次无法完成证明
> 重启后复测。仍失败则停用Keystore/KeyMint Hook与TEE模拟模块，更新或恢复官方系统

# Attestation随机挑战/密钥绑定/签名链/受信根
> 一次性随机挑战、生成密钥绑定、证书签名链或受信根校验未通过
> 停用TrickyStore/TEE/KeyMint模拟或拦截模块，恢复官方Keystore/KeyMint与可信启动链

# Attestation证书吊销状态
> Google状态列表标记证书序列号已吊销
> 更换合法未吊销的设备证明环境或恢复厂商官方方案

# Boot哈希一致性
> 使用TA辅助模块/手动更新哈希解决

# Knox Warranty检测
> Samsung Knox warranty_bit=1，代表历史e-fuse状态
> 该状态通常不可逆

# StrongBox密钥请求
> 系统声明支持StrongBox但实际结果矛盾
> 尝试更新TEE相关模块

# TEE/KeyMint证明异常（仅Ultra）
> TEE授权标签、设备标识标签、APPLICATION_ID、KeyUsage、签名算法、安全级别、证书链、Vendor补丁或超长挑战探针异常
> 按实际命中条目的详情排查，停用证明模拟模块，恢复官方KeyMint/TEE与可信启动链

# Binder调用路径一致性
> libc ioctl与直接syscall连续三次稳定分歧
> 停用Keystore/TEE Binder拦截、ioctl Hook与服务代理模块，恢复官方服务后重启

# Widevine设备ID读取/Widevine与TEE交叉验证
> Widevine初始化或设备ID读取失败，或安全级别为L3/与可信TEE结果矛盾
> 恢复官方DRM、TEE与完整固件

# KernelSU免解设备/越狱模式（综合）
> 汇总KernelSU Late-load、活动LKM、SELinux标记、持久化工件、Metamodule、Magica事务与越狱组件
> 关闭Late-load/越狱模式。KSU使用者：更新KSU管理器并重新修补镜像刷入后开启selinux_hide
> 移除Metamodule、Magica和持久化工件，恢复未集成KernelSU的官方内核后重启

# 免解机型Prop交叉检测
> 更改机型

# KernelSU Late-load脚本环境
> Native检测到KSU越狱脚本位或对应启动环境
> 关闭Late-load/越狱模式，移除启动脚本，恢复官方内核和启动镜像后重启

# KernelSU活动LKM/内核与SELinux标记
> 检测到活动KernelSU模块、ksu_file策略或明确KSU运行态
> 卸载KernelSU并恢复官方内核/SELinux策略

# KernelSU越狱持久化工件/Metamodule挂载
> 检测到明确持久化文件或source=KSU等挂载来源
> 卸载对应组件，删除对应残留，恢复被挂载文件后重启

# Magica启动事务窗口/瞬时进程
> 检测到Magica属性窗口、5555端点或瞬时命令运行态
> 停用并卸载Magica相关组件，清理启动脚本后重启

# 临时目录权限检测
> /data/local/tmp权限不是0771
> 确认修改来源后，按设备原厂设置恢复0771

# 临时目录inode检测
> 等待收集修复方式中..

# 外挂/Root工具路径清单
> 命中具体外挂/Root工具路径
> 卸载对应工具，清理确认不再需要的残留

# 异常sh文件快速扫描
> 在应用可访问的共享存储范围内检测到.sh脚本文件
> 根据详情查看具体路径，删除确认不再需要的脚本

# 应用可访问范围外的异常文件
> 扫描范围受限于应用实际可访问路径
> 不建议授予不必要的高权限来扩大扫描

# 检测失败/协议异常
> 对应检测器执行异常或Native协议不完整，结果未知
> 先重启后复测。仍然失败请记录条目名称与详情中的异常类型后提交Issue

# Native显示Must use __system_property_read_callback() to read
> 旧版Native属性API读取长属性时返回的Bionic提示
> 更新至最新版后重新检测即可

# 最后排查顺序
> 1. 重启并重新检测，排除瞬时进程、动态maps与时间同步影响。
>
> 2. 根据详情停用对应模块，不要一次叠加多个隐藏方案。
>
> 3. 取消Maple的Xposed/Zygisk/Frida/变速作用域并重启。
>
> 4. 卸载明确命中的工具，清理确认属于它的残留。
>
> 5. 属性、挂载、内核或TEE仍异常时，恢复同版本官方完整固件。
>
> 6. 确认所有分区原厂且设备支持后再回锁BL，避免变砖。
>
> 7. 仍然无法通过时，提交完整检测依据与环境信息，不要只发结果颜色。
