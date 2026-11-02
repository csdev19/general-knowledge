# Software licensing for a source-published tool

How to pick a license for a tool whose source is public and has one maintainer. Read the decision
table, check the family row, then use the mechanics section when you change anything.
Last reviewed: 2026-11-01.

> **Not legal advice.** This page summarises public license texts and public cases so a
> maintainer can ask a lawyer better questions. License wording decides, not this summary; quotes
> below are from the linked texts on the review date. Have a lawyer read the final choice,
> especially before selling a commercial license.

## Pick in one minute

Two facts decide most of it:

1. **Licenses condition copying and distributing, not running.** Under every family below, a
   company that runs your unmodified tool on its own machines owes you nothing for credit. What a
   license can require is a notice when someone copies, modifies or redistributes. (This is the
   reading of the texts below, not a quoted rule.)
2. **"Non-commercial" blocks for-profit internal use.** A company running `<tool>` in its own CI
   on its own private repositories is using it for business. Under a non-commercial license that
   is a use you sell a license for, not a free use.

| Goal | Family | Trade-off |
| --- | --- | --- |
| Maximum adoption; credit only | Permissive: MIT or Apache-2.0 | Anyone can close it, rebrand it and sell it. Credit is a notice in copies, nothing more |
| Free for everyone including companies; improvements and hosted forks must publish source | Copyleft: AGPL-3.0 (plus a trademark policy for the name) | Some companies ban AGPL outright, which costs adopters. Does not stop commercial use |
| Free to read and run, including by companies; nobody ships a competing product or hosted clone | Competing-use: FSL-1.1 (becomes Apache-2.0 or MIT after 2 years) | Not OSI open source. Some developers refuse to contribute or will fork |
| Companies may run it internally; no redistribution of any kind | Internal use: PolyForm Internal Use, or a custom internal-use license in the style of n8n | No community forks that circulate; the license is a customer license, not a community one |
| Free for personal and non-profit use; companies pay | Non-commercial + commercial license: PolyForm Noncommercial 1.0.0, dual-licensed | A for-profit team must buy before using it in CI. You must sell, enforce and support it |
| Credit the author even in forks | Any family plus a trademark policy; AGPL can add attribution terms (section 7(b)) | A credit rule attaches only to copies and modified versions that get distributed |
| Nobody may copy it at all | Not "source-available": keep it private or use a proprietary EULA | Loses the trust argument of readable code |

If the tool's core audience is companies running it in their own CI, the families that keep that
audience are permissive, AGPL and competing-use. The non-commercial family removes it unless you
accept selling licenses to that audience.

## Families compared

The questions: does it require attribution, can a company run it internally for business (own CI,
private repositories), can a third party offer it as a hosted service, is it OSI open source, and
what friction does adoption meet?

| Family and licenses | Attribution | Company internal use | Third-party hosted service | OSI open source | Adoption friction |
| --- | --- | --- | --- | --- | --- |
| **Permissive**: MIT, Apache-2.0 | Yes: copyright and license notice in copies; Apache-2.0 also carries a `NOTICE` file | Yes | Yes, no source obligation | Yes | Lowest |
| **Copyleft**: GPL-3.0, AGPL-3.0 | Yes: keep copyright notices; AGPL can add author-attribution terms | Yes | Yes; AGPL requires a modified version's source to be offered to its network users | Yes | Medium to high: some companies ban AGPL |
| **Non-commercial**: PolyForm Noncommercial 1.0.0 | Yes: `Required Notice:` lines and a copy of the terms | **No** for business use. Allowed: personal use, charities, schools, public bodies | No | No | High for companies: pay or leave |
| **Internal use**: PolyForm Internal Use 1.0.0 | No separate notice term in the text | Yes, for "internal business operations" | No: it forbids distributing | No | Medium: fine for users, no forks |
| **Competing-use**: FSL-1.1, BSL 1.1, PolyForm Shield, ELv2 | FSL: link or copy of the terms and keep copyright notices; ELv2: do not remove notices | Yes | No for a competing service (ELv2: no hosted service of substantial features at all) | No | Medium; BSL is parameterised, so each project differs |
| **Internal or non-commercial only**: n8n Sustainable Use | Do not remove notices | Yes, for own internal business purposes | No: distribution only free and non-commercial | No | Medium |
| **CC BY-NC** | Yes | Not for commercial use | No | No | **Do not use for software** |

Why CC BY-NC is not for software: Creative Commons recommends against CC licenses for software,
because they lack terms on source distribution and patents and are incompatible with major
software licenses ([CC FAQ](https://creativecommons.org/faq/#can-i-apply-a-creative-commons-license-to-software)).

Why none of the restrictive rows are open source: the Open Source Definition requires that a
license "must not restrict anyone from making use of the program in a specific field of endeavor"
([OSD, criterion 6](https://opensource.org/osd)). A non-commercial or competing-use limit is such
a restriction. Do not call such a tool "open source" in public copy.

### What each competing-use license actually blocks

- **FSL-1.1.** Allows any "Permitted Purpose" except a "Competing Use": making the software
  available in a commercial product or service that substitutes for it, substitutes for another
  product of the licensor, or offers substantially similar functionality. Internal use,
  non-commercial education and research are explicitly permitted. Each version converts to
  Apache-2.0 (FSL-1.1-ALv2) or MIT (FSL-1.1-MIT) on its second anniversary
  ([FSL text as used by Sentry](https://github.com/getsentry/sentry/blob/master/LICENSE.md),
  [fsl.software](https://fsl.software/)).
- **BSL 1.1.** A template with parameters: an "Additional Use Grant" the licensor writes, a
  Change Date (default up to four years) and a Change License. Non-production use is always
  allowed; production use only as the grant says
  ([Terraform's license](https://github.com/hashicorp/terraform/blob/main/LICENSE)). The grant
  is bespoke, so every BSL project reads differently.
- **ELv2.** Three limits: no hosted or managed service exposing a substantial set of features,
  no circumventing license-key functionality, no removing notices
  ([Elastic licensing](https://www.elastic.co/licensing/elastic-license)).
- **PolyForm Shield.** Forbids competing with the licensor's product, and says goods and services
  compete even when free or on a different platform
  ([text](https://github.com/polyformproject/polyform-licenses/blob/1.0.0/PolyForm-Shield-1.0.0.md)).

### Fair source

"Fair source" is a label, not a license. Its site lists FSL as the flagship, with BSL and the
Fair Core License as alternates, and describes the approach as an alternative to closed source
that eventually becomes open source ([fair.io](https://fair.io/),
[licenses](https://fair.io/licenses/)). Name the actual license in your repository; the label
alone tells a reader nothing about what they may do.

## Mechanics a maintainer needs

**Relicensing future versions.** The sole copyright holder may release new versions under new
terms. Past releases keep the license they shipped under: the GNU FAQ says public GPL rights
"cannot be withdrawn", and HashiCorp stated its 2023 change applied to "all future releases"
([GNU FAQ](https://www.gnu.org/licenses/gpl-faq.html),
[HashiCorp](https://www.hashicorp.com/en/blog/hashicorp-adopts-business-source-license)). Anyone
who already has a release can keep using and forking it under the old license. Moving from AGPL
or MIT to a restrictive license therefore changes only what you ship next.

**Contributors.** Every outside contribution is owned by its author unless you collect something.
Without it, relicensing needs every contributor's consent.

| Instrument | What the contributor gives | Relicensing right |
| --- | --- | --- |
| None | Contribution under the project license, by implication | No |
| DCO sign-off | A certification that they may submit it under the file's license ([DCO](https://developercertificate.org/)) | No |
| CLA | A broad license to you, for example "permission to license my contributions on any terms" ([n8n CLA](https://github.com/n8n-io/n8n/blob/master/CONTRIBUTOR_LICENSE_AGREEMENT.md)) | Yes |

If relicensing or dual licensing is a possible future, collect the CLA from the first outside
contribution. Adding it later means finding every earlier contributor. A project that takes no
outside contributions has no problem to solve.

**Dual licensing.** The copyright holder can offer the same code under several licenses at once
([GNU FAQ](https://www.gnu.org/licenses/gpl-faq.html)). Typical shape: a free license for open
use, and a paid commercial license for those who cannot or will not accept it. It requires that
you own all the code, hence the CLA above.

**Trademark: the separate tool for the name.** A software license covers code, not the name or
logo. AGPL section 7(e) lets the author decline trademark rights, and the Open Source Definition
accepts a requirement that derived works carry a different name
([AGPL text](https://www.gnu.org/licenses/agpl-3.0.txt), [OSD criterion 4](https://opensource.org/osd)).
A trademark policy states what others may do with the name, logo and mascot without asking.
Typical content: true statements such as "based on `<tool>`" are fine; a modified build shipped
under the name, or a hosted service using it, needs permission. Trademark law varies by country.

**What attribution means in each license.**

| License | What you must keep or add when you copy or distribute |
| --- | --- |
| MIT | "The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software" ([OSI](https://opensource.org/license/mit)) |
| Apache-2.0 | License copy, change notices, and the attribution notices from the `NOTICE` file, if one exists ([text, section 4](https://www.apache.org/licenses/LICENSE-2.0.txt)) |
| AGPL-3.0 | Keep copyright notices; modified versions carry notices that they were changed. Section 7(b) lets the copyright holder add "reasonable legal notices or author attributions" as a term ([text](https://www.gnu.org/licenses/agpl-3.0.txt)) |
| FSL-1.1 | Include a copy of or link to the terms, and do not remove copyright notices when redistributing |
| PolyForm family | Pass on the terms or their URL, plus every `Required Notice:` line you ship ([text](https://github.com/polyformproject/polyform-licenses/blob/1.0.0/PolyForm-Noncommercial-1.0.0.md)) |
| n8n Sustainable Use, ELv2 | Do not alter, remove or obscure licensing, copyright or other notices |

"Credit the author" is therefore a notice that travels with copies. The GNU FAQ says academic
citation "cannot be added as an additional requirement to the GPL"; attribution beyond the
notices has to ride on section 7(b) or on a trademark policy. A `NOTICE` or `Required Notice:`
file is where to put the exact credit line.

**Keep the license in one place.** The `LICENSE` file, the package manifest's SPDX identifier
(`AGPL-3.0-only` versus `AGPL-3.0-or-later`) and the README must say the same thing. "Version 3
or any later version" and "version 3 only" are different grants; a tool that reads only the manifest
and a person who reads only the file reach different answers. Put the project's notice in the README and keep the license text
unmodified, which is also what lets GitHub detect it.

## Public cases

Dates and wording from each project's own files unless noted. License files change; re-check
before you cite one.

| Project | License | What happened |
| --- | --- | --- |
| Sentry | FSL-1.1-Apache-2.0 | Moved from BSL to FSL in November 2023 because BSL's variable terms made compliance review harder ([Sentry](https://blog.sentry.io/introducing-the-functional-source-license-freedom-without-free-riding/)) |
| Terraform | BSL 1.1, change to MPL 2.0 after four years | HashiCorp switched from MPL 2.0 in August 2023; the Additional Use Grant allows production use except competitive hosted or embedded offerings, and internal use is not competitive. The community forked it as OpenTofu under the Linux Foundation ([HashiCorp](https://www.hashicorp.com/en/blog/hashicorp-adopts-business-source-license), [OpenTofu](https://opentofu.org/manifesto/)). The licensor is now IBM ([LICENSE](https://github.com/hashicorp/terraform/blob/main/LICENSE)) |
| Elasticsearch | AGPL-3.0, SSPL and ELv2 triple license | Moved to ELv2/SSPL in 2021, then announced in August 2024 that it was adding AGPL, partly to call it open source again ([Elastic](https://www.elastic.co/blog/elasticsearch-is-open-source-again), [LICENSE](https://github.com/elastic/elasticsearch/blob/main/LICENSE.txt)) |
| n8n | Sustainable Use License; `.ee` paths under a separate enterprise license | Free for own internal business use and non-commercial use; distribution only free of charge for non-commercial purposes; collects a CLA ([LICENSE](https://github.com/n8n-io/n8n/blob/master/LICENSE.md)) |
| Cap | AGPL-3.0, with MIT for the `cap-camera*` and `scap-*` crates | I found no directory under a commercial source license. Separately, its terms require a paid "Desktop License" for commercial use of the binaries it distributes, and say it does not apply to builds from source ([LICENSE](https://github.com/CapSoftware/Cap/blob/main/LICENSE), [commercial license](https://cap.so/docs/commercial-license)) |
| xlwings PRO, EPPlus 8, umbrelOS | PolyForm Noncommercial 1.0.0 | xlwings and EPPlus dual-license it with a paid commercial license; EPPlus says commercial business use needs a commercial license; Umbrel asks commercial users to contact it ([xlwings](https://github.com/xlwings/xlwings), [EPPlus](https://github.com/EPPlusSoftware/EPPlus), [Umbrel](https://github.com/getumbrel/umbrel)) |
| Pullfrog | MIT | A BYOK GitHub Actions review bot; the repository is MIT, "Copyright (c) 2026 Pullfrog, Inc." ([repository](https://github.com/pullfrog/pullfrog)) |
| PR-Agent | MIT | Donated by Qodo to a community organisation; Qodo's hosted product is separate ([repository](https://github.com/qodo-ai/pr-agent)) |
| CodeRabbit | Proprietary hosted service | I found no primary license page; secondhand sources describe a closed SaaS with a free tier for public repositories ([It's FOSS](https://itsfoss.com/news/coderabbit-free/)) |

What the cases show: two well-known competing-use moves (Terraform, Elasticsearch) each produced
a fork or a partial reversal. In the review-tool niche, the two open tools I checked (Pullfrog,
PR-Agent) are MIT, and the commercial product is a hosted service. AGPL users that charge anyway (Cap) do it
with a separate commercial terms page rather than a different source license; whether that
restriction can bind AGPL code contributed by others is a legal question this page does not
answer.

Friction evidence for AGPL: Google's policy states that AGPL code "MUST NOT be used at Google"
because network use can trigger source disclosure
([Google](https://opensource.google/documentation/reference/using/agpl-policy)). One company's
policy is not the market, but it shows the ceiling on corporate adoption.

## Measured example: two tools by one owner

Not a recommendation; the state of the owner's two published tools, from their own repository
files and decision records, on 2026-11-01.

- **Kaipu (desktop screen recorder).** `AGPL-3.0-only`, public since 2026-10-05. A CLA
  (2026-10-30) keeps relicensing possible, and a trademark policy protects the name, logo and
  mascot. The license decision record names source-available (FSL or BSL) as the fallback if a
  fork ships as a competing product. A "supporter license" that funds development without
  gating use is a proposed idea, not implemented. Until 2026-10-30 its `LICENSE` header said
  "or any later version" while its manifests said `AGPL-3.0-only`; the fix was an unmodified
  license text plus the version-3-only notice in the README.
- **A CI code-review CLI (second tool).** AGPL-3.0 chosen with the stated intent "free to use,
  attribution preserved, modifications shared", and the trade-off recorded as "AGPL deters some
  corporate adopters". Its `LICENSE` says "version 3 or any later version" while its manifest
  says `AGPL-3.0-only`: the same mismatch.
- **Lesson.** The owner's intent was "credit the author, preferably non-commercial", but the
  chosen license permits commercial use and requires only notices and source. Writing the intent
  down in plain words before choosing would have exposed the gap: the families that deliver
  "non-commercial" also remove the tool's main audience, companies running it in their own CI.

## Related

- [Legal documents](./README.md): terms, privacy, cloud and cookie templates. A license covers
  the code; these cover the service around it.
- [Filesystem-first monetization](../desktop/filesystem-first-monetization.md): the business model
  that decides what you charge for, which comes before the license.
- [Product marketing playbook](../product/product-marketing-playbook.md): the byline and
  copyright holder in `LICENSE` are separate decisions; claims about "open source" must match the
  license.
