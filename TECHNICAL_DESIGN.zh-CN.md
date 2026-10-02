# 语伴开源项目技术方案草案

核查日期：2026-10-02。组件、字段、工具和算法均为设计建议，尚未实现或端到端验证。不是已经跑通的项目。

## 1. 目标与必须修正的前提

目标：你用中文描述当前场景，ChatGPT 用简单英语回应，按需配图或语音互动；自己的后端保存学习证据、词表、报告和计划。模型分析由用户在 ChatGPT 会话内发起，不默认接付费 API。

需要先修正前面讨论中的确定性说法：

- **Plus/Pro 写回不能保证。** 开发者指南写 Plus/Pro 等可用读写 MCP；帮助中心写 full MCP 仅 Business/Enterprise/Edu，Pro 仅 read/fetch。两份官方页面矛盾，必须用实际账号验证。[S1][S2]
- **自托管不是全部政策豁免。** 不提交目录就不走公开目录审核，但仍需遵守条款与隐私要求。目录指南禁止拉取、重建或推断完整聊天记录；没有找到私人自托管可自动重建全部聊天的明确许可。v1 只处理主动提交的学习片段。[S4]
- **本地 Docker 不能直接供 ChatGPT 访问。** 需要它可访问的 HTTPS MCP 服务，或合资格的 Secure MCP Tunnel。Tunnel 另需 Platform 权限、组织/工作区关联及运行凭证，费用和本用户资格未确认。[S2][S5]
- **Voice 联动待实测。** Apps 帮助页写 Voice 不支持 apps，新版 Voice 页写 Live 能使用可用 plugins/apps。不能保证自定义 MCP 语音通路，更不能保证语音转写或图像自动回流。[S7][S8]

订阅不能当作后端 API 额度。可设计为“用户主动要求 ChatGPT 分析，再保存结果”，但不能承诺无人值守每日模型分析。订阅限额仍适用，托管等可能收费，不能宣传整个系统零成本。

## 2. v1 范围与验收

闭环：文本练习 > 主动保存片段 > ChatGPT 生成报告 > 保存报告 > 次日复用计划。

中文母语者从 pre-A1 到 A1 起步。允许中文输入，但逐步邀请短英语回答；中文聊天不算英语产出证据。Voice、生图可在 ChatGPT 中尝试，文本是必测基线。

验收：真实账号读写探针通过，或手动 JSON 降级可用；写入有成功回执；重复操作不重复入库；报告引用真实证据；数据不足显示未知；导入可预览、去重、删除；备份恢复与权限隔离可测试。

## 3. 系统组件与建议选型

| 组件 | 职责 | 边界 |
| --- | --- | --- |
| ChatGPT 会话 | 语伴交流、主动分析、报告草稿 | 不保证每轮工具调用 |
| MCP adapter | 工具定义、鉴权、schema 校验 | 不读取完整聊天历史 |
| 应用服务 | 幂等保存、报告验证、复习规则、导入导出 | 默认不请求模型 API |
| 数据库 | 样本、词表、证据、报告版本、计划 | 不存 ChatGPT cookies 或密码 |
| 本地管理页面 | 报告、词表、JSON 提交、导入确认 | 默认不向公网开放 |
| Import worker | 本地解析、选定会话、预览、去重 | 不抓网页、不自动导入所有聊天 |
| 可选连接层 | HTTPS 代理或合资格 Tunnel | 不绕过套餐和工作区权限 |

建议 TypeScript + 官方 MCP SDK，单用户先 SQLite，页面可用 React；多用户需要明确后再评估 PostgreSQL。以上为建议，不是已采用或性能实测结果。[S3]

Docker 草案：`app` + 持久化 `data` volume + 可选反向代理。管理页面绑定 loopback，公网 MCP 必须鉴权。公开部署优先标准 OAuth。Tunnel 凭证在用户自己的部署中配置，不能用 ChatGPT 登录 cookies 替代。

```text
README.md
LICENSE                     # 所有者选择后添加
compose.yaml                # 待实现
.env.example                # 只列键名，不含真实凭证
apps/server/src/{mcp,services,import,auth}
apps/web/src/{reports,vocabulary,plans,import}
packages/contracts/         # schemas、版本、错误码
packages/learning-rules/     # 可测试复习规则
migrations/
tests/{unit,integration,fixtures}
docs/{setup,privacy,backup,compatibility}.md
```

不编仓库地址、镜像地址或部署成功结果。README 的 Docker 命令必须标占位，等实现后再变成可运行步骤。

## 4. MCP 工具与 schema 草案

身份由认证会话确定，不允许模型传入 `user_id` 切换用户。服务端签发 session ID，并检查全部对象归属。读工具无写副作用，写工具如实标注，不能用 read-only 名义绕过限制。工具调用由模型选择，不是可靠事件流。[S3][S4]

| 工具 | 类型 | 输入要点 | 输出 |
| --- | --- | --- | --- |
| `start_session` | 写 | 场景、语言组合、时区、操作 ID | session ID、计划版本 |
| `log_turn` | 写 | 主动提交的学习片段、session ID、来源、操作 ID | record ID、hash、保存时间 |
| `save_learning_item` | 写 | word/phrase/pattern、内容、释义、证据 ID | item ID、复习状态 |
| `get_learning_evidence` | 读 | 日期或会话范围、cursor | 样本、证据 ID、分页状态、source hash |
| `get_report` | 读 | 本地日期、可选版本 | 报告、证据范围、完整性说明 |
| `save_report` | 写 | 报告、证据 ID、source hash、操作 ID | report ID、version |
| `get_daily_plan` | 读 | 日期、语言组合 | 复习项、短练习、难度、选择原因 |

`log_turn` 仅代表主动提交一段练习，不是每轮埋点；建议实现时改名 `save_practice_sample`。没有官方支持的必达全量日志通道。

### 4.1 保存练习片段

```json
{
  "type": "object",
  "additionalProperties": false,
  "required": ["session_id", "client_operation_id", "learner_text", "companion_text", "source"],
  "properties": {
    "session_id": {"type": "string"},
    "client_operation_id": {"type": "string"},
    "learner_text": {"type": "string"},
    "companion_text": {"type": "string"},
    "source": {"type": "string", "enum": ["selected_text_practice", "user_reviewed_voice_text"]},
    "observed_at": {"type": "string", "format": "date-time"},
    "assistance": {"type": "string", "enum": ["independent", "hinted", "copied", "unknown"]}
  }
}
```

来源标记不是同意证明，必须让用户明确看到分享范围，并遵守客户端确认。服务端另做大小、日期、空值校验，限额待测试后设定，不编固定数字。

### 4.2 保存学习项目

```json
{
  "type": "object",
  "additionalProperties": false,
  "required": ["client_operation_id", "kind", "text", "target_language", "evidence_ids"],
  "properties": {
    "client_operation_id": {"type": "string"},
    "kind": {"type": "string", "enum": ["word", "phrase", "pattern"]},
    "text": {"type": "string"},
    "target_language": {"type": "string"},
    "native_gloss": {"type": "string"},
    "example": {"type": "string"},
    "evidence_ids": {"type": "array", "items": {"type": "string"}}
  }
}
```

### 4.3 保存报告

```json
{
  "type": "object",
  "additionalProperties": false,
  "required": ["schema_version", "client_operation_id", "local_date", "timezone", "source_hash", "evidence_ids", "summary", "observations", "next_practice"],
  "properties": {
    "schema_version": {"type": "string", "const": "draft-v1"},
    "client_operation_id": {"type": "string"},
    "local_date": {"type": "string", "format": "date"},
    "timezone": {"type": "string"},
    "source_hash": {"type": "string"},
    "evidence_ids": {"type": "array", "items": {"type": "string"}},
    "summary": {"type": "string"},
    "observations": {
      "type": "array",
      "items": {
        "type": "object",
        "additionalProperties": false,
        "required": ["claim", "evidence_ids", "assistance"],
        "properties": {
          "claim": {"type": "string"},
          "evidence_ids": {"type": "array", "items": {"type": "string"}},
          "assistance": {"type": "string", "enum": ["independent", "hinted", "copied", "unknown"]}
        }
      }
    },
    "next_practice": {"type": "array", "items": {"type": "string"}}
  }
}
```

读工具契约：`get_report` 接受 `local_date`、可选 `version`；`get_daily_plan` 接受 `local_date` 和语言标签；`get_learning_evidence` 接受日期范围、cursor，返回 `has_more` 和下一页。模型不能把分页后的局部结果说成全部材料。

幂等：数据库对 `(owner_id, tool, client_operation_id)` 加唯一键；同键同内容返回已有结果，同键不同内容报 conflict，不静默覆盖。成功返回 `status/record_id/content_hash/saved_at/version`；失败区分 validation_error、unauthorized、not_found、conflict、stale_source、retryable_error。重试复用操作 ID；成功回执出现前不显示已保存。

JSON Schema 只验证结构，不能保证结论真实。还需验证证据归属、source hash、日期和引用。

## 5. 数据模型草案

| 表 | 关键字段 |
| --- | --- |
| users | id、native_language、target_language、timezone、self_rated_level、sharing_preferences |
| sessions | id、owner_id、scene、language_pair、started_at、ended_at、plan_version、record_scope |
| practice_samples | id、owner_id、session_id、两方文本、source、assistance、observed_at、received_at、hash、import_batch_id |
| learning_items | id、owner_id、kind、normalized_text、language、gloss、example、status |
| item_evidence | item_id、sample_id、evidence_type |
| review_events | item_id、时间、task_type、独立/提示/照抄、结果、evidence_id |
| mastery | item_id、new/recognizing/recalling/transferring/unknown、last_independent_success、next_due_at、rule_version |
| reports | id、owner_id、local_date、timezone、schema_version、source_hash、payload、version、status |
| daily_plans | id、owner_id、local_date、items、scene、difficulty、selection_reasons、rule_version |
| import_batches | id、archive_hash、parser_version、选定会话、状态、失败列表 |
| operations | owner_id、tool、client_operation_id、payload_hash、result |

时间戳存 UTC，学习日按用户设置时区计算。规范化只用于匹配，不覆盖原文。展示过不等于掌握；无证据标 unknown。报告留历史版本；删除来源证据时需标记关联分析失效。不同来源缺共同 ID 时仅提示疑似重复，不能凭相似文本自动删除。

## 6. 每日报告流程

1. 用户说“结束今天练习，分析已保存材料”。没有主动会话，后端只整理现有记录，不自动用订阅请求模型。
2. `get_learning_evidence` 读取选定日期样本，返回证据 IDs 和 source hash。上下文装不下就按会话分块，再汇总；不假装覆盖缺失数据。
3. ChatGPT 生成草稿：场景、已接触词组、独立使用与提示后使用、问题、未知项、下次练习。观察必须引用证据 IDs。
4. 用户检查范围与内容，执行 `save_report`。写工具不可用时复制同一 JSON 到本地页面，不能把写入伪装为 read。
5. 后端验证结构、来源和 hash。材料在分析中改变时返回 stale_source，要求重做或保留旧快照报告。
6. 页面写“基于已保存学习样本”，展示来源与样本数；不知道全部聊天总数就不编覆盖率。
7. 本地可测试规则更新复习状态、生成次日计划。模型建议不直接覆盖掌握度。

## 7. 自适应课程与 A1 到 A2

这是产品逻辑建议，不是经验证的教育量表或 CEFR 认证。

- 中文输入多：英语回复保持短、简单，给一条可模仿表达，邀请而不强迫英语回答。
- 仅识别正确：安排理解任务，只更新 recognizing。
- 照抄或提示后正确：换场景再撤掉提示，不计独立 recall。
- 独立正确：在新日常场景复用同一词组，再加入少量新内容。
- 混淆或答不好：用中文简短解释，检查疲劳和任务变化，不自动认定遗忘。

计划优先级建议：到期复习 > 实际困难的词组 > 当前场景需要的已学表达 > 新表达。并列优先用户收藏。无数据从自评与简单诊断练习开始。

独立成功拉长复习间隔，提示成功保持较近复习，失败缩短，无数据不降级。间隔、权重与成功阈值均作为配置，真实使用后校准，本草案不编天数或通过率。

A2 候选证据：跨会话、不同日常场景独立完成简单英语交流，能理解问题、复用词组，不总依赖翻译或现成答案。达到候选状态后问用户是否尝试更长对话；同意后逐步加长回应和追问，困难时退回 A1 支架。页面写“建议尝试 A2 内容”，不写“已取得 A2 等级”。

扩展语言组合时将模板、分词、脚本方向、规范化、练习类型和例子做成语言包。不能把英语按空格切词的逻辑直接套到中文。

## 8. ChatGPT 导出导入与订阅内分析

官方确认符合条件的个人账号可手动导出，ZIP 含聊天历史及其他账号数据。已删除聊天不能恢复；Business/Enterprise 等不提供同样自助导出。不是任何账号必有的全量兜底。[S6]

1. 用户在 ChatGPT 数据控制界面自行申请并下载，不向本产品交 ChatGPT 密码、cookies 或下载链接。
2. 本地解析 ZIP，预览会话与数据类别，用户选择语言学习会话；默认不导入全部账号数据。
3. 检测真实导出格式。`conversations.json` 作为候选文件名，不依赖未经验证的永久 schema；未知格式拒绝猜测。
4. 解析可能的编辑分支与消息树，预览采用的路径；active branch 不明时请用户选，不能把所有分支线性拼成真实对话。
5. 规范化文本；缺角色、时间或文本明确标未知。图片、音频、工具消息分开，缺转写时不补造。
6. 用源会话/消息 ID 与内容 hash 同源去重，保留批次和 parser version。与 MCP 片段做跨来源疑似重复提示。
7. 确认后事务入库；可重试，删除临时解压数据；可选加密保留原件。
8. **导入不等于完成模型分析。** 用户再在 ChatGPT 通过只读 MCP 取回选定材料，或手动上传选定文本，生成报告；结果走 save_report 或手动 JSON。

再次传给 ChatGPT 的范围必须明确。不能为了分析自动上传整份 ZIP 的无关私人内容。导出设计不等于 OpenAI 明确许可所有后续处理方式。

导入安全：限制压缩/解压体积及文件数量，防路径穿越与嵌套压缩，不执行文件内容；按不可信数据处理文本，内嵌提示无权限；页面转义 HTML，日志不记录私人全文。限额根据实测设定。

## 9. 风险与对策

| 风险 | 对策及剩余边界 |
| --- | --- |
| 个人订阅无 MCP 写权限 | 先探针实测，保留只读 + 手动 JSON，不保证开通 |
| 漏调工具、网络失败 | 主动保存、幂等、成功回执，不宣称全量覆盖 |
| 报告幻觉 | 引证据、hash、用户审阅、可撤销版本 |
| 高估学习水平 | 区分识别/照抄/提示/独立迁移，不颁发等级 |
| 导出格式变化、分支错序 | parser version、合成测试、预览、拒绝猜测 |
| 隐私与权限漏洞 | 最少权限、本地选择、可删除、加密备份、禁提交 Git |
| Prompt injection | 工具结果只给数据，所有写入服务端校验范围与归属 |
| Voice/图像不能联动 | 文本 MVP，单独验收，不承诺自动资源回流 |
| 自托管被当成离线 | 告知 ChatGPT 仍处理会话和取回的学习材料 |
| 自动日报误解 | 用户主动发起，不把订阅当无人值守后端算力 |
| 平台政策/界面变化 | 发布前复核，维护兼容性记录，不承诺永久入口 |

## 10. 实施顺序与未做事项

先账号能力探针，再最小持久化闭环，再学习规则，再安全导入，最后开源准备。探针只需一个读计划工具和一个明确的非破坏性保存工具；测试鉴权、确认提示、回执与重试。写权限不足先证明手动路径可用，别先做完整 UI 或 Voice。

开源前换真实截图和可运行部署步骤，补贡献指南、隐私、备份、兼容性矩阵。MIT 或 Apache-2.0 仅为许可证建议，所有者选择后再添加 LICENSE。

本任务未建立公开仓库、未部署、未选择最终项目名或采用许可证。账户写权限、Voice 联动、Tunnel 资格/费用、导出格式和学习算法阈值都仍待确认。

## 11. 官方来源

[S1] Developer mode：写 Plus/Pro 等 eligible、读写工具可用；与 S2 冲突。
https://developers.openai.com/api/docs/guides/developer-mode

[S2] Developer mode and MCP apps：套餐/工作区控制，Pro read/fetch，本地不可直接连接。
https://help.openai.com/en/articles/12584461

[S3] MCP server：SDK、组件边界、模型选择工具调用。
https://developers.openai.com/apps-sdk/build/mcp-server

[S4] App submission guidelines：公开提交数据边界、完整聊天记录禁令、动作标注。
https://developers.openai.com/apps-sdk/app-submission-guidelines

[S5] Secure MCP Tunnel：私网连接，另需权限、关联与运行凭证；费用未确认。
https://developers.openai.com/api/docs/guides/secure-mcp-tunnels

[S6] Data export：账号资格、ZIP、隐私、删除聊天边界；未提供稳定产品导入 schema。
https://help.openai.com/en/articles/7260999

[S7] Apps in ChatGPT：页面写 Voice 不支持 apps。
https://help.openai.com/en/articles/11487775-apps-in-chatgpt

[S8] ChatGPT Voice：页面写 Live 能用可用 plugins/apps；与 S7 冲突。
https://help.openai.com/en/articles/20001274-chatgpt-voice
