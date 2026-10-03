# SEP-XXXX: Structured Error Policy for MCP Failures

- **Status**: Draft
- **Type**: Standards Track
- **Created**: 2026-10-03
- **Author(s)**: Yoad Elkayam (@YoadElkayam), aurumflux20 (@aurumflux20), johnyzaguirre-glean (@johnyzaguirre-glean), with contributions from @HarperZ9, @soul-sol, @slowe89, and @devmaha (originating discussion)
- **Sponsor**: None (seeking sponsor)
- **PR**: https://github.com/modelcontextprotocol/modelcontextprotocol/pull/XXXX

> **AI assistance disclosure (per AI_POLICY.md):** parts of this document and of the
> reference implementation were drafted with AI assistance (Claude Code) under the
> direction of the authors, who understand and stand behind all of it. One
> co-author, @aurumflux20, has disclosed operating as an AI agent; every
> contribution from that account (scan datasets, conformance modes, corrections)
> was independently verified by the human authors before inclusion.

## Abstract

MCP failures surface at three layers — transport, JSON-RPC error, and tool result
with `isError: true` — and all three render as free text for the language model.
This SEP adds an optional, machine-readable **error policy** payload, attachable at
each layer, that tells a client *what kind* of failure occurred (`category`, a
closed enum mapped to three behavior classes), *whether and when* a retry may
succeed (`retryable`, "not before" timing), *whether the effect is safe to repeat*
(gated on a declared effect class), and *how to reconcile* an ambiguous outcome (a
declared read with a four-valued verdict in which "could not determine" is terminal
and never collapses into "absent"). Machine directives are separated from a short
model-facing guidance string and from operator-facing fields (`code`,
`correlationId`). Clients that understand only `category` and `retryable` still
behave correctly. The proposal is fully additive, carried in existing extension
points, with a working reference implementation and a conformance battery that has
caught three real bugs in that implementation.

## Motivation

### The gap

There is no standard way for a server to say "this failure is retryable, wait N
seconds" or "this call may have executed; check before retrying." `error.data` is
application-defined and `_meta` has no agreed shape, so the retry decision falls to
the model: it re-reads the raw error on every turn and retries on a sampling
outcome rather than a policy, burning context and sometimes double-executing tools
with real side effects.

### Fragmentation is already here

Independently built servers fill the gap incompatibly (documented in
[discussion #3188]): Docdex (custom code `-32029`, fields `retry_after_ms`/`retry_at`),
slack-mcp (dedicated exception class, `retry_after`), mcp-time-server-node
(`-32000`, `retryAfter` in seconds). Client-side symptoms include opencode aborting
in-progress tasks on rate-limit errors despite having working retry logic it cannot
trigger, and independent issues in vercel/ai and awslabs/mcp (a hallucinated retry
loop; a server wrongly marked unavailable). SEP-1686 (Tasks) explicitly deferred a
general idempotency mechanism to "a dedicated proposal."

### Inference cannot replace declaration

Two independent measurements: a static scan of 755 published MCP servers (671
scanned, 27,153 tools) found 32% of write-capable servers with no visible
idempotency guard, and annotations that exist are not necessarily consulted at
runtime. A separate careful static-inference tool for retry safety still produced
roughly one false positive in three when hand-verified. Only the tool reliably
knows whether it is safe to replay; the spec should let it say so. (Datasets and
methodology: [fencescan], [agent-money-test].)

### Field evidence that the rules change outcomes

Applying this SEP's rules to shipping software found and fixed real defects:
Agent402 shipped a client fix for an idempotency key that could not survive a
retry ([Agent402 #1126]); trigger.dev clarified its idempotency boundary in
documentation after the gap was demonstrated ([trigger.dev #4627]); further
red/green reproductions exist for multiple x402 facilitators. The reference
implementation's own conformance battery caught the reference proxy
double-executing twice and over-refusing once — recorded publicly as the argument
for declaration over inference, demonstrated on our own code.

### Cost to the model

Measured on deterministic failure scenarios: unprotected agents re-read 200–250
token error payloads per retry turn with the full conversation replayed each time;
behind a policy-aware layer the model receives one message of roughly 20 tokens
(96–98% of error-payload tokens kept out of context).

## Specification

The full normative text, JSON Schema, and conformance examples are maintained in
the working draft at [mcp-fuse/sep] and will be inlined here at the sponsor's
preference. Summary of the normative core:

### Payload

An optional object carried at three attachment points: JSON-RPC errors
(`error.data["io.modelcontextprotocol/error-policy"]`), tool results with
`isError: true` (`result._meta`, same key), and an `MCP-Error-Policy` HTTP response
header (base64url JSON) so the signal survives an opaque transport-level body.

```jsonc
{
  "version": "1",
  "category": "rate_limit",      // closed enum, 11 values, mapped to 3 classes
  "retryable": true,
  "retry": { "afterMs": 12000, "strategy": "fixed", "maxAttempts": 3 },
  "circuit": { "state": "open", "reopenAfterMs": 30000 },
  "reconcile": { "tool": "get_invoice", "verdict": "could_not_determine" },
  "agentGuidance": "This tool is rate limited. The system retries automatically; do not re-invoke it yourself.",
  "code": "payment_challenge_invalid",   // operator-facing, clients MUST NOT branch on it
  "detail": "upstream returned 429 ...", // operators/logs only, never the model
  "correlationId": "req_8f3a"
}
```

- `category` → class mapping (client behavior contract): **reliability**
  (`transient`, `rate_limit`, `timeout`, `unknown`: retry per directive, subject to
  the replay gate), **capability** (`auth`, `permission`, `invalid_input`,
  `not_found`, `resource_exhausted`, `permanent`: stop and surface the required
  out-of-band action), **governance** (`policy_blocked`: escalate for human
  approval; do not retry, do not re-auth).
- `retry.afterMs` is a "not before" bound, never "sleep this long": a client whose
  own deadline falls earlier fails fast and remembers the bound.
- Minimal-consumer rule: a consumer understanding only `category` + `retryable`
  MUST still behave correctly.

### Replay gate and effect declaration

A tool declares a three-state **effect class** (safe-to-replay / unsafe-reversible
/ unsafe-irreversible), superseding the boolean reading of `idempotentHint`.
Normative behavior:

- A client MUST NOT automatically replay an ambiguous failure (timeout, mid-stream
  reset, opaque 5xx — any received response) against a tool not declared safe to
  replay. Nothing in the text of a response proves non-execution.
- A client MAY replay a request that provably never left the client, subject to
  `retry.afterMs`.
- Where a tool accepts a caller-supplied idempotency key, its contract MUST require
  the same value for every attempt of the same logical operation and a distinct
  value per operation; the key MUST be retained until the outcome is reconciled and
  MUST NOT be rotated because an attempt failed ambiguously ("the description is
  the interface": a model re-reads parameter text on every call and executes it
  literally).

### Reconciliation

A tool not safe to replay MAY declare a reconciliation pointer: a read-only tool
that answers "did this effect happen?". Running it yields exactly one of four
verdicts: effect found once / authoritatively absent / found more than once /
**could not determine** — and the last is terminal for automatic handling: it MUST
NOT be collapsed into "absent", including by the client's own plumbing (a failed,
truncated, or success-shaped non-answer read maps to could-not-determine).
Reconcile checkers MUST be validated against positive controls (one case that must
return settled, one that must return not-settled).

## Rationale

- Directives and model-facing text are separated because the error text itself is
  what burns context; `agentGuidance` is capped, imperative, never a stack trace.
- The category enum is closed and small so clients can branch on it; granular
  server-defined codes ride in `code` for operators, and domain vocabularies
  (payments first) live as profiles rather than growing the enum.
- Replay gating on declared effect class rather than error class alone follows
  from the measured unreliability of inference (one in three false positives under
  careful static analysis).
- Prior art: `google.rpc.RetryInfo`/`ErrorInfo`, gRPC retry policy, Envoy retry
  semantics, Cloudflare's header-plus-body retry timing.

## Backward Compatibility

Fully additive. All fields are optional and carried in existing extension points.
Servers that emit nothing and clients that ignore the payload behave exactly as
today. The only constraint on existing behavior is the replay gate, which restricts
automatic retries clients were never guaranteed to be safe making. One open
question for reviewers: multiple SDKs have been observed dropping `_meta` in
transit; the authors prefer a normative preservation requirement over a dedicated
field, and flag it here because it lands on SDK maintainers.

## Reference Implementation

- **mcp-fuse** (MIT; npm `mcp-fuse`, `mcp-fuse-core`): transparent stdio proxy
  implementing both sides — synthesizes the payload for servers that do not emit it
  (classifier), enforces the structural replay gate, executes declared
  reconciliation reads with the four-valued verdict (could-not-determine terminal),
  converts out-of-budget `afterMs` into circuit cooldown, and attaches the payload
  under `_meta`. 41 tests including end-to-end integration.
- **Conformance battery** ([mcp-fuse/sep/conformance]): nine failure modes with
  direction-aware scoring (over-firing AND over-refusing) and a `not_exercised`
  verdict that is never counted as a pass. It has caught three real bugs in the
  reference implementation — the sequence is documented as part of the battery.
- **fencescan** (Apache-2.0): declaration-side scanner and dataset; a payment-shaped
  sibling battery ([hostile-facilitator]) covers facilitator-specific vectors.

## Security Implications

- `detail` may contain sensitive upstream error text: for operator logs, never
  forwarded to the model or untrusted clients by default.
- `reconcile` names a read a client may run automatically; it MUST be restricted to
  tools declared read-only, or a malicious server could use it to trigger side
  effects.
- `agentGuidance` is server-supplied model-facing text and therefore a
  prompt-injection surface, like tool descriptions today; hosts SHOULD apply the
  same trust treatment. The 500-character cap and imperative-only convention limit
  but do not remove this.
- Intermediaries that rewrite failures MUST log every suppressed raw error so
  nothing is unrecoverable.

[discussion #3188]: https://github.com/modelcontextprotocol/modelcontextprotocol/discussions/3188
[fencescan]: https://github.com/aurumflux20/fencescan
[agent-money-test]: https://github.com/aurumflux20/agent-money-test
[Agent402 #1126]: https://github.com/MikeyPetrillo/Agent402/issues/1126
[trigger.dev #4627]: https://github.com/triggerdotdev/trigger.dev/issues/4627
[mcp-fuse/sep]: https://github.com/YoadElkayam/mcp-fuse/tree/main/sep
[mcp-fuse/sep/conformance]: https://github.com/YoadElkayam/mcp-fuse/tree/main/sep/conformance
[hostile-facilitator]: https://github.com/aurumflux20/hostile-facilitator
