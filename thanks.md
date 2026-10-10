# 致谢名单 / Credits

本文档与 Maple 检测器的环境检测能力，参考、得益于以下社区公开项目与资料。排序不分先后。

## 检测逻辑与问题解决思路参考

- [Chunqiu-Detector-Problem-solution](https://github.com/mingzun09/Chunqiu-Detector-Problem-solution)：同类检测项解决方案文档，本文档的结构与排查思路大量参考该项目。
- [Duck Detector](https://github.com/eltavine/Duck-Detector)：运行时注入与环境检测字段整理。
- [AnyCheck](https://github.com/AnyCheck)：无障碍、开发者选项、模拟器与多开检测项参考。
- [DirtySepolicy](https://github.com/LSPosed/DirtySepolicy)：SELinux 策略 oracle 侧信道实现参考。
- [PathQuorum / 路径多路仲裁](https://github.com/tiann/KernelSU)：KernelSU、Magisk 上游源码审计，用于确定路径与策略判据。

## 模块与工具

- 密钥模块：[TEESimulator-RS](https://github.com/Enginex0/TEESimulator-RS)、[OhMyKeymint](https://github.com/qwq233/OhMyKeymint)、[Tricky-addon-Enhanced](https://github.com/Enginex0/tricky-addon-enhanced)
- Zygisk 实现模块：[Zygisk-Next](https://github.com/Dr-TSNG/ZygiskNext)
- 应用隐藏模块：[HMA-OSS](https://github.com/frknkrc44/HMA-OSS)、[HideMyApplist](https://github.com/dr-webmaster/HideMyApplist)
- 元模块与挂载隐藏：[Hybrid-Mount](https://github.com/Hybrid-Mount/meta-hybrid_mount)、[NoMount](https://github.com/maxsteeel/nomount)、[LKM-PathMask](https://github.com/Andrea-lyz/LKM-PathMask)
- SELinux 策略隐藏：[selinux_maf_fork](https://github.com/741afb7/selinux_maf_fork)、[selinux_hook](https://github.com/Admirepowered/selinux_hook)
- 残留与痕迹清理：[Inode-Hijacker](https://github.com/YiJieqwq/Inode-Hijacker)、[ADB-Trace-Cleaner](https://github.com/YiJieqwq/ADB-Trace-Cleaner)、[FuseFixer](https://github.com/5ec1cff/FuseFixer)
- APatch / KernelPatch 隐藏方案：[NoHello.kpm](https://t.me/welikeandroid)、[SELinux_Hook.kpm](https://t.me/APatch_nightly)

## 项目

- Root 管理器：[Magisk](https://github.com/topjohnwu/Magisk)、[KernelSU](https://github.com/tiann/KernelSU)、[KernelSU-Next](https://github.com/KernelSU-Next/KernelSU-Next)、[SukiSU-Ultra](https://github.com/SukiSU-Ultra/SukiSU-Ultra)、[APatch](https://github.com/bmax121/APatch)、[KernelPatch](https://github.com/bmax121/KernelPatch)
- OAID：[Android_CN_OAID](https://github.com/gzu-liyujiang/Android_CN_OAID)

## 文档与贡献

- 文档整理与维护：Maple
- 欢迎通过 Issue 提交新的检测依据与解决办法，或提交 PR 参与翻译与完善。

> 以上项目均为其作者所有；本文档仅引用其公开信息用于环境检测与排查说明，不对第三方项目的安全性与可靠性负责。
