# Maple_Detector_Problem_Solution

Maple_Detector 检测项解决文档 『时光流转，愿你能与珍爱之人再度重逢』🍁

> 核对版本：**1.3.0** ｜ 最后更新：2026-10 ｜ 检测项总数：日常 **112** / 游戏 **129** / Ultra 日常 **208** / Ultra 游戏 **329**
>
> 随着检测器的更新，该文档将会被修改（添加 / 删除 / 重命名）。

You can select your language below to view this document.

您可以在下方选择语言查看此文档。

## language / 语言

> Select one of the following languages to view the solution file.

> 请选择以下语言之一来查看解决方案文件。

[中文](answer_zh.md) ｜ [English](answer_en.md)

> There are many embedded links in the document (highlighted in blue). You can click on them to be redirected to the relevant address.

> 文档中有很多嵌入式链接（蓝色高亮），点击可跳转到对应地址。

## 怎么用这份文档 / How to use

1. 在 Maple 里点开命中的条目，**复制「检测依据」里的完整内容**（不是只记标题）。
2. 在本文档里 `Ctrl+F` 搜索界面上的条目名（中英版都保留中文名，方便对照）。
3. 按「解决办法」逐条处理，**一次只改一个变量**，改完重启设备再复测。
4. 界面上**看不到某一条**，说明本次没有取到可比对的数据（被拒绝 / 不支持 / 超时），不代表你已经通过。
5. 判断不了是哪一条来源异常时，按文末「最后排查顺序」从第 1 步做起。

## 反馈或参与翻译 / Feedback or participate in translation

You can branch and modify the warehouse and then pull the request, and I will check and decide whether to merge.

您可以创建分支并修改仓库，然后提交合并请求，我会检查并决定是否合并。

提交 Issue 时请附上：命中条目的完整「检测依据」、系统与内核版本（`uname -a`）、安全补丁日期、Root 管理器与模块列表、Xposed / LSPosed 模块与作用域、是否使用改机 / PIF / TEE 模拟 / 应用隐藏类模块。

## 常用模块仓库 / Common module repositories

| 用途 | 仓库 |
|---|---|
| 密钥模块（TEE 证明） | [TEESimulator-RS](https://github.com/Enginex0/TEESimulator-RS) ｜ [OhMyKeymint](https://github.com/qwq233/OhMyKeymint) ｜ [Tricky-addon-Enhanced](https://github.com/Enginex0/tricky-addon-enhanced) |
| Zygisk 实现模块 | [Zygisk-Next](https://github.com/Dr-TSNG/ZygiskNext) |
| 应用隐藏模块 | [HMA-OSS](https://github.com/frknkrc44/HMA-OSS) |
| 元模块 / 挂载隐藏 | [Hybrid-Mount](https://github.com/Hybrid-Mount/meta-hybrid_mount) ｜ [NoMount](https://github.com/maxsteeel/nomount) ｜ [LKM-PathMask](https://github.com/Andrea-lyz/LKM-PathMask) |
| SELinux 策略隐藏 | [DirtySepolicy](https://github.com/LSPosed/DirtySepolicy) ｜ [selinux_maf_fork](https://github.com/741afb7/selinux_maf_fork) ｜ [selinux_hook](https://github.com/Admirepowered/selinux_hook) |
| 残留 / 痕迹清理 | [Inode-Hijacker](https://github.com/YiJieqwq/Inode-Hijacker) ｜ [ADB-Trace-Cleaner](https://github.com/YiJieqwq/ADB-Trace-Cleaner) ｜ [FuseFixer](https://github.com/5ec1cff/FuseFixer) |
| APatch / KernelPatch 隐藏 | [NoHello.kpm](https://t.me/welikeandroid) ｜ [SELinux_Hook.kpm](https://t.me/APatch_nightly) |

## 声明 / Disclaimer

- 本文档与 Maple 检测器仅面向 **Android 技术爱好者**，用于**技术学习、环境研究与探讨**；**严禁**用于绕过应用反作弊、规避风控机制、游戏作弊等违法违规场景，后果自负。
- 文档中的操作涉及修改系统镜像、替换密钥、嵌入内核模块、执行底层 Root 指令等，均存在**极高且不可逆的风险**（变砖、无限重启、数据丢失），请自行评估并提前备份。
- 受 ROM、内核、Root 管理器与模块组合影响，任何检测项都存在概率性误报；文档给出的方案不保证对所有设备生效，检测结果仅作排查参考。
- 文中引用的第三方模块、脚本与链接均为社区公开资源，作者不对其安全性、可靠性负责，请自行甄别来源。

## License

见 [LICENSE](LICENSE)。
