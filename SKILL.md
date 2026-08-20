---
name: "phone-brand-guide"
description: "Identifies phone brand (Xiaomi/vivo/Honor etc.) from user screenshots or text, returns GitHub/jsDelivr-hosted guide images. Invoke when user uploads phone screenshots or asks phone operation guides."
---

# 手机品牌操作指引图片识别

## 用途

识别用户上传的手机系统截图或文字描述中的手机品牌与型号（小米、vivo、荣耀等），然后从 GitHub 仓库（经 jsDelivr CDN）引用对应品牌的操作指引图片链接并返回给用户。

## 配置

- **图片托管**: GitHub 仓库 `https://github.com/linknowaaa/ICS.git`（main 分支），经 jsDelivr CDN 加速（无防盗链，可直接嵌入展示）
- **图片链接前缀（IMAGE_BASE）**: `https://cdn.jsdelivr.net/gh/linknowaaa/ICS@main/`
- **图片命名规则**: `{品牌文件前缀}-{操作名}{可选-序号}.jpg`
  - 示例: `Xiaomi-BasicPermissions.jpg`、`vivo-BasicPermissions.jpg`、`OPPO-BasicPermissions-1.jpg`
- **当前仓库已上传图片**: 仅「基础权限（BasicPermissions）」操作，品牌与文件对应关系见下方映射表

## 执行流程

### 第一步：识别手机品牌与型号

#### 图片识别

当用户上传截图时，先使用 Read 工具查看图片内容，重点寻找以下线索：

1. 「关于手机」/「关于本机」页面中的品牌 Logo、设备名称、型号
2. 系统 UI 特征与系统名称：

| 系统/UI 名称 | 对应品牌 | 权限图片文件（BasicPermissions） |
|---|---|---|
| MIUI、澎湃 OS（HyperOS） | 小米 / Redmi | `Xiaomi-BasicPermissions.jpg` |
| OriginOS、Funtouch OS | vivo / iQOO | `vivo-BasicPermissions.jpg` |
| MagicOS、Magic UI | 荣耀 | 暂未上传 |
| HarmonyOS（鸿蒙） | 华为 | `HarmonyOS-BasicPermissions.jpg` |
| EMUI | 华为 | `Huawei-BasicPermissions.jpg` |
| ColorOS | OPPO / 一加 / realme | `OPPO-BasicPermissions-1.jpg`、`-2.jpg` |
| One UI | 三星 | 暂未上传 |
| iOS | 苹果 | 暂未上传 |
| Flyme | 魅族 | 暂未上传 |
| 红魔 OS | 红魔 / 努比亚 | 暂未上传 |

3. 常见型号前缀线索：`M20xx / Mi / MIX / K系列 / Note系列(Redmi)` → 小米系；`V19xx / V20xx / X系列 / S系列 / iQOO` → vivo 系；`ALN / NOH / BKL / Mate / P系列` → 华为；`HRY / ANY / 荣耀Magic系列` → 荣耀。

#### 文字识别

对用户输入文本做大小写不敏感的中英文关键词匹配：

| 用户输入关键词 | 品牌 | 权限图片文件（BasicPermissions） |
|---|---|---|
| xiaomi、小米、redmi、红米、MIUI、澎湃 | 小米 | `Xiaomi-BasicPermissions.jpg` |
| vivo、VIVO、iQOO、OriginOS | vivo | `vivo-BasicPermissions.jpg` |
| honor、HONOR、荣耀、MagicOS | 荣耀 | 暂未上传 |
| huawei、HUAWEI、华为、EMUI、鸿蒙 | 华为 | 见下方华为分系统规则 |
| oppo、OPPO、oneplus、一加、realme、真我、ColorOS | OPPO 系 | `OPPO-BasicPermissions-1.jpg`、`-2.jpg` |
| samsung、三星、galaxy、One UI | 三星 | 暂未上传 |
| apple、iphone、苹果、iOS | 苹果 | 暂未上传 |
| meizu、魅族、Flyme | 魅族 | 暂未上传 |
| nubia、努比亚、红魔 | 努比亚 | 暂未上传 |

- 若能识别出具体型号（如「小米 14 Pro」「vivo X100」），记录型号用于回复展示；但指引图片按品牌粒度返回，不区分型号。
- 荣耀与华为需严格区分：出现「HONOR / 荣耀 / MagicOS」→ 荣耀；出现「HUAWEI / 华为」→ 华为。
- 华为分系统规则：识别到「鸿蒙 / HarmonyOS」→ `HarmonyOS-BasicPermissions.jpg`；识别到「EMUI / 安卓」→ `Huawei-BasicPermissions.jpg`；两者都出现或不确定时，同时返回两张图。

### 第二步：确定操作类型

从用户问题中提取想查询的操作，映射为 `operation_name`：

| 用户问题关键词 | 操作名（operation_name） | 是否有图片 |
|---|---|---|
| 权限、授权、允许 | BasicPermissions | ✅ 有 |
| 自启动、自启、后台启动 | Autostart | ❌ 暂未上传 |
| 悬浮窗、小窗、画中画 | Floating Window | ❌ 暂未上传 |
| 分屏、多窗口 | Split Screen | ❌ 暂未上传 |
| 后台运行、保活、省电、电池优化 | Background | ❌ 暂未上传 |
| 安装、未知来源、安装包 | Install | ❌ 暂未上传 |
| 通知、消息提醒 | Notification | ❌ 暂未上传 |

> 当前仓库仅上传了「权限 / 授权 / 允许」对应的 `BasicPermissions` 图片。若用户询问其他操作，按异常处理返回提示，不要编造链接。

### 第三步：拼接链接并返回

1. 按品牌从第一步映射表查得图片文件，拼接完整 URL（前缀 `https://cdn.jsdelivr.net/gh/linknowaaa/ICS@main/` + 文件名）：
   - 小米：`https://cdn.jsdelivr.net/gh/linknowaaa/ICS@main/Xiaomi-BasicPermissions.jpg`
   - vivo：`https://cdn.jsdelivr.net/gh/linknowaaa/ICS@main/vivo-BasicPermissions.jpg`
   - 华为（鸿蒙）：`https://cdn.jsdelivr.net/gh/linknowaaa/ICS@main/HarmonyOS-BasicPermissions.jpg`
   - 华为（EMUI）：`https://cdn.jsdelivr.net/gh/linknowaaa/ICS@main/Huawei-BasicPermissions.jpg`
   - OPPO（两张）：`https://cdn.jsdelivr.net/gh/linknowaaa/ICS@main/OPPO-BasicPermissions-1.jpg` 和 `...-2.jpg`
2. 使用 Markdown 图片语法直接展示图片：

```markdown
![小米权限设置指引](https://cdn.jsdelivr.net/gh/linknowaaa/ICS@main/Xiaomi-BasicPermissions.jpg)
```

3. 回复模板：

```markdown
已识别设备：{品牌} {型号（如有）}

以下是{品牌}手机{操作名称}的操作指引：

![{品牌}{操作名称}指引]({image_url})
```

## 异常处理

- **无法识别品牌**：不要猜测，请用户提供手机品牌名称或「关于手机」页面截图。
- **品牌无图片**：荣耀、三星、苹果、魅族、努比亚的权限指引图片尚未上传到仓库，明确告知用户图片暂缺，不要编造链接。
- **操作无图片**：除「权限 / 授权 / 允许」外，其他操作（自启动、悬浮窗、分屏等）暂无图片，明确告知用户该操作指引尚未上传。
- **图片链接可能失效**：jsDelivr 依赖 GitHub 仓库，需保证仓库为公开、文件名大小写与仓库完全一致；jsDelivr 有缓存，新推送的图片若未生效可稍等或访问 `https://purge.jsdelivr.net/gh/linknowaaa/ICS@main/文件名` 刷新缓存，不要编造其他链接。
