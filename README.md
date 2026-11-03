# CS2 间歇卡顿排查报告

> 中文输入法（搜狗）在 Counter-Strike 2 中触发信任模式反复校验，导致周期性卡顿。
> 本报告记录证据链、修复方法与实测数据。

## 一、现象

- 1080p 无边框模式，平均帧率明显偏低，1% low 很差
- 间歇性卡顿，鼠标点击/移动时尤为明显
- 全屏窗口模式更严重，切换显示模式不能解决
- GPU 占用很低，温度功耗正常，无降频

## 二、被排除的怀疑

| 怀疑对象 | 做法 | 结果 |
|---|---|---|
| MPO（多平面覆盖） | 按 NVIDIA 官方文档禁用 `OverlayTestMode=5` 并重启 | 无效 |
| Steam 覆盖层 | 关闭 CS2 的 `OverlayAppEnable` | 无效（一度以为好了，随即复发） |
| DPC / 中断 | ETW 采集 | 最长 DPC 0.2182 ms，正常 |
| CPU 调度 | 线程亲和性分析 | 主线程全在 P 核，正常 |
| GPU / 呈现链路 | PresentMon + ETW | GPU busy 约 2 ms，始终 Hardware Independent Flip |
| 后台软件 | 开关对照 | 体感无差异 |

结论：问题不在 CPU、GPU、DWM 合成、显示模式或后台软件。

## 三、根因

### 3.1 直接证据

CS2 信任模式日志 `trustedlaunch.cfg` 共 163 行，其中 **51 条**全部是同一条：

```
~C:\Program Files (x86)\SogouInput\<version>\Resource.dll
```

`~` 表示**被拒绝加载**。游戏运行中反复尝试加载该文件，失败后重试。

### 3.2 单变量对照

同一进程、同一主线程、同样无边框 1080p，MPO 与 Steam 覆盖层状态不变，**唯一变化是输入法布局**（`0x8040804` 中文 → `0x4090409` ENG）：

| | 主线程样本 | `cs2.exe` 校验路径热点 |
|---|---|---|
| 中文输入法 | 17,619 | 4,428（**25.13%**） |
| ENG | 12,448 | 10（**0.08%**） |

4,428 条热门栈中有 4,422 条包含 Steam 覆盖层 + Windows 输入/加载链。

### 3.3 Valve 官方说明

Steam 支持页《CS2 - Trusted Mode》原文：

> "Counter-Strike 2 refuses injections in Trusted mode, but sometimes applications keep trying. When applications try to inject into Counter-Strike 2 at very high rates (some as frequently as 1kHz), players may notice performance issues. Sometimes applications attempt to inject during low level Windows mouse events, which results in noticeable performance degradation when the mouse is moved."

### 3.4 `Resource.dll` 是什么

PE 分析结果：

```
段表：.rdata  .rsrc(4.60MB)  .reloc
导出表：RVA 0x0，大小 0    ← 没有任何导出函数
文件描述：搜狗输入法 资源
```

它是**纯资源包**（皮肤、候选窗图片、图标），没有可执行代码。被加载进 `cs2.exe` 的唯一目的是让输入法在游戏内绘制界面。

### 3.5 完整因果链

1. 游戏内激活中文输入法 → TSF 框架把 `SogouTSF.ime` / `SogouPY.ime` 加载进游戏进程
2. 输入法需要 UI 资源 → 尝试加载安装目录下的 `Resource.dll`
3. CS2 信任模式拒绝非系统目录 DLL → 写一条日志并返回失败
4. 输入法在后续输入/鼠标事件中**再次尝试**，CS2 **再次校验**
5. 校验跑在游戏主线程 → 周期性卡顿

## 四、修复措施

### 4.1 移除资源包的加载路径（核心）

将

```
C:\Program Files (x86)\SogouInput\<version>\Resource.dll
```

改名为 `Resource.dll.bak`（保留原件备份）。

效果：`cs2.exe` 模块列表中 `Resource.dll` 消失；`SogouTSF.ime`、`SogouPY.ime` 仍在，属 TSF 机制，不影响功能。

### 4.2 冻结输入法更新（防止改动被覆盖）

| 项 | 处理 |
|---|---|
| `SogouSvc` 服务 | 启动类型改为 Disabled |
| `PinyinUp.exe`（网络更新程序） | 改名为 `PinyinUp.exe.bak` |

未改动 `userNetSchedule.exe`（账号词库同步）与 `SogouCloud.exe`（云输入），以免影响日常打字。

### 4.3 为什么不关闭 Trusted Mode

CS2 **没有**官方关闭开关，官方仅提供 `-allow_third_party_software`。放行注入只是让注入「成功一次」，并没有消除输入法代码在游戏进程内运行，不如直接移除资源加载路径。

## 五、实测效果

45 秒 PresentMon 采样，中文输入法状态下：

| 指标 | 修复前（中文） | 修复后（中文） |
|---|---|---|
| 平均 FPS | 172.6 | **267.9** |
| 1% low | 31.5 | **118.7** |
| 0.1% low | — | 42.9 |
| p99 帧间隔 | — | 6.2 ms |
| >33.3 ms 长帧 | 23 帧 / 90 秒 | **3 帧 / 45 秒** |
| 最长帧 | 88.2 ms | 48.25 ms |
| GPU busy | ~1.9 ms | 2.21 ms |
| 呈现模式 | Independent Flip | Independent Flip |

长帧按秒折算：0.256 次/秒 → 0.067 次/秒，约减少 4 倍；1% low 提升 3.8 倍。GPU 负载未变，改善全部来自 CPU 侧。

> 两段采样场景不完全相同，绝对值不可严格比较，但差距方向明确。

## 六、风险与注意事项

1. **升级会覆盖修复**：输入法更新会恢复 `Resource.dll`、`PinyinUp.exe`、`SogouSvc`。已冻结更新，若手动升级须重做 4.1/4.2。
2. **最新版同样有问题**：16.10.0.5858 安装包内仍有 `Resource.dll`（4,841,856 字节），PE 结构与 16.9 一致（导出 0、`.rsrc` 4.60 MB），架构未改。
3. **候选窗外观可能退化**：资源包缺失后皮肤图片可能不显示，输入功能不受影响。
4. **另一个潜在注入源**：通义千问输入法的 `qianwenime.dll` 也在 `cs2.exe` 中，但仅 1.07 MB 桥接层，引擎位于独立 server 进程，本次采样未表现出问题。
5. **Valve 侧进展**：官方称正在开发零开销的注入拦截机制，落地后可考虑恢复完整安装。

## 七、一键添加英文键盘（美式键盘）

> 英文键盘是 Windows **内置**的，不存在「下载」步骤。所谓安装，是把 `0409:00000409` 注册进语言列表。

以管理员身份打开 PowerShell：

```powershell
$langs = Get-WinUserLanguageList
if (-not ($langs | Where-Object { $_.LanguageTag -eq 'en-US' })) { $langs.Add('en-US') }
$en = $langs | Where-Object { $_.LanguageTag -eq 'en-US' }
if ($en.InputMethodTips -notcontains '0409:00000409') { $en.InputMethodTips.Add('0409:00000409') }
Set-WinUserLanguageList $langs -Force
Get-WinUserLanguageList | ForEach-Object { "$($_.LanguageTag): $($_.InputMethodTips -join ', ')" }
```

将英文设为默认（开机即英文，中文用 `Win + Space` 切换）：

```powershell
$langs = Get-WinUserLanguageList
$en = $langs | Where-Object { $_.LanguageTag -eq 'en-US' }
if ($en) { $langs.Remove($en); $langs.Insert(0, $en); Set-WinUserLanguageList $langs -Force }
```

## 八、回滚

恢复资源包：

```powershell
Rename-Item 'C:\Program Files (x86)\SogouInput\<version>\Resource.dll.bak' 'Resource.dll'
```

恢复更新能力（管理员）：

```powershell
Set-Service SogouSvc -StartupType Manual
Rename-Item 'C:\Program Files (x86)\SogouInput\<version>\PinyinUp.exe.bak' 'PinyinUp.exe'
```
