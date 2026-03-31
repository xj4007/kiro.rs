# Kiro Request Normalization Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Adjust the Anthropic-compatible ingress layer so client `system` is dropped, `SYSTEM_CHUNKED_POLICY` remains as an internal Kiro-side policy, `metadata` keeps its conversationId role, `thinking` remains supported, and non-Kiro extra request fields are stripped from outbound semantics.

**Architecture:** Keep Kiro traits fixed in the protocol/conversion layer and minimize merge conflicts by concentrating behavior changes in the Anthropic ingress path (`types.rs`, `handlers.rs`, `converter.rs`). Do not change Kiro transport/header generation or Kiro request model shape unless strictly necessary.

**Tech Stack:** Rust, Axum, Serde, reqwest, custom Anthropic→Kiro conversion layer.

---

## Current behavior summary

### Already aligned with the target direction
- Outbound requests are already normalized into `KiroRequest` and sent through `KiroProvider`.
- Kiro-specific protocol traits are already fixed in code:
  - headers in `src/kiro/provider.rs`
  - `conversationState` in `src/kiro/model/requests/*`
  - `agent_task_type = "vibe"`, `chat_trigger_type = "MANUAL"`, `origin = "AI_EDITOR"`
  - model mapping and tool normalization in `src/anthropic/converter.rs`
- `tool_choice` is currently accepted on input but effectively unused.
- request-side `cache_control` is not part of the outbound Kiro request construction path.

### Conflicts with the desired policy
- Client `system` is currently preserved by being rewritten into synthetic Kiro history.
- `SYSTEM_CHUNKED_POLICY` is currently appended through that client-system injection path.
- `thinking` currently survives by being converted into prompt/history tags and coupled to the synthetic history path.
- `output_config` is still consumed as part of thinking handling.
- There is no explicit request normalization policy layer; behavior is implicit and spread across handlers and converter logic.

---

## File map

### Files to modify
- `src/anthropic/types.rs`
  - Request input model for Anthropic-compatible payloads.
  - Candidate place to make “accepted but intentionally dropped” fields explicit in comments/docs.
- `src/anthropic/handlers.rs`
  - Entry-point normalization and model/thinking override behavior.
  - Candidate place to stop coupling thinking to `output_config` if required.
- `src/anthropic/converter.rs`
  - Main conversion logic.
  - Current source of system injection, `SYSTEM_CHUNKED_POLICY` appending, thinking-prefix generation, history construction.
- `src/test.rs` and/or inline tests near converter/handler modules
  - Add regression tests for normalization behavior.

### Files to leave unchanged if possible
- `src/kiro/provider.rs`
  - Already correctly owns Kiro protocol-layer headers.
- `src/kiro/model/requests/kiro.rs`
- `src/kiro/model/requests/conversation.rs`
  - Already define the Kiro request shape; changing them would increase merge risk.

---

## Target policy to implement

1. **Drop client `system` entirely**
   - Do not forward it.
   - Do not rewrite it into synthetic history.
   - Do not preserve user-provided system text in any prompt-like form.

2. **Keep `SYSTEM_CHUNKED_POLICY` as internal project policy**
   - Preserve the logic conceptually.
   - It should remain an internal Kiro-side instruction path, not a retention path for client `system`.
   - Do not tie its existence to whether the client provided `system`.

3. **Preserve `metadata` only for its current proven role**
   - Keep `metadata.user_id -> conversationId` derivation.
   - Do not attempt to preserve outbound metadata fields in Kiro request body.

4. **Preserve `thinking`**
   - Client `thinking` should remain meaningful.
   - It must not depend on retaining client `system`.
   - Remove unnecessary coupling between `thinking` and client-system synthetic history injection.

5. **Strip non-Kiro extra request fields from outbound semantics**
   - `output_config`: do not preserve as outbound semantic field.
   - `tool_choice`: explicitly ignore.
   - `cache_control`: ignore.
   - Unknown future extra fields: do not let them influence outbound Kiro body unless intentionally supported later.

6. **Keep Kiro traits in protocol/conversion layer**
   - headers
   - `conversationState`
   - `agent_task_type`
   - `chat_trigger_type`
   - `origin`
   - model mapping
   - tool normalization

---

## Chunk 1: Define normalization boundaries in the Anthropic ingress layer

### Task 1: Document the accepted-vs-effective request field policy

**Files:**
- Modify: `src/anthropic/types.rs`
- Modify: `src/anthropic/handlers.rs`
- Modify: `src/anthropic/converter.rs`
- Test: inline module tests or `src/test.rs`

- [ ] **Step 1: Add documentation comments describing normalization policy**

Update comments near `MessagesRequest` and converter entrypoints to explicitly state:
- `system` is accepted on input for compatibility but will be dropped from outbound semantics.
- `metadata.user_id` is used only for conversation ID derivation.
- `thinking` is preserved as supported functionality.
- `output_config`, `tool_choice`, and unknown extras are not part of outbound Kiro request semantics.

- [ ] **Step 2: Add a small helper or comment block that centralizes field policy**

Prefer a focused helper or explicit normalization notes rather than scattering behavior across multiple branches. Keep it local to `anthropic` to reduce conflicts.

- [ ] **Step 3: Add/adjust tests that assert accepted input fields do not imply outbound preservation**

At minimum, add tests covering:
- request with `tool_choice` does not alter conversion output
- request with `metadata.user_id` changes conversation ID derivation only
- request with unsupported extra semantic fields does not appear in outbound body

- [ ] **Step 4: Run focused tests**

Run:
```bash
cargo test anthropic -- --nocapture
```
If there is no matching filter, run:
```bash
cargo test
```
Expected: tests pass with no regression in request deserialization/conversion.

---

## Chunk 2: Remove client-system retention while preserving internal policy

### Task 2: Stop injecting client `system` into synthetic history

**Files:**
- Modify: `src/anthropic/converter.rs`
- Test: converter tests near `build_history(...)`

- [ ] **Step 1: Write failing tests for client-system dropping behavior**

Add tests that currently fail under existing behavior:
- input with `system` should not produce synthetic user/assistant history entries derived from client system text
- output history should not contain user system text
- output history should not contain the synthetic assistant message `"I will follow these instructions."` solely because client system existed

- [ ] **Step 2: Refactor `build_history(...)` so client `system` no longer contributes history messages**

Target area:
- `src/anthropic/converter.rs:582-625`

Implementation target:
- remove the branch that joins `req.system` and emits synthetic user/assistant history from it
- preserve surrounding message-history behavior for real user/assistant messages

- [ ] **Step 3: Preserve `SYSTEM_CHUNKED_POLICY` as internal policy, not client-system retention**

Do **not** delete the constant itself.
Instead, move or reinterpret its usage so that it is not dependent on the presence of client `system`.

At this stage, keep the implementation minimal and low-conflict:
- detach it from `req.system`
- do not invent a new top-level Kiro `system` field
- if no clean policy sink exists yet, park the constant as intentionally reserved internal policy and cover that decision in comments/tests

- [ ] **Step 4: Run focused tests**

Run:
```bash
cargo test build_history -- --nocapture
```
Fallback:
```bash
cargo test
```
Expected:
- client `system` no longer shows up in synthetic history
- message history behavior for normal messages stays intact

---

## Chunk 3: Keep thinking, but decouple it from client-system retention

### Task 3: Preserve thinking without requiring client-system synthetic context

**Files:**
- Modify: `src/anthropic/handlers.rs`
- Modify: `src/anthropic/converter.rs`
- Test: converter/handler thinking tests

- [ ] **Step 1: Write failing tests for the desired thinking behavior**

Add tests that assert:
- `thinking` remains meaningful when client `system` is absent
- dropping `system` does not drop `thinking`
- thinking handling does not require synthetic system/history retention

- [ ] **Step 2: Review `override_thinking_from_model_name(...)` and decide the minimal retained behavior**

Current file:
- `src/anthropic/handlers.rs:599-630`

Implementation target:
- keep model-name based thinking override only if it is still part of product behavior
- remove any dependence on `output_config` if it is no longer wanted as an effective field

- [ ] **Step 3: Refactor `generate_thinking_prefix(...)` and `build_history(...)` coupling**

Current files:
- `src/anthropic/converter.rs:546-567`
- `src/anthropic/converter.rs:585-624`

Implementation target:
- thinking must survive independently of client `system`
- client `system` dropping must not implicitly delete thinking behavior
- avoid reintroducing a disguised synthetic system retention path just to carry thinking

- [ ] **Step 4: Remove `output_config` dependency from thinking semantics if feasible**

Current use:
- `src/anthropic/converter.rs:554-563`
- `src/anthropic/handlers.rs:625-629`

Target behavior:
- `output_config` should not be required to preserve thinking
- adaptive thinking defaults should be handled internally without preserving `output_config` as an inbound semantic dependency

- [ ] **Step 5: Run focused tests**

Run:
```bash
cargo test thinking -- --nocapture
```
Fallback:
```bash
cargo test
```
Expected:
- thinking-related tests pass
- no system-derived history is needed for thinking retention

---

## Chunk 4: Make dropped fields explicit and future-proof

### Task 4: Explicitly ignore `output_config`, `tool_choice`, and request-side `cache_control`

**Files:**
- Modify: `src/anthropic/handlers.rs`
- Modify: `src/anthropic/converter.rs`
- Possibly modify comments in `src/anthropic/types.rs`
- Test: request normalization tests

- [ ] **Step 1: Add explicit tests for dropped fields**

Test cases:
- request with `tool_choice`
- request with `output_config`
- request with request-side `cache_control` equivalent if represented in test payload

Assertions:
- none of these fields appear in outbound Kiro request body
- none of them change message/history/tool conversion beyond explicitly supported behavior

- [ ] **Step 2: Remove remaining semantic reliance on `output_config`**

This is the final cleanup after Task 3. Ensure `output_config` no longer affects outbound Kiro semantics.

- [ ] **Step 3: Leave `tool_choice` and unknown future extras as non-semantic compatibility input**

Implementation target:
- accept them without crashing if already modeled
- do not let them influence outbound Kiro request unless intentionally supported later

- [ ] **Step 4: Run tests**

Run:
```bash
cargo test
```
Expected:
- full suite passes
- no regressions in messages endpoint behavior

---

## Chunk 5: Verify protocol-layer Kiro traits remain unchanged

### Task 5: Confirm Kiro-specific protocol behavior survives normalization cleanup

**Files:**
- Verify only: `src/kiro/provider.rs`
- Verify only: `src/kiro/model/requests/kiro.rs`
- Verify only: `src/kiro/model/requests/conversation.rs`
- Verify/possibly modify tests near `src/anthropic/converter.rs`

- [ ] **Step 1: Add regression tests for fixed Kiro traits**

Assert conversion still produces:
- `agent_task_type = "vibe"`
- `chat_trigger_type = "MANUAL"`
- `origin = "AI_EDITOR"`
- normalized Kiro tools in `userInputMessageContext.tools`
- mapped Kiro model IDs

- [ ] **Step 2: Verify provider headers remain untouched**

No functional changes should be required in:
- `src/kiro/provider.rs:145-198`

Add tests if the project already has provider/header tests; otherwise document that this file is intentionally unchanged for low-conflict integration.

- [ ] **Step 3: Run full verification**

Run:
```bash
cargo test
cargo build
```
Expected:
- tests pass
- build succeeds
- normalization changes do not alter Kiro transport/header layer

---

## Risk notes

### Main behavioral risk
Removing client `system` may change outputs for clients that relied heavily on upstream system prompts. This is intentional under the agreed policy, but it should be captured in tests and release notes.

### Main merge-risk mitigation
Keep changes concentrated in:
- `src/anthropic/converter.rs`
- `src/anthropic/handlers.rs`
- light comment updates in `src/anthropic/types.rs`

Avoid expanding Kiro request model or provider transport unless absolutely necessary.

### Thinking-specific caution
Because current thinking support is intertwined with synthetic history insertion, refactoring must be test-driven. Do not remove system injection first and assume thinking still works without explicit tests.

---

## Validation checklist

Before considering the work complete, verify all of the following with fresh test/build evidence:

- [ ] Client `system` no longer affects outbound Kiro history/body
- [ ] `SYSTEM_CHUNKED_POLICY` remains present only as internal project policy, not as retained client-system text
- [ ] `metadata.user_id` still influences `conversation_id`
- [ ] `thinking` still works when `system` is absent
- [ ] `output_config` no longer affects outbound semantics
- [ ] `tool_choice` remains ignored
- [ ] request-side `cache_control` remains non-semantic
- [ ] Kiro headers and protocol-layer traits remain unchanged
- [ ] `cargo test` passes
- [ ] `cargo build` succeeds

---

Plan complete and saved to `docs/superpowers/plans/2026-03-22-kiro-request-normalization.md`. Ready to execute?
