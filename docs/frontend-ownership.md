# Frontend ownership

Every file under `packages/twenty-front` is owned by exactly one of twelve
teams, declared in [`.github/CODEOWNERS`](../.github/CODEOWNERS). This document
explains the grouping so the rules can be maintained rather than merely obeyed.

Coverage is enforced, not assumed — see [Keeping it honest](#keeping-it-honest).

## Why these twelve

The teams are named after the product domains already used in
`packages/twenty-docs/user-guide/` (`ai`, `billing`, `calendar-emails`,
`dashboards`, `data-model`, `layout`, `permissions-access`, `settings`,
`views-pipelines`, `workflows`). That was the only product vocabulary in the
repo; reusing it means docs, product and ownership agree instead of introducing
a fourth taxonomy that has to be kept in sync with the other three.

Two structural facts make directory-based rules safe here:

- `packages/twenty-front/folderStructure.json` is lint-enforced. Module
  directories must be kebab-case, may recurse four levels, and may only contain
  a fixed set of children (`hooks/`, `utils/`, `states/`, `components/`, …). The
  shape the globs target cannot silently drift.
- `.oxlintrc.json` `ignorePatterns` is the repo's own definition of
  not-human-authored code. The CODEOWNERS generated-code section reuses that
  list rather than inventing one.

## The teams

| Team | Domain | Files | Owns |
|---|---|---:|---|
| `@twenty/records` | record CRUD engine | 1,527 | `object-record` (table, board, calendar, field, inline-cell, picker, drag, merge), `people`, `companies`, `pages/object-record` |
| `@twenty/layout` | layout, dashboards | 1,025 | `page-layout` and its widget host, `dashboards`, `layout-customization` |
| `@twenty/workspace-admin` | settings, billing, permissions | 827 | `settings` (roles, admin-panel, billing, security, developers, members…), `workspace*`, `users`, `domain-manager`, `support`, `analytics` |
| `@twenty/design-system` | in-app UI layer | 698 | `ui`, `blocknote-editor`, `advanced-text-editor`, `spreadsheet-import` |
| `@twenty/workflows` | automation | 652 | `workflow` (steps, diagram, variables, triggers), `logic-functions` |
| `@twenty/shell` | app chrome | 591 | `navigation*`, `command-menu*`, `side-panel`, `keyboard-shortcut-menu`, `information-banner`, `loading` |
| `@twenty/views` | views, filters, sorts | 537 | `views`, `context-store`, and the filter/sort/group subtree of `object-record` |
| `@twenty/frontend-platform` | infrastructure | 521 | `app`, `apollo`, `client-config`, `error-handler`, `browser-event`, extension runtime, `src/{utils,hooks,types,config,testing}`, generated code |
| `@twenty/data-model` | object/field metadata | 447 | `object-metadata`, `metadata-store`, `sse-db-event`, `settings/data-model` |
| `@twenty/comms` | email, calendar, tasks, notes | 394 | `activities`, `accounts`, `mention`, `file*`, `geo-map`, `settings/accounts` |
| `@twenty/ai` | agents and chat | 283 | `ai`, `settings/ai`, the ask-ai side panels |
| `@twenty/auth` | sign-in and onboarding | 179 | `auth`, `onboarding`, `captcha`, `sign-in-background-mock` |

## Four modules that cannot have one owner

The `CODEOWNERS` file is ordered broad → narrow because **the last matching
rule wins**. Four modules are large enough that a single owner would be a
fiction, so they get nested overrides below their parent rule. Moving an
override above its parent silently disables it.

- **`object-record` (1,714 files)** — the record engine belongs to `records`,
  but filtering, sorting and grouping are view semantics and go to `views`
  (`advanced-filter`, `object-filter-dropdown`, `object-sort-dropdown`,
  `record-filter`, `record-filter-group`, `record-group`, `record-sort`).
- **`settings` (877) and `pages/settings` (236)** — six unrelated domains in one
  directory. `data-model`, `ai`, `logic-functions` and `accounts` route to their
  domain teams; the remainder is `workspace-admin`.
- **`page-layout/widgets` (496)** — the container is layout's, but each widget
  body is another domain's surface re-hosted inside it. Email, calendar, notes,
  tasks, files and timeline widgets go to `comms`; the workflow widget to
  `workflows`; the record-table widget to `records`.
- **`side-panel/pages` (359)** — same shape. `shell` owns the panel; the content
  owners own what renders inside it.

## Load-bearing code that looks small

Fan-in, measured by counting import sites across all 56 modules:

| Module | Files | Modules importing it | Import sites |
|---|---:|---:|---:|
| `ui` | 526 | **46 of 56** | 6,578 |
| `object-metadata` | 165 | 29 | 1,642 |
| `auth` | 124 | 28 | 517 |
| `client-config` | 37 | 20 | 157 |
| `context-store` | 24 | 14 | 260 |
| `src/utils` | 136 | — | 513 |
| `src/testing` | 101 | — | 875 |

`object-metadata`, `auth`, `client-config` and `context-store` are small enough
to look unimportant and are anything but: a two-line change in `client-config`
reaches twenty modules. They stay with their domain teams — moving them to
platform would hollow those teams out — but treat a PR touching them as a
cross-cutting change.

`ui` is the genuine bottleneck: 46 of 56 modules import it, so
`@twenty/design-system` is in the blast radius of almost every change. That is a
fact about the architecture, not about the ownership model, and the model
reports it rather than hiding it behind a shared owner.

## Generated code

`src/generated`, `src/generated-metadata`, `src/generated-admin`,
`src/locales/generated` and `src/testing/mock-data` are emitted by
graphql-codegen, lingui and a mock-data script. They are owned by
`@twenty/frontend-platform` so CI has somewhere to send a notification when
codegen output changes shape — not because anyone should be reviewing an
8,220-line generated `graphql.ts` line by line.

## Keeping it honest

A catch-all rule at the top of `CODEOWNERS` assigns anything unmatched to
`@twenty/frontend-platform`, so a module added next quarter has an owner from
its first commit instead of falling through to nobody.

Coverage is checked, not trusted:

```bash
node scripts/ownership-coverage.mjs --repo . --package packages/twenty-front
```

It walks every source file, resolves it through the same matcher, prints a
per-team table and **exits non-zero if a single file is unowned**. Without a
gate, unowned code does not raise an error — it quietly becomes some default
team's problem, which is worse, because it looks like ordinary work.
