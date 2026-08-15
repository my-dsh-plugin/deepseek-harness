# Agent Note: 会话中途切换 agent preset（模式）

Status: implemented

[English](2026-08-15-agent-preset-mid-conversation-switch.md) | 中文

## 问题

会话的 agent preset（标准 / PTC / 极简 / 自定义）一旦对话开始就被固定：
`agentPreset.select` 对所有非空白会话一律以 `agent-preset-locked` 拒绝，用户
在创建会话时选错模式就只能重开一段对话。`dsh-agent-mode-switcher` 插件希望
模型回答完毕后，标题栏的模式标签变成可点击的切换器。

## 决策

**`agentPreset.select` 现在可以切换空闲会话，无论是否空白。** api-proxy 中的
守卫从"会话还没有产生任何内容"改为"agent 不在运行回合中"。回答进行中的切换
仍然被拒绝（`agent-preset-locked`）：那会改变正在运行的请求所依据的工具
schema。切换本身不变——`AgentPresets.recompose` 把 agent 的作用域重新挂到
目标 preset 的常驻挂载上，提交后的选择以 `agent-preset/selected` 事件追加进
会话日志，`resolveSessionPreset` 在恢复和 fork 时已经是"最新一条生效"。

**历史记录的取舍由调用方负责。** 切换已开始对话的组合，可能留下新组合无法
调用的工具调用记录。模式切换 UI 自行承担并向用户说明这一取舍；服务层契约
保留原有警告（调用方拥有检查职责）。

## 备选方案

**用插件自带的 RPC 实现切换。** 否决：插件自建 remote/RPC 需要在插件构建中
引入 Typert 生成器；标准 `agentPreset.select` RPC 已经具备完全一致的载荷、
错误码、事件和客户端摘要更新，放宽它的守卫是支持该功能的最小改动面。

## 影响

浏览器可以切换任意 agent 空闲的会话的模式，每次切换都会写入日志，恢复或
fork 时会重建同一组合。`agent-preset-locked` 现在的含义是"回合正在运行"，
而不是"对话已开始"。空白会话和随附的只读标题标签行为不变；安装了
`dsh-agent-mode-switcher` 插件的部署会用可交互切换器替换该标签。
