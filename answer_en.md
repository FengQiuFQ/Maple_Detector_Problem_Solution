# Maple Detector Problem Solution (for version 2.1.0) — English

> Compiled by: Maple | Last updated: 2026-10-22 | Verified version: 2.1.0 (total checks: Daily 157 / Game 148 / Ultra Daily 295 / Ultra Game 375; 155 / 146 / 292 / 371 with package-name detection disabled)
> Credits: [Credits list](thanks.md) | For reference only; actual results vary with device / system / kernel environment.
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

- [2.1.0 changes](#210-changes)
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

## 2.1.0 changes

<details><summary><b>What was added, merged or withdrawn in this version</b></summary>

#### Added

- **Root/SU path check (merged item)**: the per-path "Root path: …" entries and "su binary check" are merged into one card; expanding "Detection basis" shows the result for every checked location.
- **Previous SELinux domain check**: verifies whether the domain the process migrated from lies outside the application baseline.
- **SELinux domain audit-log side channel**: checks whether the denial records visible to this app contain a privileged domain of another process.
- **Kernel private-interface probe**: a kernel behaviour check that depends on neither files nor strings.
- **JIT cache injection signature**: checks whether this process's JIT cache mapping carries an injection module entry signature.
- **Hidden framework mount source**: checks whether a hidden framework name appears as a whole mount source or mount point.
- **Namespace creation time**: checks the establishment order of the mount view and the network view.

#### Criteria adjusted

- **Sideloaded third-party accessibility service**: only **enabled** services count; declared but disabled ones no longer hit.
- **User app holding a system-level permission**: system packages, updated system packages, factory preload directories and store installs are excluded.
- **Runtime framework classes / LSPosed class loading**: classes shipped inside Maple's own package no longer count.
- **APK mtime does not match install time**: only applications in the self-installable location are compared.
- **TEE and key attestation family**: the attestation generation wait is relaxed from 10 s to **20 s**, so slow TEEs are no longer truncated into "not evaluated".

#### Withdrawn

- **Install time earlier than build time**: the system image build time is later than the install time of most applications after every OTA, so the criterion has no discriminating power and was removed.

#### Display

- The "Risk reason" and "Suggested action" sections are removed from every item; only "Detection basis" and "Evidence" remain.

</details>

<details><summary><b>Root/SU path check (merged item) (Root/SU路径检测)</b></summary>

#### How it is detected
Every classic Root / SU location is checked for existence, object type and execute bit, and those observations are folded into three states: any confirmed executable object is a hit; only when every location is confirmed absent is a negative conclusion formed; if any location cannot be confirmed, the whole item forms no conclusion. Merging only collapses the display — the per-location results are all kept under "Detection basis", with hits prefixed by "重要:" (important).

#### How to fix it

- Remove unneeded su binaries and their execute bits, and clean up executable leftovers from root managers.
- General rule: **never grant the detector root**, and keep no privilege-escalation binary with an execute bit.

#### Related items

`Running su process`, `Root process check` and `检测到Root权限/Selinux修改` belong to the same family and usually hit together.

#### Notes

- "Cannot be confirmed by the current process" only means invisible — not absent, and not a pass.
- Generic directories are not root evidence by themselves; an unreadable one only leaves a note and never drags the item into not-evaluated.

</details>

<details><summary><b>Previous SELinux domain check (上一次SELinux域检测)</b></summary>

#### How it is detected
Checks whether the domain the process came from before migrating lies inside the application baseline: a zygote / init derived chain and the ordinary application domains are normal; a root-framework privileged domain is a hit; an unknown domain outside the baseline is reported as medium risk; an unreadable value is treated as **unknown**, never as "not found".

#### How to fix it

- A privileged domain means the process migrated back from a root domain: focus on the injection chain, disable global injection in the manager and narrow the scope of app-hiding modules.
- An "unknown domain outside the baseline" is usually an OEM-specific domain; confirm the system build before treating it as tampering.

#### Related items

Same family as `SELinux上下文检测`, `检测到Root权限/Selinux修改` and `SELinux域日志侧信道`; they corroborate each other.

#### Notes

- **The zygote domain is a normal baseline, not evidence**: an app process is derived from the zygote process and then migrates into its app domain, so it legitimately came from the zygote domain.
- Only the domain-type field is compared, not the whole string, so similar-looking text in the level or category never triggers.

</details>

<details><summary><b>SELinux domain audit-log side channel (SELinux域日志侧信道)</b></summary>

#### How it is detected
Checks whether "the denial records visible to this app" correspond to "the denials triggered by this process accessing other processes": take a log baseline, perform a round of accesses that cross domain boundaries, then read the log back and look for another process's privileged domain in the newly appeared records. A privileged domain is a hit; only ordinary domains means the channel works and no privileged domain was seen; when no such denial record is readable at all the item is **unknown**.

#### How to fix it

- A hit means a root-domain process exists and the system writes audit records into an app-visible buffer: clean up the root-domain process and the injection chain first.
- To close it system-side: upgrade to a build containing the procfs audit hiding patch.

#### Related items

Corroborates `上一次SELinux域检测` and `检测到Root权限/Selinux修改`; it is a different channel from the side-channel criterion inside `检测到KernelSU特征`.

#### Notes

- **When no denial record is readable the item is "not evaluated", not a pass**: a patched system never exposes such records to applications in the first place.
- Some hiding schemes rewrite a privileged domain into an ordinary application domain; this item cannot see that rewrite and needs other items as corroboration.

</details>

<details><summary><b>Kernel private-interface probe / JIT cache injection signature (内核私有接口探针 / JIT缓存注入签名)</b></summary>

#### How it is detected
Both check kernel and runtime behaviour rather than files or strings: one calls a kernel interface with a non-standard option and observes whether the caller's memory page is faulted back in — a stock kernel errors out immediately and never touches that page; the other checks whether this process's JIT cache mapping carries an injection module entry signature. The first one also runs a control measurement, and an abnormal control voids the whole measurement.

#### How to fix it

- Both are kernel / injection-side behavioural evidence; file-level or string-level hiding cannot evade them.
- On a hit, revert to an unintegrated stock kernel or disable the corresponding hiding feature.

#### Related items

Corroborates `KernelSU检测`, `检测到Root权限/Selinux修改` and the injection family.

#### Notes

- If the eviction baseline is unavailable (the kernel does not honour voluntary eviction), access is denied, or the control is abnormal, the item is "not evaluated".
- If this process has no JIT cache mapping, the item is likewise "not evaluated" and must not be read as clean.

</details>

<details><summary><b>Hidden framework mount source / Namespace creation time (隐藏框架挂载源 / 命名空间创建时间)</b></summary>

#### How it is detected
- Mount source: checks whether the **complete** hidden framework name appears as a mount source or mount point (no loose substring matching, so an ordinary module directory is not counted).
- Namespace: checks the establishment order of the mount view and the network view; a mount view clearly later than the network view means the mount view was rebuilt inside the process.

#### How to fix it

- Mount-source hit: unmount / remove the corresponding hiding module, or switch to a scheme that does not rebuild the mount view.
- Namespace time skew: look for modules that rebuild the mount view inside the application process.

#### Notes

- When the mount tables are unreadable the item is "not evaluated".
- The namespace criterion never exceeds medium risk and always prints both timestamps for manual review.

</details>

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
Verifies its own security context during the app preload stage, and, using several groups of clearly designated samples, checks whether the policy view and the results written back by the kernel agree with each other; it also checks whether privileged domain transitions and policy flag reads are still accepted by the currently effective policy.
#### How to fix it

- **KernelSU / SukiSU**: update the manager and the kernel / LKM to the latest CI, enable "Hide SELinux modifications", re-patch and flash, then reboot.
- **APatch / FolkPatch**: load an SELinux-hiding KPM ([NoHello.kpm](https://t.me/welikeandroid), [selinux_hook](https://github.com/Admirepowered/selinux_hook), [selinux_maf_fork](https://github.com/741afb7/selinux_maf_fork)).
- **Magisk**: the Magisk family needs a kernel with audit / policy spoofing support, or a KPM solution; swapping only the manager does nothing.

#### Related items

Belongs to the same criterion family as `SELinux status check (SELinux状态检测)`, `SELinux policy check (SELinux策略检测)`, `SELinux context check (SELinux上下文检测)` and `Device is rooted / jailbroken (设备取得Root权限/免解)`, and they often hit together.

#### Notes

It is **normal** for an ordinary app to be unable to read `/sys/fs/selinux/enforce`; this item hits only when the policy view really can be read by this process.

</details>

<details><summary><b>KernelSU traits detected (检测到KernelSU特征)</b></summary>

#### How it is detected
During the application preload stage, this process's own runtime state is cross-checked: whether the observations obtained along the different read paths agree under the same conditions, whether the same input yields the same result through both parsing paths, whether the process baseline still keeps its parent-derived relationship, and whether the storage view and the policy view contradict each other. Any inconsistency counts as a hit; the sub-criteria are independent, and only the ones that hit are listed; anything that cannot be read is treated as **unknown**, never as “not found”.
#### How to fix it

- **Hide the app**: every observation above uses this app's own runtime state, so adding Maple to a hiding list (a hiding tool such as HMA) directly blocks the observation surface.
- **Update to the latest CI build**: update both the Root manager and the kernel / LKM to the latest CI build, then reboot.

#### Related items

Same family as `KernelSU check / KernelSU native interface check / KernelSU active LKM / KernelSU kernel and SELinux markers`, `Root privilege / SELinux modification detected (检测到Root权限/Selinux修改)` and `Device obtained Root / jailbreak (设备取得Root权限/免解)`; they often hit together.

#### Notes

- This item contains 6 sub-criteria (side-channel runtime traits, policy view consistency, policy state, runtime environment, process baseline, storage layout). **Any one of them hitting is critical.**
- Updating to the latest CI build matters: the relevant fixes land in CI builds first and in official releases later, so comparing against an old release number will still hit.
- Fixing this item does not make the other timing, mount or policy checks disappear; **uninstalling the manager** or **not granting this app Root** does not change this criterion.

</details>

<details><summary><b>KernelSU check (KernelSU检测) / KernelSU native interface check (KernelSU原生接口检测) / KernelSU active LKM (KernelSU活动LKM) / KernelSU kernel and SELinux markers (KernelSU内核与SELinux标记)</b></summary>

#### How it is detected
Cross-checks the version and feature information exposed by the custom implementation, the proprietary marker in the policy, and the active and leftover state of kernel modules against the manager app, the resident component and its data directory structure.
#### How to fix it

- Update the KSU manager to the latest CI and re-patch (LKM mode) or re-integrate (GKI / Non-GKI mode).
- Enable the hiding features inside the manager, and add Maple to the exclusion list / the app-hiding module's blacklist template.

#### Related items

Same-family criteria are now consolidated into `KernelSU traits detected (检测到KernelSU特征)` and are reported under that item when they hit.

#### Notes

The manager app itself can be hidden by an app-hiding module; but the policy markers and leftover kernel modules live on the kernel side, and hiding a module app does not change that evidence.

</details>

<details><summary><b>KernelSU jailbreak device / jailbreak mode (KernelSU免解设备/越狱模式) (composite; prominent in Game mode)</b></summary>

#### How it is detected
Checks a late-load mode's script environment, kernel module activity, policy markers, persistence artifacts, mount sources and boot-time transients for mutual consistency, and whether the manager and related components coexist.
#### How to fix it

- Jailbreak (unroot-free bootloader) mode is not recommended.
- KSU / SukiSU: update the manager and the kernel / LKM to the latest CI, re-patch, then enable `selinux_hide`.
- Turn off Late-load, remove the Metamodule and the persistent scripts, and restore the official kernel without KernelSU integrated.

#### Related items

Same family as `KernelSU Late-load script environment (KernelSU Late-load脚本环境)`, `KernelSU jailbreak persistent artifacts (KernelSU越狱持久化工件)`, `KernelSU Metamodule mount (KernelSU Metamodule挂载)`, `Magica boot transaction window (Magica启动事务窗口)`, `Magica transient process (Magica瞬时进程)` and `Jailbreak model Prop cross-check (免解机型Prop交叉检测)`; the jailbreak load-environment criteria have been merged into `KernelSU traits detected (检测到KernelSU特征)`.

</details>

<details><summary><b>KernelSU Late-load script environment (KernelSU Late-load脚本环境) / KernelSU jailbreak persistent artifacts (KernelSU越狱持久化工件) / KernelSU Metamodule mount (KernelSU Metamodule挂载) / Magica boot transaction window (Magica启动事务窗口) / Magica transient process (Magica瞬时进程)</b></summary>

#### How it is detected
Checks whether a late-load mode's script environment, persistent artifacts and extension mount origins are consistent with that loading method, and whether the property source seen at the app preload stage and the communication endpoint appear at the same time.
#### How to fix it

Turn off Late-load / jailbreak mode, delete the corresponding leftovers and boot scripts under `/data/adb`, uninstall the Metamodule and the Magica components, restore the official kernel and boot image, then reboot.

#### Related items

The jailbreak load-environment and storage-log criteria have been merged into `KernelSU traits detected (检测到KernelSU特征)` and are reported under that item when they hit.

#### Notes

The Magica-related items are **transient**: whether they hit depends on whether the window is open at scan time, so repeated scans may produce different results.

</details>

<details><summary><b>Root/SU path check (Root/SU路径检测, merged with the former su binary check in 2.1.0) / Running su process (运行中su进程) / Running Root / cheat process (运行中Root/外挂进程) / Root process check (Root进程检测) / Visible kernel cheat runtime state (可见内核外挂运行态)</b></summary>

#### How it is detected
Checks whether su or similar binaries are present in common paths; for visible processes it cross-checks four views of process identity and requires at least two to agree before counting as su; it also takes in whether the security context belongs to a Root domain, kernel-backfilled task names and cheat daemon characteristics.
#### How to fix it

- Remove the su binaries you do not need, together with their execute permission (note that `/apex/com.android.virt/bin/su` on iQOO / vivo is a false-positive source — see Notes).
- Disable cheat / script tools and their resident daemons.
- General principle: **do not grant the detector Root privileges**.

#### Notes

On Android 16 an ordinary app can only see processes of its own UID, so "no Root process among the visible processes" is a normal result and does not mean the device is already clean; on most recent models these items produce no verdict.

</details>

<details><summary><b>Hidden path visibility check (隐藏路径可见性检测) (new in 1.3.0)</b></summary>

#### How it is detected
This item checks whether the metadata, the actual read result and the parent-directory enumeration of a read-only system path agree: if the metadata says the entry exists but opening reports it missing, or the path is absent from its parent directory listing, the three contradict each other, which shows the path is deliberately hidden.
#### How to fix it

- If it genuinely exists: uninstall the corresponding tool and clean up the leftovers.
- If there is a contradiction: disable path hiding / PathMask / SusFS path spoofing and file API hooks, then reboot and retest.

#### Related items

Same family as `Three-way path existence cross-check (路径存在性三路核对)`, `Mutually exclusive path evidence (路径证据互斥)` and `Java vs native path view divergence (Java与Native路径视图分歧)`.

</details>

<details><summary><b>SELinux status check (SELinux状态检测) / SELinux policy check (SELinux策略检测) / SELinux context check (SELinux上下文检测) / Policy serial check (Policy序列号检测) / SELinux dirty policy check (SELinux脏策略检测)</b></summary>

#### How it is detected
Checks whether the filesystem state of the security context and the load and readability state of the policy view are self-consistent, and whether Root-tool signatures and proprietary types remain in the policy view; it also checks whether the current process lands in a Root domain and whether the policy serial number has been replaced.
#### How to fix it

- Restore SELinux Enforcing (uninstall the kernel module that sets SELinux to permissive).
- Restore the official kernel / boot and the SELinux policy, without injected Root policy.
- Stop the context modifications and injection that apply to Maple.

#### Notes

"The enforce file exists but is not readable" is a normal state for an ordinary app and is not a hit.

</details>

<details><summary><b>Kernel setresuid behavior probe (内核setresuid行为探针) / SusFS-related probe (SusFS相关探针)</b></summary>

#### How it is detected
In a child process, checks whether the kernel's actual behavior on permission-change sensitive paths is self-consistent with what the system reports: whether the child is terminated early and whether it obtains permissions it should not have.
#### How to fix it

Uninstall the SusFS / Root kernel module that modifies kernel behavior and restore the official kernel.

#### Notes

This item is strong evidence, but it depends on kernel implementation details and behaves differently across kernel versions; when the expected behavior is not observed it produces no verdict.

</details>

---

## 2. TEE and key attestation checks

<details><summary><b>TEE key validity check (TEE密钥有效性检测) (new in 1.3.0)</b></summary>

#### How it is detected
Checks whether the device's declared hardware key capability is self-consistent with its certificate chain: whether the leaf certificate corresponds to the key, whether the chain resolves to a trusted root, whether this challenge is bound into it, and whether the declared security level matches. If the device declares hardware key capability but the key is unavailable, the leaf does not match the key, the chain cannot be parsed, the challenge is not bound, or the key is handled by software, it is a hit.
#### How to fix it

- Restore a normal TEE: restore the official firmware and the trusted boot chain, and make sure keystore2 / KeyMint are not intercepted.
- Users of a keybox module: update the keybox module ([TEESimulator-RS](https://github.com/Enginex0/TEESimulator-RS) / [OhMyKeymint](https://github.com/qwq233/OhMyKeymint)), configure the keybox, the security patch and the boot hash correctly, then reboot.
- If the device is genuinely in a "blown TEE" state (it hits even with no modules installed): that is a hardware / firmware-level state, and can only be resolved by restoring the official firmware or sending the device for repair.

#### Related items

Same family as `Android Key Attestation availability (Android Key Attestation可用性)`, `Attestation chain structure consistency (Attestation链结构一致性)`, `Attestation chain root and implementation level (证明链根与实现级别)` and `TEE authorization tag cross-check (TEE授权标签对照)`.

#### Notes

The key / attestation state does not change within one boot; repeated scans in the same environment should agree. If the results differ every time, investigate Keystore hook modules first.

</details>

<details><summary><b>TEE attestation unavailable (damaged / spoofed) (TEE证明不可用（损坏/伪装）)</b></summary>

#### How it is detected
Checks whether a hardware attestation chain can be obtained and parsed in this run: an empty chain, a missing attestation extension, a KeyDescription parse failure, or a timeout while waiting for the primary attestation all count as a hit. This item answers "is the attestation channel available", while "certificate chain signature verification" answers whether the signatures inside the chain are valid.
#### How to fix it

- Reboot, wait about 20 seconds, then retest; a single timeout can happen while KeyMint is busy right after boot.
- Keybox-module users: update the module and verify keybox, target list, security patch and boot hash, then reboot and retest.
- If it still hits with no module: restore the official firmware and trusted boot chain; a genuine hardware / firmware failure can only be repaired by the vendor.

#### Related items

Same family as `TEE key validity check (TEE密钥有效性检测)`, `Android Key Attestation availability (Android Key Attestation可用性)` and `Certificate chain signature verification (证书链逐级验签)`.

#### Notes

This item only means "no attestation was obtained or parsed in this run"; it does not by itself prove the device has been modified. If the other attestation items are normal in the same scan, investigate timeout / compatibility first.

</details>

<details><summary><b>Device lock state untrusted (设备锁定状态不可信)</b></summary>

#### How it is detected
Checks the BootLoaderStatus value range in the locally rebuilt payload: `-1` (attestation channel unavailable / unparsable) is treated as unevaluated and hidden; only a real out-of-range value (outside 0..3) is a hit.
#### How to fix it

- Reboot and retest first; if the attestation channel is normal but this item still appears, check whether a custom ROM / property module writes an invalid boot state.
- Keybox / property-module users: restore the original BootLoaderStatus and do not write out-of-range values other than `-1`.

#### Notes

`-1` is the collection marker for "attestation unavailable", not "device unlocked"; this item no longer shows "device lock state untrusted" because of `-1`.

</details>

<details><summary><b>Android Key Attestation availability (Android Key Attestation可用性) / Attestation chain structure consistency (Attestation链结构一致性) / Attestation chain root and implementation level (证明链根与实现级别) / Key binding and random challenge (密钥绑定与随机挑战)</b></summary>

#### How it is detected
Generates a key with a random challenge and parses its description: whether the chain anchors to the official root, whether the public key matches the leaf certificate, whether the challenge is echoed back, and whether the attestation and key security levels are self-consistent.
#### How to fix it

Disable TrickyStore / TEE / KeyMint simulation or interception modules and restore the official Keystore / KeyMint and the trusted boot chain; if you use a keybox module, complete the target list, the security patch and the boot hash as described in "Configuring modules correctly".

#### Related items

Same family as `TEE key validity check (TEE密钥有效性检测)`, `TrickyStore / similar module found (发现TrickyStore/类似模块)`, `TEE spoofing (TEE 伪造)` and `TEE spoofing (2) (Tee伪造(2))`.

</details>

<details><summary><b>TEE authorization tag cross-check (TEE授权标签对照) / TEE device identifier tag (TEE设备标识标签) / KeyUsage vs KeyPurpose consistency (KeyUsage与KeyPurpose一致性) / Signature algorithm vs issuer key consistency (签名算法与签发钥一致性) / KeyMint placeholder level inconsistency (KeyMint占位层级不一致) / KeyMint rollback resistance authorization (KeyMint回滚抵抗授权) / Attestation chain root and implementation level (证明链根与实现级别)</b></summary>

#### How it is detected
Checks several kinds of tag consistency across the attestation certificate chain: whether the leaf certificate's key usage contradicts the purpose in its extension, whether its signature algorithm matches the issuer's key algorithm, whether software-only tags appear on the hardware security level side, whether vendor-specific extension tags can still be issued, and whether the rollback resistance statement is abnormal.
#### How to fix it

Replace / update the keybox module, reboot and retest; if it still hits, the module implementation does not match the current system KeyMint version — try swapping [TEESimulator-RS](https://github.com/Enginex0/TEESimulator-RS) and [OhMyKeymint](https://github.com/qwq233/OhMyKeymint).

#### Notes

Some tag differences disappear naturally after a minor version update (the module follows the system update); these items belong to "attestation implementation consistency" and are not equivalent to Root.

</details>

<details><summary><b>TEE RootOfTrust boot state (TEE RootOfTrust启动状态) / Bootloader state check (Bootloader状态检测) / OEM unlock check (OEM解锁检测) / Bootloader property scan (Bootloader属性扫描) / Abnormal boot state (启动状态异常)</b></summary>

#### How it is detected
Checks the bootloader lock state: reads several boot-time state properties from the property source, some marking locked or unlocked, others whether boot is verified; then cross-checks them against the trusted attestation report's root-of-trust claims to see whether both describe the same device consistently.
#### How to fix it

- **Genuinely unlocked device**: use a keybox module to hide the unlock state from the detector and add Maple to the target list; or restore the complete official firmware and then relock.
- **Fake-relock / jailbreak / self-signed device**: these already present themselves as "locked", so this item usually does not hit.
- If properties have been messed up (for example a leftover `ro.secureboot.lockstate=unlocked`): run `resetprop -n ro.secureboot.lockstate locked`, or uninstall the device-spoofing / property-spoofing module and reboot.

#### Notes

Restoring factory firmware and relocking the bootloader is an **irreversible** operation; first confirm that all partitions are factory-original and that the device supports it, to avoid bricking.

</details>

<details><summary><b>Attested security patch level (Attested安全补丁级别) / Vendor patch level cross-validation (Vendor补丁级别交叉验证) / Security patch cross-validation (安全补丁交叉验证) / Security patch age (安全补丁年限) / TEE key creation time (TEE密钥创建时间)</b></summary>

#### How it is detected
Checks whether the patch levels declared in the attestation certificate match the runtime properties, and whether patch age and key creation time are consistent.
#### How to fix it

- With a keybox module: write a patch date matching the system into `/data/adb/tricky_store/security_patch.txt`, or simply delete that file so the module takes the value automatically.
- Restore the real system properties and make sure the system and vendor / boot partitions come from the same firmware version.
- On some vendors (including China-specific custom ROMs) the build time and the patch time are inconsistent by themselves, so even a device that has never been unlocked can show a difference; retest first to confirm.

#### Related items

Same family as `Build integrity check (构建完整性检测)` and `Security patch age (安全补丁年限)`; it belongs to "timeline consistency between properties and attestation".

</details>

<details><summary><b>Boot hash consistency (Boot哈希一致性)</b></summary>

#### How it is detected
Checks whether the digest of the device boot image and its boot verification data agrees with the boot verification digest in the system integrity attestation; a mismatch means the boot chain may have been altered.
#### How to fix it

Use the keybox module's add-on (TA) to configure the correct hash; on a genuinely unlocked device the hash often becomes all zeros, so you need to obtain the correct value from a trusted device and write it in.

#### Notes

The hash value changes with the vendor firmware version; do not reuse a value across versions.

</details>

<details><summary><b>Attestation certificate revocation status (CRL) (Attestation证书吊销状态（CRL）)</b></summary>

#### How it is detected
Checks the unique identifier of the last certificate in the attestation chain against the official attestation status list; a revoked certificate is a hit. When the list cannot be retrieved the item is not shown, and a failed retrieval is never counted as not revoked.
#### How to fix it

- Switch to a legitimate, non-revoked device attestation environment (keybox). Keyboxes circulating in public channels are very likely already on the revocation list.
- Restore the vendor's official solution: complete official firmware with a never-unlocked boot chain, where attestation is issued by the vendor / Google production roots.
- On network-restricted devices (no GMS, intranet), the item simply will not appear — that is normal.

#### Related items

Same family as `Proof chain root and implementation level (证明链根与实现级别)`, `TEE key validity check (TEE密钥有效性检测)` and `Android Key Attestation availability (Android Key Attestation可用性)`.

</details>

<details><summary><b>Proof chain root and implementation level (AOSP key) (证明链根与实现级别（AOSP密钥）)</b></summary>

#### How it is detected
Checks whether the root of the attestation chain is a test-purpose or non-production vendor root, and cross-checks it against the security level to see whether the attestation is a software implementation lacking hardware support. When a device is not an officially certified model, its attestation root is not a production vendor root to begin with, which is usually normal.
#### How to fix it

- Switch to a keybox module built on a production root and configure the keybox correctly, then reboot and retest.
- Restore complete official firmware and a trusted boot chain.
- If the device itself only offers software attestation (no TEE, or a broken TEE), this is a hardware / firmware state that no module can fix.

#### Notes

This item often hits together with `TEE key validity check (TEE密钥有效性检测)`: one means the root is wrong, the other means the key / attestation never reaches the hardware.

</details>

<details><summary><b>Widevine device ID read (Widevine设备ID读取) / Widevine vs TEE cross-validation (Widevine与TEE交叉验证) / StrongBox capability declaration (StrongBox能力声明) / Knox Warranty check (Knox Warranty检测)</b></summary>

#### How it is detected
Checks whether the device unique identifier, security level and system identity information declared by the device's cryptographic module are consistent with the facts reported by the hardware trust attestation, and whether the system declares a separate hardware security element. StrongBox is read-only capability declaration: no key is generated during the scan, so it does not consume KeyMint and is never judged abnormal solely because generation is unavailable.
#### How to fix it

- Restore the official DRM, TEE and complete firmware.
- StrongBox is only a capability declaration, not a Root / tampering conclusion; if another TEE item is abnormal, handle that item instead.
- The Knox warranty bit is a historical e-fuse state and is **usually irreversible**.

#### Notes

Widevine being L3 is not abnormal by itself; it is only judged when it contradicts the declaration / TEE result.

</details>

<details><summary><b>Untrusted TEE environment (Soter) (TEE环境不可信（Soter）)</b></summary>

#### How it is detected
Cross-checks whether the declared third-party security service program actually exists, and whether its security context state is consistent with that.
#### How to fix it

Use SusFS / [LKM-PathMask](https://github.com/Andrea-lyz/LKM-PathMask) to hide the Soter-related service paths, and use an app-hiding module to hide `com.tencent.soter.soterserver` from Maple.

#### Notes

On most devices this item produces no verdict: it only hits when both the Soter service program and its service property can be obtained.

</details>

---

## 3. Mount and namespace checks

<details><summary><b>Root mount check (Root挂载检测) / Bind mount specific check (Bind Mount专项检测) / Overlay mount check (Overlay挂载检测) / Cacerts mount check (Cacerts挂载检测) / System partition RW check (系统分区RW检测) / Hidden bind mounts (隐藏Bind-Mounts) / Hidden system mounts (隐藏系统挂载)</b></summary>

#### How it is detected
Checks the mount view: whether its recorded mount sources relate to known injection frameworks, whether system partitions are rewritten or remounted, whether certificate directories are replace-mounted, and whether system partitions are writable although they should not be. The mount view should stay consistent with the storage view, property sources and other read paths. It also checks whether device identity sources are individually overridden by a separate mount entry instead of the normal system mount.
#### How to fix it

- Uninstall the module that creates the mount, restore the overwritten system files, then reboot and retest.
- Use the Zygisk provider module's exclusion policy "restore mounts only", or switch metamodule ([Hybrid-Mount](https://github.com/Hybrid-Mount/meta-hybrid_mount) / [NoMount](https://github.com/maxsteeel/nomount)).
- Restore the official certificate directory and the partition writability state.

#### Notes

Ordinary OEM overlays, APEX, EROFS and dm devices are **not Root evidence by themselves** (these mounts exist in large numbers on factory devices).

</details>

<details><summary><b>Mount device consistency check (挂载设备一致性检测) / Mount ID consistency check (挂载ID一致性检测) / File descriptor mount visibility (文件描述符挂载可见性) / Mapped device visibility check (映射设备可见性检测) (the last three are new in 1.3.0)</b></summary>

#### How it is detected
Cross-checks the same fact through several read paths, asking only whether the storage view and the mount view stay consistent. ① whether the device identifier of a system path matches the mount record that overrides it; ② whether the mount identifier of the file instance behind that path matches the overriding entry; ③ whether the mount an open file points to can be traced in the mount records, judged only when it "existed at boot and has now disappeared"; ④ whether the device identifier of a memory-mapped file can be traced.
#### How to fix it

- A contradiction means a mount was hidden / unmounted inconsistently: disable mount hiding (SusFS / PathMask / the metamodule's mount-hiding feature), then reboot.
- If you are sure a certain module causes it: use the paths in the details to locate that module and uninstall it.

#### Notes

APEX itself is mounted from loop devices, the device number of an overlay-covered path may belong to the lower partition, and pseudo paths (such as `/dmabuf:`) do not correspond to any mount point - all of these are normal and do not hit.

</details>

<details><summary><b>Mount table drift check (挂载表漂移检测) / Runtime mount changes (运行时挂载变化) / Mount namespace drift check (挂载命名空间漂移检测) (new in 1.3.0)</b></summary>

#### How it is detected
Whether the mount table and the mount namespace change during runtime: mount entries are checked for additions, disappearances and attribute changes, and the namespace for being swapped while the app runs, a typical mount-hiding trick. A single count alone is not treated as a finding.
#### How to fix it

Runtime mount changes usually mean "a module remounted / unmounted while the app was running": move the module's mount timing to early boot (`post-fs-data`), or use the "restore mounts only" policy; if the system itself is mounting things during boot (such as external storage or screen casting), wait 20 seconds to 5 minutes after boot before testing.

#### Related items

Same family as `Mount device consistency check (挂载设备一致性检测)` and `Mount ID gap check (挂载ID空洞检测)`; `Mount ID gaps (挂载ID空洞)` alone **do not count as a hit** (on factory devices the gaps come from concurrent allocation during namespace cloning and are very noisy).

</details>

<details><summary><b>Magic Mount / Mount loophole / Mount namespace (Mount命名空间) / PID namespace nesting (PID命名空间嵌套) / User namespace mapping (用户命名空间映射) / Root directory redirection (根目录重定向)</b></summary>

#### How it is detected
Checks the mount traces on the system partition, the process's own namespace and identity mapping views, and whether its root directory has been redirected, and whether these agree with a normal process baseline.
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
Checks whether the file and process names observed on the device are consistent with what the memory mappings record, and whether the network listening endpoints and the actively initiated loopback connections point to the same service whose responses match the protocol it declares.
#### How to fix it

Stop `frida-server`, remove the gadget and the injection scripts, cancel injection into Maple, then reboot.

#### Notes

A resident `frida-server` triggers several items at once; if only a debug port is open without injection, only the port-related items hit.

</details>

<details><summary><b>Xposed / LSPosed check (Xposed/LSPosed检测) / Xposed class loading check (Xposed Class加载检测) / XposedBridge field check (XposedBridge字段检测) / LSPosed class loading check (LSPosed Class加载检测) / LSPosed scope check (LSPosed作用域检测) / LSPosed data directory (LSPosed数据目录) / LSPosed runtime leftovers (LSPosed运行时残留) / OAT / ODEX artifact check (OAT/ODEX工件检测) / Xposed module metadata scan (Xposed模块元数据扫描)</b></summary>

#### How it is detected
Checks for injection-framework characteristics: whether framework-specific injected classes and data fields are present, whether the call stack still carries the framework's call chain, and whether the loader relationships exposed during the preload stage and the metadata of installed modules point to the same framework. It then checks whether references to the module mechanism remain in the process space, and whether the process baseline, the program mapping table and the storage view agree. Runtime leftovers only match explicit module / package names (`zygisk_lsposed`, `org.lsposed`, `liblspd`, `lspd`, `xposed`, `edxposed`, `riru`, etc.); for `/data/app/~~...` random hash directories only the package name or leaf module name is matched, never the random hash or Maple's own `base.apk`.
#### How to fix it

- Remove Maple from the LSPosed / Xposed scope, disable the related modules and reboot.
- If OAT / ODEX artifacts remain: uninstall and reinstall Maple, then retest.
- If you need to keep the modules, use an app-hiding module to hide the module information from Maple.

#### Related items

Same family as `Framework method localization (框架方法本地化)`, `Own method localization (自身方法本地化)`, `ART bridge attribution (ART桥接归属)` and `Boot class loading attribution (引导类加载归属)`.

</details>

<details><summary><b>Zygisk / Riru check (Zygisk/Riru检测) / Zygisk environment check (Zygisk环境检测) / Zygisk module injection check (Zygisk模块注入检测) / Zygisk thread check (Zygisk线程检测) / Zygisk process check (Zygisk进程检测) / Zygisk daemon (Zygisk守护进程) / Zygisk namespace check (Zygisk Namespace检测) / Zygisk SELinux context (Zygisk SELinux上下文)</b></summary>

#### How it is detected
Checks environment variables, module entries in process names and paths, thread names, libraries in restricted paths, trace state, resident daemons, and namespace and security contexts, seeing whether each stays consistent with the process baseline and the mount view.
#### How to fix it

Turn off Zygisk / Riru or remove Maple from the scope, disable the corresponding modules and reboot; or upgrade [Zygisk-Next](https://github.com/Dr-TSNG/ZygiskNext) to the latest version.

</details>

<details><summary><b>Substrate check (Substrate检测) / ShadowHook / Dobby check (ShadowHook/Dobby检测) / Hook library enumeration check (Hook库枚举检测) / Inline hook check (Inline Hook检测) / SMAPS inline hook check (SMAPS内联钩子检测) / Hook callback check (Hook回调检测) / ECA H3</b></summary>

#### How it is detected
Checks whether the loaded-image list agrees with the process memory view and two independent read paths agree, whether function entry bytes have been altered, and whether the memory view shows unexpected executable regions or callback pointers.
#### How to fix it

Disable the hook modules that apply to Maple (SandHook / Dobby / ShadowHook / Pine / YAHFA, etc.), then reboot and retest.

#### Related items

Unrelated in category to `Native injection image attribution (Native注入映像归因)`, `Anonymous executable code check (匿名执行代码检测)` and `Signature algorithm vs issuer key consistency (签名算法与签发钥一致性)`, but they often appear at the same time.

</details>

<details><summary><b>Native injection image attribution (Native注入映像归因) / Anonymous executable code check (匿名执行代码检测) / Anonymous RWX segment check (匿名RWX段检测) / Memory injection conclusion (内存注入结论) / Memfd / JIT check (Memfd/JIT检测) / Deleted mapping check (Deleted映射检测) / ART JIT deleted memfd / Isolated process ART diff (隔离进程ART差分) / ART code page disk verification (ART代码页磁盘校验) / Critical native code page integrity (关键Native代码页完整性)</b></summary>

#### How it is detected
Checks whether named and hidden code modules in the memory mapping view, the function pointer tables registered during the preload stage, executable anonymous segments not produced by just-in-time compilation, and suspicious memory file descriptors each correspond to on-disk code pages. It also checks whether the runtime views of the main process and the isolated process agree under the same process baseline; any divergence is anomalous.
#### How to fix it

- Stop all injection, hook, speed-hacking and debugging modules targeting Maple, then reboot and retest.
- If system-level code pages were modified: restore the official ROM; if it is Maple's own artifact: uninstall and reinstall the official APK.

#### Notes

Ordinary JIT caches (`/memfd:jit-cache`) and `(deleted)` APK mappings are normal; only non-JIT executable anonymous segments and unknown memfds are abnormal.

</details>

<details><summary><b>Signal handler integrity (信号处理器完整性) / Game signal handler integrity (游戏信号处理器完整性) / Exception handler attribution (异常处理器归属) / Anti-debug / signal check (反调试/信号检测) / Debugger check (调试器检测) / Java debugger state (Java调试器状态) / TracerPid check (TracerPid检测)</b></summary>

#### How it is detected
Checks whether the entry points for key signals land in anonymous executable regions or injected modules, whether the uncaught exception handler has been replaced, and whether the process baseline agrees with the debugger connection state.
#### How to fix it

Exit the debugger (Android Studio / LLDB / ptrace / Frida) and the crash-interception and hook modules, then reboot.

#### Notes

Maple itself deliberately raises one exception to verify whether the exception handler has been taken over; that is expected behavior and does not affect the result.

</details>

<details><summary><b>Three-way path existence cross-check (路径存在性三路核对) / Mutually exclusive path evidence (路径证据互斥) / Java vs native path view divergence (Java与Native路径视图分歧) / Two-way proc read cross-check (proc读取双路核对)</b></summary>

#### How it is detected
For the paths that Root frameworks typically modify, it checks whether the evidence from several independent kernel read paths agrees, and compares that with the application-layer view; if one read path shows the path exists while another clearly shows it does not, that counts as a contradiction. For reads of the same runtime file, it checks whether the content returned by several read paths agrees.
#### How to fix it

- Genuine path: uninstall the corresponding tool and clean up the leftovers.
- Multiple contradicting views: disable file hiding, PathMask, SusFS path spoofing or file API hooks, then retest.

#### Related items

Belongs, together with `Hidden path visibility check (隐藏路径可见性检测)`, `Two-way package visibility cross-check (包可见性双路核对)` and `Binder call path consistency (Binder调用路径一致性)`, to the "view divergence" family.

</details>

<details><summary><b>Binder call path consistency (Binder调用路径一致性)</b></summary>

#### How it is detected
Checks whether the protocol version obtained through multiple read paths diverges consistently and stably.
#### How to fix it

Disable Binder interception, ioctl hooks and service-proxy modules for Keystore / TEE, restore the official services, then reboot.

</details>

<details><summary><b>Runtime environment variables (运行时环境变量) / Dynamic library path check (动态库路径检测) / Dynamic linker check (动态链接器检测) / Environment variable markers (环境变量标记) / Linker symbol check (Linker符号检测) / Linker bridge symbols (Linker桥接符号)</b></summary>

#### How it is detected
Checks whether the values of the environment variables visible during the preload stage contain paths pointing to Root or hook injection directories, and whether those variables are consistent with the bridging symbols of the dynamic linker namespaces.
#### How to fix it

Remove the injection launcher, terminal wrapper scripts and environment-variable-modifying modules, and launch Maple normally from the system launcher.

</details>

<details><summary><b>App signature self-check (应用签名自校验) / App debuggable flag (应用可调试标志) / App install source (应用安装来源) / APK install path verification (APK安装路径核对) / Own APK mapping consistency (自身APK映射一致性) / Own native library origin (自身native库来源) / Host executable (宿主可执行文件)</b></summary>

#### How it is detected
Checks whether the app's signer count, debug flag and install source all match a normal distribution channel, and whether the APK and its own libraries are mapped from system-default locations by the system-provided host process.
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
Checks whether the inventory of installed applications is complete and its versions match the machine's actual state, then matches it item by item against the built-in reference lists for risky apps, hiding tools, root managers and hook frameworks. An app missing from the results, or a version that disagrees with the record, means the two are not self-consistent.
#### How to fix it

- Keep only the tools you really need and uninstall the rest.
- If you need to keep them, use an app-hiding module ([HMA-OSS](https://github.com/frknkrc44/HMA-OSS)) to hide the corresponding apps from Maple.
- Make sure the hiding scope does not include system components (IME / accessibility services / app store), otherwise it triggers the visibility cross-check items.

#### Related items

Same family as `System component visibility cross-check (系统组件可见性交叉)`, `Two-way package visibility cross-check (包可见性双路核对)` and `Risky app / blacklisted package name directory check (风险应用/黑名单包名目录检测)`.

</details>

<details><summary><b>System component visibility cross-check (系统组件可见性交叉) (new in 1.3.0)</b></summary>

#### How it is detected
Checks whether the current input method and the apps behind enabled accessibility services, as recorded in system settings, agree with what the package list can actually find: if the record says an app is in use or enabled but the package list cannot find it, that is a hit.
#### How to fix it

In the app-hiding module, cancel the hiding of the "IME / accessibility services" presets, or remove Maple from the hiding scope, then reboot.

#### Notes

This is the most direct symptom of an app-hiding module configured too broadly; it usually appears when the app store and the IME are hidden together.

</details>

<details><summary><b>Two-way package visibility cross-check (包可见性双路核对) / PM cross-method check (PM交叉方法检测)</b></summary>

#### How it is detected
For a known root manager package, checks whether the results returned by multiple read paths are consistent with one another.
#### How to fix it

Disable HMA, package list filtering, virtual containers and PackageManager hooks, then reboot; if all paths agree, use the details to decide whether to uninstall.

</details>

<details><summary><b>Accessibility service check (无障碍服务检测) / Developer options check (开发者选项检测) / ADB debugging check (ADB调试检测) / Active ADB endpoint probing (ADB端点主动探测)</b></summary>

#### How it is detected
Checks whether the number of enabled accessibility services clearly exceeds normal needs and whether any entry carries an automation or script characteristic, and whether developer options and USB debugging are on, whether wireless debugging is enabled, and whether the on-device debugging channel is still reachable from outside.
#### How to fix it

- Turn off the automation / auto-clicker / script tool services in the system accessibility settings.
- When you are not using ADB, turn off USB debugging (`settings put global adb_enabled 0`) and turn off "Wireless debugging".
- If you need to keep it enabled long-term: use [ADB-Trace-Cleaner](https://github.com/YiJieqwq/ADB-Trace-Cleaner) to clean up the debugging traces.

#### Notes

USB debugging / developer options are **user configuration** signals, labeled separately as low risk and not carrying the same weight as other Root evidence.

</details>

<details><summary><b>Temp directory permission check (临时目录权限检测) / Temp directory inode check (临时目录inode检测) / Temp directory ownership check (临时目录归属检测) / Temp directory timestamp check (临时目录时间戳检测)</b></summary>

#### How it is detected
The shared temp directory: loosened permissions, a high inode number, the original low-privilege owner and group, and a modification time later than now or a status change time later than modification.
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
Checks every entry in the built-in high-risk path list, judging by that entry's own attributes in the storage view without following redirects elsewhere; the result lists the concrete entries only.
#### How to fix it

Confirm each **concrete path** given in the details and delete the leftovers you no longer need; for the MT2 folder you can change the path in MT Manager's settings and then delete the old directory.

#### Related items

Same family as `Fast scan for abnormal sh files (异常sh文件快速扫描)`, `Fast scan of external storage scripts (外置存储脚本快速扫描)` and `Scheduler / thermal module check (调度/温控模块检测)`.

</details>

<details><summary><b>Fast scan for abnormal sh files (异常sh文件快速扫描) / Fast scan of external storage scripts (外置存储脚本快速扫描) / Zero-width disk scan app check (零宽扫盘应用检测) / Blacklisted package name directory check (黑名单包名目录检测) / Blacklisted directory hidden app check (黑名单目录隐藏应用检测)</b></summary>

#### How it is detected
Scans app-accessible shared storage, checking whether the script files, blacklisted app directories and hidden app directories it finds match the apps installed on the device.
#### How to fix it

Check the concrete paths in the details and delete the scripts and directories you have confirmed you no longer need; anomalies in directories such as `/storage/emulated/0/Android/data/<package name>` usually come from leftovers of cheat / app-cloning tools.

#### Notes

The `.sh` extension does not by itself prove malicious behavior; a hit only means "a script file exists".

</details>

<details><summary><b>System image residue check (系统镜像残留检测) — Ultra</b></summary>

#### How it is detected
Within the shared storage the app can reach, checks whether startup images and flash backups appear there, files an ordinary app should not hold. Ultra mode additionally checks configuration files in the root of shared storage, whether their content and naming match the actual device state.
#### How to fix it

Confirm the source from the paths listed in the details and delete them; such images usually come from flashing / patching boot, and abnormal `.xml` files are usually configuration written by modding modules. Keeping them in shared storage also triggers the "modding tool residue" items.

#### Notes

A hit only means "boot image files exist in shared storage" — it does not mean the device is currently rooted, but flash backups kept there waste space and are easy to flag.

</details>

<details><summary><b>Abnormal system executable files (系统异常可执行文件) / Scheduler / thermal module check (调度/温控模块检测) / Kernel cheat driver check (内核外挂驱动检测) / Kernel module check (内核模块检测) / Kernel module leftovers (内核模块残留)</b></summary>

#### How it is detected
Checks whether executables that do not belong to the usual set appear in system-preloaded locations. Checks whether the configuration sources of scheduler- and thermal-related components agree with their runtime traces, and whether load entries outside the official configuration exist. Checks the loaded kernel module list and its driver origins against the system baseline.
#### How to fix it

Uninstall the corresponding scheduler / thermal / cheat modules and restore the official system partition files; vendor-modified builds commonly include `adb`/`fastboot` — flash back the official build, or use [LKM-PathMask](https://github.com/Andrea-lyz/LKM-PathMask) to hide them.

#### Notes

It is normal that `/proc/modules` is not readable (an ordinary app has no permission for it); in that case this item produces no verdict.

</details>

<details><summary><b>Terminal environment trace check (终端环境痕迹检测) / PTY node ownership (PTY节点归属) (new in 1.3.0)</b></summary>

#### How it is detected
Compares a terminal device node's owner and security context against Root framework traits; a trait counts as a hit, and an unreadable context gives no conclusion.
#### How to fix it

Close resident Root terminal / debug sessions, stop interactive shells opened in a Root domain, then reboot.

#### Notes

Ordinary terminals (including Maple's own pty) do not hit; only labels that fall in a Root framework domain count.

</details>

<details><summary><b>Process cgroup context (进程Cgroup上下文) / Suspicious detection runtime environment (检测运行环境可疑) / Container (容器) / App cloning (多开) / Process identity quadruple (进程身份四元组) / Data directory ownership consistency (数据目录归属一致性)</b></summary>

#### How it is detected
Checks whether the process isolation context carries markers of common injection frameworks, whether the process identity credentials agree, whether the app's data and installation directories sit where the system requires, and whether the process runs in a nested namespace.
#### How to fix it

Exit the app clone / cloud phone / virtual container, uninstall the current copy and reinstall the official APK from a trusted source.

#### Related items

Same family as `PID namespace nesting (PID命名空间嵌套)`, `User namespace mapping (用户命名空间映射)` and `Root directory redirection (根目录重定向)`.

</details>

<details><summary><b>Vold isolation property check (Vold隔离属性检测) / Log buffer property check (日志缓冲区属性检测) / Core platform API policy check (核心平台API策略检测) / Hiding solution property check (隐藏方案属性检测) (new in 1.3.0)</b></summary>

#### How it is detected
Checks four persisted settings: the app data isolation switch, the log capacity parameter, and two switches under runtime debug policy, confirming they stay consistent with the system property sources and the storage view, and whether those switches still hold a value.
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
Checks whether the device carries runtime traces of known emulation or virtualization runtimes, and cross-checks them against reference surfaces such as battery, charging and SIM state.
#### How to fix it

- Real device: disable hardware spoofing and cloud-phone / container modules, and restore the official properties, kernel and drivers.
- An emulator cannot be turned into a real device by changing properties; this item appearing on an emulator is expected.

#### Notes

Testing with no SIM card inserted while the battery is full and charging can produce a combination of features similar to an emulator; retest in a realistic state.

</details>

<details><summary><b>Build fingerprint integrity check (构建指纹完整性检测) (new in 1.3.0) / Partition fingerprint consistency check (分区指纹一致性检测) / Fingerprint cross-validation (Fingerprint交叉验证) / Fingerprint component consistency check (Fingerprint分量一致性检测) / Fingerprint component cross-validation (Fingerprint分量交叉验证)</b></summary>

#### How it is detected
The build identifier must follow the standard segmented structure; across partitions only the build posture segments are checked for consistency, while the brand and codename segments take no part; within one partition, identifier components from one source are cross-checked against independent property sources from another.
#### How to fix it

- Uninstall device-spoofing / PIF / property-spoofing modules, restore the official properties, then reboot.
- Property leftovers (for example the fingerprint was changed to only the brand name): clear the corresponding `resetprop` script or the module's `system.prop`.

#### Notes

On Xiaomi models the vendor segment is fixed to a generic base such as `mivendor_<soc>_cn`, which is **legitimately different** from the device codename and is a normal layout.

</details>

<details><summary><b>Device model spoofing check (机型伪装检测) / Jailbreak model Prop cross-check (免解机型Prop交叉检测) / OemPropertyConsistency / Property consistency check (属性一致性检查) / Property context directory enumeration (属性上下文目录枚举)</b></summary>

#### How it is detected
Checks whether vendor-specific properties and device identifiers are self-consistent: whether the values that should be present do exist, and whether values that should not appear do appear instead; it also checks whether key properties are empty and whether their values come from an abnormal source.
#### How to fix it

Remove property-filtering and device-spoofing modules and restore the real properties; if a third-party ROM is missing properties itself, you can only wait for the ROM maintainer to fix it or flash back to the official system.

</details>

<details><summary><b>Eight device property dual-source cross-check (8项设备属性双源核对) / Build.TAGS cross-check (Build.TAGS交叉) / Security patch cross-validation (安全补丁交叉验证) / Build integrity check (构建完整性检测) / Boot parameter raw value cross-validation (启动参数原始值交叉验证) / Kernel boot security context (内核启动安全上下文)</b></summary>

#### How it is detected
The same property is read from four mutually independent sources — check whether their readings agree. Then check whether boot parameters match what the system declares about its boot state, and whether they expose unlock, permissive policy, disabled verification, or custom directories and injection-framework traces.
#### How to fix it

Disable device-spoofing / PIF / resetprop modules, restore the official properties and boot partitions, then reboot.

#### Related items

Same family as `Play Integrity Fix leftovers (Play Integrity Fix残留)` and `Property consistency check (属性一致性检查)`.

</details>

<details><summary><b>Play Integrity Fix leftovers (Play Integrity Fix残留)</b></summary>

#### How it is detected
During the preload stage, checks whether the property sources carry enablement markers of a spoofing-type solution, and compares those property values against the device's actual properties.
#### How to fix it

Disable and uninstall the corresponding modules, clean up their actual leftovers (including `system.prop` and `service.d` scripts), then reboot.

</details>

<details><summary><b>Property area context check (属性区上下文检测) (new in 1.3.0) / Property area readability cross-check (属性区可读性核对)</b></summary>

#### How it is detected
Enumerates the context names of the property source visible at the preload stage; any match with known spoofing or hiding markers counts as a hit, and it also checks whether that property source is readable and whether the serial number and the info records stay self-consistent.
#### How to fix it

Uninstall the module that created the custom property area (such contexts are usually written by property-spoofing modules), then reboot.

#### Notes

Normal contexts that exist on factory devices, such as `vendor_socket_hook_prop`, do not hit.

</details>

<details><summary><b>Custom ROM check (自定义ROM检测) / Kernel version check (内核版本检测) / Kernel build signature check (内核构建特征检测) / Third-party kernel (三方内核)</b></summary>

#### How it is detected
Checks whether the system and device model features are consistent with the built-in third-party ROM and kernel keywords.
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
Checked as an ordinary app: when this app asks the kernel for network connection information, does the kernel deny it under the policy view; if a response comes back containing other apps' connection records, the kernel's query policy has been relaxed.
#### How to fix it

Restore the official kernel / SELinux policy; if some hiding solution opened up this interface, disable it and reboot.

#### Notes

A factory system rejects this query from an ordinary app anyway, so "rejected" is the normal result.

</details>

<details><summary><b>Process protection view consistency (进程防护视图一致性) (new in 1.3.0)</b></summary>

#### How it is detected
During the preload stage, several read paths are cross-checked to confirm the process's own sandbox and privilege-escalation restrictions agree: one from the status view, one from the policy view, and one from the property source. All three must match exactly; any disagreement between two is a hit.
#### How to fix it

Disable `/proc` filtering, file API hooks and hiding modules targeting Maple, then reboot.

#### Related items

Belongs, together with `Two-way proc read cross-check (proc读取双路核对)` and `Binder call path consistency (Binder调用路径一致性)`, to "multi-view observation divergence".

</details>

<details><summary><b>Zygote supplementary GID check (Zygote补充GID检测) (new in 1.3.0)</b></summary>

#### How it is detected
Checks whether the process's supplementary group list still carries the app-specific groups — the normal signature of a process forked during the preload stage; if missing, the supplementary group identity has been narrowed or rewritten, so look into restrict-preload-privileges style hiding configurations.
#### How to fix it

In the app-hiding module, turn off the "restrict zygote privileges / SHARED USER GID / APP ZYGOTE GID" style switches for Maple, then reboot.

#### Notes

On Android 14+ an app does not have GID 3009 (readproc) anyway, which is normal.

</details>

<details><summary><b>OBB path view consistency (OBB路径视图一致性) (new in 1.3.0)</b></summary>

#### How it is detected
For the app's own OBB storage location, uses multiple read paths to cross-check whether it exists and whether its own recorded attributes agree with the entries the parent directory listing shows; a contradiction between any of these is a hit.
#### How to fix it

Disable hiding modules aimed at scanning / file reads; if necessary, turn off path interception policies other than "restrict zygote privileges" for Maple.

</details>

<details><summary><b>Clock source consistency (时钟源一致性) / Kernel clock cross-validation (内核时钟交叉验证) / Boot time consistency (开机时刻一致性) / Process start time consistency (进程启动时刻一致性)</b></summary>

#### How it is detected
Whether the time readings returned by multiple time-reading paths on the device contradict each other over the same stable observation window — that is, whether the differences between those readings persist and fail to converge over time.
#### How to fix it

Turn off speed hacking, time hooks and virtual containers, restore the system's automatic date and time, then reboot; if it keeps hitting, investigate clock-related kernel modules.

</details>

---

## 8. Hardware and sensor cross-validation

<details><summary><b>CPU / GPU / platform three-way validation (CPU/GPU/Platform三向验证) / CPU + GPU cross-validation (CPU+GPU交叉验证) / Hardware cross-validation report (硬件交叉验证报告) / Chip identity recognition (芯片身份识别) / CPU feature verification (CPU特性校验) / sysfs cross-validation (sysfs交叉验证) / Direct Vulkan physical GPU enumeration (Vulkan物理GPU直枚举) / GPU driver verification (GPU Driver验证)</b></summary>

#### How it is detected
Checks whether the hardware identity evidence obtained from multiple read paths at the preload stage, system property sources, driver nodes, device information reported by the graphics interface, CPU topology and device nodes contradicts itself.
#### How to fix it

Disable device-spoofing, KPM file overwriting and hardware-spoofing modules, and restore the official kernel, drivers and system properties.

#### Notes

Different device models with the same SoC may have different frequency tables; Maple avoids single-point differences by only judging when multiple sources disagree.

</details>

<details><summary><b>CPU frequency (CPU频率) / CPU topology (CPU拓扑) / Total memory check (内存总量检测) / Storage space check (存储空间检测) / Temperature sensor fingerprint (温度传感器指纹) / Thermal zone sensor count (热区传感器计数) / Thermal zone temperature read (热区温度读取) / Power supply check (电源检测) / Input device check (输入设备检测)</b></summary>

#### How it is detected
Checks whether the frequency table and the topology are consistent. Checks whether the memory and storage capacities match the device model. Checks whether the number, types and readings of thermal zone sensors are abnormal. Checks whether the input devices are virtual devices.
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
