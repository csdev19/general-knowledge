# Playbook: create a transactional email package

Reference: [Rakoi's implemented pattern](./transactional-email-package.md).
Last reviewed: 2026-09-15.

## Outcome and scope

Create `@scope/infra-email`: exported JSX templates, a typed template registry,
HTML and plain-text rendering, and an interchangeable delivery provider wired
by the server. Use this playbook in an existing monorepo or adapt the same
folders into a server application without workspace packages.

This is an implementation recipe. The companion pattern describes what Rakoi
already implements. Localization, operational controls and framework integration
below are adoption steps, not claims about existing Rakoi capabilities.

## 1. Inspect the destination before creating files

Read its repository instructions and an existing infrastructure package. Record:

| Input | Resolve before implementation |
| --- | --- |
| Package location/name | Workspace package or server-local module; replace `scope` everywhere |
| Runtime | Node, Worker or another server target; existing bundler and JSX configuration |
| Package conventions | Manager, catalog, TypeScript base config, source/dist exports and test runner |
| First consumer | One actual auth hook, invitation use case or scheduled notification |
| First template | Required data, subject, action URL, language and expiry wording |
| Sender | Configured From and optional Reply-To; provider domain setup |
| Execution lifetime | Await delivery or use the runtime's supported background mechanism |

For Niway-owned apps, use the [shared sender convention](../infra/transactional-email.md).
For other projects, use their own agreed sender. Do not copy Rakoi branding,
recipients, invite codes, quotas or historical dependency versions.

## 2. Scaffold the package

```text
packages/infra-email/
  package.json
  tsconfig.json
  src/
    index.ts
    email-service.ts
    providers/
      email-provider.interface.ts
      resend.provider.ts
    templates/
      action-email.tsx
      types.ts
      index.ts
    __tests__/
      templates.test.tsx
      email-service.test.ts
      resend.provider.test.ts
```

Use the repository's package scaffolding and
[build/export strategy](./shared-package-build-strategy.md). Include dependencies
for `react`, `react-dom`, `@react-email/components`, `@react-email/render` and
`resend`, plus the project's TypeScript/React types and test tooling. Match its
React version and dependency catalog. Verify installed SDK APIs before adapting
these examples; do not treat them as a version-pinned starter.

Enable the project's React JSX transform for `.tsx`. Expose `src/index.ts` to
workspace consumers only if their toolchain compiles dependency source; expose
built JS/types where required. Add the package to the server's dependencies and
the workspace's typecheck/build graph.

For a repository without workspaces, put this structure under a server-only
`email/` module and use relative imports in the composition example.

## 3. Define the provider boundary

`src/providers/email-provider.interface.ts`:

```ts
export interface EmailMessage {
  to: string;
  from: string;
  subject: string;
  html: string;
  text: string;
}

export interface IEmailProvider {
  send(message: EmailMessage): Promise<void>;
}
```

This interface separates rendering from transport inside infrastructure. If an
application use case needs a delivery port, define that port in domain/application
and implement it here. Never make domain import React, Resend or infra-email.
Do not add a second port unless a consumer actually needs that boundary.

## 4. Create the first exported JSX template

`src/templates/action-email.tsx`:

```tsx
import {
  Body, Button, Container, Head, Html, Preview, Text,
} from "@react-email/components";

export interface ActionEmailData {
  lang: string;
  subject: string;
  preview: string;
  heading: string;
  message: string;
  actionLabel: string;
  actionUrl: string;
  footer: string;
}

export function ActionEmail(data: ActionEmailData) {
  return (
    <Html lang={data.lang}>
      <Head />
      <Preview>{data.preview}</Preview>
      <Body style={{ backgroundColor: "#f6f9fc", fontFamily: "sans-serif" }}>
        <Container style={{ backgroundColor: "#fff", padding: "32px", maxWidth: "480px" }}>
          <Text style={{ fontSize: "24px", fontWeight: 700 }}>{data.heading}</Text>
          <Text>{data.message}</Text>
          <Button href={data.actionUrl}>{data.actionLabel}</Button>
          <Text>{data.footer}</Text>
        </Container>
      </Body>
    </Html>
  );
}
```

This generic action template is a starting point, not a required abstraction for
all emails. Add dedicated components and typed data for invitations, receipts or
other structurally different messages. Use trusted application-generated action
URLs; validate external input at the caller boundary. Templates display URLs and
expiry copy, but do not generate or validate auth tokens.

If the project supports multiple languages, resolve subject, preview, body,
button and footer from its i18n package before calling the service. Supply `lang`
explicitly and document a fallback. For a single-language project, start with
that language; localization is not a prerequisite for adopting the package.

## 5. Register templates with their data types

`src/templates/types.ts`:

```ts
import type { ActionEmailData } from "./action-email";

export const EMAIL_TEMPLATE_VALUES = { ACTION: "ACTION" } as const;
export type EmailTemplate =
  (typeof EMAIL_TEMPLATE_VALUES)[keyof typeof EMAIL_TEMPLATE_VALUES];
export type TemplateDataMap = {
  [EMAIL_TEMPLATE_VALUES.ACTION]: ActionEmailData;
};
```

`src/templates/index.ts`:

```ts
import type { JSX } from "react";
import { ActionEmail } from "./action-email";
import { EMAIL_TEMPLATE_VALUES, type EmailTemplate, type TemplateDataMap } from "./types";

export const EmailTemplateMap: {
  [K in EmailTemplate]: {
    subject: (data: TemplateDataMap[K]) => string;
    component: (data: TemplateDataMap[K]) => JSX.Element;
  };
} = {
  [EMAIL_TEMPLATE_VALUES.ACTION]: {
    subject: (data) => data.subject,
    component: ActionEmail,
  },
};
```

Use an explicit mapped-type annotation so the service retains the relationship
between each identifier and its payload. Adding a template means adding its
component/data, identifier, map entry and exports; missing entries should fail
typechecking. Do not erase this relationship with `any`.

## 6. Render once per format and delegate delivery

`src/email-service.ts`:

```ts
import { render } from "@react-email/render";
import type { IEmailProvider } from "./providers/email-provider.interface";
import { EmailTemplateMap } from "./templates";
import type { EmailTemplate, TemplateDataMap } from "./templates/types";

export class EmailService {
  constructor(
    private readonly provider: IEmailProvider,
    private readonly from: string,
  ) {}

  async sendEmail<T extends EmailTemplate>(
    template: T,
    to: string,
    data: TemplateDataMap[T],
  ): Promise<void> {
    const entry = EmailTemplateMap[template];
    const subject = entry.subject(data);
    const element = entry.component(data);
    const [html, text] = await Promise.all([
      render(element),
      render(element, { plainText: true }),
    ]);
    await this.provider.send({ to, from: this.from, subject, html, text });
  }
}
```

Keep components synchronous and render-safe. Let errors propagate to the caller;
do not return success after swallowing a render/provider error. The caller owns
the user-facing failure behavior and any retry policy.

## 7. Implement Resend and public exports

`src/providers/resend.provider.ts`:

```ts
import { Resend } from "resend";
import type { EmailMessage, IEmailProvider } from "./email-provider.interface";

export class ResendEmailProvider implements IEmailProvider {
  private readonly client: Resend;

  constructor(apiKey: string) {
    this.client = new Resend(apiKey);
  }

  async send(message: EmailMessage): Promise<void> {
    const { error } = await this.client.emails.send(message);
    if (error) throw new Error(`Resend error: ${error.message}`);
  }
}
```

`src/index.ts`:

```ts
export { EmailService } from "./email-service";
export { ResendEmailProvider } from "./providers/resend.provider";
export type { IEmailProvider, EmailMessage } from "./providers/email-provider.interface";
export { ActionEmail } from "./templates/action-email";
export type { ActionEmailData } from "./templates/action-email";
export { EmailTemplateMap } from "./templates";
export { EMAIL_TEMPLATE_VALUES } from "./templates/types";
export type { EmailTemplate, TemplateDataMap } from "./templates/types";
```

This exports JSX as requested by the Rakoi pattern as well as the sending service.
If client-side previews are needed, add a templates-only subpath export so the
preview does not depend on the server/provider entry point. Keep API keys server-side.

## 8. Wire one real consumer

At the server composition boundary, after validating runtime configuration:

```ts
import {
  EmailService, ResendEmailProvider, EMAIL_TEMPLATE_VALUES,
} from "@scope/infra-email";

const emails = new EmailService(
  new ResendEmailProvider(env.RESEND_API_KEY),
  env.EMAIL_FROM,
);

await emails.sendEmail(EMAIL_TEMPLATE_VALUES.ACTION, recipientEmail, {
  lang: "en",
  subject: "Verify your email",
  preview: "Confirm your email to finish setting up your account.",
  heading: "Verify your email",
  message: "Use the button below to confirm this email address.",
  actionLabel: "Verify email",
  actionUrl: verificationUrl,
  footer: "If you did not request this, you can ignore this email.",
});
```

`env`, `recipientEmail` and `verificationUrl` come from the real server/hook;
they are placeholders, not globals supplied by this package. Reuse existing env
names such as `RESEND_FROM_EMAIL` or `AUTH_EMAIL_FROM` when the project has them.
Document names in its env schema/example and configure each deployment separately.

For auth, consume the library-generated verification/reset URL in the appropriate
hook. The auth library owns token expiry, single use and session invalidation.
For business notifications, connect the actual use case/job; an exported template
alone does not constitute an integration.

Await the send or attach it to the runtime's supported background lifecycle.
Do not launch an untracked promise in a Worker and assume it will finish.
Verify whether the calling framework suppresses hook errors; an HTTP success
response is not proof that a provider accepted an email.

## 9. Add only operational features the consumer requires

| Need | Explicit extension |
| --- | --- |
| Reply-To | Add optional field to the message/config and map it to the installed provider SDK |
| Retries or duplicate job execution | Define a stable event identity and idempotency/delivery record before retrying blindly |
| Queued delivery | Put a queue around sending; template rendering remains independent |
| Bounce/delivery tracking | Record provider identifiers and handle delivery events; `send()` alone does not track inbox arrival |
| User-triggered resend | Rate-limit at the endpoint; do not send again merely because a screen refreshes |

Avoid logging auth URLs, tokens, full email bodies or API keys. Log enough event
context to diagnose failures without turning logs into a second copy of messages.
These capabilities are not included in the minimal implementation above.

## 10. Verify before handing it over

Use the project's test runner with a fake provider; unit tests must not send real mail.

1. **Typecheck:** wrong/missing payload fields and missing registry entries fail.
   Include a compile-time check if the runner does not typecheck tests.
2. **Template content:** render representative data; assert the correct action
   URL, subject/copy, escaped user text and meaningful plain text.
3. **Service behavior:** one invocation produces one provider call with the
   configured sender/recipient and both formats. A provider rejection propagates.
4. **Adapter:** mock the installed SDK; cover accepted responses, returned errors
   and thrown network failures.
5. **Integration:** exercise the actual hook/job with a fake provider, including
   its failure behavior and configured locale. Verify it is actually called.
6. **Build/runtime:** run package/server typecheck and build, plus a smoke check
   in the deployment runtime; Node tests do not prove Worker compatibility.
7. **Visual review:** render fixtures to HTML, inspect narrow/wide layouts and
   long localized text, then inspect a controlled test email in target clients.
   Browser previews do not prove mail-client rendering.
8. **Controlled delivery:** when authorized, send to a known test inbox and verify
   From, Reply-To if used, links and arrival. Do not equate provider acceptance
   with inbox delivery or click real one-use tokens as a passive preview.

Use the destination's actual commands, not invented package scripts. Record each
command and exit status. Run its required checks and report skipped runtime,
visual or delivery checks explicitly. The snippets in this playbook derive from
Rakoi's implementation but have not been compiled as a standalone starter.

## Completion checklist

- [ ] Package/module follows the destination's export and build conventions.
- [ ] JSX components and typed template data are publicly exported.
- [ ] HTML and plain text are generated; provider is injected.
- [ ] A real consumer is wired with validated runtime configuration.
- [ ] Branding, sender, locale and URLs belong to the destination project.
- [ ] Render/provider failures are observable and handled by the caller.
- [ ] Tests, build, visual review and delivery evidence are reported separately.
- [ ] Project documentation links to this playbook and records local deviations.

## Handoff template

```text
Implemented: <package/module, template and real consumer>
Configuration: <variable names and runtime, no secret values>
Local choices: <sender policy, locale, hook/job, optional features>
Checks: <commands, exit codes, meaningful results>
Visual/delivery verification: <performed checks or explicit gaps>
Follow-up: <remaining work, if any>
```
