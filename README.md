# Android-Guru-Agent-Release

[Android-Guru-Agent](https://github.com/AceGuru-mjh/Android-Guru-Agent) 的**发布仓库** —— 只存放构建产物，不含源码。

## 这里有什么

| 内容 | 说明 |
| --- | --- |
| **[Releases](https://github.com/AceGuru-mjh/Android-Guru-Agent-Release/releases)** | 每个版本的 APK（arm64 / universal）与增量补丁 |
| **[version.json](./version.json)** | 最新版本清单（版本号 / 下载地址 / SHA-256），供 App 内检查更新使用 |
| **[patches.json](./patches.json)** | 历史补丁全量索引（from→to 链），App 跨版本链式增量更新的数据源 |

## 发布流程（全自动）

```
开发仓库 push tag v*.*.*
        │
        ▼
开发仓库 CI 构建 APK（universal + arm64）
        │
        ├─ 生成 version.json（含 SHA-256 校验）
        ├─ 与上一版本生成 xdelta3 增量补丁（LZMA 二级压缩）
        ├─ 累积维护 patches.json（历史补丁全量索引）
        ▼
通过跨仓库 PAT 发布到本仓库
        │
        ├─ 创建 Release + 上传 APK / 补丁 / version.json
        └─ 提交 version.json + patches.json 到 main 分支（供 App 内更新检查）
```

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

> **增量更新（v1.4.5+）**：App 下载补丁后在应用内**自动合成新 APK 并直接拉起安装**
> （内置纯 Kotlin VCDIFF 解码器，无需命令行、无需 xdelta3）；跨任意多个小版本时
> 自动按 patches.json 解析补丁链逐段应用，链体积超过全量包 60% 时才回退全量。
