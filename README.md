# Android-Guru-Agent-Release

[Android-Guru-Agent](https://github.com/AceGuru-mjh/Android-Guru-Agent) 的**发布仓库** —— 补丁发布主体：全部更新资产（APK / vcdiff 增量 / 热更包）与两份客户端清单都在这里，不含源码。

## 这里有什么

| 内容 | 说明 |
| --- | --- |
| **[Releases](https://github.com/AceGuru-mjh/Android-Guru-Agent-Release/releases)** | 每个版本的 APK（arm64 / universal）、增量补丁与热更包 |
| **[version.json](./version.json)** | 最新版本清单（版本号 / 下载地址 / SHA-256 / 热更档 / 签名指纹），供 App 内检查更新使用 |
| **[patches.json](./patches.json)** | 历史补丁全量索引（from→to 链），App 跨版本链式增量更新的数据源 |
| **[HOT_UPDATE.md](./HOT_UPDATE.md)** | 热更新通道规范与发布手册（hot 包格式 / version.json 新字段 / **应急热修操作手册** / FAQ） |
| **清单校验门禁** | `.github/workflows/validate-manifests.yml` —— 两份清单的结构/自洽/单调性守门（手改清单必须过 PR 校验） |
| **应急热修工作流** | `.github/workflows/hotfix.yml` —— 数据层坏内容紧急重发（零 APK 发版，约 2 分钟送达全部客户端） |

## 发布流程（全自动）

```
开发仓库 push tag v*.*.*（或 PR 合并进 main）
        │
        ▼
开发仓库 CI 构建 APK（universal + arm64）
        │
        ├─ 生成 version.json（含 SHA-256 校验 / 签名证书指纹 / commit）
        ├─ 与上一版本生成 xdelta3 增量补丁（LZMA 二级压缩）
        ├─ 累积维护 patches.json（历史补丁全量索引）
        ├─ data-only 判定：本版变更全在热载路径（assets/skills、
        │   assets/mcp_catalog）→ 额外产出热更包 hot_v*.zip（免安装通道）
        ▼
通过跨仓库 PAT 发布到本仓库
        │
        ├─ 创建 Release + 上传 APK / 补丁 / 热更包 / version.json
        └─ 提交 version.json + patches.json 到 main 分支（供 App 内更新检查）
        │
        ▼
本仓库 validate-manifests 门禁自动复核清单（结构 / 自洽 / 单调性）
```

### 应急热修（本仓库独立执行，不走开发仓库发版）

已发布版本的数据层（技能 / 目录）出了坏内容时：开发仓库合入仅触碰
assets 数据的修复 → 本仓库 Actions → **Emergency hot update** →
工作流打包快照、挂到当前版本 Release、重写 hot 档（target 不变 +
新指纹）→ 客户端凭包指纹自动重新应用（零安装）。白名单守卫与开发仓库
CI 同尺：含代码变更的修复拒绝走此通道。操作手册见
**[HOT_UPDATE.md](./HOT_UPDATE.md)** 「应急热修」一节。

源代码、Issue、PR 请移步开发仓库：**https://github.com/AceGuru-mjh/Android-Guru-Agent**

## 下载哪个 APK？

| 变体 | 适用设备 | 体积 |
| --- | --- | --- |
| `*-arm64-*.apk` | 现代手机（arm64-v8a，**首选**） | ~300 MB |
| `*-universal-*.apk` | 旧设备 / 模拟器（含全部 3 ABI） | ~900 MB |

> 两个变体均内置 Ubuntu 24.04 完整 CLI 环境（gcc / python3 / git / vim 开箱即用），
> 首次使用离线解包约 2~5 分钟，零网络依赖。

## App 内检查更新

应用「设置 → 关于」中可检查更新；更新清单固定读取：

```
https://raw.githubusercontent.com/AceGuru-mjh/Android-Guru-Agent-Release/main/version.json
https://raw.githubusercontent.com/AceGuru-mjh/Android-Guru-Agent-Release/main/patches.json
```

App 会按以下优先级自动选择更新通道（v1.4.7+）：

| 优先级 | 通道 | 适用 | 是否安装 |
| --- | --- | --- | --- |
| 1 | **热更新**（`version.json` 的 `hot` 档） | data-only 版本（只改了技能 / MCP 目录）与应急修订 | **零安装，下载后即时生效** |
| 2 | 增量补丁（vcdiff） | 官方渠道安装且签名一致 | 合成新 APK 后系统安装器确认 |
| 3 | 全量包 | 兜底 | 系统安装器确认 |

> **增量更新（v1.4.5+）**：App 下载补丁后在应用内**自动合成新 APK 并拉起安装**
> （内置纯 Kotlin VCDIFF 解码器，无需命令行、无需 xdelta3）；跨任意多个小版本时
> 自动按 patches.json 解析补丁链逐段应用，链体积超过全量包 60% 时才回退全量。
>
> **热更新（v1.4.7+）**：本版只改了数据层（技能 / MCP 目录）时，App 直接出现
> 「热更新 · 免安装」卡片 —— 下载几 MB 的 hot_v*.zip，校验落位后技能与目录
> **即时生效**。全程不碰 APK、不做签名校验 —— **本地构建 / 分叉构建用户从此
> 不再撞签名冲突**。数据层发布后发现问题可**同版本应急重发**（本仓库 hotfix
> 工作流，客户端凭包指纹自动重新应用）。含代码变更的版本仍走增量/全量，且
> App 会比对 `version.json` 的 `signingCertSha256` 与本地签名指纹，不一致时
> **提前警告**（不再下载几百 MB 后才在安装器里报错）。
> 详见 **[HOT_UPDATE.md](./HOT_UPDATE.md)**。
