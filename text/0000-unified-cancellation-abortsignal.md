# RFC: Unified cancellation via AbortSignal across React async APIs

- Start Date: 2025-10-23
- RFC PR: (leave this as a link to the PR)
- React Issue: n/a

## Summary

Introduce an opt-in, web-standard cancellation model using `AbortSignal` that flows consistently through React async features: `startTransition`, Suspense and `use`, and server actions. The goal is to stop stale work predictably on both client and server, reduce wasted computation and I/O, and align React with platform semantics.

## Motivation

React now spans client and server execution with transitions, Suspense, `use`, and server actions. Today, when a user navigates away or starts a superseding transition, previously initiated work often continues to run. Examples:

- A transition that becomes obsolete continues resolving data and scheduling renders.
- A promise read via `use` keeps resolving after its Suspense boundary is abandoned.
- A server action keeps running even if the user leaves the page, causing unnecessary server load.

Lack of a shared cancellation model makes performance tuning and correctness harder. A unified `AbortSignal` based approach would provide a predictable, composable way to stop irrelevant work and free resources early.

## Detailed design

This RFC proposes a minimal cross-cutting integration that threads `AbortSignal` through core async APIs. The design is opt-in and advisory: React remains free to commit already-completed urgent work, but ongoing async work observes cancellation consistently.

### 1) Transitions

```ts
type TransitionOptions = {
  signal?: AbortSignal;
};

type TransitionHandle = {
  signal: AbortSignal;
  cancel: () => void;
};

declare function startTransition(
  fn: () => void,
  options?: TransitionOptions
): TransitionHandle;
```

Behavior

- `startTransition` returns a handle `{ signal, cancel }`.
- React aborts `signal` automatically when the transition is superseded by a newer transition or when the associated UI path is abandoned.
- Users may call `cancel()` to abort explicitly.
- Any async work within the transition that accepts a signal should observe `signal.aborted === true` and stop.

### 2) Suspense and `use`

```ts
declare function use<T>(
  resource: Promise<T> | { read: () => T },
  options?: { signal?: AbortSignal }
): T;
```

Behavior

- When a Suspense boundary unmounts or becomes hidden such that its work is not going to be displayed, React aborts the associated signal.
- Fetches or data loaders that participate in `use` should pass the signal to underlying I/O so that in-flight work can be cancelled by the platform.

### 3) Server actions

```ts
// Server
export async function action(
  formData: FormData,
  ctx: { signal: AbortSignal }
) {
  // Pass ctx.signal to fetch and other cancellable work
}
```

Behavior

- The client side transition or navigation creates a controller whose `signal` is propagated to server actions.
- If the client navigates away, supersedes the transition, or cancels explicitly, the server receives an aborted signal that propagates to `fetch` and other cancellable I/O.

### 4) DevTools and tracing

- DevTools can show cancelled work items with a simple trace: what was cancelled, by whom (for example, superseding transition or navigation), and when.
- This is diagnostic only. No behavior depends on DevTools being present.

### Scheduling and guarantees

- Cancellation is best-effort. If an update is already committed, React will not roll it back due to an abort.
- React continues to coalesce updates and preserve scheduling semantics. The `signal` allows userland and platform I/O to stop ongoing work.

## Examples

#### Search box with superseding transitions

```tsx
function SearchBox() {
  const [query, setQuery] = useState("");

  const { signal, cancel } = startTransition(async () => {
    const res = await fetch(`/api/search?q=${encodeURIComponent(query)}`, {
      signal,
    });
    const data = await res.json();
    // set state with results
  });

  useEffect(() => () => cancel(), [cancel]); // cancel on unmount

  return <input value={query} onChange={e => setQuery(e.target.value)} />;
}
```

#### Suspense + `use` with cancellation

```tsx
function UserPanel({ id }) {
  const controller = new AbortController();
  useEffect(() => () => controller.abort(), []); // boundary cleanup

  const user = use(fetch(`/api/user/${id}`, { signal: controller.signal }));

  return <div>{user.name}</div>;
}
```

#### Server action

```ts
export async function savePost(formData: FormData, { signal }: { signal: AbortSignal }) {
  const res = await fetch("https://api.example.com/save", {
    method: "POST",
    body: formData,
    signal,
  });
  return res.ok;
}
```

## How we teach this

- Teach that React does not invent a new cancel primitive. It uses the platform standard `AbortController` and `AbortSignal`.
- Show patterns for threading a signal through data loaders, fetch calls, and custom async functions.
- Emphasize that cancellation is opt-in and advisory, and that React will not undo already committed UI.

## Drawbacks

- Additional API surface on `startTransition` and `use`.
- Requires library authors to plumb `AbortSignal` through their async layers to capture full benefits.
- Misuse is possible if code ignores the signal, leading to partial adoption and inconsistent results.

## Alternatives

- Status quo: continue to cancel at the library level only, or rely on ad hoc patterns per feature.
- Userland state libraries: provide signal-like patterns, but lack a unified story across transitions, Suspense, and server actions.
- Promise cancellation tokens: non-standard and not aligned with the Web Platform.

## Prior art

- Web Platform `AbortController` and `AbortSignal` widely used with `fetch`.
- Data fetching libraries that support cancellation via `AbortSignal`.
- Community requests for cancellation hooks around transitions, Suspense, and server actions.

## Unresolved questions

- Should `startTransition` always return a handle, or only when an option is provided.
- Interaction with `useOptimistic` state and partial pre-rendering on the server.
- Router integration points for navigation driven cancellation.
- What minimal telemetry should DevTools surface by default.

## Adoption strategy

- Keep all new parameters optional.
- Encourage library authors and frameworks to thread `AbortSignal`.
- Provide examples and cookbook entries in docs so teams can adopt incrementally.

## Future work

- Integrations with router transitions and form actions in popular frameworks.
- Ergonomics helpers for composing controllers across nested transitions and boundaries.
