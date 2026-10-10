# Maple Detector Problem Solution (for version 1.3.0) — English

> Compiled by: Maple | Last updated: 2026-10 | Verified version: 1.3.0 (total checks: Daily 114 / Game 131 / Ultra Daily 211 / Ultra Game 332)
> Credits: [Credits list](https://github.com/FengQiuFQ/Maple_Detector_Problem_Solution) | For reference only; actual results vary with device / system / kernel environment.
> Items marked with "?" are unconfirmed and are for troubleshooting reference only.
> Document link: [github](https://github.com/FengQiuFQ/Maple_Detector_Problem_Solution/)

---

<details><summary><b>Disclaimer</b></summary>

**1. Compliance and intended use**

This document and the environment detection and remediation solutions provided by the Maple detector are **intended solely for Android technology enthusiasts**, and only for **technical learning, environment research and discussion**. It is **strictly forbidden** to use any solution in this document to bypass application anti-cheat, evade risk-control mechanisms, cheat in games, or for any other illegal or non-compliant purpose. Any consequences and liability arising from misuse are borne entirely by the user.

**2. Risk warning**

All operation flows, automation scripts and module configurations recorded in this document are for technical reference only. Operations that involve modifying the system image, replacing hardware keys, embedding kernel modules (KPM), or executing low-level Root commands all carry **extremely high irreversible risk**, and may brick the device, cause endless reboots, or lose user data. All operation risks are borne by the user.

**3. About the results**

**Not seeing a certain item in the UI means that no comparable data was obtained this run — it does not mean you passed that item.**

**4. About false positives**

Because of ROM, kernel, root manager and module combinations, any check can produce probabilistic false positives. If you are sure a finding is a false positive, submit your detection evidence and environment details as described at the end of this document.

</details>

---

## Notes and feedback

<details><summary><b>Checks that still fail after trying on your own</b></summary>

Please open an Issue and provide:

1. The **full details of the check that hit** (the "detection evidence" shown after expanding the item — do not just send the color or the title);
2. System version, kernel version (`uname -a`), security patch date;
3. Root manager and its version, module list (`ls /data/adb/modules`);
4. Xposed / LSPosed module list and scope;
5. Whether device-spoofing / PIF / TEE simulation / app-hiding modules are in use.

Do not just send "third-party detected" or "Root detected". Without the detection evidence there is no way to tell which source is abnormal.

</details>

<details><summary><b>General troubleshooting method</b></summary>

1. **Reboot first, then retest**: transient processes, dynamic maps, boot mounts that have not yet converged, and time synchronization all affect the result. After a fresh boot, wait 20 seconds to 5 minutes.
2. **Retest with a minimal set**: keep only the root manager and the required Zygisk provider module, reboot and scan, and confirm whether the hit is still there.
3. **Bisect to locate it**: enable only one module at a time → reboot → rescan, converging step by step on the actual trigger (change only one variable each time).
4. **Handle mount-related items first**: metamodules, the Zygisk provider module's exclusion policy ("restore mounts only"), SusFS / PathMask style hiding.
5. **Keybox / TEE items**: after changing `keybox.xml`, `target.txt` or the security patch sync you **must reboot** before retesting.
6. **Do not stack multiple hiding solutions**: several hiding setups conflict with each other and create new contradictory items, which makes the device easier to flag, not harder.
7. **A few items are side-channel / unstable checks**: repeated scans in the same environment may disagree; rule out the intermittent case first, then locate the cause.

⚠️ **Safety note**: any operation that replaces the keybox, replaces RKP keys, changes the relock state, or changes the security patch sync alters the device's key and attestation state and is **not necessarily reversible**; assess the risk before acting, and back up the relevant directories if necessary.

</details>

---

## Table of contents

- [Prologue](#prologue)
- [Terminology](#terminology)
- [1. Root privileges and SELinux checks](#1-root-privileges-and-selinux-checks)
- [2. TEE and key attestation checks](#2-tee-and-key-attestation-checks)
- [3. Mount and namespace checks](#3-mount-and-namespace-checks)
- [4. Injection, hook and runtime tampering checks](#4-injection-hook-and-runtime-tampering-checks)
- [5. Environment, process and file checks](#5-environment-process-and-file-checks)
- [6. Device identity, property and kernel feature checks](#6-device-identity-property-and-kernel-feature-checks)
- [7. Side-channel and view consistency checks (new)](#7-side-channel-and-view-consistency-checks-new)
- [8. Hardware and sensor cross-validation](#8-hardware-and-sensor-cross-validation)
- [Built-in lists and properties](#built-in-lists-and-properties)
- [Final troubleshooting order](#final-troubleshooting-order)

---

## Prologue

<details><summary><b>Guide to building a minimal perfect hidden environment</b></summary>

- Genuinely unlocked device: **keybox/attestation module + Zygisk provider module + app-hiding module**
- Fake-relock / jailbreak / self-signed device: **Zygisk provider module + app-hiding module**

For **APatch / FolkPatch** users, loading **[NoHello.kpm](https://t.me/welikeandroid)** is also recommended to guard against side-channel checks; newer managers may have a built-in SELinux hook feature (which must be enabled manually), and on older versions you can additionally load **[SELinux_Hook.kpm](https://t.me/APatch_nightly)**.

For **KernelSU / SukiSU** users, update the manager to the latest CI, re-patch the image, and enable "Hide SELinux modifications / SELinux Hide" in the manager; AVC-log checks additionally require log spoofing on the manager side or a kernel audit patch.

> **Less is more**: one stable hiding solution plus clean properties passes more easily than three stacked layers of hiding.

</details>

<details><summary><b>Recommended modules (in no particular order)</b></summary>

#### Keybox / attestation module (TEE attestation)

- [TEESimulator-RS](https://github.com/Enginex0/TEESimulator-RS): the best-known keybox module after TrickyStore, updated frequently.
- [OhMyKeymint](https://github.com/qwq233/OhMyKeymint): behaves closer to AOSP, possibly with lower IO overhead.
- [Tricky-addon-Enhanced](https://github.com/Enginex0/tricky-addon-enhanced): a WebUI extension module usable with TS / TEES-RS.
- The original TrickyStore is updated slowly and has fallen behind the modules above; it is not recommended to keep using it.

#### Zygisk provider module

- [Zygisk-Next](https://github.com/Dr-TSNG/ZygiskNext): the most widely used standalone Zygisk implementation.
- Enable "Use Zygisk connector" and "Use anonymous memory"; for the exclusion policy, prefer "restore mounts only".

#### App-hiding module

- [HMA-OSS](https://github.com/frknkrc44/HMA-OSS): available in both Zygisk and Xposed builds, used to hide the root manager, Xposed modules and risky apps.

#### Metamodule / mount hiding

- [Hybrid-Mount](https://github.com/Hybrid-Mount/meta-hybrid_mount): a widely used third-party metamodule.
- [NoMount](https://github.com/maxsteeel/nomount): a VFS-oriented path redirection framework.
- [LKM-PathMask](https://github.com/Andrea-lyz/LKM-PathMask): path-based hiding (**not focused on environment hiding — use with caution**).

#### SELinux policy hiding

- [DirtySepolicy](https://github.com/LSPosed/DirtySepolicy): a reference implementation on the SELinux policy oracle side.
- [selinux_maf_fork](https://github.com/741afb7/selinux_maf_fork): an access filter adapted to the Magisk family.
- [selinux_hook](https://github.com/Admirepowered/selinux_hook): commonly used with KernelSU / APatch.

#### Other tools

- [Inode-Hijacker](https://github.com/YiJieqwq/Inode-Hijacker): fixes abnormal inode values of `/data/local/tmp`.
- [ADB-Trace-Cleaner](https://github.com/YiJieqwq/ADB-Trace-Cleaner): cleans up USB debugging traces.
- [FuseFixer](https://github.com/5ec1cff/FuseFixer): fixes the blind spot in risky-app scanning caused by restricted shared-storage reads.

</details>

<details><summary><b>Configuring modules correctly</b></summary>

#### Keybox / attestation module

a. Add Maple to the target app list (`/data/adb/tricky_store/target.txt`, or configure it through the WebUI; writing `com.maple.detect!` forces processing);
b. Configure the security patch date correctly (or simply delete `security_patch.txt`);
c. Configure the boot hash correctly (it is set automatically under normal conditions).

#### Zygisk provider module

Enable "Use Zygisk connector" and "Use anonymous memory"; choose either "restore mounts only" or the root manager's "kernel-level module unmount" — do not enable both at the same time.

#### App-hiding module

- **Blacklist mode**: apps this mode is applied to cannot see the apps in the blacklist template;
- **Whitelist mode**: apps this mode is applied to can only see the apps in the whitelist template.

Before it takes effect on Maple, first confirm that the hiding scope **contains only the apps that need hiding**; do not hide system components (IME, accessibility services, app store) along with them — that directly triggers Maple's visibility cross-check items.

</details>

---

## Terminology

<details><summary><b>Terminology and conventions</b></summary>

#### Genuinely unlocked device

The bootloader (ABL) unlock flag is genuinely set, and the system lets through low-level images that have not passed signature verification. The unlock state is faithfully reflected in the low-level system properties and in the KeyMint hardware-backed credentials (attestation).

#### Fake-relock device

On a genuinely unlocked device, a custom payload is planted and executed early in the boot chain to rewrite the in-memory unlock state and report it, so the device presents itself as "locked" while it actually still loads tampered boot / init_boot images.

#### Jailbreak device (unroot-free bootloader)

With ABL staying genuinely locked, Root is obtained through an Android / userspace privilege-escalation vulnerability (often accompanied by SELinux being set to permissive). This kind of Root is usually not persistent and must be re-exploited on every boot.

#### Self-signed device

The signing key trusted by the device's boot chain can be obtained or reproduced by a third party, so the user can sign tampered images with that key and have them pass verification while ABL stays locked.

#### Root manager

The complete set of privilege-management components: the user-facing app, the userspace daemon service, and the persistent hooks or memory patches deployed in the kernel or ramdisk.

#### Metamodule

A top-level module of the module-manager type. It provides no spoofing functionality itself; its core job is managing submodules and providing mount functionality. Only one metamodule can be installed on a device at a time.

#### Keybox / attestation module

A module that intercepts / replaces the KeyMint attestation path inside keystore2 and uses a keybox to generate "device is locked" attestation.

#### Zygisk provider module

A module that provides the Zygisk runtime: it injects code into Zygote / app processes and provides the runtime environment for other Zygisk modules.

#### App-hiding module

A module that operates on "package visibility": it intercepts the package query path in the target process or in system processes and hides the visibility of selected apps according to its configuration.

</details>

---

## 1. Root privileges and SELinux checks

<details><summary><b>Root privileges / SELinux modification detected (检测到Root权限/Selinux修改) (Daily basic)</b></summary>

#### How it is detected

Reads its own SELinux context inside the **app_zygote domain**, and uses sentinel contexts to build three control groups — positive control / negative control / file control — writes to `/sys/fs/selinux/access` through raw `selinuxfs`, and verifies the result the kernel writes back; at the same time it probes whether the `shell -> su` transition, the `magisk` / `ksu` / `apatch` domains, and reads of `ksu_file` / `lsposed_file` / `magisk_file` are still accepted by the live policy.

#### How to fix it

- **KernelSU / SukiSU**: update the manager and the kernel / LKM to the latest CI, enable "Hide SELinux modifications", re-patch and flash, then reboot.
- **APatch / FolkPatch**: load an SELinux-hiding KPM ([NoHello.kpm](https://t.me/welikeandroid), [selinux_hook](https://github.com/Admirepowered/selinux_hook), [selinux_maf_fork](https://github.com/741afb7/selinux_maf_fork)).
- **Magisk**: the Magisk family needs a kernel with audit / policy spoofing support, or a KPM solution; swapping only the manager does nothing.

#### Related items

Belongs to the same criterion family as `SELinux status check (SELinux状态检测)`, `SELinux policy check (SELinux策略检测)`, `SELinux context check (SELinux上下文检测)` and `Device is rooted / jailbroken (设备取得Root权限/免解)`, and they often hit together.

#### Notes

It is **normal** for an ordinary app to be unable to read `/sys/fs/selinux/enforce`; this item hits only when the policy view really can be read by this process.

</details>

<details><summary><b>KernelSU check (KernelSU检测) / KernelSU native interface check (KernelSU原生接口检测) / KernelSU active LKM (KernelSU活动LKM) / KernelSU kernel and SELinux markers (KernelSU内核与SELinux标记)</b></summary>

#### How it is detected

Reads KernelSU's version / feature interfaces, the `ksu_file` policy marker, active LKM and leftover kernel modules, and KernelSU-specific types in the SELinux policy, then cross-checks them against the manager package, the daemon process and the `/data/adb/ksu` structure.

#### How to fix it

- Update the KSU manager to the latest CI and re-patch (LKM mode) or re-integrate (GKI / Non-GKI mode).
- Enable the hiding features inside the manager, and add Maple to the exclusion list / the app-hiding module's blacklist template.

#### Notes

The manager app itself can be hidden by an app-hiding module; but the policy markers and leftover kernel modules live on the kernel side, and hiding a module app does not change that evidence.

</details>

<details><summary><b>KernelSU jailbreak device / jailbreak mode (KernelSU免解设备/越狱模式) (composite; prominent in Game mode)</b></summary>

#### How it is detected

Aggregates KernelSU Late-load, active LKM, SELinux markers, persistent artifacts, Metamodule, the Magica transaction window and jailbreak capability components, and checks whether the manager signature and the Magica components exist at the same time.

#### How to fix it

- Jailbreak (unroot-free bootloader) mode is not recommended.
- KSU / SukiSU: update the manager, re-patch, then enable `selinux_hide`.
- Turn off Late-load, remove the Metamodule and the persistent scripts, and restore the official kernel without KernelSU integrated.

#### Related items

Same family as `KernelSU Late-load script environment (KernelSU Late-load脚本环境)`, `KernelSU jailbreak persistent artifacts (KernelSU越狱持久化工件)`, `KernelSU Metamodule mount (KernelSU Metamodule挂载)`, `Magica boot transaction window (Magica启动事务窗口)`, `Magica transient process (Magica瞬时进程)` and `Jailbreak model Prop cross-check (免解机型Prop交叉检测)`.

</details>

<details><summary><b>KernelSU Late-load script environment (KernelSU Late-load脚本环境) / KernelSU jailbreak persistent artifacts (KernelSU越狱持久化工件) / KernelSU Metamodule mount (KernelSU Metamodule挂载) / Magica boot transaction window (Magica启动事务窗口) / Magica transient process (Magica瞬时进程)</b></summary>

#### How it is detected

On the native side it scans jailbreak script locations, `late-load` / `--magica` commands, persistent artifacts under `/data/adb`, Metamodule mounts whose source is KSU, and the transient combination of the Magica property window plus the 5555 endpoint.

#### How to fix it

Turn off Late-load / jailbreak mode, delete the corresponding leftovers and boot scripts under `/data/adb`, uninstall the Metamodule and the Magica components, restore the official kernel and boot image, then reboot.

#### Notes

The Magica-related items are **transient**: whether they hit depends on whether the window is open at scan time, so repeated scans may produce different results.

</details>

<details><summary><b>su binary check (su二进制检测) / Running su process (运行中su进程) / Running Root / cheat process (运行中Root/外挂进程) / Root process check (Root进程检测) / Visible kernel cheat runtime state (可见内核外挂运行态)</b></summary>

#### How it is detected

Looks for su and similar binaries in common paths; for visible processes it cross-checks four identity views (comm / argv[0] / exe / status Name — at least two must agree before it counts as su), and combines that with whether `attr/current` falls in a Root domain, kernel task names, and cheat daemon characteristics.

#### How to fix it

- Remove the su binaries you do not need, together with their execute permission (note that `/apex/com.android.virt/bin/su` on iQOO / vivo is a false-positive source — see Notes).
- Disable cheat / script tools and their resident daemons.
- General principle: **do not grant the detector Root privileges**.

#### Notes

On Android 16 an ordinary app can only see processes of its own UID, so "no Root process among the visible processes" is a normal result and does not mean the device is already clean; on most recent models these items produce no verdict.

</details>

<details><summary><b>Hidden path visibility check (隐藏路径可见性检测) (new in 1.3.0)</b></summary>

#### How it is detected

For read-only system paths of the su kind it performs three observations at once — `lstat`, `open` and parent-directory listing — and **any contradiction between them** is a hit: the metadata exists but opening returns "not found", or the metadata exists but the directory listing does not contain it. That is the typical signature of a path-hiding solution (PathMask / SusFS path hide).

#### How to fix it

- If it genuinely exists: uninstall the corresponding tool and clean up the leftovers.
- If there is a contradiction: disable path hiding / PathMask / SusFS path spoofing and file API hooks, then reboot and retest.

#### Related items

Same family as `Three-way path existence cross-check (路径存在性三路核对)`, `Mutually exclusive path evidence (路径证据互斥)` and `Java vs native path view divergence (Java与Native路径视图分歧)`.

</details>

<details><summary><b>SELinux status check (SELinux状态检测) / SELinux policy check (SELinux策略检测) / SELinux context check (SELinux上下文检测) / Policy serial check (Policy序列号检测) / SELinux dirty policy check (SELinux脏策略检测)</b></summary>

#### How it is detected

The SELinux filesystem state, the readability of the enforce and policy files, whether magisk / ksu / apatch / zygisk keywords and proprietary types remain in the policy, whether the current process has entered a Root domain, and whether the policy serial number has been replaced.

#### How to fix it

- Restore SELinux Enforcing (uninstall the kernel module that sets SELinux to permissive).
- Restore the official kernel / boot and the SELinux policy, without injected Root policy.
- Stop the context modifications and injection that apply to Maple.

#### Notes

"The enforce file exists but is not readable" is a normal state for an ordinary app and is not a hit.

</details>

<details><summary><b>Kernel setresuid behavior probe (内核setresuid行为探针) / SusFS-related probe (SusFS相关探针)</b></summary>

#### How it is detected

Abnormal kernel behavior on sensitive paths such as `setresuid` (child process killed / unexpected permissions), plus SusFS UID hook behavior.

#### How to fix it

Uninstall the SusFS / Root kernel module that modifies kernel behavior and restore the official kernel.

#### Notes

This item is strong evidence, but it depends on kernel implementation details and behaves differently across kernel versions; when the expected behavior is not observed it produces no verdict.

</details>

---

## 2. TEE and key attestation checks

<details><summary><b>TEE key validity check (TEE密钥有效性检测) (new in 1.3.0)</b></summary>

#### How it is detected

Generates a one-off hardware key and **actually signs with it**, then verifies the signature with the leaf certificate of the certificate chain, checking the challenge binding, the certificate chain signature, the trusted root, and the key / attestation security level. As long as the device declares hardware key capability and key generation fails, the leaf certificate does not match the key, the chain cannot be parsed, the challenge is not bound, or the key falls back to a software implementation, it is a hit.

#### How to fix it

- Restore a normal TEE: restore the official firmware and the trusted boot chain, and make sure keystore2 / KeyMint are not intercepted.
- Users of a keybox module: update the keybox module ([TEESimulator-RS](https://github.com/Enginex0/TEESimulator-RS) / [OhMyKeymint](https://github.com/qwq233/OhMyKeymint)), configure the keybox, the security patch and the boot hash correctly, then reboot.
- If the device is genuinely in a "blown TEE" state (it hits even with no modules installed): that is a hardware / firmware-level state, and can only be resolved by restoring the official firmware or sending the device for repair.

#### Related items

Same family as `Android Key Attestation availability (Android Key Attestation可用性)`, `Attestation chain structure consistency (Attestation链结构一致性)`, `Attestation chain root and implementation level (证明链根与实现级别)` and `TEE authorization tag cross-check (TEE授权标签对照)`.

#### Notes

The key / attestation state does not change within one boot; repeated scans in the same environment should agree. If the results differ every time, investigate Keystore hook modules first.

</details>

<details><summary><b>Android Key Attestation availability (Android Key Attestation可用性) / Attestation chain structure consistency (Attestation链结构一致性) / Attestation chain root and implementation level (证明链根与实现级别) / Key binding and random challenge (密钥绑定与随机挑战)</b></summary>

#### How it is detected

Generates a key with a random challenge and parses the KeyDescription: chain length, chain signature, whether it is anchored to the official root, whether the public key matches the leaf certificate, whether the challenge is echoed back, and whether the attestation and key security levels are self-consistent.

#### How to fix it

Disable TrickyStore / TEE / KeyMint simulation or interception modules and restore the official Keystore / KeyMint and the trusted boot chain; if you use a keybox module, complete the target list, the security patch and the boot hash as described in "Configuring modules correctly".

#### Related items

Same family as `TEE key validity check (TEE密钥有效性检测)`, `TrickyStore / similar module found (发现TrickyStore/类似模块)`, `TEE spoofing (TEE 伪造)` and `TEE spoofing (2) (Tee伪造(2))`.

</details>

<details><summary><b>TEE authorization tag cross-check (TEE授权标签对照) / TEE device identifier tag (TEE设备标识标签) / KeyUsage vs KeyPurpose consistency (KeyUsage与KeyPurpose一致性) / Signature algorithm vs issuer key consistency (签名算法与签发钥一致性) / KeyMint placeholder level inconsistency (KeyMint占位层级不一致) / KeyMint rollback resistance authorization (KeyMint回滚抵抗授权) / Attestation chain root and implementation level (证明链根与实现级别)</b></summary>

#### How it is detected

Runs several kinds of tag consistency validation on the attestation certificate chain: whether the leaf certificate's KeyUsage contradicts the KeyPurpose inside the extension, whether the leaf certificate's signature algorithm matches the issuer key algorithm, whether software-only tags appear in teeEnforced, whether vendor placeholder KeyMint tags can still be issued, whether the rollback resistance declaration is abnormal, and whether sensitive device identifier tags are not rejected.

#### How to fix it

Replace / update the keybox module, reboot and retest; if it still hits, the module implementation does not match the current system KeyMint version — try swapping [TEESimulator-RS](https://github.com/Enginex0/TEESimulator-RS) and [OhMyKeymint](https://github.com/qwq233/OhMyKeymint).

#### Notes

Some tag differences disappear naturally after a minor version update (the module follows the system update); these items belong to "attestation implementation consistency" and are not equivalent to Root.

</details>

<details><summary><b>TEE RootOfTrust boot state (TEE RootOfTrust启动状态) / Bootloader state check (Bootloader状态检测) / OEM unlock check (OEM解锁检测) / Bootloader property scan (Bootloader属性扫描) / Abnormal boot state (启动状态异常)</b></summary>

#### How it is detected

Reads bootloader lock-state properties (`ro.boot.flash.locked`, `ro.boot.verifiedbootstate`, `ro.boot.vbmeta.device_state`, `ro.secureboot.lockstate`, etc.) and cross-checks them against RootOfTrust (deviceLocked / verifiedBootState) in the trusted attestation report.

#### How to fix it

- **Genuinely unlocked device**: use a keybox module to hide the unlock state from the detector and add Maple to the target list; or restore the complete official firmware and then relock.
- **Fake-relock / jailbreak / self-signed device**: these already present themselves as "locked", so this item usually does not hit.
- If properties have been messed up (for example a leftover `ro.secureboot.lockstate=unlocked`): run `resetprop -n ro.secureboot.lockstate locked`, or uninstall the device-spoofing / property-spoofing module and reboot.

#### Notes

Restoring factory firmware and relocking the bootloader is an **irreversible** operation; first confirm that all partitions are factory-original and that the device supports it, to avoid bricking.

</details>

<details><summary><b>Attested security patch level (Attested安全补丁级别) / Vendor patch level cross-validation (Vendor补丁级别交叉验证) / Security patch cross-validation (安全补丁交叉验证) / Security patch age (安全补丁年限) / TEE key creation time (TEE密钥创建时间)</b></summary>

#### How it is detected

Cross-checks the OS / Vendor / Boot patch levels in the attestation certificate against the real runtime properties, and checks whether the security patch age and the key creation time are self-consistent.

#### How to fix it

- With a keybox module: write a patch date matching the system into `/data/adb/tricky_store/security_patch.txt`, or simply delete that file so the module takes the value automatically.
- Restore the real system properties and make sure the system and vendor / boot partitions come from the same firmware version.
- On some vendors (including China-specific custom ROMs) the build time and the patch time are inconsistent by themselves, so even a device that has never been unlocked can show a difference; retest first to confirm.

#### Related items

Same family as `Build integrity check (构建完整性检测)` and `Security patch age (安全补丁年限)`; it belongs to "timeline consistency between properties and attestation".

</details>

<details><summary><b>Boot hash consistency (Boot哈希一致性)</b></summary>

#### How it is detected

Compares the boot / vbmeta hash (such as `ro.boot.vbmeta.digest`) with VerifiedBootHash in the attestation.

#### How to fix it

Use the keybox module's add-on (TA) to configure the correct hash; on a genuinely unlocked device the hash often becomes all zeros, so you need to obtain the correct value from a trusted device and write it in.

#### Notes

The hash value changes with the vendor firmware version; do not reuse a value across versions.

</details>

<details><summary><b>Attestation certificate revocation status (CRL) (Attestation证书吊销状态（CRL）)</b></summary>

#### How it is detected

Takes the serial number of the leaf certificate in the attestation chain and compares it with the **official Google attestation status list** (`android.googleapis.com/attestation/status`); a revoked certificate is a hit. Offline, on timeout or when the request is blocked the item is not shown, and a list that cannot be fetched is never treated as "not revoked".

#### How to fix it

- Switch to a legitimate, non-revoked device attestation environment (keybox). Keyboxes circulating in public channels are very likely already on the revocation list.
- Restore the vendor's official solution: complete official firmware with a never-unlocked boot chain, where attestation is issued by the vendor / Google production roots.
- On network-restricted devices (no GMS, intranet), the item simply will not appear — that is normal.

#### Related items

Same family as `Proof chain root and implementation level (证明链根与实现级别)`, `TEE key validity check (TEE密钥有效性检测)` and `Android Key Attestation availability (Android Key Attestation可用性)`.

</details>

<details><summary><b>Proof chain root and implementation level (AOSP key) (证明链根与实现级别（AOSP密钥）)</b></summary>

#### How it is detected

Checks whether the root of the attestation chain is the **AOSP test root / software attestation root** instead of a vendor or Google production root, cross-checked with whether the attestation security level is a software implementation. On non-GMS devices the attestation root is not a Google root to begin with, which is normal.

#### How to fix it

- Switch to a keybox module built on a production root and configure the keybox correctly, then reboot and retest.
- Restore complete official firmware and a trusted boot chain.
- If the device itself only offers software attestation (no TEE, or a broken TEE), this is a hardware / firmware state that no module can fix.

#### Notes

This item often hits together with `TEE key validity check (TEE密钥有效性检测)`: one means the root is wrong, the other means the key / attestation never reaches the hardware.

</details>

<details><summary><b>Widevine device ID read (Widevine设备ID读取) / Widevine vs TEE cross-validation (Widevine与TEE交叉验证) / StrongBox capability declaration (StrongBox能力声明) / Knox Warranty check (Knox Warranty检测)</b></summary>

#### How it is detected

Reads MediaDrm's unique device ID, security level (L1 / L3) and SystemId, and cross-checks them against the TEE attestation result; it also checks whether the system declares StrongBox, and the Knox warranty bit on Samsung devices.

#### How to fix it

- Restore the official DRM, TEE and complete firmware.
- If StrongBox is declared but the result contradicts it: update the TEE-related module, or restore the official KeyMint.
- The Knox warranty bit is a historical e-fuse state and is **usually irreversible**.

#### Notes

Widevine being L3 is not abnormal by itself; it is only judged when it contradicts the declaration / TEE result.

</details>

<details><summary><b>Untrusted TEE environment (Soter) (TEE环境不可信（Soter）)</b></summary>

#### How it is detected

Cross-checks whether the Tencent Soter service program exists and the state of its service properties.

#### How to fix it

Use SusFS / [LKM-PathMask](https://github.com/Andrea-lyz/LKM-PathMask) to hide the Soter-related service paths, and use an app-hiding module to hide `com.tencent.soter.soterserver` from Maple.

#### Notes

On most devices this item produces no verdict: it only hits when both the Soter service program and its service property can be obtained.

</details>

---

## 3. Mount and namespace checks

<details><summary><b>Root mount check (Root挂载检测) / Bind mount specific check (Bind Mount专项检测) / Overlay mount check (Overlay挂载检测) / Cacerts mount check (Cacerts挂载检测) / System partition RW check (系统分区RW检测) / Hidden bind mounts (隐藏Bind-Mounts) / Hidden system mounts (隐藏系统挂载)</b></summary>

#### How it is detected

Scans `mounts` / `mountinfo` for magisk / `/data/adb` / KernelSU / APatch sources, overlay and bind rewrites of system partitions, remounted certificate directories, whether the system partition is writable, and whether CPU / SoC identity files (`/proc/cpuinfo`, `/sys/devices/soc0/serial_number`) are individually over-mounted.

#### How to fix it

- Uninstall the module that creates the mount, restore the overwritten system files, then reboot and retest.
- Use the Zygisk provider module's exclusion policy "restore mounts only", or switch metamodule ([Hybrid-Mount](https://github.com/Hybrid-Mount/meta-hybrid_mount) / [NoMount](https://github.com/maxsteeel/nomount)).
- Restore the official certificate directory and the partition writability state.

#### Notes

Ordinary OEM overlays, APEX, EROFS and dm devices are **not Root evidence by themselves** (these mounts exist in large numbers on factory devices).

</details>

<details><summary><b>Mount device consistency check (挂载设备一致性检测) / Mount ID consistency check (挂载ID一致性检测) / File descriptor mount visibility (文件描述符挂载可见性) / Mapped device visibility check (映射设备可见性检测) (the last three are new in 1.3.0)</b></summary>

#### How it is detected

Multiple observations of the same fact: ① whether the `st_dev` device number of a system path agrees with the mount record that overrides it; ② after opening a system path, whether the `mnt_id` in `/proc/self/fdinfo` agrees with the overriding record in the mount table; ③ whether the mount that a current process fd points to can be traced in the mount table (only judged when it "existed at boot and has now disappeared"); ④ whether the device number of a memory-mapped file can be traced in the mount table.

#### How to fix it

- A contradiction means a mount was hidden / unmounted inconsistently: disable mount hiding (SusFS / PathMask / the metamodule's mount-hiding feature), then reboot.
- If you are sure a certain module causes it: use the paths in the details to locate that module and uninstall it.

#### Notes

APEX itself is mounted from loop devices, the device number of an overlay-covered path may belong to the lower partition, and pseudo paths (such as `/dmabuf:`) do not correspond to any mount point - all of these are normal and do not hit.

</details>

<details><summary><b>Mount table drift check (挂载表漂移检测) / Runtime mount changes (运行时挂载变化) / Mount namespace drift check (挂载命名空间漂移检测) (new in 1.3.0)</b></summary>

#### How it is detected

Whether the mount table and the mount namespace change during runtime: the **mount table** is checked for added / disappeared / attribute-changed entries, and the **namespace** for `/proc/self/ns/mnt` being swapped (being moved into another mount namespace while the app runs is a typical mount-hiding trick). Mount ID gaps are for reference only and are not judged on their own.

#### How to fix it

Runtime mount changes usually mean "a module remounted / unmounted while the app was running": move the module's mount timing to early boot (`post-fs-data`), or use the "restore mounts only" policy; if the system itself is mounting things during boot (such as external storage or screen casting), wait 20 seconds to 5 minutes after boot before testing.

#### Related items

Same family as `Mount device consistency check (挂载设备一致性检测)` and `Mount ID gap check (挂载ID空洞检测)`; `Mount ID gaps (挂载ID空洞)` alone **do not count as a hit** (on factory devices the gaps come from concurrent allocation during namespace cloning and are very noisy).

</details>

<details><summary><b>Magic Mount / Mount loophole / Mount namespace (Mount命名空间) / PID namespace nesting (PID命名空间嵌套) / User namespace mapping (用户命名空间映射) / Root directory redirection (根目录重定向)</b></summary>

#### How it is detected

Traces of a module's magic mount on the system partition, the `/proc/self/ns/*`, `NSpid` / `uid_map` views, and whether `/proc/self/root` has been redirected, to determine whether the process has been chrooted / containerized.

#### How to fix it

- Use the Zygisk provider module's "restore mounts only", or switch metamodule.
- Exit the app clone / cloud phone / virtual container and run Maple directly from the normal system install path.

#### Notes

On most devices the `mount_namespace` item only provides information and does not directly reach a verdict (it is normal for an ordinary app to be unable to read the init namespace).

</details>

---

## 4. Injection, hook and runtime tampering checks

<details><summary><b>Frida check (Frida检测) / Frida file check (Frida文件检测) / Frida process check (Frida进程检测) / Frida port scan (Frida端口扫描) / Active probing of Frida candidate ports (Frida候选端口主动探测) / Frida-compatible protocol probe (Frida兼容协议探针)</b></summary>

#### How it is detected

File and process names, thread names, frida-agent / gadget in maps, sockets and listening endpoints, and active loopback connections plus D-Bus compatible protocol responses.

#### How to fix it

Stop `frida-server`, remove the gadget and the injection scripts, cancel injection into Maple, then reboot.

#### Notes

A resident `frida-server` triggers several items at once; if only a debug port is open without injection, only the port-related items hit.

</details>

<details><summary><b>Xposed / LSPosed check (Xposed/LSPosed检测) / Xposed class loading check (Xposed Class加载检测) / XposedBridge field check (XposedBridge字段检测) / LSPosed class loading check (LSPosed Class加载检测) / LSPosed scope check (LSPosed作用域检测) / LSPosed data directory (LSPosed数据目录) / LSPosed runtime leftovers (LSPosed运行时残留) / OAT / ODEX artifact check (OAT/ODEX工件检测) / Xposed module metadata scan (Xposed模块元数据扫描)</b></summary>

#### How it is detected

Xposed classes and fields, call stacks, ClassLoader chains, the metadata of installed modules (`META-INF/xposed`), the LSPosed data directory and fd / maps leftovers, compile artifacts (OAT / ODEX) and runtime mappings.

#### How to fix it

- Remove Maple from the LSPosed / Xposed scope, disable the related modules and reboot.
- If OAT / ODEX artifacts remain: uninstall and reinstall Maple, then retest.
- If you need to keep the modules, use an app-hiding module to hide the module information from Maple.

#### Related items

Same family as `Framework method localization (框架方法本地化)`, `Own method localization (自身方法本地化)`, `ART bridge attribution (ART桥接归属)` and `Boot class loading attribution (引导类加载归属)`.

</details>

<details><summary><b>Zygisk / Riru check (Zygisk/Riru检测) / Zygisk environment check (Zygisk环境检测) / Zygisk module injection check (Zygisk模块注入检测) / Zygisk thread check (Zygisk线程检测) / Zygisk process check (Zygisk进程检测) / Zygisk daemon (Zygisk守护进程) / Zygisk namespace check (Zygisk Namespace检测) / Zygisk SELinux context (Zygisk SELinux上下文)</b></summary>

#### How it is detected

Environment variables, `magisk.process`, module paths in maps, thread names, SO files in restricted paths, TracerPid, daemons, and namespace and SELinux context anomalies.

#### How to fix it

Turn off Zygisk / Riru or remove Maple from the scope, disable the corresponding modules and reboot; or upgrade [Zygisk-Next](https://github.com/Dr-TSNG/ZygiskNext) to the latest version.

</details>

<details><summary><b>Substrate check (Substrate检测) / ShadowHook / Dobby check (ShadowHook/Dobby检测) / Hook library enumeration check (Hook库枚举检测) / Inline hook check (Inline Hook检测) / SMAPS inline hook check (SMAPS内联钩子检测) / Hook callback check (Hook回调检测) / ECA H3</b></summary>

#### How it is detected

Hook library / class names, maps and `dl_iterate_phdr`, rewritten function prologues (inline hook opcode comparison), and executable anonymous segments and callback table anomalies in smaps.

#### How to fix it

Disable the hook modules that apply to Maple (SandHook / Dobby / ShadowHook / Pine / YAHFA, etc.), then reboot and retest.

#### Related items

Unrelated in category to `Native injection image attribution (Native注入映像归因)`, `Anonymous executable code check (匿名执行代码检测)` and `Signature algorithm vs issuer key consistency (签名算法与签发钥一致性)`, but they often appear at the same time.

</details>

<details><summary><b>Native injection image attribution (Native注入映像归因) / Anonymous executable code check (匿名执行代码检测) / Anonymous RWX segment check (匿名RWX段检测) / Memory injection conclusion (内存注入结论) / Memfd / JIT check (Memfd/JIT检测) / Deleted mapping check (Deleted映射检测) / ART JIT deleted memfd / Isolated process ART diff (隔离进程ART差分) / ART code page disk verification (ART代码页磁盘校验) / Critical native code page integrity (关键Native代码页完整性)</b></summary>

#### How it is detected

Cross-references named / hidden ELF files in maps, JNI table hooks, executable anonymous segments that are not JIT, suspicious memfds and on-disk code pages; the ART runtime views of the main process and of the isolated process are diffed against each other.

#### How to fix it

- Stop all injection, hook, speed-hacking and debugging modules targeting Maple, then reboot and retest.
- If system-level code pages were modified: restore the official ROM; if it is Maple's own artifact: uninstall and reinstall the official APK.

#### Notes

Ordinary JIT caches (`/memfd:jit-cache`) and `(deleted)` APK mappings are normal; only non-JIT executable anonymous segments and unknown memfds are abnormal.

</details>

<details><summary><b>Signal handler integrity (信号处理器完整性) / Game signal handler integrity (游戏信号处理器完整性) / Exception handler attribution (异常处理器归属) / Anti-debug / signal check (反调试/信号检测) / Debugger check (调试器检测) / Java debugger state (Java调试器状态) / TracerPid check (TracerPid检测)</b></summary>

#### How it is detected

Whether the handlers of key signals (SIGTRAP / SIGSEGV, etc.) point to anonymous executable regions or injected modules, whether the uncaught exception handler has been replaced, and `TracerPid` plus the debugger connection state.

#### How to fix it

Exit the debugger (Android Studio / LLDB / ptrace / Frida) and the crash-interception and hook modules, then reboot.

#### Notes

Maple itself deliberately raises one exception to verify whether the exception handler has been taken over; that is expected behavior and does not affect the result.

</details>

<details><summary><b>Three-way path existence cross-check (路径存在性三路核对) / Mutually exclusive path evidence (路径证据互斥) / Java vs native path view divergence (Java与Native路径视图分歧) / Two-way proc read cross-check (proc读取双路核对)</b></summary>

#### How it is detected

For paths sensitive to Root frameworks it collects evidence with three independent kernel observations (statx / openat / directory entry traversal) and adds the Java-layer view on top; if one path says it exists and another one definitely says it does not exist, that is a contradiction. `Two-way proc read cross-check (proc读取双路核对)` compares the results of reading the same `/proc` file through libc and through a direct syscall.

#### How to fix it

- Genuine path: uninstall the corresponding tool and clean up the leftovers.
- Multiple contradicting views: disable file hiding, PathMask, SusFS path spoofing or file API hooks, then retest.

#### Related items

Belongs, together with `Hidden path visibility check (隐藏路径可见性检测)`, `Two-way package visibility cross-check (包可见性双路核对)` and `Binder call path consistency (Binder调用路径一致性)`, to the "view divergence" family.

</details>

<details><summary><b>Binder call path consistency (Binder调用路径一致性)</b></summary>

#### How it is detected

Compares whether the Binder protocol version obtained through libc and through a direct syscall diverges consistently and stably (a leftover PLT hook).

#### How to fix it

Disable Binder interception, ioctl hooks and service-proxy modules for Keystore / TEE, restore the official services, then reboot.

</details>

<details><summary><b>Runtime environment variables (运行时环境变量) / Dynamic library path check (动态库路径检测) / Dynamic linker check (动态链接器检测) / Environment variable markers (环境变量标记) / Linker symbol check (Linker符号检测) / Linker bridge symbols (Linker桥接符号)</b></summary>

#### How it is detected

Whether `LD_PRELOAD`, `LD_LIBRARY_PATH`, `CLASSPATH`, `PATH` and `TMPDIR` contain Root / hook injection paths, and whether the linker namespace and the bridge symbols are abnormal.

#### How to fix it

Remove the injection launcher, terminal wrapper scripts and environment-variable-modifying modules, and launch Maple normally from the system launcher.

</details>

<details><summary><b>App signature self-check (应用签名自校验) / App debuggable flag (应用可调试标志) / App install source (应用安装来源) / APK install path verification (APK安装路径核对) / Own APK mapping consistency (自身APK映射一致性) / Own native library origin (自身native库来源) / Host executable (宿主可执行文件)</b></summary>

#### How it is detected

Multiple-signer detection, `FLAG_DEBUGGABLE`, install source (installer), whether the mapping paths of the APK and of its own so files fall in system-allowed locations, and whether the host executable is the system `app_process`.

#### How to fix it

- Uninstall the modified build and install the official Release APK from a trusted source.
- Exit the container / app clone, disable injection, then reinstall.
- If you use a self-compiled debug build: `FLAG_DEBUGGABLE` will be set — that is expected; retest with the Release build.

#### Notes

This item only shows a single-signature digest for manual verification; it performs no online validation.

</details>

---

## 5. Environment, process and file checks

<details><summary><b>Risky app check (风险应用检测) / Hiding tool check (隐藏工具检测) / Root manager check (Root管理器检测) / Hook framework app check (Hook框架应用检测) / Virtual framework check (虚拟框架检测) / Third-party environment tools (三方环境工具) / Hidden manager check (隐藏Manager检测)</b></summary>

#### How it is detected

Reads all installed packages (with versions) and matches them exactly against the built-in lists of risky apps / hiding tools / root managers / hook frameworks / virtual frameworks / tools (see the appendix for the lists).

#### How to fix it

- Keep only the tools you really need and uninstall the rest.
- If you need to keep them, use an app-hiding module ([HMA-OSS](https://github.com/frknkrc44/HMA-OSS)) to hide the corresponding apps from Maple.
- Make sure the hiding scope does not include system components (IME / accessibility services / app store), otherwise it triggers the visibility cross-check items.

#### Related items

Same family as `System component visibility cross-check (系统组件可见性交叉)`, `Two-way package visibility cross-check (包可见性双路核对)` and `Risky app / blacklisted package name directory check (风险应用/黑名单包名目录检测)`.

</details>

<details><summary><b>System component visibility cross-check (系统组件可见性交叉) (new in 1.3.0)</b></summary>

#### How it is detected

Takes the current IME (`Settings.Secure`'s default_input_method) and the packages of all enabled accessibility services, and cross-checks them against PackageManager query results: the system thinks it is in use / enabled **but the package query cannot see it** — that is a hit.

#### How to fix it

In the app-hiding module, cancel the hiding of the "IME / accessibility services" presets, or remove Maple from the hiding scope, then reboot.

#### Notes

This is the most direct symptom of an app-hiding module configured too broadly; it usually appears when the app store and the IME are hidden together.

</details>

<details><summary><b>Two-way package visibility cross-check (包可见性双路核对) / PM cross-method check (PM交叉方法检测)</b></summary>

#### How it is detected

Whether different query paths — `getInstalledPackages`, `getPackageInfo`, `createPackageContext`, etc. — return consistent results for a known root manager package.

#### How to fix it

Disable HMA, package list filtering, virtual containers and PackageManager hooks, then reboot; if all paths agree, use the details to decide whether to uninstall.

</details>

<details><summary><b>Accessibility service check (无障碍服务检测) / Developer options check (开发者选项检测) / ADB debugging check (ADB调试检测) / Active ADB endpoint probing (ADB端点主动探测)</b></summary>

#### How it is detected

The number of enabled accessibility services and automation keywords (autojs / autox / script / auto-clicker / macro), the developer options and USB debugging switches, ADB over WiFi, and whether the loopback 5037 / 5555 endpoints are connectable.

#### How to fix it

- Turn off the automation / auto-clicker / script tool services in the system accessibility settings.
- When you are not using ADB, turn off USB debugging (`settings put global adb_enabled 0`) and turn off "Wireless debugging".
- If you need to keep it enabled long-term: use [ADB-Trace-Cleaner](https://github.com/YiJieqwq/ADB-Trace-Cleaner) to clean up the debugging traces.

#### Notes

USB debugging / developer options are **user configuration** signals, labeled separately as low risk and not carrying the same weight as other Root evidence.

</details>

<details><summary><b>Temp directory permission check (临时目录权限检测) / Temp directory inode check (临时目录inode检测) / Temp directory ownership check (临时目录归属检测) / Temp directory timestamp check (临时目录时间戳检测)</b></summary>

#### How it is detected

The permissions of `/data/local/tmp` (default 0771), whether the inode is abnormally high (threshold 10000), whether the owner / group is shell(2000), and whether the modification time is later than the current time or the status change time is clearly later than the modification time.

#### How to fix it

- Permissions: `su -c chmod 771 /data/local/tmp`
- Owner / group: `su -c chown shell:shell /data/local/tmp`
- Abnormally high inode (the directory was deleted / recreated): use [Inode-Hijacker](https://github.com/YiJieqwq/Inode-Hijacker) or factory reset; on f2fs the inode can naturally be high, so this is a reference signal.
- Abnormal timestamp: delete the leftovers whose timestamps were altered, then reboot; do not rewrite the times with a script (that introduces new status-change differences).

#### Notes

"Directory not accessible" is recorded as unknown in Maple and is **not** treated as a hit.

</details>

<details><summary><b>Cheat / Root tool path list (外挂/Root工具路径清单) / Abnormal files (异常文件) / MT Manager (MT2 folder) / abnormal files (Mt管理器（MT2文件夹）/异常文件) / Other leftovers check (其他残留检测) / Root leftovers check (Root残留检测) / Root leftovers (extended) (Root残留(扩展)) / Data leftovers (数据残留)</b></summary>

#### How it is detected

Runs `lstat` on every entry of the built-in high-risk path list (see the appendix), and lists the concrete paths that hit.

#### How to fix it

Confirm each **concrete path** given in the details and delete the leftovers you no longer need; for the MT2 folder you can change the path in MT Manager's settings and then delete the old directory.

#### Related items

Same family as `Fast scan for abnormal sh files (异常sh文件快速扫描)`, `Fast scan of external storage scripts (外置存储脚本快速扫描)` and `Scheduler / thermal module check (调度/温控模块检测)`.

</details>

<details><summary><b>Fast scan for abnormal sh files (异常sh文件快速扫描) / Fast scan of external storage scripts (外置存储脚本快速扫描) / Zero-width disk scan app check (零宽扫盘应用检测) / Blacklisted package name directory check (黑名单包名目录检测) / Blacklisted directory hidden app check (黑名单目录隐藏应用检测)</b></summary>

#### How it is detected

Within the shared storage the app can access, it performs a bounded scan (depth 4, 256 directories, 4000 entries, 1.5 seconds) for `.sh` scripts, blacklisted package name directories and hidden app directories.

#### How to fix it

Check the concrete paths in the details and delete the scripts and directories you have confirmed you no longer need; anomalies in directories such as `/storage/emulated/0/Android/data/<package name>` usually come from leftovers of cheat / app-cloning tools.

#### Notes

The `.sh` extension does not by itself prove malicious behavior; a hit only means "a script file exists".

</details>

<details><summary><b>System image residue check (系统镜像残留检测) — Ultra</b></summary>

#### How it is detected

Bounded scan inside the shared storage the app can reach for boot images and flash backups such as `boot.img` / `init_boot.img` / `vbmeta.img`, plus abnormal `.xml` files in the root of shared storage (Ultra-mode item).

#### How to fix it

Confirm the source from the paths listed in the details and delete them; such images usually come from flashing / patching boot, and abnormal `.xml` files are usually configuration written by modding modules. Keeping them in shared storage also triggers the "modding tool residue" items.

#### Notes

A hit only means "boot image files exist in shared storage" — it does not mean the device is currently rooted, but flash backups kept there waste space and are easy to flag.

</details>

<details><summary><b>Abnormal system executable files (系统异常可执行文件) / Scheduler / thermal module check (调度/温控模块检测) / Kernel cheat driver check (内核外挂驱动检测) / Kernel module check (内核模块检测) / Kernel module leftovers (内核模块残留)</b></summary>

#### How it is detected

Unusual system executables such as `/system/bin/adb` and `/system/bin/fastboot`; scheduler / thermal module signatures such as `/dev/cpuset/AppOpt`, `/data/encore/*`, `encore_profiler`, `yshell`; loaded kernel modules and cheat driver names.

#### How to fix it

Uninstall the corresponding scheduler / thermal / cheat modules and restore the official system partition files; vendor-modified builds commonly include `adb`/`fastboot` — flash back the official build, or use [LKM-PathMask](https://github.com/Andrea-lyz/LKM-PathMask) to hide them.

#### Notes

It is normal that `/proc/modules` is not readable (an ordinary app has no permission for it); in that case this item produces no verdict.

</details>

<details><summary><b>Terminal environment trace check (终端环境痕迹检测) / PTY node ownership (PTY节点归属) (new in 1.3.0)</b></summary>

#### How it is detected

Reads the owner and the SELinux label of `/dev/pts/*`; if Root framework labels such as `ksu_file` / `magisk_file` / `apatch_file` appear, it is a hit (an unreadable label is only recorded as unknown).

#### How to fix it

Close resident Root terminal / debug sessions, stop interactive shells opened in a Root domain, then reboot.

#### Notes

Ordinary terminals (including Maple's own pty) do not hit; only labels that fall in a Root framework domain count.

</details>

<details><summary><b>Process cgroup context (进程Cgroup上下文) / Suspicious detection runtime environment (检测运行环境可疑) / Container (容器) / App cloning (多开) / Process identity quadruple (进程身份四元组) / Data directory ownership consistency (数据目录归属一致性)</b></summary>

#### How it is detected

Whether `/proc/self/cgroup` contains magisk / zygisk / ksu / apatch / lsposed / frida markers; whether the `Uid` / `Gid` quadruple is fully equal; whether `dataDir` / `sourceDir` match the locations required by the system; and whether the process is in a nested namespace.

#### How to fix it

Exit the app clone / cloud phone / virtual container, uninstall the current copy and reinstall the official APK from a trusted source.

#### Related items

Same family as `PID namespace nesting (PID命名空间嵌套)`, `User namespace mapping (用户命名空间映射)` and `Root directory redirection (根目录重定向)`.

</details>

<details><summary><b>Vold isolation property check (Vold隔离属性检测) / Log buffer property check (日志缓冲区属性检测) / Core platform API policy check (核心平台API策略检测) / Hiding solution property check (隐藏方案属性检测) (new in 1.3.0)</b></summary>

#### How it is detected

Reads `persist.sys.vold_app_data_isolation_enabled`, `persist.logd.size*` (non-empty is a hit), `persist.debug.dalvik.vm.core_platform_api_policy`, and the properties of path-hiding solutions aimed at detectors.

#### How to fix it

- Vold isolation: turn off "Vold app data isolation" in HMA / the app-hiding module; if it remains, run `resetprop -p --delete persist.sys.vold_app_data_isolation_enabled` and reboot.
- Log buffer properties: delete the scripts / modules that write these properties.
- Hiding solution properties: uninstall the corresponding path-hiding script (these properties show that a detector-targeted hiding solution has been run on the device).

#### Notes

A `persist.logd.size*` property that exists with an empty value is normal.

</details>

---

## 6. Device identity, property and kernel feature checks

<details><summary><b>Emulator property check (模拟器属性检测) / Emulator path check (模拟器路径检测) / GPU emulation check (GPU模拟检测) / CPU emulator signatures (CPU模拟器特征) / Virtual framework check (虚拟框架检测)</b></summary>

#### How it is detected

Runtime evidence of QEMU / goldfish / ranchu / VirtualBox / VMware / Genymotion / Nox / LDPlayer (device nodes, properties, drivers, process names), combined with reference surfaces such as battery / charging and SIM state.

#### How to fix it

- Real device: disable hardware spoofing and cloud-phone / container modules, and restore the official properties, kernel and drivers.
- An emulator cannot be turned into a real device by changing properties; this item appearing on an emulator is expected.

#### Notes

Testing with no SIM card inserted while the battery is full and charging can produce a combination of features similar to an emulator; retest in a realistic state.

</details>

<details><summary><b>Build fingerprint integrity check (构建指纹完整性检测) (new in 1.3.0) / Partition fingerprint consistency check (分区指纹一致性检测) / Fingerprint cross-validation (Fingerprint交叉验证) / Fingerprint component consistency check (Fingerprint分量一致性检测) / Fingerprint component cross-validation (Fingerprint分量交叉验证)</b></summary>

#### How it is detected

① The build fingerprint must satisfy the standard segmented structure (`brand/product/device:release/id/incremental:type/tags`); ② between partition fingerprints only the **build posture segments** are compared (release / id / incremental / type / tags) — the brand and codename segments do not participate; ③ within the same partition the fingerprint components are cross-checked against independent properties.

#### How to fix it

- Uninstall device-spoofing / PIF / property-spoofing modules, restore the official properties, then reboot.
- Property leftovers (for example the fingerprint was changed to only the brand name): clear the corresponding `resetprop` script or the module's `system.prop`.

#### Notes

On Xiaomi models the vendor segment is fixed to a generic base such as `mivendor_<soc>_cn`, which is **legitimately different** from the device codename and is a normal layout.

</details>

<details><summary><b>Device model spoofing check (机型伪装检测) / Jailbreak model Prop cross-check (免解机型Prop交叉检测) / OemPropertyConsistency / Property consistency check (属性一致性检查) / Property context directory enumeration (属性上下文目录枚举)</b></summary>

#### How it is detected

Whether vendor-specific properties and device identifiers on OPPO / vivo and similar devices are self-consistent (missing when they should be present / present when they should not be), whether key properties are empty or come from an abnormal source, and whether property context directories have been mounted additionally.

#### How to fix it

Remove property-filtering and device-spoofing modules and restore the real properties; if a third-party ROM is missing properties itself, you can only wait for the ROM maintainer to fix it or flash back to the official system.

</details>

<details><summary><b>Eight device property dual-source cross-check (8项设备属性双源核对) / Build.TAGS cross-check (Build.TAGS交叉) / Security patch cross-validation (安全补丁交叉验证) / Build integrity check (构建完整性检测) / Boot parameter raw value cross-validation (启动参数原始值交叉验证) / Kernel boot security context (内核启动安全上下文)</b></summary>

#### How it is detected

Whether the results of reading the same property from four sources — Java / Build constants / `getprop` / the native property area — agree; whether `cmdline` / `bootconfig` agree with `ro.boot.*`, and whether they contain unlock, permissive, verity-disabled, `/data/adb`, KernelSU / APatch / SUSFS markers.

#### How to fix it

Disable device-spoofing / PIF / resetprop modules, restore the official properties and boot partitions, then reboot.

#### Related items

Same family as `Play Integrity Fix leftovers (Play Integrity Fix残留)` and `Property consistency check (属性一致性检查)`.

</details>

<details><summary><b>Play Integrity Fix leftovers (Play Integrity Fix残留)</b></summary>

#### How it is detected

The native layer reads the properties of enabled PIF / PIHooks / PixelProps / property-spoofing solutions (`persist.sys.pihooks*`, `persist.sys.pixelprops*`, `persist.sys.spoof.gms`, etc.).

#### How to fix it

Disable and uninstall the corresponding modules, clean up their actual leftovers (including `system.prop` and `service.d` scripts), then reboot.

</details>

<details><summary><b>Property area context check (属性区上下文检测) (new in 1.3.0) / Property area readability cross-check (属性区可读性核对)</b></summary>

#### How it is detected

Enumerates property area context names from the process's own mappings; a hit on spoofing / hiding markers (spoof / pif / pixelprop / hide / ksu / magisk / apatch / lsposed / xposed / tricky, etc.) counts as a hit; it also checks whether the property area files are readable and whether the serial number and the info files are self-consistent.

#### How to fix it

Uninstall the module that created the custom property area (such contexts are usually written by property-spoofing modules), then reboot.

#### Notes

Normal contexts that exist on factory devices, such as `vendor_socket_hook_prop`, do not hit.

</details>

<details><summary><b>Custom ROM check (自定义ROM检测) / Kernel version check (内核版本检测) / Kernel build signature check (内核构建特征检测) / Third-party kernel (三方内核)</b></summary>

#### How it is detected

Matches system / device model features against the built-in third-party ROM and kernel keywords (LineageOS, the `-dirty` suffix of self-compiled kernels, third-party kernel names, etc.).

#### How to fix it

- Restore the official ROM and kernel; or use a kernel-level solution such as SUSFS to hide the kernel name (note that such hiding can itself trigger other items).
- If the kernel version contains `-dirty`: recompile a clean build, or restore the official boot.

#### Notes

A third-party ROM's own characteristics cannot be fully eliminated with modules; this is an "inherent environment difference".

</details>

---

## 7. Side-channel and view consistency checks (new)

<details><summary><b>Netlink query policy check (Netlink查询策略检测) (new in 1.3.0)</b></summary>

#### How it is detected

Issues one query to the kernel's `sock_diag` as an ordinary app: under a normal policy it should be rejected; if a response comes back (especially **socket records of other UIDs**), the kernel netlink query policy has been relaxed.

#### How to fix it

Restore the official kernel / SELinux policy; if some hiding solution opened up this interface, disable it and reboot.

#### Notes

A factory system rejects this query from an ordinary app anyway, so "rejected" is the normal result.

</details>

<details><summary><b>Process protection view consistency (进程防护视图一致性) (new in 1.3.0)</b></summary>

#### How it is detected

Cross-checks three paths: Seccomp / NoNewPrivs from `/proc/self/status`, a direct `prctl` query, and a libc read; if any two disagree it is a hit (typically `/proc` reads or syscalls are being intercepted).

#### How to fix it

Disable `/proc` filtering, file API hooks and hiding modules targeting Maple, then reboot.

#### Related items

Belongs, together with `Two-way proc read cross-check (proc读取双路核对)` and `Binder call path consistency (Binder调用路径一致性)`, to "multi-view observation divergence".

</details>

<details><summary><b>Zygote supplementary GID check (Zygote补充GID检测) (new in 1.3.0)</b></summary>

#### How it is detected

Checks whether the process's supplementary GIDs still contain the **app-specific groups** (`20000+appId`, `50000+appId`) — an inevitable feature of processes forked from Zygote; if they are missing, the supplementary groups have been restricted (common with "restrict zygote privileges" style hiding configurations).

#### How to fix it

In the app-hiding module, turn off the "restrict zygote privileges / SHARED USER GID / APP ZYGOTE GID" style switches for Maple, then reboot.

#### Notes

On Android 14+ an app does not have GID 3009 (readproc) anyway, which is normal.

</details>

<details><summary><b>OBB path view consistency (OBB路径视图一致性) (new in 1.3.0)</b></summary>

#### How it is detected

Reads the app's own OBB path in four ways — "does the file exist", `stat`, `lstat` and parent directory listing — and any contradiction between the results is a hit (it means a module is intercepting reads of that path).

#### How to fix it

Disable hiding modules aimed at scanning / file reads; if necessary, turn off path interception policies other than "restrict zygote privileges" for Maple.

</details>

<details><summary><b>Clock source consistency (时钟源一致性) / Kernel clock cross-validation (内核时钟交叉验证) / Boot time consistency (开机时刻一致性) / Process start time consistency (进程启动时刻一致性)</b></summary>

#### How it is detected

Whether multiple clock sources (monotonic / boottime / uptime / kernel btime / process starttime) diverge persistently within a stable sampling window.

#### How to fix it

Turn off speed hacking, time hooks and virtual containers, restore the system's automatic date and time, then reboot; if it keeps hitting, investigate clock-related kernel modules.

</details>

---

## 8. Hardware and sensor cross-validation

<details><summary><b>CPU / GPU / platform three-way validation (CPU/GPU/Platform三向验证) / CPU + GPU cross-validation (CPU+GPU交叉验证) / Hardware cross-validation report (硬件交叉验证报告) / Chip identity recognition (芯片身份识别) / CPU feature verification (CPU特性校验) / sysfs cross-validation (sysfs交叉验证) / Direct Vulkan physical GPU enumeration (Vulkan物理GPU直枚举) / GPU driver verification (GPU Driver验证)</b></summary>

#### How it is detected

Whether the hardware identities from `cpuinfo`, system properties, driver nodes, Vulkan physical devices, sysfs, CPU topology and device nodes contradict each other (including ISO source, frequency table, core count, SoC serial number).

#### How to fix it

Disable device-spoofing, KPM file overwriting and hardware-spoofing modules, and restore the official kernel, drivers and system properties.

#### Notes

Different device models with the same SoC may have different frequency tables; Maple avoids single-point differences by only judging when multiple sources disagree.

</details>

<details><summary><b>CPU frequency (CPU频率) / CPU topology (CPU拓扑) / Total memory check (内存总量检测) / Storage space check (存储空间检测) / Temperature sensor fingerprint (温度传感器指纹) / Thermal zone sensor count (热区传感器计数) / Thermal zone temperature read (热区温度读取) / Power supply check (电源检测) / Input device check (输入设备检测)</b></summary>

#### How it is detected

Whether the frequency table and topology are self-consistent, whether the memory and storage capacities match the device model, whether the number / type / readings of thermal zone sensors are abnormal, and whether input devices are virtual devices.

#### How to fix it

- Hardware spoofing / device-spoofing modules: uninstall them and restore the official properties.
- Thermal / scheduler modules that rewrite thermal zone nodes: uninstall the corresponding modules.
- Virtual input devices (auto-clicker / mapping tools): turn off the corresponding tool's accessibility or input injection service.

#### Notes

Some models (especially European / Indian variants) have a sensor layout different from the China variant; single-point differences are normal.

</details>

---

## Built-in lists and properties

<details><summary><b>Appendix A: key system properties read by Maple (partial)</b></summary>

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

<details><summary><b>Appendix B: high-risk files and directories Maple watches (partial)</b></summary>

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

> System paths such as `/proc`, `/sys`, `/dev` and `/system` are objects the detector **reads** — **do not delete them**.

</details>

<details><summary><b>Appendix C: risky app / hiding tool / root manager package names (partial)</b></summary>

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

> The lists are updated with each version; the in-app details are authoritative for the complete lists (the package names and versions that hit are listed directly in the detection evidence).

</details>

---

## Final troubleshooting order

<details><summary><b>Recommended order</b></summary>

1. **Reboot and scan again**, to rule out the influence of transient processes, dynamic maps and boot mounts that have not yet converged (after a fresh boot, wait 20 seconds to 5 minutes).
2. **Disable the corresponding module based on the details**, changing only one variable at a time; do not stack several hiding solutions at once.
3. **Remove Maple from the Xposed / Zygisk / Frida / speed-hack scopes** and reboot.
4. **Uninstall the tools that clearly hit**, and clean up the leftovers that are confirmed to belong to them (paths, properties, scripts).
5. **If properties / mounts / kernel / TEE are still abnormal**, restore the complete official firmware of the same version.
6. **Only relock the bootloader after confirming that all partitions are factory-original and that the device supports it**, to avoid bricking.
7. **If it still does not pass**, submit the complete detection evidence and environment information — do not just send the result color.

</details>
