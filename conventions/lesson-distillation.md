# Lesson distillation

**A lesson that stays in the conversation is lost with it.** Skills and hub
pages only improve if what happened in practice flows back into them, and
that flow does not happen by itself: at the end of a session the lesson is
obvious, and by the next session it is gone.

The executable version of this convention is the
[`cs-distill-lesson` skill](https://github.com/niway-dev/skills/tree/main/skills/cs-distill-lesson). This page is the reasoning;
the skill is the procedure.

## The rule

Every lesson has exactly one home, chosen by what it is about:

| It is about | Home |
| --- | --- |
| how a step is done | the skill |
| why a rule exists, or a trade-off that turned out differently | the hub page |
| a recurring situation nothing covers | a new skill, with its page |
| this project only | the project's docs, ADR or `CLAUDE.md` |
| something that will not recur | nowhere, and said so |

Product-agnostic knowledge lives in the hub; a project's application of it
lives in the project. Never both, because two copies drift and the reader
cannot tell which one is current.

## Why the pair stays in sync

A skill is a procedure; its page is the reasoning. When a rule changes, the
procedure and the reasoning change together, in the same sitting, or the
next reader finds a skill doing something its page says not to. When only
the *how* changes, only the skill moves.

## Why the form matters

**An event, not a story.** The lesson is named once as what the skill said
versus what happened. The artifact gets the generalised form: a conditional
on something the agent can observe. "In the session where..." never
appears in a skill or a page; narrative is what the reader has to translate
back into a rule, and they will translate it differently each time.

**A conditional, not an exception.** "Unless X" appended to a rule reopens
the rule every time it is read. "If X, then Y" is a second rule that stands
on its own.

**Shape is preserved.** A lesson that does not fit an existing section is
usually classified wrong. Growth by appended sections is how a skill becomes
a document nobody follows.

## Why the change is tested

The wording that seems obvious to the person who lived the event is the one
a fresh agent skips. A changed skill is handed to an agent with no memory of
the session and the scenario that produced the lesson; if it does not act
right unprompted, the words are not binding yet.

## Related

- [verifiable-handoffs](./verifiable-handoffs.md): the PR that ships a
  distilled lesson names the event it protects against.
- [specs-and-plans-workflow](./specs-and-plans-workflow.md): a lesson that
  becomes a new skill goes through the brainstorm first.
