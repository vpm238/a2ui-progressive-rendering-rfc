# Draft discussion post — for google/A2UI/discussions

**Category:** 💡 Ideas (or 💬 General — Ideas feels closer)

**Title:**
`Experience report + draft RFC: three progressive-rendering primitives (pending, streaming flag, append op)`

**Body:**

---

Hi all 👋 — first-time poster here. I've spent the past few weeks building A2UI agents on both native (SwiftUI) and the browser (using `@a2ui/lit` + a custom catalog extension), and I kept running into the same three gaps around **progressively rendering LLM-streamed data** into surfaces. I wrote them up as a small draft RFC with a live, interactive demo, and wanted to share it here for feedback before taking it further.

There's meaningful overlap with @Emiyaaaaa's [#727 (Incremental Streaming Support via `updateDataModel`)](https://github.com/google/A2UI/discussions/727) — that post was actually the nudge that made me write this up as something shareable rather than vendor-specific notes.

## The three gaps

Each of these is solvable per-client today, but inconsistent across clients. Today every renderer invents its own answer; I'd like the protocol to nail down a default.

### 1. Pending state for unresolved bindings
A surface can arrive ahead of its data (skeleton-first). `{path: "/reply/headline"}` resolves to `undefined` for a few seconds. Clients render this as blank / spinner / shimmer / error / "null" — whatever the author thought of. Proposal: standardize on a shimmer placeholder as the default behavior, with an explicit opt-out.

### 2. A `streaming` lifecycle flag on `updateDataModel`
While a value is mid-stream vs. final, renderers need to know. Typewriter cursor, disabled actions, a11y live-region announcements — all of these want a protocol signal, not a heuristic. Proposal: an optional `streaming: true/false` field on `updateDataModel`.

### 3. An `append` patch op
This is basically @Emiyaaaaa's #727. Streaming a 10,000-char response today takes ~500 KB on the wire because every delta re-sends the accumulated value. With `append`, ~10 KB. I've written it up as part of the same document because it completes the picture with the other two — happy to fold it back into #727 if that's preferred.

## All three are additive and backward-compatible

v0.9 servers/clients ignore the new fields and keep working. An upgraded client against a v0.9 server degrades gracefully. No breaking changes.

## Live demo + implementations

- **Interactive demo:** https://vpm238.github.io/a2ui-progressive-rendering-rfc/
  Each proposal has a stage showing the rendered UI on the left and the exact wire messages on the right — poke the buttons, see the effect + the bytes saved.
- **Draft RFC document:** [`RFC.md`](https://github.com/vpm238/a2ui-progressive-rendering-rfc/blob/main/RFC.md)
- **End-to-end implementations** (so the proposals aren't theoretical):
  - [vpm238/a2ui-starter-swiftui](https://github.com/vpm238/a2ui-starter-swiftui) — reference Swift app, all 3 primitives wired up
  - [vpm238/a2ui-starter-web](https://github.com/vpm238/a2ui-starter-web) — uses the official `@a2ui/web_core` + `@a2ui/lit` with a custom extended catalog, live at https://vpm238.github.io/a2ui-starter-web/
  - The underlying libraries: [a2ui-swiftui](https://github.com/vpm238/a2ui-swiftui), [a2ui-skills-swiftui](https://github.com/vpm238/a2ui-skills-swiftui)

## What I'd love feedback on

1. Are these three primitives the right cut, or are there obvious adjacent gaps I missed?
2. For the `streaming` flag — should it also cover `updateComponents` (for progressive tree growth), or is data-only the right scope?
3. Anyone else hitting #1 (pending state) and building around it? Curious how Flutter / Angular renderers handle it today.
4. @Emiyaaaaa — happy to converge #727 with this if you'd prefer, or keep them separate as "narrow proposal" + "integrated RFC."

Thanks for reading — excited to be part of the conversation.

---

## Notes for posting (NOT part of the post)

- Remove this "Notes" block before pasting
- Pick category: 💡 Ideas (maps best)
- Don't cross-post to Show & Tell — one post, one conversation
- After posting, give it ~48h before any follow-ups
- If sunnypurewal shows up and asks about the SwiftUI renderer naming overlap, the honest answer is: their renderer covers the basicCatalog; mine is scoped to the 3 progressive-rendering primitives + 2 agent-specific components (OptionsGrid, RichMessageCard). Different design goals, room for both.
