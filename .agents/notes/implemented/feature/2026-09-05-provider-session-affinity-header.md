# Agent Note: Provider session-affinity header configuration

Status: implemented

English | [中文](2026-09-05-provider-session-affinity-header.zh.md)

## Problem

Session-affinity gateways route and optimize by a per-conversation id carried on a request header. The OpenCode Go gateway announced that requests without `x-opencode-session` would be refused from 2026-09-06 (upstream discussion #5495), and its telemetry showed the Harness sending session context on some catalog-served models but almost never on settings-declared routes: `deepseek-v4-flash`, the highest-volume model in that measurement, carried it on about 2.5% of requests. pi-ai owns a `sendSessionAffinityHeaders` compat switch, but it is catalog-owned — this package's drift gates withhold it from configuration, it defaults to false, its wire formats emit `session_id`, `x-client-request-id`, and `x-session-affinity` rather than `x-opencode-session`, and no catalog entry for the Go gateway enables it. The adapter already receives the conversation id on every request (`GenerateOptions.sessionId`), and static profile `headers` cannot vary per conversation, so a deployment had no way to comply.

## Decision

The provider profile gains `sessionHeader`: the name of the request header that carries the conversation id. The adapter injects it at its single `streamSimple` call site — where `options.sessionId` is already stringified for pi-ai — so every protocol gets the same treatment with no per-protocol work: when the profile names a header and the request carries a session id, that header is sent with `String(options.sessionId)`; otherwise nothing is sent. The session header replaces the same name in static `headers` (a fixed value cannot do a per-conversation id's job), and Harness attribution still wins reserved names. Resolution refuses an empty name and a name Fetch cannot represent, alongside the existing header validation.

Coverage: adapter specs pin header presence with a session id, absence without one, and replacement of a same-named static header; config specs pin the empty-name and unrepresentable-name refusals plus passthrough resolution.

## Alternatives considered

- **Turning on pi-ai's `sendSessionAffinityHeaders` for the gateway.** Lost twice over: the switch is catalog-owned and withheld from settings by design, and enabling it would still emit the wrong header names for this gateway — so it needs a pi-ai upstream change (a new affinity format) before it helps, while the maintainer discussion points at this package normalizing provider particulars.
- **Hardcoding the header for `opencode*` routes** (the approach of the community branch referenced in discussion #5495). Rejected: a deployment-varying choice keyed by route-name prefix lives in code instead of configuration, and a different gateway wanting the same behavior gets nothing.
- **A static `headers` entry with a fixed id.** Rejected as the mechanism (it remains an emergency stop): it satisfies header presence but collapses every conversation into one affinity bucket, defeating the routing and cache locality the gateway optimizes for.

## Consequences

- A session-affinity gateway is one settings line away: `sessionHeader: x-opencode-session` on the route. Routes that do not set the field send byte-identical requests to before.
- The value is the Harness conversation id as-is (stable across turns, resume, compaction, and retries), not a minted UUID; a gateway demanding a specific shape is a new explicit transformation, not a silent one.
- Only the pi-ai seam is covered. `web-search-deepseek` opens its own Anthropic-compatible fetch outside `ctx.llm` and sends no session header; routing web search through such a gateway keeps that gap until that provider grows the same field.
