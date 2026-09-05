# Agent Note: Provider session-affinity header configuration

Status: implemented

[English](2026-09-05-provider-session-affinity-header.md) | 中文

## Problem

具备会话亲和性的网关依据请求头中携带的每会话 ID 进行路由与优化。OpenCode Go 网关宣布自 2026-09-06 起拒绝缺少 `x-opencode-session` 的请求（上游 discussion #5495），其遥测显示 Harness 在部分由目录直接服务的模型上会发送会话上下文，但在 settings 声明的路由上几乎从不发送：该测量中调用量最大的 `deepseek-v4-flash` 仅有约 2.5% 的请求携带。pi-ai 自带 `sendSessionAffinityHeaders` 兼容开关，但它归目录所有——本包的漂移门禁止其进入配置，默认值为 false，其线上格式发送的是 `session_id`、`x-client-request-id` 和 `x-session-affinity` 而非 `x-opencode-session`，且 Go 网关没有任何目录条目启用它。适配器在每个请求中已收到会话 ID（`GenerateOptions.sessionId`），而静态的 profile `headers` 无法按会话变化，部署因此无从合规。

## Decision

provider profile 新增 `sessionHeader`：携带会话 ID 的请求头名称。适配器在其唯一的 `streamSimple` 调用点注入——正是为 pi-ai 字符串化 `options.sessionId` 的位置——因此所有协议获得一致处理，无需逐协议工作：当 profile 命名了该头且请求携带会话 ID 时，以 `String(options.sessionId)` 发送该头；否则不发送。会话头替换静态 `headers` 中的同名条目（固定值无法承担每会话 ID 的职责），Harness 归因头仍在保留名上获胜。解析阶段在既有头部校验之外，拒绝空名称与 Fetch 无法表示的名称。

覆盖：adapter 规格固定有会话 ID 时的头部存在、无会话 ID 时的缺席、以及对同名静态头部的替换；config 规格固定空名称与不可表示名称的拒绝及解析透传。

## Alternatives considered

- **为网关打开 pi-ai 的 `sendSessionAffinityHeaders`。** 双重不成立：该开关归目录所有且被设计为不进入 settings，而且即使启用，对此网关发送的头名称仍然不对——因此需要先做 pi-ai 上游修改（新增亲和格式）才有帮助，而 maintainer 讨论指向的正是由本包负责归一化 provider 的特殊性。
- **按 `opencode*` 路由硬编码该头**（discussion #5495 中引用的社区分支的做法）。已拒绝：按路由名前缀硬编码部署相关选择，使其存在于代码而非配置中，而其他想要同样行为的网关一无所获。
- **在静态 `headers` 中写入固定 ID。** 已拒绝作为机制（它仍可作为应急手段）：它满足头部存在性，但把所有会话折叠进同一个亲和桶，破坏网关所优化的路由与缓存局部性。

## Consequences

- 会话亲和网关距离一行 settings 之遥：路由上写 `sessionHeader: x-opencode-session`。未设置该字段的路由发送与之前逐字节相同的请求。
- 值原样使用 Harness 会话 ID（跨轮次、resume、压缩与重试保持稳定），而非新铸的 UUID；若网关要求特定形状，那是新的显式转换，而非隐式行为。
- 仅覆盖 pi-ai 这一接缝。`web-search-deepseek` 在 `ctx.llm` 之外发起自己的 Anthropic 兼容 fetch，不发送会话头；经此类网关路由 web 搜索的部署保持该缺口，直至该 provider 增加同样字段。
