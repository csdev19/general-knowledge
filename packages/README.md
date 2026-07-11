# Packages

How to build and structure workspace packages in a monorepo: the `infra-*` naming convention, package exports/build strategy (source in dev vs built dist), where repository contracts vs implementations live, and a worked case study of creating a new shared package end to end.

## Contents

| File | Summary |
| ---- | ------- |
| [infrastructure-naming.md](./infrastructure-naming.md) | The `infra-<capability>` naming convention for infrastructure packages and why it works. |
| [shared-package-build-strategy.md](./shared-package-build-strategy.md) | Consuming packages as source in dev (HMR) vs built `dist` for prod, the tsdown `devExports` pattern, and the Node main-process gotcha. |
| [repository-contracts-and-implementations.md](./repository-contracts-and-implementations.md) | Repository contracts (interfaces) live in the domain; implementations live in infrastructure — rules, naming, and examples. |
| [case-study-design-tokens-package.md](./case-study-design-tokens-package.md) | Worked example: building a shared design-tokens package end to end, with generated committed CSS, drift tests, and phased delivery. |
