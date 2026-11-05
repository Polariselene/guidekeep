# 代码来源与第三方资料分发边界

核对日期：2026-11-05。此文件随完整技能目录一起复制，以保留来源说明；无需在普通采集时加载。

## 本项目编写的文件

`assets/archive.html` 的阅读界面、搜索和数据嵌入实现，以及 `codex-online.md` 的标准库客户端、`bilibili-login.md` 的兼容入口/调用器/合成测试，均在本项目开发过程中编写。协议客户端依据 [OpenAI App Server 文档](https://learn.chatgpt.com/docs/app-server)，B站适配依据其SDK接口及 [Agent Reach 上游问题说明](https://github.com/Panniantong/Agent-Reach/pull/866)。这些文件没有捆绑上游库实现或外部网页脚本。

项目自有文件适用MIT许可证，版权人为2026 Siyi Qi；项目根目录及技能目录内的LICENSE文本保持一致，复制技能时须保留目录内的许可证。引用接口名、公开URL或调用外部程序，不表示拥有外部程序、服务或返回内容的版权。

## 外部工具与库

以下为适配示例涉及的外部项目，均由宿主另行安装，不包含在本项目分发文件中：

| 外部项目 | 核对的许可标识 | 依据与边界 |
| --- | --- | --- |
| [Agent Reach](https://github.com/Panniantong/Agent-Reach) | MIT | 许可见其仓库LICENSE；Guidekeep不分发其安装环境 |
| [bilibili-cli](https://github.com/jackwener/bilibili-cli) 0.6.2 | Apache-2.0 | 已安装版本包元数据；仅调用其已有接口 |
| [bilibili-api-python](https://github.com/Nemo2011/bilibili-api) 17.4.2 | GPL-3.0-or-later | 已安装版本包元数据；兼容示例引用其SDK，不能将SDK重新标为MIT |
| [qrcode](https://github.com/lincolnloop/python-qrcode) 8.2 | BSD | 已安装版本包元数据及LICENSE；仅在已有B站工具环境中调用 |

该表不是上述工具的完整传递依赖清单。各版本原始LICENSE及版权声明仍适用；如将来复制、修改、打包或随安装器分发外部实现，须另行核对相应版本的许可和组合分发要求，保留必要版权/许可/NOTICE并履行适用义务。本表不将外部代码统一重新许可为MIT，也不承诺尚未审查的组合发行符合全部许可。

## 图片、数据、文章与字幕

当前项目只含空归档模板、指令、公开来源链接与测试结果描述，不分发测试采集的文章全文、字幕全文、图片、音视频、原始响应或用户认证数据。测试和用户归档保存在项目之外，项目MIT许可证不覆盖这些资料。

以后拟加入公开项目的第三方素材，须先记录实际来源URL、作者、明确的许可或授权依据、适用范围、署名要求及是否允许改编/再分发。无法确认时不得随项目发布全文、图片或数据；可保留来源链接和允许范围内的自行改写说明。原作者署名、转载标记、网页公开可访问、站点主题的MIT许可证，都不能代替素材本身的再分发授权。

`.gitignore` 用于降低误加入采集产物和凭据的风险，不能授予素材权利，也不能清除文档正文或既有历史中已经包含的内容。
