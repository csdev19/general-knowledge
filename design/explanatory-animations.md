# Explanatory animations: minimal first, theme after

_A recipe for building a diagram that tells a product's story step by step: get the story right on a plain version, freeze it, then skin it._

**Status:** a recipe, not a rigid rule. Proven on one project so far (a developer-tool landing page,
2026-11). Adjust it when the situation asks for it, and refine this page after a second project.

Unlike the app-type pages in this folder, this page is a process, not a taste entry. It holds no
visual preference of its own: the theme in step 5 comes from the project's design brief and the
[principles](./principles.md).

## The process

1. **Start from the idea alone, in words.** Write down the actors, what flows between them, the order
   of the steps, and the one thing the viewer must understand at the end. Ask only about missing
   relationships that change the story.
2. **Build a minimal, neutral, single-file animation.** Plain shapes, one accent colour, no theme, no
   component kit. It exists only to get the story right: who acts, what travels, in what order.
3. **Refine the story with the owner in short rounds on that minimal version.** Change actors,
   connections, sequence and captions until the viewer can explain the intended relationship. The
   owner's judgment of the running animation is the milestone. Do not theme during this phase.
4. **Freeze that base as the reference.** Keep the file. It is the baseline the themed version is
   compared against, and the place to restart from if the story changes.
5. **Apply the theme as a skin over the frozen base.** Keep actors, connections, sequence and timing;
   change only the visual language. Example (illustrative): a neobrutalist skin where the active node
   lifts like a button with a hard shadow and visited nodes press flat. That contrast gives the
   sequence a rhythm the neutral version lacked.
6. **Check the states a final screenshot hides.**
   - Mobile width.
   - Light and dark.
   - Reduced motion (the story must still be readable without movement).
   - Mid-transition states: screenshots taken while the animation runs, not only the end state.
   - Label overlap, and contrast of the active state.

## What a theme may change

| Diagram meaning | Theme treatment |
|---|---|
| Which actor is active | fill, outline, shadow or lift |
| What travels between actors | connector and marker appearance |
| Order and pauses | easing and transition character, without hiding the order |
| Caption emphasis | underline, marker sweep or another readable treatment |

A theme is more than a colour swap. Fonts and shadows change dimensions, dark mode can break
contrast, and a moving highlight can fail between its start and end. Recheck the whole sequence after
skinning. Do not carry meaning by colour alone.

## Why it works

- Separating "is the story right" from "does it look right" stops theme work from hiding a wrong
  story. A polished diagram that explains the wrong thing is harder to criticise than a plain one.
- Restarting from the idea beats patching a confused diagram. When rounds on the minimal version stop
  converging, rewrite the idea in words and rebuild; the minimal file is cheap to throw away.
- A frozen base makes the skin reviewable: any change in actors or order after step 4 is a deliberate
  edit, not drift from styling.

## Anti-patterns

- **Theming first.** Styling decisions leak into the story and the wrong flow survives because it looks good.
- **Asking for a generic graph framework.** One animation does not justify scene-data, timing and token
  abstractions. Build that only when a second real use case needs it.
- **Judging only the final frame.** The failures live between frames: overlaps, a highlight that never
  reaches its target, a caption that disappears too early.

## When not to use

A static diagram, or an animation whose story is already agreed and only needs a new theme, does not
need steps 1 to 4.

## Measured example

Proven once, on one developer-tool landing page (2026-11): the minimal version fixed the actors and
sequence before any skin existed, and the themed version kept both unchanged. One project is not
evidence that the recipe transfers.

## Related

- [web-playwright.md](../verification/web-playwright.md): taking the mid-animation screenshots in step 6.
- [landing.md](./landing.md): the app type where explanatory animations usually live.
- [principles.md](./principles.md): the register and tokens the skin in step 5 comes from.
