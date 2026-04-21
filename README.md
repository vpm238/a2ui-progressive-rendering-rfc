# a2ui-progressive-rendering-rfc

**A draft RFC proposing three protocol additions to [A2UI](https://a2ui.org/) v1.0:** `pending` state for unresolved bindings, a `streaming` lifecycle flag on `updateDataModel`, and an `append` patch op for efficient text streaming.

Includes a **[live demo](https://vpm238.github.io/a2ui-progressive-rendering-rfc/)** you can interact with — every UI change is paired with the exact A2UI wire message that would produce it.

## The gap this addresses

A2UI v0.9 treats UI description as a single transaction: server says "here's the UI," client renders it. In practice, agents generate UI progressively — skeleton first, data streaming in over seconds. Today there is no standard way to:

1. Render **pending** bindings before their data arrives (every client invents its own).
2. Signal that a value is **mid-stream** vs. **final** (so renderers can show typewriter cursors, disable incomplete actions, announce accessibility updates).
3. **Append** to long streamed text without re-sending the whole accumulated value each delta (a 10,000-char response today takes ~500 KB; with append, ~10 KB).

The three proposals are all backward-compatible and address these in minimal surface-area additions.

## Files

| File | What |
|---|---|
| [`RFC.md`](RFC.md) | The draft proposal. Wire-format examples, ASCII lifecycle diagram, migration path, non-goals. |
| [`index.html`](index.html) | Fully client-side demo — no backend, no build step. Shows each proposal's effect on a rendered surface with synchronized wire log. Hosted via GitHub Pages at the URL above. |

## Running the demo locally

Just open the file. Zero dependencies, zero build step:

```bash
git clone https://github.com/vpm238/a2ui-progressive-rendering-rfc
cd a2ui-progressive-rendering-rfc
open index.html   # macOS; or any static server
```

Or view hosted: **https://vpm238.github.io/a2ui-progressive-rendering-rfc/**

## Status

This is an **experience-report-style proposal**: it's grounded in building one production A2UI application (a Claude-powered multi-turn consultant) and the gaps we hit integrating streaming LLM output into an A2UI surface. Intended for discussion on [google/A2UI/discussions](https://github.com/google/A2UI/discussions) and the A2UI contributor community.

See [RFC.md](RFC.md) for the full proposal and reasoning.

## License

MIT. See [LICENSE](LICENSE).
