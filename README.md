# Guidekeep — 把会消失的帖子，变成找得回的攻略

*Because the post might not be there tomorrow.*

Guidekeep（存照）是一份 Agent Skill：给 Agent 一个公开网页或视频链接，由它在线读取资料、整理操作步骤和条件路线，再保存成可搜索、可离线打开的 HTML。

项目名为 `guidekeep`，调用名为 `$guide-keep`。项目由指令、参考规范和 HTML 模板组成，采集与文件写入由宿主 Agent 执行；没有独立 CLI、服务端或构建命令。

## 解决什么问题

收藏链接不能保证以后还能看到正文；教程里的图片、代码、字幕可能一起消失。即使原帖还在，按系统、版本或设备区分的操作路线，也常需要重新翻找。

Guidekeep 在允许保存的范围内保留来源资料，再提炼攻略。每个来源放进按内容概括命名的文件夹，方便在文件系统中直接辨认；HTML 内可检索步骤、原文、条件和失败原因。过时、冲突或暂时无法验证的内容保留并标注，置信度表示证据状态，不保证内容正确。

## 主要功能

| 功能 | 实际行为 |
| --- | --- |
| 网页归档 | 保存可合法保留的正文、代码、表格和可取得的原图；不完整时标记缺失 |
| 视频字幕 | 优先读取已有字幕并保留时码；缺字幕时仅在已有转写能力和授权允许的情况下尝试转写 |
| 攻略与条件路线 | 整理目标、前置条件、步骤、注意事项；按系统、版本、环境等条件选择路径，并保留来源定位 |
| 证据标注 | 使用“已核实、可能过时、存疑、无法验证”四级及理由，不因存疑删除已保存内容 |
| 独立来源文件夹 | 用“主题 + 用途”概括命名，每个来源有自己的 `index.html`；重名区分来源，更新另存版本 |
| 单文件阅读 | HTML 内嵌已保存资料与图片，可通过 `file://` 打开，无需服务器、CDN或相邻 JSON/Markdown |
| 搜索与筛选 | 文本搜索、标签与置信度组合筛选、条件折叠；命中折叠内容时自动展开，图片编码不参与搜索 |
| 失败记录 | 区分站点拒绝、工具连接故障和采集不完整，不把工具成功响应当作完整归档证明 |

默认每个独立链接分别输出；只有明确要求合集时才额外生成合集。现有旧目录和合并页不会自动迁移。

## 安装

### 环境要求

- 能加载本地技能、读取网页并写入文件的宿主 Agent；下面的加载方法针对 Codex。
- 至少一种实际可用且获准的在线读取能力。Agent Reach 和本项目的 MCP 配置均为可选通道，复制技能不会自动安装或授权它们。
- 阅读 HTML 需要支持 JavaScript 的浏览器。单文件离线阅读不代表采集时不需要联网。

技能本身不要求安装 Python、Node.js 或 npm/pip 包；开发测试使用的工具不属于运行依赖。

### 方法一：在本项目中使用

将项目文件放到本机，在 Codex 中打开 `guidekeep` 项目目录。项目已经包含 `.agents/skills/guide-keep/`，无需再次复制或执行安装脚本。

Codex 的本地技能目录包括项目 `.agents/skills` 和个人 `$HOME/.agents/skills`。技能未显示时检查目录和名称，必要时重启宿主；加载规则见 [OpenAI 官方技能文档](https://learn.chatgpt.com/docs/build-skills)。打开父目录不会向下自动加载本项目子目录中的技能和 MCP 配置。

### 方法二：复制到另一个项目

在 **Guidekeep 项目根目录**打开 Windows PowerShell，修改第一行的目标项目路径后执行：

```powershell
$targetProject = 'D:\YourProject' # 替换为你要使用技能的项目目录
$source = (Resolve-Path -LiteralPath '.\.agents\skills\guide-keep').Path
$skillsDir = Join-Path $targetProject '.agents\skills'
$destination = Join-Path $skillsDir 'guide-keep'
if (Test-Path -LiteralPath $destination) {
    throw "目标已存在，请先检查版本并备份：$destination"
}
New-Item -ItemType Directory -Path $skillsDir -Force | Out-Null
Copy-Item -LiteralPath $source -Destination $destination -Recurse -ErrorAction Stop
Test-Path -LiteralPath (Join-Path $destination 'SKILL.md')
```

最后应返回 `True`。请复制整个技能目录，保留 `LICENSE`、`references/`、`assets/` 和 `agents/`，不要只复制 `SKILL.md`。技能目录内的许可证与项目根目录一致，来源说明也随目录分发。该命令不会覆盖已存在的同名技能，也不会复制项目根目录的 MCP 配置。随后在 Codex 中打开目标项目。

如需个人范围使用，也可手动将完整 `guide-keep` 文件夹复制到 `$HOME/.agents/skills/guide-keep`。避免在同一宿主范围内同时保留多个同名副本，以免选错版本。

上述复制命令已在本机 Windows PowerShell 5.1 中使用临时目标项目验证：完整文件复制、内容哈希一致、重复安装停止。技能自动发现尚未用全新 Agent 会话重新验收，详见 [测试记录](TESTING.md)。

### 配置在线读取能力

优先使用宿主已有且能实际调用的网页工具。工具缺失或读取不完整、且没有站点访问限制时，可使用已安装且获准的 Agent Reach；它不依赖 Edge 扩展或当前对话的 MCP 工具注入。

如果本机已经安装 Agent Reach，可以在 PowerShell 检查其版本和声明的渠道：

```powershell
agent-reach version
agent-reach channels
```

这两条命令已在 Agent Reach 1.6.1 环境验证；其他版本的动作、参数以当地工具为准。它们只检查工具，不证明目标网页或字幕可读。若出现 `uv trampoline` 脚本路径或uv缓存访问错误，先通过宿主正常许可流程对照复测：受限环境失败、正常授权环境成功时，使用获准环境继续读取，无需重装；两者均失败才诊断真实安装故障。不能仅凭PATH中存在命令或历史成功判定当前可用。安装和平台登录属于 Agent Reach 自己的配置，Guidekeep 不自动执行；调用约定见 [Agent Reach 参考](.agents/skills/guide-keep/references/agent-reach-online.md)。

本项目另含可选的 [.codex/config.toml](.codex/config.toml)，配置 `guidekeep_online`，地址为 `https://agent.tinyfish.ai/mcp`，仅允许 `fetch_content`。使用前需由宿主加载受信任项目配置、完成所需授权并确认实际工具可调用；复制项目不会复制认证状态。该配置不包含其他自动化工具，服务可用性、额度和价格以服务方为准。入口缺失时的已有授权连接调用路径见 [Codex 在线读取参考](.agents/skills/guide-keep/references/codex-online.md)。

## 使用方法

以下是**发给 Agent 的聊天提示词**，不是 PowerShell 命令。项目没有 `guidekeep run` 或类似可执行入口。

### 快速归档

```text
用 $guide-keep 在线读取 https://rust-lang.org/tools/install/，整理为可搜索的攻略，输出到 D:\guides\。
```

默认快速模式：读取来源及正文分页、图片或字幕，整理攻略并标注证据状态，不搜索独立旁证、不截图。没有指定输出位置时，使用当前工作目录下的 `guidekeep-archive/` 作为父目录；请将实际归档放在本项目之外。

### 视频字幕

```text
用 $guide-keep 将 https://www.bilibili.com/video/BV1fNVd6WEhY/ 整理成攻略，使用已安装的 Agent Reach 读取字幕，输出到 D:\guides\。保留时码，并明确说明缺失信息。
```

本机已在用户本人登录后读到该样例首P的41段字幕。它不是“任意B站视频均可读取”的承诺；视频简介不作为字幕，多P范围、语言和字幕类型仅记录实际取得的信息。无字幕且不能转写时，只保存失败元数据，不猜测攻略内容。

### 深查证

```text
用 $guide-keep --deep 归档 https://developer.mozilla.org/en-US/blog/getting-started-with-css-container-queries/，交叉查证关键步骤并保存关键段落截图，输出到 D:\guides\。
```

只有本次请求明确包含 `--deep` 才进入深查证；它是对话触发词，不是项目 CLI 参数。该模式为关键步骤寻找至少两个独立旁证，并尝试保存段落截图或视频关键帧。旁证不足或截图能力不可用时记录限制，不保证每个论断都能标为“已核实”。

## 输入输出示例

输入是公开来源 URL，可附输出父目录、模式和已授权的读取通道。文件夹名依据实际正文概括，不直接照抄夸张标题，也不只使用文章ID或日期。

例如，分别输入前面的 Rust 页面和B站视频后，输出布局可以是：

```text
D:\guides\
├── Rust安装与系统架构选择\
│   ├── index.html
│   ├── guide.json
│   └── 原文快照.md
└── Python代码打字练习与专项训练\
    ├── index.html
    ├── guide.json
    ├── 攻略.md
    └── 时码索引.json
```

目录名是内容概括示例，不保证每次逐字相同。每个 `index.html` 默认只含对应来源；`guide.json` 保存元数据、步骤、分支、置信度、来源和阶段事件。稳定 `id` 可以与文件夹名不同。命名、重复处理和完整数据契约见 [格式规范](.agents/skills/guide-keep/references/guide-format.md)。

| 采集结果 | 文件与页面内容 |
| --- | --- |
| 可完整保存的图文 | `原文快照.md`、`guide.json`、内嵌正文及已取得原图的 HTML |
| 可保存的字幕或转录 | `字幕全文.md` 或 `转录全文.md`，包含实际时码；机器转写明确标注 |
| 已读正文但仅允许摘要 | 自行改写的攻略和必要定位；`status=partial`、`original=null`，不生成伪全文文件 |
| 完全未读到正文 | `guide.json` 和显示失败原因的 HTML，无原文快照和虚构步骤 |
| 深查证实际取得截图 | 另有 `截图/`，图片同时内嵌 HTML；快速模式不生成截图 |

B站样例的已交付版本为摘要版：9条要点及41段时码索引，没有保存逐字字幕全文；缺少的工具名称、下载入口、脚本参数和快捷键明确标注。并非所有视频都具有这些文件或相同条目数。

打开对应文件夹中的 `index.html` 即可阅读。搜索原文、步骤或条件时会展开命中区域，标签与置信度可组合过滤。HTML 内嵌已保存资料，单独复制这一文件仍可离线阅读；来源链接需主动联网打开，HTML 不包含视频文件。

每次交付应包含实际文件夹和 HTML 路径、采集状态、模式、缺失范围、验证结果及系统实测字节数。≥20 MiB 提示分卷，>50 MiB 必须分卷并重测；页面上的 Blob 估算不作为判据。分卷时在相应来源文件夹或明确要求的合集目录内生成静态总目录，列出各卷主题/年份、条目数与入口，并提示“搜索不跨卷”。单条资料无法保留完整内容并满足上限时报告未完成分卷，不宣称达标。

## 已验证范围与限制

- 已有真实网页、字幕和深查证图文测试；内容命名的单来源文件夹已用保存资料生成并进行离线验证。最近的两份独立样例均可检索，转载页45张内嵌图片可离线解码。
- 知乎样例原站仍返回403；不保证所有网站可访问。遇登录墙、付费墙、验证码或明确拒绝时停止受限尝试，不规避限制。浏览器宿主故障与网站403分别记录。
- 使用同文公开转载须获用户同意，并标明实际来源及“转载版本、未与原文核对”；不能把转载可读称为原站已恢复。用户要求在线采集时，不用手工导入冒充完成。
- 内容保存受来源许可和工具复制限制约束；摘要版不能代替全文备份，采集到字幕也不意味着可以无限制复制全文。
- 新 Agent 会话的自动路由、多P完整采集、真实音频转写、视频关键帧、大体积分卷压力及并发中断恢复尚未完整验收。
- 去重、暂存、备份和更新另存版本是宿主遵循的指令规则，没有独立构建器提供整体原子事务保证；新指令不会自动更新旧 HTML。
- 不自动安装采集器、升级平台工具、代替用户登录或启用收费转录。宿主模型和外部服务仍可能产生自身费用。

验证记录及每项边界见 [TESTING.md](TESTING.md)。官方 `quick_validate.py` 已验证技能格式，但它不是项目自带文件，也不能证明某个网站在线可读。

## 项目结构

```text
guidekeep/
├── README.md
├── LICENSE
├── TESTING.md
├── .codex/config.toml                 # 可选在线连接
└── .agents/skills/guide-keep/
    ├── LICENSE                       # 随技能复制的MIT许可证
    ├── SKILL.md                      # 工作流程与约束
    ├── agents/openai.yaml            # 技能显示信息
    ├── assets/archive.html           # 自包含阅读模板
    └── references/
        ├── guide-format.md           # 数据、命名与安全写入
        ├── agent-reach-online.md     # 已安装 Agent Reach 的读取路径
        ├── bilibili-login.md         # 特定版本登录故障与字幕验收
        ├── codex-online.md           # 已授权 MCP 的按需调用路径
        └── third-party-notices.md    # 示例来源及外部许可边界
```

## 许可

项目使用 [MIT License](LICENSE)。该许可仅覆盖本项目文件，不授予第三方网页、图片、字幕或视频的再分发权。

代码示例来源、外部工具许可证及第三方资料的分发边界见 [第三方与来源说明](.agents/skills/guide-keep/references/third-party-notices.md)。当前项目不捆绑外部库、采集正文或图片；宿主另行安装的工具保留各自许可证，不能一并按 MIT 再许可。本文的 Windows 绝对路径均为可替换的使用示例，不是运行依赖或开发者个人目录。
