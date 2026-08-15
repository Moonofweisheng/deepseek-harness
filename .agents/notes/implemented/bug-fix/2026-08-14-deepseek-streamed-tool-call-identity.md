# Agent Note: DeepSeek streamed tool-call identity

Status: implemented

English | [中文](2026-08-14-deepseek-streamed-tool-call-identity.zh.md)

## Problem

DeepSeek-compatible streaming responses may split a tool-call id and function name across several deltas, then emit empty-string or null placeholders for the same wire index. The DeepSeek translator replaced identity fields whenever another delta carried them, so a later placeholder could erase an already assembled id or name. The resulting completed block could reach tool lookup with an empty identity instead of the call emitted by the provider.

Tool-call ids also connect the assistant call, tool execution, tool result, and the next provider request. Repairing only lookup or weakening durable-session validation would leave those records unable to prove that they describe the same call.

## Decision

The DeepSeek translator owns wire-fragment assembly. For each tool-call wire index, it concatenates every string fragment of `id`, function `name`, and arguments in arrival order. Empty strings append nothing, while null or absent fields leave accumulated values unchanged. Each emitted delta reports the identity accumulated so far.

When `[DONE]` closes a tool-call block, the translator rejects an absent id or function name with `MALFORMED_RESPONSE`. Session validation remains strict: completed tool-call blocks and the call ids recorded across execution events must still be usable and consistent. The adapter's nullable wire types describe provider placeholders without broadening the completed harness type.

## Alternatives considered

**Use the first non-empty id and name.** Rejected because providers may fragment either value into multiple meaningful strings. Concatenation matches argument assembly and preserves every ordered fragment.

**Allow a completed empty identity and make tool lookup recover.** Rejected because there is no reliable tool name or cross-round call id to recover. Failing at stream completion reports a provider protocol defect before execution creates ambiguous records.

**Relax session call-id validation.** Rejected because the wire parser, not durable session storage, owns reconstruction. Accepting empty or inconsistent ids would weaken replay and history serialization for every provider.

**Reject intermediate deltas that lack complete identity.** Rejected because incomplete intermediate deltas are valid streaming input. Completeness is required only when `[DONE]` closes the assembled block.

## Consequences

Fragmented DeepSeek tool calls retain their full identity across assistant history, execution, results, and the following request. Empty and null placeholders cannot erase earlier fragments. A provider stream that never supplies identity fails deterministically instead of invoking an unknown tool.

Unit coverage pins fragmented identity, placeholder handling, parallel indices, and missing final fields. A keyless headless snapshot runs the shipped DeepSeek HTTP/SSE adapter, assembles `call_streamed` and `todo_write` from fragments, executes the real todo tool, and verifies that the next provider request carries the same call id and function name.
