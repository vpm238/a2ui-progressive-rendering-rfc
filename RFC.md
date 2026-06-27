# RFC: Progressive rendering primitives for A2UI v1.0

Status: **Draft**
Target spec: A2UI v0.10 → v1.0
Authors: (fill in)
Related: [a2ui.org/concepts/data-binding](https://a2ui.org/concepts/data-binding/), [reference/messages](https://a2ui.org/reference/messages/)
Live demo: `docs/demo/progressive-rendering.html` (host on GitHub Pages; see README)

## Summary

A2UI v0.9 was designed as a "describe the UI, render it" protocol. In practice,
agents generate UI progressively: structure first, data streaming in over
seconds. The spec doesn't formalize this lifecycle, forcing every renderer
and every agent author to reinvent conventions.

This RFC proposes **four backward-compatible additions** to A2UI v0.9. The
first three make the protocol aware of *time*; the fourth makes it aware of
*who wrote what to the data model*:

1. **`pending` as a first-class resolution state** for unresolved path bindings.
2. **A `streaming` lifecycle flag on `updateDataModel`** to signal mid-stream
   vs final values. (Not adding streaming — A2UI already streams. Adding the
   protocol-level lifecycle signal so renderers can render consistent UX.)
3. **An `append` patch operation** for efficient streaming of long text.
4. **Write bindings + origin tracking** so components can mutate the data
   model client-side, and the agent automatically sees what the user
   changed without the server having to ship its own state back to itself.

Each is optional, each is small, and together they cover ~95% of the
progressive-rendering and agent-context use cases authors end up building
by hand today.

## Motivation

### The scenario the protocol doesn't describe

An agent emits a rich-message card skeleton. Its `title` and `body` fields
are `{"path": "/reply/title"}` etc. The agent then streams text into those
paths character-by-character via repeated `updateDataModel`.

Questions the current spec doesn't answer:

1. What should the client render for `headline` before any data arrives?
   (undefined? empty string? placeholder? shimmer? spinner?)
2. How does the client know that the current value is "partial mid-stream" vs
   "final"? (For cursor indicators, typewriter effects, disabling actions,
   accessibility announcements, etc.)
3. How does a server efficiently stream long text without re-sending the whole
   accumulated value on every delta? (A 500-char text field × ~100 deltas
   under the current `set` semantics = ~50 KB on the wire for what could be
   ~500 bytes of deltas.)

### Concrete today: what implementers build around the gap

A production multi-turn assistant built on A2UI v0.9 (Claude-brain agent,
Lit+Flutter renderers) ran into each of these:

- Client treats unresolved `{path}` resolution as "render a shimmer placeholder"
  — but there is zero spec guidance, so two implementations of the same catalog
  will differ visually the moment data hasn't arrived. Agents emit identical
  JSON; hosts render it inconsistently.
- Server sends full-value `updateDataModel` on every text delta. Wasteful
  (~50× overhead on long text) but the only op v0.9 supports.
- Action buttons with path-bound event names have to be client-side-disabled
  until both `event.name` and `label` resolve. Every renderer reinvents this
  guard independently.

Every A2UI consumer hitting real streaming will rediscover these patterns.
The protocol should bless them so renderers are interoperable by default, not
by each host's best guess.

### Why not handle this outside the protocol

Proposals to solve this in skills / host / catalog (not in A2UI itself) hit
the interop ceiling: if Claude Desktop and the Gemini app and a custom web
client all use A2UI but render shimmer states differently (or not at all), the
same agent behaves differently across hosts. The streaming lifecycle is not a
host decision; it is part of what the protocol describes.

## Proposal 1 — `pending` as a first-class resolution state

### Spec change

When a path binding `{"path": "/foo"}` resolves to `undefined`, the client
MUST render a **pending placeholder** for that binding. Components declare how
their bound fields should render in the pending state.

Catalog schema gains one optional property per bindable field:

```json
{
  "component": "Text",
  "text": {
    "$ref": "common_types.json#/$defs/DynamicString",
    "pending": "shimmer"
  }
}
```

Allowed `pending` values:

- `"shimmer"` — animated pulsing line (sized to typography) — default for text
- `"spinner"` — small activity spinner — default for non-text fields
- `"skeleton"` — static gray block sized to expected content
- `"hidden"` — render nothing (collapse in layout)
- `"previous"` — keep the last-known value if any (default when path was
  previously resolved; useful for live-updating dashboards)

Default inference (when a catalog field omits `pending`):
- Text-type fields → `"shimmer"`
- Reference-type fields (component child/children) → `"hidden"`
- All others → `"skeleton"`

### Renderer obligations

Every A2UI renderer MUST:

1. Detect when a binding resolves to `undefined` at render time
2. Apply the catalog's declared pending behavior (or the default)
3. Re-render when the binding resolves to a defined value (already required by v0.9)

### Backward compatibility

No breaking change. v0.9 catalogs without `pending` fields get the default
behavior. v0.9 renderers that ignore the field keep their current (undefined)
behavior; they're just missing out on the standard visual.

## Proposal 2 — `streaming` lifecycle flag on `updateDataModel`

### What A2UI already does, and what this adds

A2UI v0.9 already supports streaming in the practical sense: the wire is
JSONL, the data model is observable, and repeated `updateDataModel` patches
at the same path re-render the bound components. Servers stream text into
fields today by sending successive sets to the same path.

**The gap is the lifecycle signal.** There is no protocol-level way for a
client to distinguish "this is the latest value, more is coming" from "this
is the latest value, it's final." Without that signal, renderers can't
reliably:

- Show a typewriter caret (when do you stop animating?)
- Disable an action button whose label depends on a still-streaming field
  (every renderer reinvents the "are we done yet" heuristic)
- Announce "loading" to screenreaders and switch to "complete" at the right
  moment
- Decide when client-side validation should run on a bound input

This proposal adds the lifecycle signal — not streaming itself.

### Spec change

Add an optional `streaming` boolean to the `updateDataModel` message:

```json
{
  "version": "v0.9",
  "updateDataModel": {
    "surfaceId": "msg_1",
    "path": "/reply/headline",
    "value": "Before anything else —",
    "streaming": true
  }
}
```

Semantics:

- `streaming: true` — the value at this path is **partial**; more patches are
  expected for the same path. Renderers MAY show a typewriter cursor, animated
  caret, or other "live" indicator.
- `streaming: false` (or omitted — the v0.9 default) — the value is final.
  Remove any live indicators.
- A server sending a `streaming: true` patch MUST eventually send a
  `streaming: false` patch to the same path (or close the surface via
  `deleteSurface`). Renderers MAY time out and clear indicators after a
  reasonable inactivity window (suggested: 5s).

### Use cases

- **Typewriter effect**: catalog field's `pending` is `shimmer`; once streaming
  starts, renderer switches to a caret + streamed text; on `streaming:false`
  the caret disappears.
- **Accessibility**: screen readers announce streaming text progressively and
  know when to stop polling for updates.
- **Action guards**: buttons with path-bound event names stay disabled while
  `streaming: true`, become enabled on `streaming: false`. (Replaces the
  ad-hoc "check label AND event.name resolve" pattern we wrote by hand.)

### Lifecycle diagram

```
Client render              Wall     Wire message
                           clock
──────────────────────────────────────────────────────────────────────────
┌─────────────────────┐    T=0      updateComponents: {
│  ░░░░░░░░░░░░░░░░   │              id:"card", component:"Card",
│  ░░░░░░░░░░░░░      │                title: {path:"/r/title"},
│  ░░░░░░░░░░░░░░░░   │                body:  {path:"/r/body"} }
└─────────────────────┘             [shimmer per Proposal 1]
                                    
┌─────────────────────┐    T=0.6s   updateDataModel {
│  Hello ▌            │              path:"/r/title", value:"Hello",
│  ░░░░░░░░░░░░░      │              streaming:true }
└─────────────────────┘             [caret appears; shimmer removed]

┌─────────────────────┐    T=1.2s   updateDataModel {
│  Hello, world ▌     │              path:"/r/title", value:", world",
│  ░░░░░░░░░░░░░      │              streaming:true, patch:"append" }
└─────────────────────┘             [text grows, caret stays]

┌─────────────────────┐    T=1.4s   updateDataModel {
│  Hello, world       │              path:"/r/title", streaming:false }
│  ░░░░░░░░░░░░░      │             [caret disappears; title final]
└─────────────────────┘

┌─────────────────────┐    T=1.6s   updateDataModel {
│  Hello, world       │              path:"/r/body", value:"Nice...",
│  Nice to see you ▌  │              streaming:true }
└─────────────────────┘             [body caret appears, body shimmer gone]

... body streams ... T=3.0s: body streaming:false ... caret gone ... done.
```

Three discrete visual states per field: `pending` → `streaming` → `final`.
Each renderer can style each state its own way, but the lifecycle is
protocol-guaranteed.

### Backward compatibility

No breaking change. Pre-v1.0 servers don't emit the flag → renderers treat
everything as final (current v0.9 behavior). v0.9 renderers ignoring the flag
also treat everything as final — no regression.

## Proposal 3 — `append` patch operation on `updateDataModel`

### Spec change

Add an optional `patch` field selecting the update op. Default `"set"`
(current v0.9 behavior, overwrite value at path).

```json
{
  "version": "v0.9",
  "updateDataModel": {
    "surfaceId": "msg_1",
    "path": "/reply/headline",
    "patch": "append",
    "value": "else — see a derm.",
    "streaming": true
  }
}
```

Allowed `patch` values:

- `"set"` (default) — replace value at path. Current v0.9 behavior.
- `"append"` — if the existing value at path is a string, append `value`
  (must also be a string). If the path is missing or not a string, initialize
  to `value`. (Arrays: append item. Objects: shallow merge.)
- `"prepend"` — mirror of append, for prepending (rarer, but useful for
  logs/timelines).
- `"remove"` — remove the path. Used for dismissing / clearing.

### Motivating benchmark

Bytes on the wire for streaming a text field via 100 deltas, measured as
(path + envelope + payload) summed across all messages:

| Text size | `set` (current v0.9) | `append` (proposed) | Ratio |
|---|---|---|---|
| 100 chars | 5.0 KB | 0.3 KB | 16× |
| 500 chars | 25.0 KB | 0.8 KB | 31× |
| 2,000 chars (long rationale / article) | 100 KB | 2.3 KB | 43× |
| 10,000 chars (full chat response) | 500 KB | 10.3 KB | 48× |

The overhead scales with *message count × accumulated length* under `set`,
but only with *total characters* under `append`. Longer text = wider gap.

Corollary: `set` is fine for short fields (a headline, a button label).
`append` is essential for long fields (rationales, articles, code blocks,
chat transcripts). Most servers will use both — `set` for static or final
values, `append` for mid-stream text.

### Backward compatibility

`patch` field is optional. Servers that don't send it get `"set"` behavior
(v0.9-compatible). Renderers that don't implement `append` can either:

1. Fall back to fetching the current value and concatenating (easy), or
2. Declare non-support via capability negotiation (see "Discovery" below)

### Discovery (bonus, not core to this RFC)

v0.9 defines `a2uiClientCapabilities` as part of the A2A handshake. We suggest
adding:

```json
{
  "supportedPatchOps": ["set", "append", "prepend", "remove"]
}
```

So servers know what to emit. If a client doesn't support `append`, the
server can fall back to `set` for that connection. This belongs in a
separate capability-negotiation RFC; mentioning here for completeness.

## Proposal 4 — Write bindings + origin tracking

### What A2UI v0.9 already does informally

A2UI v0.9 supports client-side data-model writes today, but only as a side
effect of input components. `TextField` writes the user's typed value to
its bound path. `Slider` writes the value the user dragged. `CheckBox`
writes the checked state. `ChoicePicker` writes the selection. These are
all bidirectional bindings, and they work — but the protocol doesn't
formalize the concept. Each input component just happens to mutate locally
because its widget implementation does.

### The gap

Three problems with the informal status quo:

1. **No vocabulary for "this binding writes back" outside of basicCatalog
   input components.** A custom `PaginatedTable` that wants to update
   `/page_meta` on user pagination has no protocol-level way to declare
   "this is a write binding." The semantics live in the widget code.
2. **`sendDataModel` is the wrong abstraction.** It's all-or-nothing
   (true echoes everything including state the server itself wrote;
   false echoes nothing including state the user just changed). The
   real question — "what did the user mutate via UI?" — is hidden under
   a coarse switch.
3. **Pattern B (paginated table with server-fetched buffer, client-side
   page navigation) has no clean expression.** Either the server ships
   100 rows of buffer back to itself on every turn, or the agent has no
   way to know what page the user is on.

### The proposal

Three small additions that compose into one solution:

#### (a) Explicit write bindings

A path binding may carry an optional `write` flag:

```json
{
  "component": "PaginatedTable",
  "rows":     { "path": "/buffer" },                          // read-only
  "page":     { "path": "/page_meta/page", "write": true },   // read + write
  "visible":  { "path": "/visible_drinks", "write": true }    // read + write
}
```

| Form | Semantics |
|---|---|
| `{ "path": "/x" }` (default) | Component reads `/x` and re-renders on change. Cannot mutate. |
| `{ "path": "/x", "write": true }` | Component reads `/x` and MAY mutate it. Mutations propagate through the data model exactly like server-sent `updateDataModel` patches: downstream bindings observe and re-render. |

This formalizes existing input-component behavior — `TextField`, `Slider`,
`CheckBox`, `ChoicePicker` are retroactively `write: true` on their value
bindings. No breaking change.

#### (b) Origin tracking

The host MUST track the origin of every data-model entry:

- **Server-origin** — set by an `updateDataModel` message from the server
- **Client-origin** — set by a component via a write binding

Origin is per-path and last-write-wins. If the server sets `/foo` and then
a component mutates `/foo`, the entry becomes client-origin. If the server
re-sends, it flips back to server-origin.

Hosts maintain this metadata internally; it never appears in catalog
schemas or user-facing JSON.

#### (c) Origin-aware echo

`sendDataModel` gains a third mode:

```json
{
  "version": "v0.9",
  "createSurface": {
    "surfaceId": "menu",
    "catalogId": "starter/cafe-companion@v0.2",
    "sendDataModel": "client-origin"
  }
}
```

| Value | Echoed in client → server messages |
|---|---|
| `false` (default, v0.9) | Nothing |
| `true` (v0.9) | Entire data model — all origins |
| `"client-origin"` (new) | Only paths the client has mutated since the last server message |

The agent automatically sees what the user changed via UI without the
server having to subscribe to anything, ship its own state back to
itself, or maintain a separate exposure list.

### Worked example: Pattern B (paginated table with server-fetched buffer)

A cafe-companion shows a menu of 100 drinks, fetched in one server call
but paginated 20 at a time client-side.

```json
// 1. Surface created with origin-aware echo.
{
  "version": "v0.9",
  "createSurface": {
    "surfaceId": "menu",
    "catalogId": "starter/cafe-companion@v0.2",
    "sendDataModel": "client-origin"
  }
}

// 2. Server fetches the buffer once and ships it. Server-origin.
{ "version": "v0.9", "updateDataModel": {
    "surfaceId": "menu", "path": "/buffer",
    "value": [/* 100 drinks */]
}}

// 3. PaginatedTable mounts. Its write bindings to /page_meta and
//    /visible_drinks compute initial values and write them. Client-origin.
//
//    /page_meta = { page: 1, page_size: 20 }
//    /visible_drinks = buffer[0..19]

// 4. User clicks "Next page" four times. Each click is a pure client-side
//    pagination — the component mutates /page_meta + /visible_drinks. No
//    agent round-trip. The new entries stay client-origin.

// 5. User types "tell me about the 3rd one". Client → agent:
{
  "type": "userMessage",
  "text": "tell me about the 3rd one",
  "dataModel": {
    "menu": {
      "page_meta":      { "page": 5, "page_size": 20 },     // client-origin
      "visible_drinks": [ /* 20 rows of page 5 */ ]         // client-origin
      // /buffer is NOT included — it's server-origin
    }
  }
}
```

The agent has exactly the state it needs to resolve "the 3rd one" to
`visible_drinks[2]` with page context from `page_meta.page`. The 100-row
buffer never travels across the wire to the agent.

No exposure list to maintain. No data-layout coupling. The right state
ships because the right entity (the component) wrote it.

### Composition with Proposals 1–3

Orthogonal. Write bindings don't change what's rendered (Proposal 1),
when values arrive (Proposal 2), or how patches accumulate (Proposal 3).
Origin tracking is a metadata concern; the streaming lifecycle continues
to apply to all paths regardless of who wrote them.

### Backward compatibility

- v0.9 input components keep working unchanged. Their writes were always
  implicit; the spec just formalizes them. Pre-v1.0 catalogs that don't
  declare `write: true` on input components are interpreted as if they
  did (legacy compatibility rule for basicCatalog input components).
- `sendDataModel: true` and `false` semantics unchanged.
- `"client-origin"` is opt-in. Surfaces that don't request it get v0.9
  behavior.
- Pre-v1.0 hosts that don't understand `"client-origin"` SHOULD treat
  unknown values as `false` (default-deny — safer than default-allow).

### Why not selective path exposure (`sendDataModel: ["/a", "/b"]`)?

An earlier draft of this proposal used a JSON Pointer array. Origin
tracking subsumes it and is strictly better:

- **No list to maintain.** Add a new write binding → it auto-echoes.
  No risk of forgetting to update the exposure list.
- **No coupling to data layout.** Refactor `/page_meta` to
  `/menu/state/page` and nothing breaks.
- **Captures the actual semantic.** "What did the user change via UI?"
  is the question the agent needs answered. Origin tracking answers it
  directly; path enumeration is a proxy.
- **Subsumes the use cases.** Privacy (credit-card field) — that field
  has no write binding, so it stays out. Token thrift — server-origin
  caches stay out. Pagination — client writes are echoed.

A future revision MAY add path-array form as an additional override for
edge cases (e.g., echoing server-set state on demand). Not needed for v0.1.

### Open questions

- **Delta echoes vs full echoes.** Should client-origin echoes ship the
  *new value at the path* or the *diff between the last server message
  and now*? Diff is more efficient for large objects; full value is
  simpler. Recommend full value for v0.1, deltas as a v1.1 optimization.
- **Lifecycle of client-origin markers.** Are markers cleared when the
  echo ships, or kept until the next server `updateDataModel` overrides
  them? The proposal says "since last server message" — needs to be
  precise about what counts as "last."
- **Component-internal state vs data-model writes.** A component may
  also have purely-internal state that never touches the data model
  (e.g., hover focus). Spec is silent; that's a component-design concern,
  not a protocol concern.
- **Reactivity of write bindings.** A component's write to `/page_meta`
  causes a re-render of all bindings to `/page_meta`. Does it also cause
  the component's OWN write binding to re-evaluate? Recommend yes;
  components should treat their own writes as just another data-model
  patch.

## Composition

The three proposals work together. End-to-end lifecycle for a single
streaming text field:

```
# Server emits a skeleton: a Card with path-bound title.
updateComponents → { "id": "card", "component": "Card",
                     "title": { "path": "/reply/title" } }

# Client resolves /reply/title to undefined → renders shimmer per Proposal 1.

# Server starts streaming:
updateDataModel → { path: "/reply/title", value: "Welcome",
                    streaming: true, patch: "append" }
# Client appends (Prop 3), swaps shimmer for caret + text (Prop 2).

updateDataModel → { path: "/reply/title", value: " back to your dashboard",
                    streaming: true, patch: "append" }
# Text grows; caret still visible.

updateDataModel → { path: "/reply/title", streaming: false }
# Value omitted → no content change; just flag flip.
# Client hides caret. Final state.
```

Each proposal stands alone. Combined, they formalize the entire streaming
lifecycle.

## Open questions

- **Alternative to `streaming:false` with empty value**: use a dedicated
  `endStream` message type? (Cleaner, but adds surface area.) Current proposal
  favors the flag-flip approach to minimize new message types.
- **Should `pending` be per-instance overridable?** Currently declared only in
  catalogs. A per-component `{pending: "spinner"}` override would allow
  agents to customize; adds complexity. Recommend keeping catalog-only for v1.0.
- **Timeout semantics for orphaned `streaming: true`**: spec suggests 5s
  inactivity. Configurable? Client-vendor-specific? Probably the latter, with
  spec-level SHOULD guidance.
- **Batching**: should a server be able to send multiple `patch` ops in one
  message? Currently each op is a separate message. Fine for clarity;
  potential future optimization.

## Migration path

1. **v0.10 draft** includes all three proposals as optional fields. Reference
   renderers (Flutter, Lit, Angular, React) gain support.
2. **v1.0 stable** promotes them to recommended. Default `pending` behavior
   is mandatory; flag + append are optional but recommended.
3. **v1.1+** could mandate append support for text fields based on adoption.

## Prior art

- **GraphQL `@defer` / `@stream`** — handles progressive response in query
  results. Different target (data fetching, not UI), but the streaming
  lifecycle is analogous.
- **HTMX `hx-swap-oob` + SSE** — progressive HTML swapping. Proves the
  server-driven progressive render pattern at web scale.
- **React Suspense / RSC** — has `pending` / `ready` as first-class component
  states. Inspiration for Proposal 1.
- **Anthropic Messages API `input_json_delta`** — progressive tool-use JSON
  streaming. The upstream source of most of this need.

## Non-goals

This RFC does NOT propose:

- **Skeleton / template as a first-class surface concept.** A skeleton is
  just a normal `updateComponents` message emitted server-side before data
  arrives. Whether to use a skeleton is a host/skill concern, not A2UI's.
- **Protocol-level knowledge of LLM providers.** Proposals are source-agnostic:
  Claude tool_use, Gemini structured output, hand-coded servers all use the
  same wire format.
- **Component-level macros / composition.** Separate RFC.
- **Catalog registries / capability negotiation.** Separate RFCs.

---

*A reference implementation (Lit web client + Python agent host, Claude-brain)
ships a shimmer-on-pending pattern matching Proposal 1 and a full-value
`updateDataModel` streaming pattern matching Proposal 3 — both hand-rolled
today because the spec doesn't bless them yet. Link available on request;
omitted from the public draft to keep the proposal framework-agnostic.*
