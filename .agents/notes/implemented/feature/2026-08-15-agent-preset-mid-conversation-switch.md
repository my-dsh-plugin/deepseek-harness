# Agent Note: switching a session's agent preset mid-conversation

Status: implemented

English | [中文](2026-08-15-agent-preset-mid-conversation-switch.zh.md)

## Problem

A session's agent preset (standard / PTC / minimal / custom) was fixed once
its conversation started: `agentPreset.select` refused every non-blank session
with `agent-preset-locked`, so a user who picked the wrong mode at session
creation had to start a new conversation. The `dsh-agent-mode-switcher` plugin
wants the header's preset chip to become a live switcher after the model
finishes answering.

## Decision

**`agentPreset.select` now swaps an idle session, blank or not.** The guard in
the api-proxy moves from "the session has produced nothing" to "the agent is
not running a turn". A mid-answer swap is still refused (`agent-preset-locked`):
it would change the tool schemas the running request was assembled against.
The swap itself is unchanged — `AgentPresets.recompose` re-links the agent's
scope to the target preset's standing mount, and the committed selection is
appended to the session log as `agent-preset/selected`, which
`resolveSessionPreset` already reads newest-first on resume and fork.

**The history trade-off is the caller's to own.** Swapping a started
conversation's composition may leave logged tool calls the new composition
cannot make. The mode-switcher UI owns that trade-off and communicates it;
the service contract keeps its warning (the caller owns the check).

## Alternatives considered

**Ship the swap as a plugin-only RPC.** Rejected: adding a plugin-owned
remote/RPC method requires the Typert generator in the plugin build; the
standard `agentPreset.select` RPC already carries the exact payload, error
codes, event, and client summary updates, so relaxing its guard is the
smallest surface that enables the feature.

## Consequences

The browser can switch the preset of any session whose agent is idle, and the
log records each switch so a resumed or forked session rebuilds the same
composition. The `agent-preset-locked` error now means "a turn is running",
not "the conversation started". Existing behavior is unchanged for blank
sessions and for the shipped read-only header label; the
`dsh-agent-mode-switcher` plugin replaces that label with the interactive
switcher in deployments that install it.
