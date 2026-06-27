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

### Pick a concrete component to talk about

Imagine a `PaginatedTable` of 100 drinks. The server fetches all 100 once.
The component paginates client-side, 20 at a time. Two things have to be
true for it to work:

- **Pagination is local.** When the user clicks "Next page," the component
  re-renders without a server round-trip. (Otherwise the buffer is useless.)
- **The agent still knows which page is visible** when the user later
  types "tell me about the 3rd one." (Otherwise the user's reference can't
  be resolved.)

The data model has three entries:

| Path | Who set it | Size |
|---|---|---|
| `/buffer` | server (once) | 100 rows — big |
| `/page_meta` | the component (on each pagination) | small |
| `/visible_drinks` | the component (on each pagination) | 20 rows |

Two questions A2UI v0.9 doesn't answer:

1. **How does the component update `/page_meta` and `/visible_drinks`?**
   `updateDataModel` is server-to-client. There's no formal client-to-data-model write.
2. **How does the agent see `/page_meta` and `/visible_drinks` without
   also receiving the 100-row `/buffer` it already knows about?**
   `sendDataModel` is boolean — all or nothing.

This proposal answers both.

### v0.9 already does half of this — for one component family

A `TextField` bound to `/form/name`: user types "Vishal" → the value at
`/form/name` becomes "Vishal" → bindings to that path re-render. **The
component wrote to the data model.** That works today. Every input component
in basicCatalog does it: `TextField`, `Slider`, `CheckBox`, `ChoicePicker`.

But the protocol doesn't formalize the concept. There's no JSON way to say
"this binding writes back" — `TextField` just *does* it because its widget
code mutates the bound path. Custom components (like `PaginatedTable`)
have no vocabulary to declare it. And when the form gets submitted,
`TextField`'s write travels to the agent via `sendDataModel: true` —
along with every other field on the surface, including ones the agent set.

Proposal 4 extends what `TextField` does today to be:

1. **Declarable.** Any binding can say `"write": true`.
2. **Attributable.** The host tracks who wrote each entry — server or client.
3. **Selectively shippable.** The agent receives only what the user changed.

### Spec change #1 — declare write bindings

Any path binding may carry a `"write": true` flag:

```json
{ "path": "/page_meta/page", "write": true }
```

| Form | Component can… |
|---|---|
| `{ "path": "/x" }` (default) | Read `/x`. Re-render on change. |
| `{ "path": "/x", "write": true }` | Read `/x`. Mutate it. Mutations propagate through the data model like any server-sent `updateDataModel` patch. |

`TextField`, `Slider`, `CheckBox`, `ChoicePicker` are retroactively
`write: true` on their value bindings (their widgets already write —
the spec just formalizes it). No breaking change.

### Spec change #2 — origin tracking

The host MUST track who set each data-model entry. Server-origin entries
were set by `updateDataModel`; client-origin entries were set by a
component's write binding.

Origin is per-path and last-write-wins. If the server sets `/foo` and
then a component mutates it, the entry is now client-origin. If the
server re-sends, it's server-origin again.

Origin lives only in the host's memory. It never appears in catalog
schemas or surface JSON. It's metadata the host needs in order to do
the next part.

### Spec change #3 — origin-aware echo

`sendDataModel` gains a third mode:

```json
"sendDataModel": "client-origin"
```

| Value | The agent receives in client → server messages |
|---|---|
| `false` (default — v0.9 unchanged) | Nothing |
| `true` (v0.9 unchanged) | Entire data model — all origins |
| `"client-origin"` (new) | Only paths the client wrote since the last server message |

That's it. The agent automatically receives what the user changed via
UI. No exposure list to maintain.

### What `PaginatedTable` looks like

Now we can express it:

```json
{
  "id": "table",
  "component": "PaginatedTable",
  "rows":      { "path": "/buffer" },                          // read-only
  "page":      { "path": "/page_meta/page",   "write": true },  // writes
  "visible":   { "path": "/visible_drinks",   "write": true },  // writes
  "pageSize":  20,
  "onExhausted": { "event": { "name": "fetch_more", "context": {} } }
}
```

The widget code (each host implements its own):

1. Renders `buffer[page * 20 : (page + 1) * 20]`.
2. On "Next page": mutates `/page_meta/page` (its `write: true` binding
   makes this legal). Recomputes the visible slice and mutates
   `/visible_drinks` too.
3. The host marks both writes as client-origin.
4. No agent round-trip. The next page paints instantly.
5. When the buffer is exhausted, fires `onExhausted` → agent fetches
   more → emits `updateDataModel` for `/buffer` (server-origin) and
   the cycle continues.

### End-to-end: what the agent sees when the user types

```json
// Setup
{ "createSurface": {
    "surfaceId": "menu",
    "catalogId": "starter/cafe-companion@v0.2",
    "sendDataModel": "client-origin"
}}
{ "updateDataModel": {
    "surfaceId": "menu", "path": "/buffer", "value": [ /* 100 drinks */ ]
}}

// User clicks Next four times. Component mutates /page_meta and
// /visible_drinks each click. No client→agent messages.

// User types "tell me about the 3rd one":
{
  "type": "userMessage",
  "text": "tell me about the 3rd one",
  "dataModel": {
    "menu": {
      "page_meta":      { "page": 5, "page_size": 20 },   // ⬜ client-origin
      "visible_drinks": [ /* 20 rows of page 5 */ ]        // ⬜ client-origin
      // /buffer NOT included — server set it; agent already knows it
    }
  }
}
```

Agent resolves `visible_drinks[2]` to the 3rd drink shown on the user's
screen. The 100-row buffer never crosses the wire to the agent.

### Backward compatibility

- v0.9 input components keep working unchanged.
- `sendDataModel: true` and `false` semantics unchanged.
- `"client-origin"` is opt-in. Pre-v1.0 hosts that don't recognize it
  SHOULD treat unknown as `false` (default-deny — safer than default-allow).

### Why not just enumerate paths?

The earlier draft of this proposal had
`sendDataModel: ["/page_meta", "/visible_drinks"]` — explicit JSON
Pointer paths. Origin tracking is strictly better:

- **No list to maintain.** Add a new write binding → it auto-echoes.
- **No data-layout coupling.** Refactor `/page_meta` → `/menu/state/page`
  and nothing breaks.
- **Captures the semantic directly.** "What did the user change via UI?"
  is the question the agent needs answered. Origin answers it. Path
  enumeration is a proxy.

### Open questions

- **Echo granularity.** Ship the new value at the path, or the diff
  since the last echo? Diff scales better; full value is simpler.
  Recommend full value in v0.1, diffs as v1.1.
- **Marker lifecycle.** Are client-origin markers cleared on echo, or
  kept until the next server `updateDataModel` overrides them?
  Recommend cleared on echo, so each user message ships only what's new.
- **Conflict resolution.** Server `updateDataModel` arrives while a
  path is client-origin — server's value wins and the path flips back
  to server-origin.

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
