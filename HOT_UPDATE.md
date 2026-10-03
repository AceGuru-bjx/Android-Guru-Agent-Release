# 热更新通道（hot update channel）规范与发布手册

> 适用版本：App v1.4.7+ / 开发仓库 PR（`feat(update): 热更新体系 v1`）合入后。
> 完整设计文档见开发仓库 `docs/hot-update-pipeline.md`，本文是**发布仓库侧**
> 的数据契约、校验门禁说明与应急热修操作手册。

## 本仓库在更新体系中的角色

Android-Guru-Agent-Release 是**补丁发布主体** —— 全部更新资产与两份清单
都在本仓库名下：

| 资产 / 文件 | 位置 | 写入方 |
| --- | --- | --- |
| `version.json`（版本清单） | main 分支根（客户端检查更新的固定读取点） | 开发仓库 release.yml CI / 本仓库 hotfix 工作流 |
| `patches.json`（补丁全量索引） | main 分支根（跨版本链式增量底座） | 开发仓库 release.yml CI |
| APK（arm64 / universal） | 各版本 GitHub Release | 开发仓库 release.yml CI |
| vcdiff 增量补丁 | 各版本 GitHub Release | 开发仓库 release.yml CI |
| **热更包（apex-hot-v1）** | 各版本 GitHub Release | 开发仓库 CI（data-only 版本）/ **本仓库 hotfix 工作流（应急重发）** |
| **清单校验门禁** | `.github/workflows/validate-manifests.yml` | 本仓库自动运行 |
| **应急热修发布** | `.github/workflows/hotfix.yml` | 本仓库手动触发（workflow_dispatch） |

## 为什么需要热更新

现有增量链路（vcdiff 补丁 → App 内合成新 APK → 系统安装器）的终点是
**覆盖安装**。而发布 APK 以 debug 密钥签名（密钥是构建机器本地的），
所以：

```
本地构建 / 分叉构建（fork 自编译）的已装 APK
    ≠ CI 产物签名
    → 覆盖安装 = INSTALL_FAILED_UPDATE_INCOMPATIBLE（签名冲突）
```

热更新通道只携带**数据层**内容（技能 / MCP 精选目录），App 下载后校验
→ 原子落位 → 即时生效 —— 全程不碰 APK，**签名冲突在这条通道上从根上
不存在**，任何签名来源的安装（官方 / 本地构建 / 分叉构建）都可用。

## 通道怎么工作

```
开发仓库 PR 合并 → CI 发布
        │
        ├─ compare(上一版 commit → 本版 commit) 变更集
        │
        ├─ 变更全部在热载白名单 ──── 出热更包 hot_v*.zip + version.json 带 hot 档
        │   （白名单：app/src/main/assets/skills/*.json
        │         与 app/src/main/assets/mcp_catalog/*.json）
        │
        └─ 含任何白名单外变更 ──── 不出热更包（本版必须重装 APK）
            （version.json 仍带 signingCertSha256 → App 签名预检提前告知）

App「关于 → 检查更新」
        │
        ├─ effective = max(包 versionCode, 已应用热更目标)
        ├─ 远端更新 且 hot 档过门 → 主推「立即热更新」（零安装，即时生效）
        └─ 无 hot 档 → 原增量（vcdiff）/ 全量路径（含签名预检警告）
```

## version.json 新增字段（v1.4.7 起，CI 自动生成）

```json
{
  "versionName": "1.4.7",
  "versionCode": 49,
  "download": { "arm64": { "…": "…" }, "universal": { "…": "…" } },
  "patch": { "…": "…" },

  "hot": {
    "baseVersionCode": 49,
    "targetVersionCode": 49,
    "targetVersionName": "1.4.7",
    "url": "https://github.com/AceGuru-mjh/Android-Guru-Agent-Release/releases/download/v1.4.7/hot_v1.4.7.zip",
    "sizeBytes": 1048576,
    "sha256": "…"
  },
  "commitSha": "<本版完整 commit SHA（40 hex）>",
  "signingCertSha256": "<发布 APK 签名证书 SHA-256 指纹（64 hex）>"
}
```

| 字段 | 语义 | 缺省 |
| --- | --- | --- |
| `hot` | 热更包档；data-only 版本由 CI 写入，应急修订由本仓库 hotfix 工作流重写 | `null` |
| `hot.baseVersionCode` | 热更适用的最低数据层 versionCode —— **恒为热通道能力地板 49**（v1.4.7，首个热通道客户端；见下节「基底语义」） | — |
| `hot.targetVersionCode` | 热更目标 versionCode（**== 本清单 `versionCode`**，App 侧与校验门禁双向强校验） | — |
| `hot.sha256` | 包 ZIP 指纹 —— 整包校验 + **应急重发判重基准**（见「应急热修」） | — |
| `commitSha` | 本版 commit —— 下一版 data-only 判定的比对基准 + **应急热修白名单比对的 base** | `null`（v1.4.6 及更早清单） |
| `signingCertSha256` | 发布 APK 签名证书指纹 —— App 端签名预检（本地指纹不一致 = 覆盖安装必报签名冲突，提前警告） | `null`（提取失败） |

老客户端（≤ v1.4.6）读新清单：`ignoreUnknownKeys` 自动忽略新字段，
行为完全不变；新客户端读老清单：`hot` 档缺省 `null`，走原增量/全量路径。

### 基底语义：为什么 baseVersionCode 恒为 49

热更包是**累积快照**（skills + 目录全量、自包含），对任何热通道客户端
（v1.4.7+）都可以安全应用，基底不需要随版本收缩。而签名冲突用户 ——
本地构建 / 分叉构建，本通道的主要服务对象 —— **永远无法重装 APK**：
如果基底随版本推进（例如取上一版 versionCode），他们只要跳过一次发版
就会被永久锁在数据通道之外。地板以下的老客户端本就不认识 `hot` 字段，
不受影响。

## 热更包格式（apex-hot-v1）

`hot_v{versionName}.zip`（应急修订为 `hot_v{versionName}.r{N}.zip`），
布局与 APK 热载路径同构：

```
hot_v1.4.7.zip
├── hotmanifest.json          ← 包内清单（schema / target / 逐文件 SHA-256）
├── skills/*.json             ← 技能清单快照（与 assets/skills 同构）
└── mcp_catalog/*.json        ← MCP 精选目录快照（分类文件）
```

`hotmanifest.json`：

```json
{
  "schema": "apex-hot-v1",
  "targetVersionCode": 49,
  "targetVersionName": "1.4.7",
  "entries": [
    { "path": "skills/api-design.json", "sha256": "…" },
    { "path": "mcp_catalog/browser.json", "sha256": "…" }
  ]
}
```

安全链：ZIP 整包 SHA-256（对 version.json `hot` 档）→ 包内清单 schema
校验 + 路径合法性（拒绝绝对路径 / `..` / 反斜杠）→ **逐文件 SHA-256
复核**（对 hotmanifest entries）→ 解压走 SafeZipExtractor（zip-slip
canonical 校验 + zip bomb 三重上限）→ 原子落位（tmp + renameTo，失败
回滚）。

## 版本口径（阅读 version.json 时的关键语义）

- `packageVersionCode`：已装 APK 的 versionCode（二进制层）；
- `appliedTargetVersionCode`：已应用热更的目标 versionCode（数据层）；
- **effective = max(两者)** —— App 更新检查的比对口径：
  - 数据层已热更到 N 的设备，对远端 N 版判「已是最新」；
  - 远端 N+1 无 hot 档（含代码变更）时照常推增量/全量；
- APK 重装追平（`packageVersionCode >= appliedTarget`）→ 热更 overlay
  自动退役（新 APK 的 assets 已携带 ≥ 热更内容）。

## 应急热修（hotfix 工作流操作手册）

**场景**：已发布版本的数据层（某个技能清单 / 目录条目）出了坏内容，
需要**立刻**修复 —— 走开发仓库发版要完整构建（约 40 分钟），且对签名
冲突用户毫无意义。本仓库的 `Emergency hot update` 工作流可独立完成
数据层应急重发（约 2 分钟）。

**操作步骤**：

1. 在开发仓库把修复（只能改 `app/src/main/assets/skills/*.json` 或
   `app/src/main/assets/mcp_catalog/*.json`）合入 main —— **不能携带
   任何代码变更**，含代码的修复必须走正常发版；
2. 本仓库 Actions → **Emergency hot update** → Run workflow：
   - `dev_ref`：携带修复的 ref（默认 `main`，也可填分支 / tag / SHA）；
   - `note`：修复说明（写入提交信息与 Run Summary，供溯源）；
3. 工作流自动执行：
   - **白名单守卫**：compare（本版 `commitSha` → `dev_ref`）的变更集
     必须全部落在热载白名单（与开发仓库 release.yml 同一把尺）；ref
     必须是本版 commit 的后代；compare 截断（>300 文件）保守拒绝；
   - 按 apex-hot-v1 打包数据层快照 → 挂到**当前版本的既有 Release**
     （`hot_v{版本}.r{N}.zip`，N 自动递增）；
   - 重写 version.json 的 `hot` 档（**targetVersionCode 不变 + 新指纹**，
     其余字段一概不动）→ 提交 main → validate-manifests 门禁自动复核。

**客户端传播机制（同版本重发语义）**：v1.4.7+ 检查更新时凭**包指纹**
判重 —— 同 target + 新指纹 → 关于页出现热更卡片，一键重新应用（零安装，
即时生效）；同指纹 → 幂等跳过；旧 target → 永不回退。已应用过旧修订的
设备同样自动收到新修订。

**前置条件**：本版 version.json 必须带 `commitSha`（v1.4.7 起由开发仓库
CI 注入）—— 它是白名单比对的 base。v1.4.6 及更早版本无法走应急通道，
请升级到 v1.4.7+ 后再使用。

**回滚**：把 `dev_ref` 指向修复前的 commit 重跑一次 —— 快照内容回到
旧数据，指纹变化触发全体客户端重新应用。

## 清单校验门禁（validate-manifests）

本仓库 main 上的两份清单是全体客户端的更新入口，**任何形状错误都会同时
打挂所有客户端的检查更新**。`validate-manifests.yml` 在以下时机自动校验：

- push 到 main（触碰 version.json / patches.json 时）—— 兜住 CI 自动提交
  与 hotfix 工作流提交；
- 任何触碰这两文件的 PR —— 人工手改（紧急止血 / 修正）必须过校验再合并。

校验内容（纯结构判定，零网络零 Token）：

| 对象 | 检查 |
| --- | --- |
| version.json | 必填字段 / 类型 / URL 必须指向本仓库 Release 资产 / sha256 为 64 位 hex / tag↔versionName 自洽 / **hot.targetVersionCode == versionCode** / base ≤ target |
| patches.json | 条目结构 / variant 合法 / fromTag ≠ toTag / URL 挂在 toTag 名下 / 无重复 (variant, fromTag, toTag) |
| 跨文件 | patches.latestTag == version.json.tag |
| 单调性 | push 到 main 时 versionCode 相对上一提交**不得回退**（回退会让已升级客户端收到降级误报） |

## 自举时间线

| 版本 | commitSha | hot 档 | 说明 |
| --- | --- | --- | --- |
| ≤ v1.4.6.x | 无 | 无 | 旧清单（热更体系之前） |
| v1.4.7（首个新 CI 版本） | ✅ | ❌ | 携带 commitSha；上一版清单无 commitSha → 无比对基准，不出热更包 |
| v1.4.7 之后首个 data-only 版本 | ✅ | ✅ | **热更通道正式生效**（此后应急热修通道亦可用） |

## FAQ

**Q：我是 fork 用户（自己编译的 APK），补丁更新装不上怎么办？**
A：v1.4.7 起，data-only 版本会直接出现「热更新 · 免安装」卡片 —— 点它，
下载几 MB 即时生效，不用装 APK。含代码变更的版本会提前显示签名警告：
请卸载重装完整包，或自己从源码构建（`assembleRelease`）。

**Q：为什么有的版本没有热更包？**
A：该版本改了代码 / 资源 / manifest —— Android 不允许 App 无 root 替换
自身 APK，这类版本必须重装。版本 Release 页会写明；App 端也有签名预检
提前告知。

**Q：热更包会覆盖我的技能设置吗？**
A：不会。技能走版本化幂等安装（`installBundled`）：同版本跳过、新版
本升级且**保持用户启停状态**。

**Q：热更内容什么时候被清掉？**
A：重装 APK（versionCode ≥ 已应用热更目标）后，overlay 自动退役 ——
新 APK 已随包携带同样的（或更新的）数据。

**Q：热更包与 vcdiff 补丁是什么关系？**
A：互补关系，按优先级排序：**热更（零安装）> vcdiff 增量（合成新 APK
后系统安装器确认）> 全量包**。热更包只放本仓库 Release（与 vcdiff 补丁
同策略，开发仓库镜像只收 APK + SHA 清单 + version.json）。

**Q：应急热修会不会让版本号混乱？**
A：不会。应急修订**不是新版本**：versionCode / versionName / tag / APK
下载档全部不动，只有 `hot` 档指向新包（`targetVersionName` 形如
`1.4.7-r2`，仅展示用）。下一个正常发版的 data-only 判定与应急修订
互不干扰（比对基准始终是 `commitSha` 指向的发布 commit）。

**Q：我可以手改 version.json 吗？**
A：可以但必须走 PR —— validate-manifests 门禁会做完整结构校验；直接
push 到 main 同样会被门禁兜底检查，校验失败 = 红叉可见。热更档的
`targetVersionCode` 必须与 `versionCode` 一致，`url` 必须指向本仓库
Release 资产，这两条是客户端硬校验。
