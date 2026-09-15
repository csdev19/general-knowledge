# Product legal documents

Reusable starting point for small teams operating evolving apps and optional cloud services.
Last reviewed: 2026-09-14.

## Templates

- [Terms and conditions](terms-and-conditions-template.md): product changes, free offers, availability, ownership, suspension, termination and proportionate liability limits.
- [Privacy policy](privacy-policy-template.md): data inventory, purposes, providers, international processing, diagnostics, retention and requests.
- [Cloud and acceptable use](cloud-and-acceptable-use-template.md): promotional capacity, anti-abuse controls, over-quota accounts and shutdown.
- [Cookies and local storage](cookies-template.md): sessions, preferences and diagnostics.

These are original adaptations of the structure used in Niway's Winku and Sentir projects, tailored first for Kaipu. They are not a legal opinion or a guarantee against claims. Do not transplant Sentir's healthcare/AI clauses into unrelated products or reuse blanket statements such as “we never share data with third parties” when infrastructure providers process it.

## Adapt before publishing

Replace every `{{...}}` token. Confirm the operating entity, business address, contact email, jurisdiction and any required company identifiers. Niway's current public email is **contacto@niway.dev**; legacy `contacto.niway@gmail.com` references should not be copied into new policies.

Then verify the statements against the actual product:

1. Whether local features require an account, and whether uploads are explicit.
2. File, metadata, hosting and diagnostics providers; actual regions and cross-border safeguards. A provider name is not proof of a particular storage location.
3. Exact data/events collected, permissions, cookies, retention, support and deletion processes. The example mentions PostgreSQL, a 10-minute session cache, a one-year language cookie, disabled session replay and device diagnostics: replace or remove these details if they differ.
4. Any optional analytics or marketing consent requirements. A policy or signup notice does not implement a consent mechanism. Do not silently call diagnostic identifiers “anonymous”.
5. The free quota and notice periods. The example promises **at least 30 days** for ordinary Cloud quota reductions and shutdown, and no automatic deletion merely for exceeding a lowered quota. These are operational commitments, not generic filler.
6. Existing paid commitments, open-source licenses, and local rights that cannot be waived. Do not promise unilateral changes to already-paid benefits or apply a zero-fee liability cap to all consumers.
7. Privacy/account-deletion contact handling, applicable complaints mechanisms, business processing agreements and legal review appropriate to actual markets. An ordinary contact email is not automatically a statutory complaints book.

## What flexibility means here

Routine interface, feature, compatibility, integration and provider changes can proceed without individual notice. Material changes affecting contractual rights or stored content have notice and transition rules. Security, abuse and legal emergencies allow prompt, proportionate action. Promotions are not perpetual entitlements, and do not become paid subscriptions automatically.

Do not interpret “subject to applicable law” as making an otherwise abusive waiver valid. The template preserves non-waivable rights and does not purport to bar complaints or all liability.

## Delivery pattern

Publish stable, unauthenticated terms, privacy, cloud and cookie URLs. Link them from the footer and next to account creation, preserving the form while documents open. Keep a version date, archive prior versions when revised, and communicate material changes through a channel you actually operate. Keep short promotional copy and the full Cloud policy aligned.

The first Kaipu implementation includes a signup notice, **not server-side evidence of acceptance**. For auditability, add storage of the accepted terms version, timestamp and relevant action in the account service, with an appropriate retention policy. Do not claim this exists simply because links are visible.

Templates are source material, not live policies shared across products. Each product owns a reviewed, versioned adaptation and links back here for guidance; changes to the template must not silently change customer agreements.

## First adaptation: Kaipu

- Operator copied from Winku/Sentir: Niway S.A.C., Lima, Peru. Confirm full statutory details for launch.
- Contact confirmed by the owner: contacto@niway.dev.
- Files: Cloudflare R2. Account/session/Cloud metadata: Neon PostgreSQL. Diagnostics in enabled desktop builds: PostHog.
- Local capture/edit/export is separate from Cloud upload.
- Initial promotional capacity: 1 GB in decimal units per verified account.
- No automatic billing, no promised perpetual storage, no automatic purge solely because a quota is reduced.
- Region, production retention settings and a complete operational consent/deletion/complaints workflow were not audited by the document implementation.

## References used

Primary starting points are local Niway projects: `winku-monorepo/apps/winku-web/src/routes/legal/` and `salud-mental-monorepo/apps/web/src/routes/legal/`. Kaipu privacy statements were checked against its authentication schema, Cloud storage implementation and PostHog configuration.

Limited background references consulted before drafting:

- [Peruvian Consumer Protection Code](https://diariooficial.elperuano.pe/Normas/obtenerDocumento?idNorma=17): non-waivable protections and limits on contract clauses.
- [Personal data protection regulation, entry into force](https://elperuano.pe/noticia/267273-llamadas-comerciales-sin-tu-permiso-conozca-el-nuevo-reglamento-que-protege-tus-datos): privacy notices do not replace permissions required for processing.
- [Cloudflare privacy policy](https://www.cloudflare.com/privacypolicy/): infrastructure processing roles.

The owner requested reuse of Winku/Sentir rather than further legal research. This package is scoped accordingly.
