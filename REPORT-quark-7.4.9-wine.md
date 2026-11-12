# Quark Cloud Drive PC 7.4.x under Wine: a documented failure record

# 夸克网盘 PC 7.4.x 在 Wine 下无法运行：一份现场记录

**Version documented / 记录版本**: Quark PC **7.4.9.1045** (current latest as of 2026-11-13; earlier reports of 7.4.5.1027 show the same behavior)
**Date / 日期**: 2026-11-13
**Tested by / 测试者**: Linux user (Thunderobot 911, i7-9750H, GTX 1660 Ti), assisted by ZCode agent

---

## TL;DR

Quark PC 7.4.x (Alibaba's cloud drive client, rebuilt in 2026-10 on their in-house Chromium fork + "Node Quantum" AI engine) fails under Wine without an error message. Across two instrumented runs:

- 121–145 distinct processes spawn, then all terminate within ~2 minutes
- Every process exits cleanly (exit code 0) — no unhandled exception, no crash dialog, no crashpad dump
- The captured Wine log ends during initialization of a vkd3d/DXVK descriptor heap
- One concrete loader failure: `onnxruntime.dll` (the AI engine runtime) reports `dbghelp.dll` as not found — even though dbghelp.dll is present in the prefix
- Contrast: Quark 7.3.5.810 (previous-generation kernel) runs without problems in the same setup, same runner, same machine

中文摘要：新版夸克在 Wine 下没有报错，也没有崩溃对话框——121~145 个进程在约两分钟内全部退出，退出码均为 0，日志终止在渲染进程初始化中途，未留崩溃转储。另有一个具体的加载缺陷：AI 引擎的 `onnxruntime.dll` 报 `dbghelp.dll not found`，而该文件实际存在于容器中。

---

## Environment / 环境

| Item | Value |
|---|---|
| OS | Linux Mint 22 (Ubuntu 24.04 base), X11, kernel 6.8+ |
| GPU | NVIDIA GTX 1660 Ti, driver 580-open (single GPU, no iGPU) |
| Wine | Kron4ek wine-11.21-staging (amd64), via Bottles Flatpak |
| Prefix | Fresh "Application" bottle `基准容器` (win64) |
| App | Quark PC installed by official online stub installer → 7.4.9.1045 |
| Logs | Full stderr captured to file for both runs; `WINEDEBUG=+seh` on run 2 |

## Reproduce / 复现步骤

1. Fresh Bottles (Flatpak) Application bottle, runner Kron4ek 11.21-staging, DXVK 3.1.1, no DLL overrides.
2. `wine C:/QuarkPC-latest.exe` (official stub, 3.2 MB; it downloads and silently Inno-installs 7.4.9.1045 into `C:\users\<user>\AppData\Local\Programs\Quark`).
3. `wine ... quark.exe` — UI may flash briefly; within ~2 minutes every process is gone. Exit code 0.

---

## Findings / 发现（按证据强度排序）

### 1. Processes terminate on their own, without an error / 进程自行退出，没有报错

Run 2 with `WINEDEBUG=+seh`: **zero** unhandled exceptions. The only exception codes seen:

| Code | Count | Meaning |
|---|---|---|
| 0x40010006 | 28 | DBG_PRINTEXCEPTION_C (OutputDebugString) — benign |
| 0x406d1388 | 5 | Thread-name setting — benign |
| 0x6ba | 3 | RPC_S_SERVER_UNAVAILABLE — Wine lacks the RPC endpoint |

Wine itself reports `Result=success, ExecMainStatus=0`. The processes end on their own.

中文：全程没有未处理异常，wine 返回"正常退出"。进程是自行结束的——这对应普通用户看到的"突然消失/卡死"。

### 2. Renderer initialization interrupted / 渲染进程初始化中途停止

Both runs end in the middle of vkd3d/DXVK descriptor-heap setup:

```
info:  Memory type mask for buffer resources: 0x3b, usage: 0x219ff
info:  Creating sampler descriptor heap (8 kB)
info:  Graphics pipeline libraries supported
<log ends>
```

No further output. The next renderer/GPU process never finishes coming up.

### 3. `onnxruntime.dll` fails to load — dbghelp.dll "not found" although present / AI 引擎运行时加载失败

```
04a4:err:module:import_dll Library dbghelp.dll (which is needed by
L"C:\\users\\cyc-mint\\AppData\\Local\\Programs\\Quark\\7.4.9.1045\\onnxruntime.dll") not found
```

Verified facts making this odd / 已验证的情况:

- `dbghelp.dll` **exists** as a 64-bit PE in `windows/system32` (883 KB, wine builtin copy)
- The runner ships `lib/wine/x86_64-windows/dbghelp.dll`
- `onnxruntime.dll` is x86-64
- No DLL overrides configured; UCRT base (`ucrtbase.dll`) identical to a known-working prefix
- This is the runtime of Quark's in-house AI stack ("Node Quantum" service, visible in process names)

We have not identified the loader-level mechanism behind this message; pointing it out for upstream investigation. / 我们尚未查明这条报错在加载器层面的成因，留给上游调查。

### 4. WinRT / COM gaps / WinRT 与 COM 缺口

```
err:combase:RoGetActivationFactory Failed to find library for L"Windows.Devices.Input.PenDevice"
err:ole:create_server class {aafe3a0e-952f-5312-9e85-35f2130387a7} not registered   (×6, several processes)
```

### 5. Chromium-sandbox-adjacent Wine stubs hit / 沙箱依赖的底层机制撞上 Wine stub

```
fixme:ntdll:NtSetInformationToken TokenIntegrityLevel stub!
fixme:process:SetProcessMitigationPolicy (8, ...): stub        ← PROCESS_MITIGATION_BINARY_SIGNATURE_POLICY
fixme:process:NtQueryInformationProcess ProcessHandleCount / ProcessHandleTable stub
fixme:thread:NtSetInformationThread ThreadPowerThrottling stub!
```

The renderer/broker processes query exactly the process-mitigation & token-integrity machinery Chromium's Windows sandbox relies on.

### 6. Quark's own crash collection captured nothing / 夸克自带的崩溃收集一无所获

- `User Data/Crashpad/reports/` — **empty**
- `quark_wer.dll` registers via `WerRegisterRuntimeExceptionModule` — Wine stubs this API (`fixme`), so Alibaba's custom WER channel does not function
- Per-process `ulog` files (proprietary `ULMF` binary format) exist but contain no plaintext

### 7. Installer behavior (context, not a bug) / 安装器行为

The official stub silently installs with:

```
/VERYSILENT /DIR=C:\users\...\Programs\Quark
/MINISETDEFBROWSER=1 /MINISETDEFPDF=1 /MINISETDEFVIDEO=1 /MINISETDEFIMAGE=1
/MINIENABLEAITOOLBAR=1
```

i.e. it requests to take over default browser/PDF/video/image handlers and install an AI toolbar. In Wine these registration steps partially no-op — which may itself feed the failing COM/RPC paths above.

---

## Ruled out / 已排除

- GPU/driver: same machine runs Quark 7.3.5.810 perfectly with the same DXVK/Vulkan stack
- Runner choice: 7.4.5.1027 failed identically on Kron4ek 11.21/11.18, Caffe 10.0, Vaniglia 10.19
- UCRT/runtime base: byte-compared against the working 7.3.5 prefix — identical
- Memory exhaustion: instrumented runs capped at 4 GiB; death occurs far below the cap, with exit code 0 (the OOM behavior only appears when users disable GPU acceleration — software rendering then balloons memory)
- `--disable-gpu`, `--no-sandbox`, disabling DXVK-NVAPI: all tried by the community/users, none change the outcome

## Contrast / 对照组

Quark **7.3.5.810** (previous kernel generation) installs and runs stably in the same Bottles setup on the same machine — login, upload, download all work. The regression is therefore **kernel-generation-specific**, not environmental.

---

## What would help / 希望得到的帮助

1. Why does `import_dll` report `dbghelp.dll` "not found" when a valid 64-bit copy sits in system32? (Happens for `onnxruntime.dll` only; other modules import fine.)
2. Is there a known Wine gap for the stub combination in Finding 5 that would explain the processes exiting on their own with exit code 0?
3. AppDB has **no entry** for QuarkCloudDrive — this document can seed one.

Contact: via this post's author / 通过发帖人联系。

## Artifacts / 证据文件

- `logs/install.log` — run 1, 1751 lines (install + first run + death)
- `logs/run2.log` — run 2, 2592 lines, `WINEDEBUG=+seh`, 121 processes, 36 SEH dispatch traces
- `evidence/ulog/*.ulog` — Quark's proprietary per-process logs (ULMF binary, offered for reverse engineering)
- `evidence/debug.log` — Quark's own text log (rollback manager lines only)
- `evidence/installer-stub.exe` — the exact official stub used (SHA256 available on request)

---

*This report was produced on 2026-11-13. Reproduction instructions are complete; the artifacts can be re-shared with upstream (WineHQ AppDB / Wine GitLab) if considered useful. — 中文版见上半部分。*
