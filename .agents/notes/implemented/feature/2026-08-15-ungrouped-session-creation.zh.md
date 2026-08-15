# Agent Note: Ungrouped session creation from the resident bucket

Status: implemented

[English](2026-08-15-ungrouped-session-creation.md) | 中文

## Problem

侧边栏分组视图只在存在游离 Session 时才派生 Ungrouped 桶，因此一旦有了 Workspace 且所有 Session 都已入账，就再也没有任何界面能在所有 Workspace 之外启动 Session。主视觉区编辑器也让这类 Session 即便通过其他途径存在也无法使用：没有归属 Workspace 的空白 Session 会渲染「选择工作区」chip 占位符和只读 textarea，用户必须先选一个 Workspace 才能输入。

## Decision

**Ungrouped 桶始终作为树的尾部分组存在。** `deriveGroups` 不再在桶为空时将其隐去，因此分组视图始终提供其 ＋。分组空状态（「暂无会话」）随之消失，因为桶头取代了它；单列表保留自己的空状态。

**桶的 ＋ 启动一条游离 Session。** 新的 `WorkspaceRuntime.startUngroupedSession` 操作复用既有的未分组空白 Session（blank、未归档、不在任何 Workspace 记账内），否则调用 `session.create({})`——Host 既有的无工作区创建，回退到 Host cwd 且不挂任何记账——然后将其打开，沿用 `connectWorkspace` 复用或新建的形态与非致命失败姿态。该操作通过新的 `WorkspaceBrowserInjected.startUngroupedSession` 到达浏览器；`SessionsPort` 的 create 面加宽为允许省略 Workspace，`TestWorkspaces` double 记录该操作。

**未分组 Session 保持主视觉区编辑器可用。** `ConversationRoot` 的 chip 标签推导在 Workspace 列表就绪且无归属 Workspace 时解析为未分组桶标签，而非「选择工作区」占位符，因此 `inert` 不再成立，textarea 可以输入。chip 仍可打开选择器；选择某个 Workspace 会跳转到该 Workspace 的空白 Session（既有的 `selectWorkspace` 语义——不存在挂接操作，见 Consequences）。

## Alternatives considered

**chip 显示 cwd 的 basename。** 否决：侧边栏把这类 Session 归入 Ungrouped 桶，而删除 Workspace 的场景刻意不显示已删除文件夹名——桶标签与 cwd 无关，对两种情况都诚实。

**保留占位符并要求先选 Workspace。** 否决：需求明确是未分组 Session，编辑器对它没有其他理由保持惰性。

**新增 Host 挂接操作，让选择 Workspace 收编当前 Session。** 否决：选择器 chip 的既有语义是重新定向 New Session 流程，收编是另一个产品缺口（记录在 ui-workspace README 的 Known Limitations 中）。

## Consequences

分组侧边栏始终以 Ungrouped 头收尾，其 ＋ 会在所有 Workspace 之外启动 Session；在那里诞生的 Session 立即可用，主视觉区 chip 显示桶标签。删除 Workspace 后其 Session 也直接可用，不再强制先选。分组空状态消失（单列表保留「暂无会话」）。游离 Session 仍没有收编入口：chip 的选择会跳转到所选 Workspace 的空白 Session，而不是挂接当前这条。分组空状态分支及其覆盖测试随桶的常驻一并移除。
