# guide.json 结构规范（1.0）

使用 UTF-8 JSON。每条攻略一个对象；HTML 数据区存放这些对象的数组。空集合用 `[]`，未知字符串用 `null`；不要用虚构内容填满字段。时间采用带时区 ISO 8601，核查日期可用 `YYYY-MM-DD`。

## 目录命名与单来源入口

- 用户给出的路径是输出父目录。默认每个独立网页/视频在其中建立一个内容概括文件夹，内含自己的 `index.html`、`guide.json` 和已保存资料；HTML 数据区只包含该来源对象 `[guide]`。同一教程的正文分页和该次已采集字幕范围属于同一来源；互不相关的输入链接分别输出。
- 根据实际读到的正文，用“核心主题 + 操作目的”概括文件夹名，通常约8–30个中文字符，英文技术名可保留，不机械凑字数。例如 `Python代码打字练习与专项训练`、`VSCode配置LaTeX论文写作环境`。去掉夸张标题、平台尾缀和无辨识度的口号；必要时保留版本、系统或 `首P`。title 仍保存原题，不能用概括标题替换来源标题。
- 清除控制字符和 Windows 禁用字符 `< > : " / \\ | ? *`，去掉末尾空格及句点；避开 `.`、`..`、CON/PRN/AUX/NUL/COM1–9/LPT1–9 等保留名（含带扩展名的形式），按实际路径长度缩短。简体中文和空格可以保留，不为机器标识改成纯数字/哈希。无法读取正文时不猜主题：有可靠标题时用“标题概括-未采集”，否则用“未采集-站点名-短ID”。
- 已有同名目录时先核对其中来源、采集范围和内容。同源同范围且内容未变，复用已有独立 HTML；不同来源则加简短站点名或短来源ID，仍以主题为前缀。更新另存“主题-更新日期”，同日冲突再加序号。不能只因同名覆盖或把不相关网页合并。
- `guide.id` 是稳定标识，可与易读文件夹名不同，不因概括命名而改写旧 ID。既有同源资料只有旧合并页、缺少独立 HTML 时，可从已保存对象生成新的内容概括文件夹及单来源 HTML，不重复联网、不移动或删除旧资料；在 archive 事件说明复用。已有独立文件夹则保持原路径，用户要求改名或迁移时再处理。
- 输出父目录只作为容器，不自动生成或更新父目录的合并 `index.html`。用户明确要求合集时才额外生成合集，仍须保留各来源入口。下文“归档根”默认指本次单来源文件夹，明确生成合集时则指另设的合集目录；分卷总目录放在相应归档根内。

## 顶层必填字段

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| schema_version | string | 固定 `1.0` |
| id | string | 卷内稳定唯一 ID；可与易读文件夹名不同，不含路径分隔符 |
| title | string | 原标题；不可得则“标题未知” |
| url | string | 输入 URL |
| final_url | string/null | 实际公开页面 URL |
| captured_at | string | 抓取时间，含时区 |
| author | string/null | 作者 |
| published_at | string/null | 原文时间；不把抓取时间冒充发表时间 |
| kind | string | `article` 或 `video` |
| mode | string | `quick` 或 `deep` |
| status | string | `success`、`partial`、`failed` |
| tags | string[] | 标签 |
| applicability | object | `{versions: string[], as_of: string/null, notes: string}` |
| original | object/null | 实际保存的原文与资源，见下文；未保存原文时为 null，摘要不可填入此字段 |
| goal | string/null | 目标 |
| prerequisites | string[] | 前置条件 |
| steps | object[] | 步骤 |
| branches | object[] | 决策树根节点；无分支时空数组 |
| pitfalls | string[] | 坑与注意事项；不确定提醒关联 claims |
| claims | object[] | 关键论断的置信度 |
| sources | object[] | 原始来源、字幕和独立旁证 |
| screenshots | object[] | 深查证截图；快速模式为空数组 |
| events | object[] | 采集、查证、截图、归档的状态和错误 |

## 原文 original

`{filename, format, method, language, completeness, missing, text, images}`。

- `filename`：`原文快照.md`、`字幕全文.md` 或 `转录全文.md`。
- `format`：固定 `markdown`；字幕转为带时码 Markdown，保留原始 VTT/SRT 文件作为额外采集证据（可得时）。
- `method`：`article`、`captions`、`transcription`；不要把转写标成原有字幕。
- `language`：语言或 null。`completeness`：`complete` 或 `partial`。
- `missing`：缺失范围 string[]；完整时为空数组。
- `text`：保存的全文文件内容，包含元数据头和全文；不得仅有摘要、路径或网络链接。HTML 必须内嵌这一字段。
- `images`：`{caption, source_url, data_uri}` 数组。只接受 base64 PNG/JPEG/WebP 数据 URI；图片原始 URL 仅作引用，不能充当离线图片。无法取得时记入 events，必要时降级 partial。

原始快照与 `original.text` 应逐字符一致；使用 SHA-256 核对。HTML 中的正文不能因搜索、筛选或深查证被永久删除。

已读到正文但因复制限制仅保存自行改写的攻略时，使用 `status=partial`、`original=null`，标签注明“摘要版”。collect 的 warning 说明实际读取范围、未保存全文的原因及已保留的定位；步骤可通过 sources 和 original_refs 指向章节或时码。可保存 `攻略.md` 和不含逐字内容的时码索引；不生成 `原文快照.md`、`字幕全文.md` 或 `转录全文.md`，不将摘要称为完整快照。若允许保存部分逐字原文，则 original.completeness=partial 且明确 missing。完全未读到正文与读到后未保存全文是不同状态，前者仍为 failed、无攻略步骤。

## 步骤与分支

步骤：`{id, title, instruction, original_refs, source_refs}`。

- `id`：条目内唯一，供分支和论断引用。
- `instruction`：具体操作；保留关键命令、数量、版本限制，注明作者所述或 Agent 补充。
- `original_refs`：string[]，例如“原文：配置容器一节，第 2 段”或 `00:00:02.000 --> 00:00:06.000`。
- `source_refs`：本条 `sources[].id` 的数组。

分支节点：`{id, label, condition, step_ids, source_refs, children}`。

- `condition`：明确的自然语言条件，例如 `Windows + MSVC`，全文检索必须包含该条件。
- `step_ids`：该路径的步骤 ID 数组，按执行顺序排列。共享步骤可被多条路线引用；不互相串联互斥路径。
- `children`：相同结构的子节点数组；叶子节点为空数组。节点不能形成循环。
- `source_refs`：支持分路的原文来源 ID；条件不明时注明未知，不能猜测。

## 置信度 claims

论断：`{id, text, step_ids, original_refs, confidence, reason, version, checked_at, source_refs, origin}`。

- `confidence`：仅允许 `已核实`、`可能过时`、`存疑`、`无法验证` 四种字符串。
- `reason`：必填；说明证据支持/缺失、冲突、版本风险或主观经验；快速模式明确“未外部核查”。
- `version`：适用版本 string/null；`checked_at`：评估日期。仅有日期不代表联网核查。
- `origin`：`source` 或 `agent`，区分作者观点和 Agent 补充。
- `step_ids`：所标注的步骤，通篇性论断可为空。
- `original_refs`：原文定位，外部补充可为空并通过 source_refs 定位。
- `source_refs`：与论断直接相关的来源 ID。快速模式通常只引用原文且使用“无法验证”，或对原文明确标注的旧版本使用“可能过时”。

深查证的“已核实”必须有至少两个不同独立来源组的 `corroboration`，且 `checked_at` 非空；原帖不充当独立旁证。来源不足则如实降级。冲突与未验证内容保留，不删除。

## 来源 sources

来源：`{id, url, title, role, independence_group, accessed_at, locator}`。

- `role`：`original`、`captions` 或 `corroboration`。
- `independence_group`：按实际作者/组织及证据来源分组；同一组织两页、转载或镜像归同组。未确认独立性时用 null，不能计入两个独立旁证。
- `locator`：章节、段落、字幕时码或支持论断的简短说明。不要大段复制旁证正文。
- `accessed_at`：实际访问时间；仅搜索到但未读的网页不作为已核实证据。

用户已授权采用同文公开版本时，保留原 URL 的 failed 记录；替代页另建条目，url/final_url 对应该替代页的实际请求及最终地址。该条目 sources 中的 original 指实际采集的版本，不代表知乎等首发站已读成功。在标题、快照元数据头和 collect 事件中标明版本关系（转载为“转载版本、未与原文核对”，已确认的作者自发版本据实标注）、原始请求 URL、实际来源及可得版本日期；同主题或仅引用原链接的文章不视为同文。success/partial 仅评估替代页本身的采集完整性；不扩展 schema，不把转载列为独立旁证，也不因页面声明许可而推定其有权授权原文。

## 截图 screenshots

截图：`{id, kind, filename, data_uri, caption, locator, timestamp, source_ref}`。

- `kind`：`paragraph` 或 `frame`。`filename`：相对条目目录，例如 `截图/frame-001.png`。
- `data_uri`：base64 PNG/JPEG/WebP；HTML 同时内嵌，不依赖此路径。
- `locator`：图文段落或视频画面定位；视频 `timestamp` 必填（`HH:MM:SS.mmm`），图文为 null。
- `source_ref`：原文来源 ID。无法获取真实画面时留空数组并写失败事件，不伪造截图。

## 事件 events 与一致性

事件：`{stage, status, message, at}`。

- `stage`：`collect`、`structure`、`verify`、`screenshot`、`archive`。
- `status`：`ok`、`skipped`、`warning`、`failed`。每个非 ok 事件必须带可读原因，在 HTML 中显示。
- 采集通道失败或切换时，在 `message` 中记录工具名称、访问 URL、可得的 HTTP 状态（未知则注明）、具体错误与回退结果；用 `at` 记录该次访问时间。区分站点拒绝、宿主联网限制和原因未确认，不把沙箱错误记为 HTTP 403。回退成功时保留首次失败事件，同时使用成功读取的正文与 final_url；采集状态可为 success，避免将单一工具失败误记为整页失败。不新增必填字段，既有模板直接显示这些事件。
- Agent Reach 的 read 响应使用顶层 items[].text；把本次返回的 channel/backend、error.code 与 meta.attempts 中的失败阶段记入 collect.message，不新增 schema 字段。元数据为 null 时不猜测；正文缺图、代码结构或分页时先记 partial，正常补全后才更新完整性。视频 read 的简介不作为字幕，代理读取服务地址也不作为文章 final_url。
- 快速模式记录 verify/screenshot 的 skipped，说明模式选择；不把有意跳过标成操作失败。
- `failed` 条目：original/goal 为 null，prerequisites/steps/branches/pitfalls/claims/screenshots 为空数组；仅保留输入元数据、已知原始来源和失败事件。
- `partial` 条目：保留实际内容，列出缺失范围；不得声称完整。截图或旁证失败但原文完整时可保持采集 success，同时显示阶段失败。
- 检查所有 ID 引用存在、ID 无重复、路径不越界；快照全文与 original.text 一致；快速模式没有旁证或截图；深查证不足两个来源的论断不能标“已核实”。

## HTML 数据嵌入契约

把数组序列化后写入模板 `script#guidekeep-data`。将实际的 `< > &` 以及 U+2028/U+2029 分别转义成 `\u003c`、`\u003e`、`\u0026`、`\u2028`、`\u2029`；保留合法 JSON，勿对整个 JSON 做 HTML 实体编码。JSON 空数组代表空归档，不含任何示例正文。

模板按纯文本显示来源内容，分支直接列出所引用步骤；搜索范围包含已保存正文、步骤、分支条件、来源、置信度理由与错误，不包含 data_uri 图片编码。匹配可读内容时自动展开对应折叠区及祖先，包括分支引用的步骤。置信度筛选匹配条目中任一论断，不隐藏匹配条目的其他原文；失败条目可用“全部”查看。标签、置信度和搜索组合采用 AND。

## 追加与安全写入

1. 先按上节定位本次来源的文件夹与已有独立入口；正常归档不把输出父目录的旧合并页当成待更新入口。需要复用旧合并资料或用户要求更新合集时，再读取其数据区；分卷时读取静态总目录所链接的各卷数据。保留所有原有对象，包括只有 HTML 内嵌数据、旁边文件已缺失的记录。待复用或替换的入口不能解析、ID 重复或卷链接越界时，停止覆盖并报告问题，不当作空归档重建。新建来源文件夹须位于输出父目录内，卷和资料路径须位于该归档根内，拒绝绝对路径和含 `..` 的相对路径；条目 ID 还须拒绝 `.`、路径分隔符及平台不支持的文件名。
2. 根据实际来源与采集范围识别重跑。只移除已确认的跟踪参数；保留影响正文、分页、视频分P、语言和版本的参数。相同来源、范围及内容未变时复用已有条目，不重复加入；用户要求更新或内容变化时，创建带日期/唯一后缀的新版本 ID 和目录，保留旧版并在 collect 事件记录关系。此前 failed 记录也保留，后续成功另建条目。无法确认是否同一范围时不自动合并。
3. 在输出父目录内的独立暂存区生成新的内容概括文件夹及其单来源 HTML；明确更新已有归档/合集时，先记录将替换文件的校验值，再暂存对应完整结果，不原地改旧正文。验证 JSON/ID 引用、原资料保留、离线交互及实际字节数；默认单来源页还须核对数据数组长度为1、来源匹配且 HTML 位于内容概括文件夹内。无法完成的检查明确报告。提交前备份待替换入口/受影响卷并再次核对校验值；发现其他进程改动就停止，不覆盖并发更新。
4. 将已验证的新条目放入不存在的目标目录，再用同一文件系统的替换操作提交 HTML，根入口最后替换。分卷重排须使用新的卷目录，旧总入口仍能引用旧卷；入口切换前不移动或删除旧卷。失败时保留备份、旧有效入口及暂存资料并说明恢复路径，不宣称已完成。多文件写入并非整体原子事务；最后打开实际交付入口核对条目数、检索与大小。不得为清理暂存而删除原有条目。

## 文件大小与跨卷入口

每个 `index.html` 以 UTF-8 写入后，用系统命令读取实际文件字节数；PowerShell 使用 `(Get-Item -LiteralPath './index.html').Length`，POSIX 使用 `wc -c < ./index.html`，均在对应文件目录执行。每次重写后重测，在交付信息中记录路径、字节数与 MiB（1 MiB = 1,048,576 字节）。模板的 Blob 估算值不作为分卷判据，也不写入 guide.json 充当实测值。

实测 ≥20 MiB（20,971,520 字节）提示分卷，>50 MiB（52,428,800 字节）必须按主题或年份分卷；逐卷重测至各卷 ≤50 MiB。每卷数据区只内嵌本卷条目，保留全文和原始资料。单条仍超限且无法保留全文并满足上限时，明确报告未完成分卷，不能仅以单独成卷视为达标。

分卷时，归档根目录 `index.html` 为静态总目录页，逐卷列出主题/年份、条目数和指向子目录中 `index.html` 的相对链接，并显示“搜索不跨卷”。各卷 `index.html` 保持自包含与卷内全文索引；总目录不合并全文索引、不依赖网络，离线核对链接与条目数。单卷时内容概括文件夹内的 `index.html` 直接为攻略页，不生成额外总目录；只有用户明确要求合集时才另设合集根。跨卷信息属于归档级交付信息，不新增 guide.json 字段。
