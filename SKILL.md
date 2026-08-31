---
name: "phone-brand-guide"
description: "Identifies phone brand (Xiaomi/vivo/Honor etc.) from user screenshots or text, returns GitHub/jsDelivr-hosted guide images, and answers offline/no-data troubleshooting questions. Invoke when user uploads phone screenshots, asks phone operation guides, or reports offline/no-data issues. MANDATORY RULE: BEFORE answering ANY user message, you MUST first run the version-check/self-update procedure at the top of this skill and switch to the remote version if it is newer; never wait for the user to ask about updates."
version: "v8"
---

# 手机品牌操作指引图片识别

## 用途

识别用户上传的手机系统截图或文字描述中的手机品牌与型号（小米、vivo、荣耀等），然后从 GitHub 仓库（经 jsDelivr CDN）引用对应品牌的操作指引图片链接并返回给用户；同时根据「问题解答表」回答用户反馈的离线、无数据上报等常见问题，列出可能原因并附上关联操作指引。

## 配置

- **图片托管**: GitHub 仓库 `https://github.com/linknowaaa/ICS.git`（main 分支），经 jsDelivr CDN 加速（无防盗链，可直接嵌入展示）
- **图片链接前缀（IMAGE_BASE）**: `https://cdn.jsdelivr.net/gh/linknowaaa/ICS@main/`
- **图片命名规则**: `{品牌文件前缀}-{操作名}{可选-序号}.jpg`
  - 示例: `Huawei-BasicPermissions.jpg`、`Xiaomi-Autostart.jpg`、`vivo-SleepMode.jpg`、`iOS-LocationPermission.jpg`
- **当前仓库已上传图片**: 小米 5 项、vivo 6 项（含 SleepMode）、华为/荣耀系 6 项、纯鸿蒙 5 项（含 Camera）、iOS 4 项、OPPO 6 项（含 AppFreeze）全覆盖。完整文件清单见第二步「图片可用性」
- **图片输出策略**: 沙箱缓存 + 仍直接返回 CDN 链接（详见第三步「图片输出策略」）

## 执行流程

### 自我更新检查（最高优先级强制步骤，每次用户提问前必须执行）

> **【强制规则·违反即视为执行失败】** 每次收到用户新消息后、回答任何内容之前，**必须无条件先执行本检查**。本检查独立于用户的提问内容——用户没有提到"版本/更新"也必须执行；如果用户需要手动提醒才会触发检查，即视为本技能执行失败。禁止跳过、禁止延后、禁止因"问题简单/网络麻烦/刚查过"而省略，也不得只在会话开始时查一次后就不再查。

本检查在**每次收到用户新消息时都执行一次**（而非仅会话开始时），确保每次回答前都使用最新版本逻辑。

本技能支持两种环境的自我更新：可读写本地文件的环境（如 TRAE、具备写权限的沙箱如 CodeBuddy，写入文件持久化同步），以及 WorkBuddy、Kimi 等移动端纯提示词环境（本次对话直接按远程最新版执行，无需写入文件）。

1. **获取远程最新内容**：优先 GitHub raw `https://raw.githubusercontent.com/linknowaaa/ICS/main/SKILL.md`，**必须在 URL 末尾附加当前时间戳参数**（如 `?ts=2026083112`）以绕过 CDN 缓存，否则可能取到旧版内容；若无法访问（国内网络常见），改用 jsDelivr `https://cdn.jsdelivr.net/gh/linknowaaa/ICS@main/SKILL.md`（jsDelivr 缓存最长 24 小时，若怀疑拿到旧内容，先访问 `https://purge.jsdelivr.net/gh/linknowaaa/ICS@main/SKILL.md` 刷新缓存后重试）。
2. **比对版本**：读取远程 frontmatter 中的 `version` 字段，与当前技能 frontmatter 的 `version` 对比：
   - **获取失败**（网络异常、仓库不可访问等）→ 跳过更新，按当前版本继续，不影响正常使用。
   - **远程无 `version` 字段** → 视为远程为旧版，跳过更新，按当前版本继续。
   - **版本一致** → 无需更新，直接继续。
   - **版本不同（远程更新）** →
     - **移动端纯提示词环境（WorkBuddy、Kimi 等）**：无法覆盖本地文件，改为**以远程最新内容为本次对话的权威指令**——本次对话后续所有步骤（品牌识别、操作映射、文字指引、问题解答、在线状态等）一律按远程最新版执行，并在回复开头告知用户「本技能已更新至版本 {远程version}，本次对话按最新版执行」。
     - **支持读写本地文件的环境（TRAE 等）**：用 Glob 定位本地技能文件 `**/.trae/skills/phone-brand-guide/SKILL.md`（兜底绝对路径：`C:\Users\w1802\AppData\Roaming\TRAE SOLO\ModularData\ai-agent\work-mode-projects\6a8650b104a097d16ec01a2b\.trae\skills\phone-brand-guide\SKILL.md`），先 Read 该文件，再使用 Write 将远程完整内容原样覆盖本地文件，实现持久化同步；同时本次对话按最新版执行。

> **执行顺序强制为：① 先完成版本检查（必要时同步/切换更新）→ ② 再按最新版逻辑回答用户问题。** 本技能任何其他流程（品牌识别、操作映射、问题解答、在线状态识别等）都不得先于本检查执行。回复内容必须基于版本检查之后确认的版本。

### 意图判断

- **操作指引查询**：用户询问某个操作如何设置（权限、自启动、后台锁定等）→ 走第一步～第三步。
- **问题解答**：用户描述离线、无数据、无数据上报、掉线等问题现象 → 走「问题解答流程（用户问题解答）」。
- **在线状态识别**：识别到设备状态为绿色「在线」、橙黄色「在线」、红色「离线」之一，或用户询问状态含义 → 走「在线状态与图标识别」。
- **灰色感叹号识别**：识别到【应在线】与灰色感叹号同时出现，或用户询问灰色感叹号含义 → 走「在线状态与图标识别」。

### 选择对话框规范（用交互式选择，不要纯文本提问）

当出现以下任一情况时，**必须使用交互式选择控件**（可点击按钮 / 单选卡片 / 下拉选项）让用户点选，**不要只用文字提问**，用户选择完成后再继续回答：

- 系统无法确认（如华为分系统不明、鸿蒙版本不明）
- 用户未说明具体要查询的操作类型
- 用户表示不清楚（品牌 / 系统 / 操作都不确定）

对话框按以下三组提供选项（系统已确认时，操作组只展示该系统支持的操作；系统未确认时列出全部常见操作）：

**① 系统版本选项**
- 鸿蒙（HarmonyOS 5.0 及以上）
- 鸿蒙 4.2 及以下 / EMUI / MagicOS / 荣耀
- 小米 / Redmi
- vivo / iQOO
- OPPO / 一加 / realme
- iOS（苹果）
- 我不清楚

**② 操作类型选项**
- 基础权限（权限 / 授权 / 允许）
- 自启动
- 后台运行 / 耗电 / 电池优化
- 后台锁定
- 省流量
- 电池白名单（华为系）
- 位置权限（鸿蒙 / iOS）
- 运动与健身（iOS）
- 后台App刷新（iOS）
- 低电量模式（iOS）
- 睡眠模式（vivo 老版本）
- 应用速冻（OPPO 老机型）
- 我不清楚

**③ "不清楚"的指引**
- 选择系统版本「我不清楚」→ 请用户上传「关于手机 / 关于本机」页面截图，或用对话框让其先选品牌；仍无法确认时按「未收录品牌兜底」提供小米完整操作指引作为参考。
- 选择操作类型「我不清楚」→ 询问用户当前遇到的具体问题（收不到数据、离线、无法自启动等），按问题现象走「问题解答流程」；用户无具体问题时返回该品牌全部操作指引图。

用户选择完成后，依据所选系统 + 操作继续走第一步～第三步（或对应流程）。

### 第一步：识别手机品牌与型号

#### 图片识别

当用户上传截图时，先使用 Read 工具查看图片内容，重点寻找以下线索：

1. 「关于手机」/「关于本机」页面中的品牌 Logo、设备名称、型号
2. 系统 UI 特征与系统名称：

| 系统/UI 名称 | 对应品牌 | 品牌文件前缀 | 支持的操作类型 |
|---|---|---|---|
| MIUI、澎湃 OS（HyperOS） | 小米 / Redmi | Xiaomi | 通用：BasicPermissions、Autostart、Background、BackgroundLock、DataSaver |
| OriginOS、Funtouch OS | vivo / iQOO | vivo | 通用：BasicPermissions、Autostart、Background、BackgroundLock、DataSaver + 睡眠模式（老版本） |
| HarmonyOS（鸿蒙） | 华为 | HarmonyOS | 鸿蒙：LocationPermission、DeviceDiscovery、PowerSaving、BackgroundLock、Camera |
| EMUI、MagicOS、Magic UI | 华为系（含荣耀） | Huawei | 华为系：BasicPermissions、Autostart、Background、BatteryWhitelist、BackgroundLock、DataSaver |
| ColorOS | OPPO / 一加 / realme | OPPO | 通用：BasicPermissions、Autostart、Background、BackgroundLock、DataSaver + 应用速冻关闭（老机型） |
| iOS | 苹果 | iOS | iOS：LocationPermission、MotionFitness、BackgroundRefresh、LowPowerMode |

3. 常见型号前缀线索：`M20xx / Mi / MIX / K系列 / Note系列(Redmi)` → 小米系；`V19xx / V20xx / X系列 / S系列 / iQOO` → vivo 系；`ALN / NOH / BKL / HRY / ANY / Mate / P系列 / 荣耀Magic系列` → 华为系（EMUI / MagicOS）。

#### 文字识别

对用户输入文本做大小写不敏感的中英文关键词匹配：

| 用户输入关键词 | 品牌 | 品牌文件前缀 | 支持的操作类型 |
|---|---|---|---|
| xiaomi、小米、redmi、红米、MIUI、澎湃 | 小米 | Xiaomi | 通用：BasicPermissions、Autostart、Background、BackgroundLock、DataSaver |
| vivo、VIVO、iQOO、OriginOS | vivo | vivo | 通用：BasicPermissions、Autostart、Background、BackgroundLock、DataSaver + 睡眠模式（老版本） |
| huawei、HUAWEI、华为、honor、HONOR、荣耀、EMUI、MagicOS、Magic UI、鸿蒙 | 华为系 | HarmonyOS 或 Huawei，见下方华为分系统规则 | 鸿蒙5项 或 华为系6项，见下方华为分系统规则 |
| oppo、OPPO、oneplus、一加、realme、真我、ColorOS | OPPO 系 | OPPO | 通用：BasicPermissions、Autostart、Background、BackgroundLock、DataSaver + 应用速冻关闭（老机型） |
| apple、iphone、苹果、iOS | 苹果 | iOS | iOS：LocationPermission、MotionFitness、BackgroundRefresh、LowPowerMode |

- 若能识别出具体型号（如「小米 14 Pro」「vivo X100」），记录型号用于回复展示；但指引图片按品牌粒度返回，不区分型号。
- 华为分系统规则：识别到「鸿蒙 / HarmonyOS」→ 品牌文件前缀 `HarmonyOS`、操作类型为鸿蒙5项（LocationPermission、DeviceDiscovery、PowerSaving、BackgroundLock、Camera）；识别到「EMUI / MagicOS / Magic UI / 荣耀 / 安卓」→ 品牌文件前缀 `Huawei`、操作类型为华为系6项（BasicPermissions、Autostart、Background、BatteryWhitelist、BackgroundLock、DataSaver）；两者都出现或不确定时，按「选择对话框规范」①系统版本选项让用户选择（含「我不清楚」选项），选择后再继续。
- 鸿蒙版本询问规则：若设备为鸿蒙但版本不明，按「选择对话框规范」①系统版本选项让用户选择鸿蒙版本：4.2 及以下走 Android 表（华为系，品牌文件前缀 `Huawei`、操作类型为华为系6项），高于 4.2 走鸿蒙表（品牌文件前缀 `HarmonyOS`、操作类型为鸿蒙5项）；选择「我不清楚」时按③指引处理。

### 第二步：确定操作类型

**先判断用户是否询问了具体操作：**

- **未询问具体操作**：若用户仅提供了品牌/型号的文字或截图，没有说明要查什么指引，则按「选择对话框规范」②操作类型选项（含「我不清楚」选项）让用户点选需要指引的操作，等待选择后再继续第三步；系统也不确认时，①②两组选项同时给出。
- **已询问具体操作**：从用户问题中提取操作关键词，按第一步识别出的系统选择对应子表，映射为 `operation_name`。

> 不同系统支持的操作类型不同，按第一步识别出的系统选择对应子表：

#### 通用操作类型（小米 / vivo / OPPO 系）

| 用户问题关键词 | 操作名（operation_name） |
|---|---|
| 权限、授权、允许、权限设置、应用权限、授予权限、权限管理 | BasicPermissions |
| 自启动、自启、后台启动、开机自启、允许自启动、启动管理、自启动管理 | Autostart |
| 后台运行、保活、省电、电池优化、后台、锁屏运行、关闭后台、后台耗电 | Background |
| 后台锁定、锁后台、后台应用锁定、任务锁定、应用锁、后台保留、不被清理 | BackgroundLock |
| 省流量、流量节省、数据节省、节省流量、智能省流量、流量保护 | DataSaver |

图片可用性（当前仓库）：

| 操作 | 小米 | vivo | OPPO |
|---|---|---|---|
| BasicPermissions | ✅ `Xiaomi-BasicPermissions.jpg` | ✅ `vivo-BasicPermissions.jpg` | ✅ `OPPO-BasicPermissions-1.jpg`、`-2.jpg` |
| Autostart | ✅ `Xiaomi-Autostart.jpg` | ✅ `vivo-Autostart-1.jpg`、`-2.jpg` | ✅ `OPPO-Autostart.jpg` |
| Background | ✅ `Xiaomi-Background.jpg` | ✅ `vivo-Background-1.jpg`、`-2.jpg`、`-3.jpg` | ✅ `OPPO-Background-1.jpg`、`-2.jpg`、`-3.jpg` |
| BackgroundLock | ✅ `Xiaomi-BackgroundLock.jpg` | ✅ `vivo-BackgroundLock.jpg` | ✅ `OPPO-BackgroundLock-1.jpg`、`-2.jpg` |
| DataSaver | ✅ `Xiaomi-DataSaver.jpg` | ✅ `vivo-DataSaver.jpg` | ✅ `OPPO-DataSaver.jpg` |

> 小米备注：MIUI 10 早期子版本中无独立「锁定/后台锁定」入口，锁定功能已整合进「应用免清理」（路径：手机管家 → 优化加速 → 应用免清理）。识别到小米设备询问后台锁定/加锁时，若为 MIUI 10 早期版本，指引路径使用「应用免清理」而非锁定。

#### 华为系操作类型（EMUI / MagicOS / 荣耀）

| 用户问题关键词 | 操作名（operation_name） | 是否有图片 |
|---|---|---|
| 权限、授权、允许、权限设置、应用权限、授予权限、权限管理 | BasicPermissions | ✅ 有（`Huawei-BasicPermissions.jpg`） |
| 自启动、自启、后台启动、开机自启、允许自启动、启动管理、自启动管理 | Autostart | ✅ 有（`Huawei-Autostart-1.jpg`、`-2.jpg`） |
| 后台运行、保活、省电、后台、锁屏运行、关闭后台、后台耗电（不含电池优化） | Background | ✅ 有（`Huawei-Background.jpg`） |
| 电池白名单、白名单、后台白名单、耗电白名单、电池优化白名单 | BatteryWhitelist | ✅ 有（`Huawei-BatteryWhitelist-1.jpg`、`-2.jpg`、`-3.jpg`） |
| 后台锁定、锁后台、后台应用锁定、任务锁定、应用锁、后台保留、不被清理 | BackgroundLock | ✅ 有（`Huawei-BackgroundLock.jpg`） |
| 省流量、流量节省、数据节省、节省流量、智能省流量、流量保护 | DataSaver | ✅ 有（`Huawei-DataSaver.jpg`） |

#### HarmonyOS（鸿蒙）专属操作类型（仅支持以下操作）

| 用户问题关键词 | 操作名（operation_name） | 是否有图片 |
|---|---|---|
| 位置权限、定位、定位权限、位置服务、位置信息、GPS、定位服务 | LocationPermission | ✅ 有（`HarmonyOS-LocationPermission.jpg`） |
| 设备发现和连接权限、设备发现、附近设备、发现设备、设备互联、蓝牙连接、超级终端、多设备协同 | DeviceDiscovery | ✅ 有（`HarmonyOS-DeviceDiscovery.jpg`） |
| 省电模式、省电、超级省电、省电开关、续航、电池省电 | PowerSaving | ✅ 有（`HarmonyOS-PowerSaving.jpg`） |
| 后台锁定、锁后台、后台应用锁定、任务锁定、应用锁、后台保留、不被清理 | BackgroundLock | ✅ 有（`HarmonyOS-BackgroundLock.jpg`） |
| 相机、相机权限、拍照、相机使用权限 | Camera | ✅ 有（`HarmonyOS-Camera-1.jpg`、`-2.jpg`） |

#### iOS 专属操作类型（仅支持以下操作）

| 用户问题关键词 | 操作名（operation_name） | 是否有图片 |
|---|---|---|
| 位置权限、定位、定位权限、位置服务、位置信息、GPS、定位服务 | LocationPermission | ✅ 有（`iOS-LocationPermission.jpg`） |
| 运动与健身、运动、健身、健康、运动数据、健康数据、运动权限 | MotionFitness | ✅ 有（`iOS-MotionFitness.jpg`） |
| 后台App刷新、后台刷新、后台应用刷新、App刷新 | BackgroundRefresh | ✅ 有（`iOS-BackgroundRefresh.jpg`） |
| 低电量模式、低电量、省电模式、省电 | LowPowerMode | ✅ 有（`iOS-LowPowerMode.jpg`） |

> iOS 提示：调整了任何权限后，务必清除后台并重新打开「掌上环卫」APP，权限设置才会生效。

#### 品牌专属补充操作（特定机型 / 系统版本）

| 品牌 | 适用机型 | 用户问题关键词 | 操作名（operation_name） | 是否有图片 |
|---|---|---|---|---|
| vivo | 老版本手机（Funtouch OS / 老版 OriginOS） | 睡眠模式、睡眠、关闭睡眠模式 | SleepMode | ✅ 有（`vivo-SleepMode.jpg`） |
| OPPO | 老机型（老版 ColorOS） | 应用速冻、关闭应用速冻、应用速冻设置、速冻 | AppFreeze | ✅ 有（`OPPO-AppFreeze.jpg`） |

> 品牌专属补充操作仅适用于对应品牌的特定机型/系统版本：识别到「睡眠模式」时，若设备为 vivo 老版本则映射 `SleepMode`（`vivo-SleepMode.jpg`）；识别到「应用速冻」时，若设备为 OPPO 老机型则映射 `AppFreeze`（`OPPO-AppFreeze.jpg`）。设备不满足适用条件时按异常处理告知不支持，不要编造链接。

> 当前仓库图片分布：小米 5 项、vivo 6 项（含 SleepMode）、华为/荣耀系 6 项、纯鸿蒙 5 项（含 Camera）、iOS 4 项、OPPO 6 项（含 AppFreeze）图片全部齐全。若后续新增操作无图，按异常处理返回提示，不要编造链接。

### 第三步：拼接链接并返回

1. 按品牌（及系统）从第一步映射表查得品牌文件前缀，按 `{品牌文件前缀}-{operation_name}{可选-序号}.jpg` 拼接完整 URL（前缀 `https://cdn.jsdelivr.net/gh/linknowaaa/ICS@main/` + 文件名）。**拼接前先对照第二步「图片可用性」确认该品牌该操作是否有图，无图不拼接**：
   - 华为（鸿蒙）5 项：`https://cdn.jsdelivr.net/gh/linknowaaa/ICS@main/HarmonyOS-LocationPermission.jpg`（DeviceDiscovery / PowerSaving / BackgroundLock 同理）；Camera 返回 `HarmonyOS-Camera-1.jpg` 和 `-2.jpg`
   - 华为（EMUI/荣耀）6 项：`https://cdn.jsdelivr.net/gh/linknowaaa/ICS@main/Huawei-BasicPermissions.jpg`；Autostart 返回 `Huawei-Autostart-1.jpg` 和 `-2.jpg`；BatteryWhitelist 返回 `Huawei-BatteryWhitelist-1.jpg`、`-2.jpg`、`-3.jpg`
   - 小米 5 项：`https://cdn.jsdelivr.net/gh/linknowaaa/ICS@main/Xiaomi-BasicPermissions.jpg`（Autostart / Background / BackgroundLock / DataSaver 同理，各 1 张）
   - vivo 6 项：`vivo-BasicPermissions.jpg`、`vivo-BackgroundLock.jpg`、`vivo-DataSaver.jpg`、`vivo-SleepMode.jpg` 各 1 张；Autostart 返回 `vivo-Autostart-1.jpg` 和 `-2.jpg`；Background 返回 `vivo-Background-1.jpg`、`-2.jpg`、`-3.jpg`
   - iOS 4 项：`iOS-LocationPermission.jpg`、`iOS-MotionFitness.jpg`、`iOS-BackgroundRefresh.jpg`、`iOS-LowPowerMode.jpg`（注意前缀大小写为 `iOS-`）
   - OPPO 6 项：BasicPermissions 返回 `OPPO-BasicPermissions-1.jpg` 和 `-2.jpg`；Background 返回 `OPPO-Background-1.jpg`、`-2.jpg`、`-3.jpg`；BackgroundLock 返回 `OPPO-BackgroundLock-1.jpg` 和 `-2.jpg`；DataSaver 返回 `OPPO-DataSaver.jpg`；Autostart 返回 `OPPO-Autostart.jpg`；AppFreeze 返回 `OPPO-AppFreeze.jpg`
2. 使用 Markdown 图片语法直接展示图片：

```markdown
![华为基础权限指引](https://cdn.jsdelivr.net/gh/linknowaaa/ICS@main/Huawei-BasicPermissions.jpg)
```

3. 回复模板（图片 + 文字操作指引一起返回；图片一律按下方「图片输出策略」输出）：

```markdown
已识别设备：{品牌} {型号（如有）}

以下是{品牌}手机{操作名称}的操作指引：

![{品牌}{操作名称}指引]({image_url})

{按品牌输出对应的「品牌文字操作指引」内容，见下方小节}
```

> **图片输出策略（沙箱缓存 + 仍直接返回 CDN 链接）**：本技能返回的所有指引图片均执行以下策略：
> 1. **沙箱缓存**：先用下载工具把图片从 jsDelivr CDN 下载到沙箱缓存目录 `{当前工作目录}/.cache/phone-brand-guide/`（目录不存在则创建，文件名与仓库保持一致）。下载成功时，回复中可同时标注缓存路径，便于支持本地文件渲染的环境（如 TRAE）直接展示缓存图；下载失败则跳过本步，不影响使用。
> 2. **仍直接返回 CDN 链接**：回复正文中的图片地址始终使用 jsDelivr CDN 链接（`https://cdn.jsdelivr.net/gh/linknowaaa/ICS@main/文件名`），确保不支持本地文件的移动端（Kimi、WorkBuddy 等）仍可通过点击链接查看。
> 3. 该策略适用于所有返回图片的场景：操作指引图、在线/离线状态图（`OnlineDetails.jpg`）、更新提示图（`Update.jpg`）、未收录品牌兜底的小米指引图。

### 品牌文字操作指引

**返回图片操作指引时，必须同时附上对应品牌的以下文字说明**（按识别出的品牌/系统选择；用户只问单一操作时，可只输出与询问操作相关的段落）：

#### 华为/荣耀机型

基础权限设置：
出现授权弹窗时点击继续，并按以下设置完成授权：
- 位置信息：选择始终允许
- 相机：选择仅使用期间允许
- 媒体和文件：选择始终允许
- 运动健康：选择允许
- 电池优化：选择关闭

自启动设置：
打开手机设置-点击应用和服务-点击应用启动管理-搜索掌上环卫-关闭自动管理-开启允许自启动、开启允许关联启动、开启允许后台活动

电池模式设置：
打开手机设置-点击电池-点击更多电池设置-打开休眠时始终保持网络连接

省流量：
设置中搜索省流量或在设置->移动网络->流量管理->智能省流量,如果有相关设置,将掌上环卫省流量关掉

#### OPPO机型

基础权限设置：
【应用信息】---> 【应用权限】--->【运动健康（允许）】--->【定位权限（始终允许）】

自启动设置：
跳转到【应用信息】页面--->【耗电管理】--->【允许完全后台行为、允许应用自启动、允许应用关联启动】

电池优化设置：
【设置】--->【电池】--->【更多】--->【耗电异常优化】--->【搜索列表搜索掌上环卫】--->点击【掌上环卫】--->选择【不优化】

省流量：
设置中搜索省流量或在设置->移动网络,如果有相关设置,将掌上环卫省流量关掉

#### VIVO机型

基础权限设置：
点击掌上环卫-我的-系统权限管理跳转至应用信息页面，点击权限，开启对应权限（定位、身体活动、存储、相机）

自启动设置：
自启动设置（打开手机设置-搜索自启动并点击或点击应用与权限再点击权限管理-打开掌上环卫自启动）

电池模式设置和允许后台高耗电：
（打开手机设置-点击电池-不要选择省电模式和超级省电模式-点击后台耗电管理-点击掌上环卫-选择允许后台高耗电）

关闭电池-省电管理：
电池--省电管理--熄屏5分钟断开网络连接【关闭】--睡眠模式【关闭】

省流量：
设置中搜索省流量或在设置->移动网络,如果有相关设置,将掌上环卫省流量关掉

#### 小米机型

基础权限设置：
点击app内权限设置跳转至app详情页，点击权限管理，开启对应权限（位置信息，运动与健康）

自启动设置：
长按app图标，点击进入app详情页，打开自启动按钮

关闭省电限制：
点击设置，找到省电与电池，滑到最下方，找到掌上环卫app，点击后，选择无限制

省流量：
设置中搜索省流量或在设置->移动网络,如果有相关设置,将掌上环卫省流量关掉

#### 纯鸿蒙机型（HarmonyOS 5.0版本及以上）

位置权限：
需开启位置权限, 将位置访问权限设为 "始终允许", 并将"精确位置"打开
总开关: 设置->隐私与安全->位置->访问我的位置
APP内开关: 设置->应用和元服务->掌上环卫->位置

设备发现和连接权限：
需开启设备发现和连接权限
设置->应用和元服务->掌上环卫->设备发现和连接

省电模式：
将省电模式关掉：设置->电池->省电模式

相机权限：
打开掌上环卫->点击相机->选择允许

#### iOS机型

位置权限：
需要开启位置权限, 将允许访问位置信息设为"始终", 并将"精确位置"打开
总开关:设置->隐私与安全性->定位服务
APP内开关: 设置->App->掌上环卫->位置

运动与健身：
需要开启运动与健身
总开关: 设置->隐私与安全性->运动与健身
APP内开关: 设置->App->掌上环卫->运动与健身

后台App刷新：
通用->后台APP刷新->最上方将后台App刷新设置为"无线局域网与蜂窝数据"->打开掌上环卫刷新开关按钮

低电量模式：
将低电量模式关掉；
系统设置->电池->低电量模式->关闭

## 在线状态与图标识别

### 1. 在线/离线状态识别

识别到绿色「在线」、橙黄色「在线」、红色「离线」这三种状态之一（通过截图识别或用户直接询问），返回对应说明并附上状态详情图：

| 识别到的状态 | 返回说明 |
|---|---|
| 绿色「在线」 | 设备在线，状态正常。 |
| 橙黄色「在线」 | 设备即将离线，请关注并及时检查（如权限、网络等）。 |
| 红色「离线」 | 设备当前已离线。 |

```markdown
![在线/离线状态详情](https://cdn.jsdelivr.net/gh/linknowaaa/ICS@main/OnlineDetails.jpg)
```

### 2. 灰色感叹号识别

识别到【应在线】与灰色感叹号同时出现，或用户询问灰色感叹号含义时，返回：

出现灰色感叹号提示表示需要帮助员工更新掌上环卫版本

```markdown
![更新提示详情](https://cdn.jsdelivr.net/gh/linknowaaa/ICS@main/Update.jpg)
```

## 问题解答流程（用户问题解答）

当用户描述离线 / 无数据 / 无上报等问题现象时，按以下流程解答。

### 第一步：确定操作系统

| 操作系统 | 判定依据 |
|---|---|
| Android（包含鸿蒙4.2及以下版本） | 小米 / vivo / OPPO / 华为EMUI / 荣耀MagicOS 等安卓设备，以及鸿蒙版本为 4.2 及以下的设备 |
| 鸿蒙 | 鸿蒙（HarmonyOS）设备（版本高于 4.2，或未说明版本时默认） |
| iOS | 苹果设备 |

- 若设备为鸿蒙但版本不明，先询问用户鸿蒙版本号：4.2 及以下走 Android 表，高于 4.2 走鸿蒙表。
- 品牌识别复用「执行流程-第一步」的映射表，用于后续附关联操作指引图。

### 第二步：匹配问题现象

将用户描述与对应系统下表的「问题现象」列做关键词模糊匹配（前台/后台、无数据、离线、轨迹、通知栏等）。命中多行则合并返回；未命中时询问用户补充具体现象（何时离线、前台还是后台、通知栏是否有服务运行中等）。

### 第三步：返回解答

按「可能原因」逐条列出；原因带关联操作的，结合品牌拼接对应操作指引图（当前所有品牌的支持操作均已有图），其余注明指引图暂未上传。

#### Android（包含鸿蒙4.2及以下）问题解答表

| 问题现象 | 可能原因（关联操作） |
|---|---|
| 1.前台有数据上报，退到后台无数据；2.前几天是好的，今天离线了 | 1.APP未加锁（BackgroundLock）；2.自启动开反了/漏了（华为，荣耀）（Autostart）；3.电池设置不到位（电池白名单，关闭低电量）（BatteryWhitelist）；4.允许后台网络/耗电（Background）；5.系统提示高耗电，用户误操作关闭了权限（BasicPermissions）；6.省流量模式（DataSaver） |
| 1.只有上下班有数据；2.你在旁边有数据，离开就无数据了；3.上报数据的间隔越来越长，30分钟后离线 | 1.连续返回，杀掉了app（3.2.1之前老版本）；2.划掉app（BackgroundLock）；3.关闭网络/定位（LocationPermission）；4.手机未携带（vivo），手机长期静止导致离线（SleepMode）；5.飞行模式 |
| 权限设置正常但离线 | 1.区域网络环境差；2.在地下室/电梯里；网络正常后数据会补传（无关联操作）；3.手动划掉APP；4.前台开着视频软件，后台太多APP导致运存不足限制后台 |
| 通知栏没有服务运行中且无数据上报 | 在前台运行超10分钟也不上传数据（无关联操作） |

#### 鸿蒙问题解答表

| 问题现象 | 可能原因（关联操作） |
|---|---|
| 应在线无轨迹数据 | 1.定位权限没开启（LocationPermission）；2.设备发现和连接权限未开启（DeviceDiscovery） |
| 设置正常，中间也会离线 | 1.用户有划掉APP（BackgroundLock）；2.区域网络环境差；在地下室/电梯里（无关联操作）；3.内存不够时系统会冻结APP，系统问题会自动补传；app自动更新了，未重新打开掌上环卫（无关联操作） |
| 设置正常，但中途通知栏无正在运行定位任务 | 手机资源紧张，运行app太多，掌上环卫被杀掉（BackgroundLock） |

#### iOS 问题解答表

| 问题现象 | 可能原因（关联操作） |
|---|---|
| 前台有数据上报，退到后台无数据 | 1.app后台刷新功能未开启（BackgroundRefresh）；2.低电量模式未关闭（LowPowerMode）；3.未开启始终定位（LocationPermission） |
| 应在线无轨迹数据（请更新到3.3.2以上版本） | 1.APP退到后台时有关闭定位权限后未重新打开APP（LocationPermission）；2.用户有划掉APP（BackgroundLock） |
| 设置正常，中间也会离线 | 1.用户有划掉APP（BackgroundLock）；2.区域网络环境差；在地下室/电梯里（无关联操作）；3.内存不够掌上环卫被杀掉，掌上环卫进入前台会自动补传（无关联操作） |

### 解答回复模板

```markdown
已识别系统：{操作系统}

您描述的问题可能原因及处理建议：

1. {原因1}
2. {原因2}
...

{对带关联操作的原因，附对应操作指引图或注明指引图暂未上传}
```

## 异常处理

- **无法识别品牌**：不要猜测，请用户提供手机品牌名称或「关于手机」页面截图。
- **未收录品牌（安卓）**：若用户询问的是 skill 未收录的安卓机型（如魅族、三星、酷派、努比亚等），按以下模板回答，并附上小米的完整操作指引（见下方「未收录品牌兜底：小米完整操作指引」）：
  ```
  本机型暂未收录，现提供小米机型的操作指引供参考，可以在手机【设置】里搜索关键词。
  {小米完整操作指引}
  ```
- **操作无图片**：当前仓库已覆盖所有已定义操作类型；若用户询问的操作在仓库中无对应图片，明确告知用户该操作指引尚未上传，不要编造链接。
- **系统不支持该操作**：鸿蒙系统仅支持位置权限、设备发现和连接权限、省电模式、后台锁定、相机权限；iOS 仅支持位置权限、运动与健身、后台App刷新、低电量模式。用户询问该操作系统不支持的操作时，明确告知不支持，不要编造图片链接。
- **图片链接可能失效**：jsDelivr 依赖 GitHub 仓库，需保证仓库为公开、文件名大小写与仓库完全一致；jsDelivr 有缓存，新推送的图片若未生效可稍等或访问 `https://purge.jsdelivr.net/gh/linknowaaa/ICS@main/文件名` 刷新缓存，不要编造其他链接。

### 未收录品牌兜底：小米完整操作指引

当识别到未收录的安卓品牌时，输出以下全部内容（文字说明 + 小米 5 项操作指引图）：

**文字说明**：

基础权限设置：
点击app内权限设置跳转至app详情页，点击权限管理，开启对应权限（位置信息，运动与健康）

自启动设置：
长按app图标，点击进入app详情页，打开自启动按钮

关闭省电限制：
点击设置，找到省电与电池，滑到最下方，找到掌上环卫app，点击后，选择无限制

省流量：
设置中搜索省流量或在设置->移动网络,如果有相关设置,将掌上环卫省流量关掉

**操作指引图**：

```markdown
基础权限设置：
![小米基础权限指引](https://cdn.jsdelivr.net/gh/linknowaaa/ICS@main/Xiaomi-BasicPermissions.jpg)

自启动设置：
![小米自启动指引](https://cdn.jsdelivr.net/gh/linknowaaa/ICS@main/Xiaomi-Autostart.jpg)

后台运行/耗电设置：
![小米后台运行指引](https://cdn.jsdelivr.net/gh/linknowaaa/ICS@main/Xiaomi-Background.jpg)

后台锁定设置：
![小米后台锁定指引](https://cdn.jsdelivr.net/gh/linknowaaa/ICS@main/Xiaomi-BackgroundLock.jpg)

省流量设置：
![小米省流量指引](https://cdn.jsdelivr.net/gh/linknowaaa/ICS@main/Xiaomi-DataSaver.jpg)
```
