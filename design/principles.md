# Principles

## Calm by default

Restrained and quiet, "simple, with character". The default register for every app type except
[playful tools](./playful-tools.md). Character comes from one deliberate element inside the calm
register, not from decoration spread across the screen.

## The words rule

If a screen needs helper text, taglines or secondary labels to make sense, fix the layout instead.
Words stay for four cases:

1. error messages;
2. destructive confirmations;
3. form field labels;
4. empty states with one action.

Cutting a visible label never removes the accessible name (`aria-label`, role, button text). Agents
and assistive technology operate the UI by accessible name; a control that lost it is unusable to both.

## Tokens: one source of truth

While the project has no tokens in code, the `design-brief` holds them. Once code has them (CSS
variables, a Tailwind `@theme`, a theme file), code is the source of truth and the brief names the file
and is regenerated from it. One line in the brief says which: `Tokens source: brief` or
`Tokens source: <path>`.

## Components

### Secondary surfaces reuse the page's primary and secondary buttons
- **When:** a banner, modal, toast or notice has actions. Its main action uses the page's primary button and its other action uses the page's secondary button: the same component and variant.
- **When not:** never a button style made just for that surface.
- **Reference:** [wachiman.dev](https://wachiman.dev) cookie banner: Accept uses the "Install for free" style, Decline uses the "Read the docs" style. Owner's words, translated: "the main CTA must have the page's styles."
- **Since:** 2026-11-25. Refined: —

## Precedence and floor

Brief > this profile > `frontend-design`, inside platform conventions and above an accessibility floor.
See the [README](./README.md#precedence).
