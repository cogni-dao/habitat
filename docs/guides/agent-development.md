---
id: agent-development-guide
type: guide
title: Agent Development Guide
status: draft
trust: draft
summary: Billing-safe rules for calling an LLM from this node — every LLM call MUST flow through the graph executor.
read_when: Adding any AI feature that calls an LLM (chat, agent, research/generate panel).
owner: derekg1729
created: 2026-06-24
tags: [ai, agents, dev, billing]
---

# Agent Development Guide

> Quick reference for adding AI features to this node.

## Billing-safe AI: the ONLY way to call an LLM

> [!CRITICAL]
> **Every LLM call in a node MUST flow through the graph executor.** It is the only path that runs the preflight credit gate AND records a usage receipt. Calling the raw LLM transport directly silently bills the end user with no credit check and no model choice — the exact defect that shipped in a forked node's "Research/Generate" panels (bug.5042). Fair billing is mission-critical; do not work around it.

**ALWAYS — pick one sanctioned entrypoint:**

1. **A catalog graph** invoked via `GraphExecutorPort.runGraph` — for chat, agents, tool-using flows.
2. **The `completionStream` facade** (`@/app/_facades/ai/completion.server`) — for a one-shot server-side "research / generate / summarize". It starts the same billed `GraphRunWorkflow`. For a non-streaming result, drain the stream and `await final`:

   ```ts
   import { completionStream } from "@/app/_facades/ai/completion.server";

   const { stream, final } = await completionStream(
     { messages, modelRef, sessionUser /* who gets billed */ },
     ctx,
   );
   for await (const _ of stream) {
     /* drain — required so billing fires (BILLING_INDEPENDENT_OF_CLIENT) */
   }
   const result = await final; // credit-gated + receipt recorded
   ```

   Pass `modelRef` from the user's `ModelPicker` selection — never hardcode a model. Insufficient credits surface as a terminal `error` event before the response commits; show it, don't swallow it.

**NEVER:**

- ❌ Inject `LlmService` into a feature/route and call `.completionStream(...)`. This skips the `PreflightCreditCheckDecorator` + `UsageCommitDecorator` stack. **CI fails** (`no-direct-completion-executestream.stack.test.ts`).
- ❌ Reach for a non-streaming `LlmService.completion(...)` — it was removed for this reason. Drain the stream + `await final` instead.
- ❌ Hardcode a "free" model to dodge the credit gate. Free vs paid is resolved server-side from the model catalog; the gate already returns 0 credits for free models.

**The ONE exception — platform/system billing (not the end user):** background jobs or platform features that must bill a _system_ account require **explicit human sign-off** and STILL route through the graph executor, bound to a designated system billing account. There is no sanctioned path that skips the executor. If you think you need one, file a bug linking your work item and ask first.

`LlmService` (`@/ports/llm.port`) is **executor-internal**: its only sanctioned consumers are `features/ai/services/completion.ts` and the in-proc completion adapter. The `BILLABLE_AI_THROUGH_EXECUTOR` and `CREDITS_ENFORCED_AT_EXECUTION_PORT` invariants govern this seam.
