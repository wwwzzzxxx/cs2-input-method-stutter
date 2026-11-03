# CS2 Intermittent Stutter — Root Cause Report

> A Chinese IME (Sogou Pinyin) triggers repeated Trusted Mode validation inside Counter-Strike 2, causing periodic stutter.
> This report records the evidence chain, the fix, and measured results.

## 1. Symptom

- 1080p borderless: average frame rate noticeably low, poor 1% low
- Intermittent stutter, most obvious on mouse click / movement
- Worse in fullscreen-windowed mode; switching display mode does not help
- GPU utilization very low, temperatures and power draw normal, no throttling

## 2. Ruled Out

| Suspect | Action | Result |
|---|---|---|
| MPO (Multi-Plane Overlay) | Disabled `OverlayTestMode=5` per NVIDIA documentation, rebooted | No effect |
| Steam overlay | Disabled CS2 `OverlayAppEnable` | No effect (briefly seemed fixed, then returned) |
| DPC / ISR | ETW capture | Max DPC 0.2182 ms, normal |
| CPU scheduling | Thread affinity analysis | Main thread entirely on P-cores, normal |
| GPU / present path | PresentMon + ETW | GPU busy ~2 ms, always Hardware Independent Flip |
| Background software | A/B toggle | No perceptible difference |

Conclusion: not CPU, GPU, DWM composition, display mode, or background software.

## 3. Root Cause

### 3.1 Direct evidence

CS2's Trusted Mode log `trustedlaunch.cfg` has 163 lines, of which **51 identical entries**:

```
~C:\Program Files (x86)\SogouInput\<version>\Resource.dll
```

`~` means **load rejected**. The game repeatedly attempts to load this file and retries after each failure.

### 3.2 Single-variable comparison

Same process, same main thread, same borderless 1080p, MPO and Steam overlay unchanged — **the only variable was the input layout** (`0x8040804` Chinese → `0x4090409` ENG):

| | Main-thread samples | `cs2.exe` validation-path hotspot |
|---|---|---|
| Chinese IME | 17,619 | 4,428 (**25.13%**) |
| ENG | 12,448 | 10 (**0.08%**) |

Of the 4,428 hot stacks, 4,422 include the Steam overlay + Windows input / loader chain.

### 3.3 Valve's own documentation

From the Steam Support article "CS2 - Trusted Mode":

> "Counter-Strike 2 refuses injections in Trusted mode, but sometimes applications keep trying. When applications try to inject into Counter-Strike 2 at very high rates (some as frequently as 1kHz), players may notice performance issues. Sometimes applications attempt to inject during low level Windows mouse events, which results in noticeable performance degradation when the mouse is moved."

### 3.4 What `Resource.dll` actually is

PE analysis:

```
Sections:  .rdata  .rsrc (4.60 MB)  .reloc
Exports:   RVA 0x0, size 0    <- no exported functions at all
File desc: Sogou Input Method — Resources
```

It is a **pure resource pack** (skins, candidate-window images, icons) with no executable code. Its only purpose inside `cs2.exe` is letting the IME draw its UI in-game.

### 3.5 Full causal chain

1. A Chinese IME is active in-game → the TSF framework loads `SogouTSF.ime` / `SogouPY.ime` into the game process
2. The IME needs UI assets → tries to load `Resource.dll` from its install directory
3. CS2 Trusted Mode rejects non-system-directory DLLs → logs one entry and returns failure
4. The IME **retries** on subsequent input / mouse events; CS2 **validates again**
5. Validation runs on the game's main thread → periodic stutter

## 4. Fix Applied

### 4.1 Remove the resource pack from the load path (core fix)

Rename

```
C:\Program Files (x86)\SogouInput\<version>\Resource.dll
```

to `Resource.dll.bak` (keep a backup of the original).

Result: `Resource.dll` disappears from the `cs2.exe` module list; `SogouTSF.ime` and `SogouPY.ime` remain (TSF mechanism, harmless).

### 4.2 Freeze IME updates (prevent the fix from being overwritten)

| Item | Action |
|---|---|
| `SogouSvc` service | Startup type set to Disabled |
| `PinyinUp.exe` (network updater) | Renamed to `PinyinUp.exe.bak` |

`userNetSchedule.exe` (account/word-list sync) and `SogouCloud.exe` (cloud input) were left untouched to avoid breaking daily typing.

### 4.3 Why not just disable Trusted Mode

CS2 has **no** official toggle. The only documented escape hatch is `-allow_third_party_software`. Allowing injection merely lets the injection succeed once; it does not remove IME code from the game process. Removing the resource load path is strictly better.

## 5. Measured Results

45-second PresentMon capture, with the Chinese IME active:

| Metric | Before | After |
|---|---|---|
| Average FPS | 172.6 | **267.9** |
| 1% low | 31.5 | **118.7** |
| 0.1% low | — | 42.9 |
| p99 frame interval | — | 6.2 ms |
| Frames > 33.3 ms | 23 per 90 s | **3 per 45 s** |
| Longest frame | 88.2 ms | 48.25 ms |
| GPU busy | ~1.9 ms | 2.21 ms |
| Present mode | Independent Flip | Independent Flip |

Long stutters per second: 0.256/s → 0.067/s, roughly a 4× reduction; 1% low improved 3.8×. GPU load unchanged — the entire improvement is on the CPU side.

> The two captures were not taken in identical scenes, so absolute values are not strictly comparable; the direction of the difference is unambiguous.

## 6. Risks and Caveats

1. **Updating overwrites the fix**: an IME update restores `Resource.dll`, `PinyinUp.exe` and `SogouSvc`. Updates have been frozen; a manual upgrade requires redoing 4.1/4.2.
2. **The latest version has the same issue**: installer 16.10.0.5858 still ships `Resource.dll` (4,841,856 bytes) with an identical PE layout to 16.9 (0 exports, `.rsrc` 4.60 MB). The architecture is unchanged.
3. **Candidate window may look degraded**: without the resource pack, skin images may not render. Typing still works.
4. **Another potential injector**: Tongyi Qianwen IME's `qianwenime.dll` is also loaded in `cs2.exe`, but it is only a 1.07 MB bridge; the engine lives in a separate server process. It showed no problem in this capture.
5. **Valve's side**: Valve states it is developing a near-zero-overhead injection interception mechanism. Once shipped, a full IME install can be restored.

## 7. One-liner: add the US keyboard

> The US keyboard is **built into Windows**; there is nothing to download. "Installing" it means registering `0409:00000409` in the language list.

Open PowerShell as Administrator:

```powershell
$langs = Get-WinUserLanguageList
if (-not ($langs | Where-Object { $_.LanguageTag -eq 'en-US' })) { $langs.Add('en-US') }
$en = $langs | Where-Object { $_.LanguageTag -eq 'en-US' }
if ($en.InputMethodTips -notcontains '0409:00000409') { $en.InputMethodTips.Add('0409:00000409') }
Set-WinUserLanguageList $langs -Force
Get-WinUserLanguageList | ForEach-Object { "$($_.LanguageTag): $($_.InputMethodTips -join ', ')" }
```

Make English the default (English at login, `Win + Space` to switch to Chinese):

```powershell
$langs = Get-WinUserLanguageList
$en = $langs | Where-Object { $_.LanguageTag -eq 'en-US' }
if ($en) { $langs.Remove($en); $langs.Insert(0, $en); Set-WinUserLanguageList $langs -Force }
```

## 8. Rollback

Restore the resource pack:

```powershell
Rename-Item 'C:\Program Files (x86)\SogouInput\<version>\Resource.dll.bak' 'Resource.dll'
```

Restore update capability (Administrator):

```powershell
Set-Service SogouSvc -StartupType Manual
Rename-Item 'C:\Program Files (x86)\SogouInput\<version>\PinyinUp.exe.bak' 'PinyinUp.exe'
```
