# Transactional email package: Rakoi reference pattern

Reviewed against the local Rakoi source on 2026-09-15.

For implementation steps and adaptable code examples, follow the
[transactional email playbook](./transactional-email-playbook.md).

## Reuse this pattern

Use `@scope/infra-email` for JSX email templates, rendering and delivery adapters.
Rakoi already implements this pattern with React Email and Resend; new Niway
apps should use it as a reference instead of redesigning the same layers.
This is a reusable package structure, not an instruction to import the
Rakoi-branded package directly into another product.

Sender policy is a separate concern: follow the
[Niway transactional email convention](../infra/transactional-email.md).
Existing Rakoi examples containing `rakoi.app` are not the shared sender policy.

## Existing package structure

```text
packages/infra-email/src/
  index.ts
  email-service.ts
  providers/
    email-provider.interface.ts
    resend.provider.ts
  templates/
    index.ts
    types.ts
    invitation.tsx
    billing-beginning.tsx
    billing-mid-month.tsx
    billing-end-month.tsx
    utils.ts
  __tests__/
```

- **Public exports:** JSX components such as `InvitationEmail`, their data
  interfaces, template identifiers, registry, `EmailService`, `IEmailProvider`,
  `EmailMessage` and `ResendEmailProvider`.
- **Typed template selection:** `EMAIL_TEMPLATE_VALUES` defines identifiers;
  `TemplateDataMap` associates each identifier with its required data.
  `EmailTemplateMap` associates each identifier with a subject function and a
  component returning `JSX.Element`.
- **Rendering service:** `sendEmail<T extends EmailTemplate>(template, to,
  data: TemplateDataMap[T])` selects the template, computes the subject and
  renders both HTML and plain text using `@react-email/render`. The two renders
  run with `Promise.all` before one provider call.
- **Delivery contract:** `IEmailProvider.send(message)` accepts `to`, `from`,
  `subject`, `html` and `text`. The interface currently lives inside infra-email.
- **Resend adapter:** wraps the Resend SDK and throws when the SDK returns an
  error. Provider acceptance does not by itself prove inbox delivery.
- **Composition:** the server constructs `EmailService(provider, from)` using
  runtime configuration. Templates do not read credentials.

The package uses `@react-email/components`, `@react-email/render`, React and
Resend. Its workspace export points to source; its publish configuration points
to built output. Follow the consuming project's existing
[package build strategy](./shared-package-build-strategy.md), and resolve
versions for that project instead of copying historical version pins.

## Verified consumers and limits

`apps/rakoi-api/src/cron/billing-cron.ts` constructs the service from
`RESEND_API_KEY` and `RESEND_FROM_EMAIL` and sends the three billing templates.
The invitation JSX component is exported, registered and exercised by service
and template tests. A source search found no production invocation of the
`INVITATION` template in Rakoi's apps or packages at review time. Do not describe
invitation delivery as verified end to end based only on the template existing.

Rakoi's invitation includes both a code and a link. Those are invitation-specific
product choices, not requirements for verification or password recovery emails.

## Applying the pattern to Kaipu

Reuse the structure in `@kaipu/infra-email` with Kaipu-specific verification,
password reset and beta approval templates. The agreed auth flows use web links,
not six-digit codes; do not copy the invitation's code UI.

Localization is an adaptation still to implement: Rakoi currently embeds Spanish
invitation copy and English billing copy directly in templates and subjects,
and infra-email has no i18n dependency. For Kaipu, keep translated subject,
preview and body copy in `@kaipu/i18n` (es/en), while JSX layout and rendering
stay in infra-email. Select the locale explicitly with a documented fallback.
Do not report this as an existing Rakoi capability.

Keep React and provider SDKs out of domain/application. If application use cases
need to request email delivery, define their delivery port in domain/application
and implement it in infra-email; do not import Rakoi's infra-local
`IEmailProvider` into domain. Better Auth hooks can be wired at the server
composition boundary. The existing infra-local provider abstraction separates
rendering from transport; it is not already a domain port.

Rakoi's current message interface has no Reply-To or idempotency field. Add
capabilities deliberately when required; do not imply retry, deduplication,
queueing or delivery tracking are already provided by this service.

## Verification when adopting

Rakoi has template, service and provider tests under `src/__tests__`. Use these
as references for checking subjects, populated HTML/plain text and provider
error propagation. This documentation review read the implementation and tests;
it did not execute them or send email.

For a new integration, also verify localized copy, rendered links, expiry copy,
provider failure handling and runtime compatibility in its actual server target.
Inspect representative rendered emails before claiming visual readiness.

## Source references

Paths are relative to `Niway/rakoi-monorepo` in the owner's workspace:

- `packages/infra-email/package.json`
- `packages/infra-email/src/index.ts`
- `packages/infra-email/src/email-service.ts`
- `packages/infra-email/src/templates/types.ts`
- `packages/infra-email/src/templates/index.ts`
- `packages/infra-email/src/templates/invitation.tsx`
- `packages/infra-email/src/providers/email-provider.interface.ts`
- `packages/infra-email/src/providers/resend.provider.ts`
- `apps/rakoi-api/src/cron/billing-cron.ts`
