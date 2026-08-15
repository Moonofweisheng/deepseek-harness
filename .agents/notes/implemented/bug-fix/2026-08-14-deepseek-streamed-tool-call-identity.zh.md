# Agent Note: DeepSeek 流式工具调用身份

Status: implemented

[English](2026-08-14-deepseek-streamed-tool-call-identity.md) | 中文

## Problem

DeepSeek 兼容流式响应可能把工具调用 id 与函数名拆到多个 delta 中，随后再为同一协议 index 发送空字符串或 null 占位符。DeepSeek 转换器会在后续 delta 携带身份字段时覆盖已有值，因此较晚的占位符可能清除已经组装的 id 或名称。最终完成的 block 可能以空身份进入工具查找，而不是使用提供方发出的调用身份。

工具调用 id 还会关联 assistant 调用、工具执行、工具结果与下一次提供方请求。只修复查找或放宽持久 Session 校验，会让这些记录无法证明它们描述的是同一次调用。

## Decision

DeepSeek 转换器负责组装协议分片。对于每个工具调用协议 index，它会按到达顺序拼接 `id`、函数 `name` 与参数的每个字符串分片。空字符串不会追加内容，null 或缺失字段不会改变已累积值。每个已发送 delta 都会报告当时累积的身份。

当 `[DONE]` 关闭工具调用 block 时，转换器会对缺失的 id 或函数名抛出 `MALFORMED_RESPONSE`。Session 校验继续保持严格：已完成的工具调用 block，以及执行事件之间记录的调用 id，仍必须可用且一致。适配器的可空协议类型只描述提供方占位符，不会扩大已完成 harness 类型。

## Alternatives considered

**采用首个非空 id 与名称。** 拒绝，因为提供方可能把任一值拆成多个有意义的字符串。拼接与参数组装一致，并保留每个有序分片。

**允许已完成的空身份，再由工具查找恢复。** 拒绝，因为没有可靠的工具名称或跨轮调用 id 可以恢复。在流完成时失败，可以在执行产生含糊记录前报告提供方协议缺陷。

**放宽 Session 调用 id 校验。** 拒绝，因为重建责任属于协议解析器，而非持久 Session 存储。接受空或不一致 id 会削弱所有提供方的重放与历史序列化。

**拒绝身份尚不完整的中间 delta。** 拒绝，因为不完整的中间 delta 是有效的流式输入。只有 `[DONE]` 关闭已组装 block 时才要求完整。

## Consequences

碎片化 DeepSeek 工具调用会在 assistant 历史、执行、结果和后续请求之间保留完整身份。空字符串与 null 占位符无法清除较早分片。始终没有提供身份的流会确定性失败，而不会调用未知工具。

单元覆盖固定了身份分片、占位符处理、并行 index 与最终字段缺失。无密钥 headless snapshot 会运行交付的 DeepSeek HTTP/SSE 适配器，从分片组装 `call_streamed` 与 `todo_write`，执行真实 todo 工具，并验证下一次提供方请求携带相同的调用 id 与函数名。
