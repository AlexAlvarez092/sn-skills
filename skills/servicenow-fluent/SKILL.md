---
name: servicenow-fluent
description: Help develop ServiceNow applications with the Fluent language (TypeScript-based `.now.ts` metadata) and the ServiceNow SDK (`now-sdk`). Looks up the SDK's own official, version-matched documentation with `now-sdk explain <topic>` (tables, business rules, ACLs, flows, client scripts, catalog items, UI, AI agents, ATF…) and covers the basic dev loop (init, build, install). Use when the user writes, reviews, or converts Fluent code, or asks how to define some ServiceNow metadata in Fluent.
license: MIT
metadata:
  author: Alejandro Alvarez
  version: "1.0.0"
compatibility: Requires Node.js 20+ and the `@servicenow/sdk` CLI (`now-sdk` v4+, or via `npx @servicenow/sdk`). `explain` is offline and needs no credentials. `install`, `download`, `transform` and `query` need an authenticated instance.
---

# ServiceNow Fluent

**Fluent** is a TypeScript-based DSL for defining ServiceNow application
metadata (tables, ACLs, business rules, flows, UI, tests…) in `.now.ts`
files, imported from `@servicenow/sdk/core`. The **`now-sdk`** CLI builds and
installs it.

**Source of truth: `now-sdk explain`.** The docs are bundled with the
installed SDK, so they match its version, work offline and need no
credentials. Prefer them over memory or web search: Fluent APIs change
between SDK versions and models often invent signatures. Never guess an API
shape — look it up first.

## Before you start

1. `now-sdk --version`. If missing, check `node --version` (needs 20+) and
   tell the user to install/upgrade Node if needed. Offer
   `npm install -g @servicenow/sdk` (ask before a global install) or use
   `npx @servicenow/sdk <command>` for everything below.
2. If the project already exists, read its `package.json`, `now.config.json`
   and `src/fluent/**/*.now.ts` to follow its conventions (scope prefix,
   folder layout, naming).

## Looking up documentation

Always pass `--format raw` (the default `pretty` is meant for terminals).

```bash
now-sdk explain --list                      # all topics (~260) with keywords
now-sdk explain --list <filter>             # e.g. flow, table, acl, catalog
now-sdk explain <topic> --peek --format raw # one-line summary, to check relevance
now-sdk explain <topic> --format raw        # full document
```

Workflow:

1. **First time in a project: read the overview and cross-cutting topics.**
   `now-sdk explain fluent-overview --format raw` (file structure,
   `@servicenow/sdk/core` imports, `$meta`) and
   `now-sdk explain fluent-language --format raw`, which lists the
   cross-cutting topics (`Now.include`, `Now.attach`, `Now.ref`, `Now.del`,
   data helpers such as `Duration`/`TemplateValue`, `$override`; they apply
   to every `.now.ts` file). Read the relevant ones by name, e.g.
   `now-sdk explain override-guide --format raw`.
2. **Find the topic** with `--list <keyword>`. Match on the record or table
   name too (e.g. `sys_security_acl`, `sys_script`) since listings show
   keywords.
3. **Peek before reading** when several topics look relevant: each one can be
   hundreds of lines. Pipe long output through `head -n 200`, then read more.
4. **Read the `-api` topic for exact signatures and the `-guide` topic for
   patterns and how-tos.** They pair up: `business-rule-guide` and
   `businessrule-api`, `table-guide` and `table-api`, `acl-api`,
   `flow-api`/`wfa-api`/`wfa-flow-guide`, `client-script-guide`, and so on.
5. **Write code from the documented examples**, adapting names/scope. If an
   API isn't covered by any topic, say so instead of improvising.

General topics worth knowing: `faq`, `keys-file` (sys_id management),
`now-config-reference` (`now.config.json`), `ci-integration` (headless
build/deploy and CI auth), `service-portal-ootb-reference`.

An unknown topic returns `No matching topics`; retry with
`--list <keyword>`.

## Development loop (reference)

```bash
now-sdk init --appName "<name>" --packageName <npm-name> --scopeName <scope> --template typescript.basic
now-sdk build [source]      # compile Fluent -> installable package; local only
now-sdk install -a <alias>  # push to an instance (see warning)
```

- **`init`** also converts a legacy app: `--from <sys_id | directory>`. The
  scope must start with the vendor prefix if applicable (e.g. `x_<vendor>_`)
  and be at most 18 characters. Templates: `base`, `javascript.basic`,
  `javascript.react`, `typescript.basic`, `typescript.react`,
  `typescript.vue`, `javascript.aiux`, `javascript.aiux-extension`.
- **`build`** is safe and local; run it after edits to catch type and
  metadata errors. Use `--frozenKeys` in CI to validate sys_ids are
  up to date.
- **`install`** modifies the target instance. Confirm which instance/alias
  with the user first (`now-sdk auth --list`), and never use `--reinstall`
  without explicit approval: metadata created on the instance that isn't in
  the local source is **lost**. Use `--info` to see the last install.
- `download <dir>` / `transform` pull existing metadata from an instance and
  convert XML to Fluent. `dependencies` fetches configured dependencies.
- Authentication is interactive, per person, and cannot be done by the
  agent: ask the user to run
  `now-sdk auth --add <instance-url> --type oauth --alias <alias>` in their
  own terminal, then `now-sdk auth --list` to confirm. For details run
  `now-sdk explain ci-integration --format raw`.

To check facts on a live instance (field names, existing records, trigger
metadata) use read-only `now-sdk query` (see the `servicenow-query` skill)
instead of guessing.

## Tips

- Fluent files are TypeScript: run the build and fix compiler errors rather
  than silencing them.
- Keep server-side logic in separate `.js`/`.ts` modules and reference it with
  `Now.include` as the docs show, instead of inlining large scripts.
- Prefer the documented `Now.ref`/data helpers over hardcoded sys_ids.
- If `explain` output seems to disagree with the installed behavior, check
  `now-sdk --version` and suggest upgrading `@servicenow/sdk`.
