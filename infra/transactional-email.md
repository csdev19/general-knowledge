# Transactional email for Niway apps

Decision confirmed by the owner: 2026-09-15.

## Scope and decision

Apps belonging to Niway use Resend with `updates.niway.dev` as their shared
transactional sender domain. This includes Kaipu and Ripuy; do not generalize
this convention to products outside Niway without an explicit decision.

Use an app-specific sender and a configurable environment variable:

| App | Sender (`AUTH_EMAIL_FROM`) |
| --- | --- |
| Ripuy | `Ripuy <no-reply-ripuy@updates.niway.dev>` |
| Kaipu | `Kaipu <no-reply-kaipu@updates.niway.dev>` |

The Kaipu address records the agreed naming convention, not a claim that its
email integration is already deployed. Niway's public contact remains
`contacto@niway.dev`; it can be used as Reply-To when replies should reach support.

Verification codes, password recovery and account/access notices use this
transactional channel. Keep marketing/newsletter sending outside this channel.
The intent is to protect transactional delivery from unrelated bulk-mail
activity and reuse the Niway email setup. This is an operating decision, not a
guarantee against spam classification or complete reputation isolation.

Do not propose moving each app to its own sender domain merely because its
website uses a different domain. A previous assistant comment in Ripuy suggested
that migration; the owner clarified that it was advice, not an accepted decision.
The `updates` label alone is not evidence that this channel sends newsletters.

## Implementation boundary

Keep the sender configurable. Reuse this convention when wiring each app's
email provider; do not infer that DNS authentication, a Resend domain or an
individual deployment has been verified from this document alone.

This documentation update does not modify DNS, provider settings or running
applications. Check the actual sending configuration before changing any SPF,
DKIM or DMARC records; do not overwrite existing records based on a generic
setup example.

## References

- [Niway public contact](../legal/README.md)
- [DNS preservation during domain migration](./custom-domain-migration.md)
- Existing implementation: `trip-planner/packages/backend/convex/lib/email.ts`
- Ripuy operational notes: `trip-planner/apps/docs/src/content/docs/changelog/2026-08-19-password-recovery-and-visibility.mdx`

## Package implementation

Use the [Rakoi email package pattern](../packages/transactional-email-package.md) for exported JSX templates, typed data, rendering and provider composition.

For a new project, follow the [email package creation playbook](../packages/transactional-email-playbook.md).
