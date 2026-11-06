# Large views: keep render cost local, and find the views that are not

*How to architect a big React screen (an editor, a dashboard, a workspace) so one control's change re-renders only what depends on it, the measurement that proves it, and a supervisor that finds the views that grew past the line.*

A screen with many panels, lanes and inspectors tends to become one component that owns every
piece of state. It works until the data gets big: then a slider at one corner rebuilds thousands
of DOM nodes at the other, and the app "feels slow" with nothing in the profiler pointing at a
single cause. The cause is structural: **one component, one state, one render tree**.

## The symptom and the diagnosis

- A continuous control (slider, drag handle, scrubber) lags only on large inputs. On the demo
  fixture it feels fine.
- The expensive work you suspect first (a simulation, a path builder) measures cheap, and is
  already coalesced per frame.
- The page component is over ~1 000 lines, calls more than ~10 feature hooks, and nothing below
  it is wrapped in `memo`.

Confirm before touching code: count renders, not milliseconds. See **Measure** below. A page that
re-renders every child once per pointer move is the diagnosis; the exact milliseconds matter less.

## The architecture: zones, slices, stable edges

Three rules, in the order they pay off.

1. **Stable edges.** Everything a child receives must keep its identity until its *meaning*
   changes. The usual offenders, in order of how often they appear:
   - the object a state hook returns (`return { scene, undo, commit, … }` is a new object every
     render; `useMemo` it over its fields);
   - callbacks that list that object as a dependency, so they are new on every render;
   - inline arrows, arrays and objects at the call site (`onSelect={(id) => …}`,
     `lanes={[…]}`, `ghost={{ … }}`);
   - inline arrows inside an intermediate component that is itself memoised, which silently
     breaks memo for everything beneath it.

   For handlers that only need the *latest* state when they run (pointer and keyboard events), a
   `useLatest(value)` ref removes the dependency: the handler reads `ref.current` and keeps one
   identity across state changes. The ref is written in a layout effect, so a child that calls
   the handler from its own layout effect would read the previous render; document that.

2. **Memo on the leaves, then on the containers.** Wrap every lane, panel and list in `memo`
   once its props are stable, keeping the export name (`export const Lane = memo(function Lane…)`).
   A container that receives a list of elements (`extraLanes`) re-renders whenever one element
   changes; that is acceptable until the next rule applies.

3. **Zones with their own subscriptions.** When memo is not enough, the page stops passing data
   down and each zone (preview, timeline, inspector, exports, shortcuts) reads the slice it needs
   from a small context or store. The page becomes composition only, a few hundred lines. This is
   the step that needs a design decision and, if it introduces a store the repo does not have, an
   ADR.

Do them as separate PRs. Rule 1 plus 2 is a behaviour-preserving PR with a measurement; rule 3
changes the tree and is reviewed on its own.

### Continuous controls hand the model one update per frame

A controlled slider that forwards every `change` event to the model forces one full render per
event, and a pointer fires several per frame. Coalesce **at the control**, not in the model:

- keep a local draft while dragging, so the input shows the pointer position immediately;
- queue the latest value and schedule one `requestAnimationFrame`;
- queue the *handler with the value*. The inspector above may not be keyed by item, so a selection
  change mid-drag swaps the handler under a mounted control, and the pending value must reach the
  item it was dragged on;
- on pointer up, cancel or unmount: cancel the frame, flush the pending value, then end the
  gesture, then clear the draft. Undo semantics stay "one step per drag, last value wins".

Coalescing in the model instead (deferring the state set) hands hooks that read the model at call
time a stale value within the frame. Only do that when every writer reads through a ref.

## Measure: a render-count test that stays as a guard

Milliseconds in jsdom are only a ratio. Render counts are exact and deterministic, so the
measurement doubles as the regression guard:

1. Build a **large synthetic fixture** in the test and assert its size (thousands of marks, ~100
   segments), so the test proves it is large and not a toy.
2. Mock each leaf module with a plain counting wrapper around the real export
   (`vi.mock(path, async (orig) => ({ …, Lane: (p) => { count("lane"); return <Orig {...p}/> } }))`).
   A plain wrapper counts "the parent re-rendered this child", which is the number the
   architecture controls.
3. Drive the control: pointer down, N `change` events, **one stubbed animation frame per event**
   (run only the control's frame; drop the page's own queued frames so the number is render cost,
   not the simulation), pointer up.
4. Assert: unrelated leaves `0`; the dragged leaf `> 0` and `≤ N + k` (lower bound too, or a leaf
   that silently stops updating passes); the result still holds (last value shown, one undo step).
5. Print one evidence line behind an env flag for PR reports; keep CI logs quiet.

Record the baseline **from the test alone, before the change**, in the test's first commit
message. The two numbers go in the PR as evidence (see
[evidence-driven optimization](../conventions/evidence-driven-optimization.md)).

### Measured example (owner's desktop video editor, 2026-11-06)

Synthetic one-hour recording: 72 000 cursor samples, 1 800 clicks, 100 zoom segments; 60 moves
of the zoom Level slider. Counts are renders per lane.

| | activity | zoom | audio | privacy | strip | ms (median of 3, jsdom) |
| --- | --- | --- | --- | --- | --- | --- |
| before (one state, no memo) | 63 | 63 | 63 | 63 | 63 | 1 144 |
| after rules 1 + 2 + control coalescing | 0 | 60 | 0 | 0 | 63 | 268 |

The strip still renders once per frame because the dragged lane's element lives in its
`extraLanes` prop; rule 3 removes that. The suspected cause, the camera-path simulation, measured
3 to 7 ms per rebuild on real tracks and was already coalesced: measuring first avoided
optimising the wrong thing.

## A supervisor for oversized views

A view does not announce when it has grown past the line. Run a supervisor in CI or as a weekly
report that lists candidates, and treat each as a ticket to design, not a lint error to silence.
Signals, in order of how well they predict the problem:

| Signal | Threshold that has mattered | Why |
| --- | --- | --- |
| Component body length | > 800 lines | Past this, nobody holds the whole state in their head |
| Feature hooks called from one component | > 8 | The component is the composition root of too many features |
| `useState` + `useCallback` + `handle*` in one component | > 40 combined | Logic that belongs to zones is living in the page |
| Children rendered with inline arrows/objects/arrays | any, under a component that renders lists | Memo below is already broken |
| Leaves rendering collections with no `memo` | any lane/list over data that can be large | The next continuous control will rebuild them |
| Hook returning a fresh object literal | any hook whose result goes in dependency lists | Every dependent callback is unstable |

Implementation that is enough: a script over the renderer tree (ts-morph or a `@typescript-eslint`
rule set) that reports the table per file, plus a render-count test per large screen as in
**Measure**. The report lands in the repo's briefings, and a row that crosses a threshold opens a
design ticket whose first deliverable is the measurement, not the refactor.

What the supervisor must not do: auto-split components. Where a cut goes (zones, slices) is a
design decision with trade-offs, taken in a design conversation with options and gaps.

## Related

- [evidence-driven optimization](../conventions/evidence-driven-optimization.md): evidence,
  budget, then optimisation; the render-count test is the evidence row.
- [renderer architecture](../desktop/renderer-architecture.md): pages compose, features own
  blocks; the zones above are that rule applied when data gets large.
- [verifiable handoffs](../conventions/verifiable-handoffs.md): before/after numbers belong in the
  PR's Proof table.
