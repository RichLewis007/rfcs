# RFC: Unified cancellation via AbortSignal across React async APIs

## Summary
Introduce an opt-in, web-standard cancellation model using AbortSignal that flows through startTransition, Suspense/use, and server actions.

## Motivation
Async work spans client and server; stale work currently continues running. Unified cancellation reduces wasted work and aligns with platform fetch semantics.

## Detailed design
- startTransition(fn, { signal? }) returns { signal, cancel }.
- use(resource, { signal }) aborts when boundary unmounts/hides.
- Server actions receive { signal } and propagate to fetch/I/O.
- DevTools surfaces cancelled work and its source.

## Drawbacks
API surface growth; scheduling semantics must remain predictable.

## Alternatives
Library-level patterns; per-feature ad hoc cancellation.

## Prior art
Link to related issues you found and AbortController spec.

## Open questions
Interaction with useOptimistic, partial pre-render, router integrations, and transitions that finish synchronously.
