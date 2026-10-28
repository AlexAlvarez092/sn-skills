---
name: servicenow-query
description: Run read-only Table API queries against a live ServiceNow instance using the official `now-sdk query` CLI (@servicenow/sdk) — e.g. confirm exact field names/types via sys_dictionary before writing a GlideRecord query, inspect Business Rule/ACL/Client Script trigger metadata, resolve names to sys_ids, or check table inheritance. Use whenever a task needs facts from the actual instance instead of guessing.
license: MIT
metadata:
  author: Alejandro Alvarez
  version: "1.1.1"
compatibility: Requires Node.js 20+ and the `@servicenow/sdk` CLI (`now-sdk` v4+, or via `npx`), one interactive OAuth sign-in per instance, and network access to the instance. Read-only (GET only).
---

# ServiceNow Query (live instance, read-only)

Usage guidance for ServiceNow's own **`now-sdk query`** command, which wraps
the standard Table API (`/api/now/table/<table>`). No custom code to maintain.

## Safety rules

- **Read-only.** `now-sdk query` only issues GET requests. Writes, background
  scripts, or changing config are out of scope: tell the user what to change
  and let them do it in the instance UI (or via `now-sdk install`).
- **Never query production** unless the user explicitly says that's allowed
  for this task. Many organizations forbid AI agents from reading production
  data. When unsure which environment an alias points to, ask. Resolve an
  alias's `host` with `now-sdk auth --list` before using it; an existing
  alias is not permission to query it.
- **Use OAuth, never Basic Auth** (`--type oauth`), so every call runs under
  the developer's own identity, roles and ACLs.
- **Never reuse another person's alias.** If `auth --list` shows credentials
  that don't obviously belong to the current user, ask first.

## Before querying

1. **Check the CLI:** `now-sdk --version`. If missing, check
   `node --version` (needs 20+); tell the user to install/upgrade Node if
   needed. Offer `npm install -g @servicenow/sdk` (ask before a global
   install) or use `npx @servicenow/sdk query ...` instead.
2. **Ask which instance** if it isn't obvious from the context. Don't guess.
3. **List aliases:** `now-sdk auth --list` (shows `[alias]`, `host`, `type`;
   the default is marked `*`).
4. **Alias exists** → pass `--auth <alias>` per call, or run
   `now-sdk auth --use <alias>` once for several queries.
5. **No alias** → the agent cannot create one: sign-in is an interactive,
   personal browser step. Ask the user to run, in their own terminal:

   ```bash
   now-sdk auth --add https://<instance>.service-now.com --type oauth --alias <alias>
   ```

   The URL may be a custom domain instead of `*.service-now.com`; use
   whatever URL the user opens in the browser.

   Then re-run `now-sdk auth --list` and continue. Manage credentials with
   `now-sdk auth --use <alias>` and `now-sdk auth --delete <alias>`.

If the instance sits behind a VPN/IP allowlist, a connection failure or
timeout usually means the VPN is off.

## Usage

```bash
now-sdk query <table> -q '<encoded_query>' -f '<fields>' -o json -a <alias>
```

- `-q/--query` encoded query (required) · `-f/--fields` comma-separated
  fields — **always pass it**, a bare query returns every column.
- `--limit` (default 100) / `--offset` — paging. With `-o json` the output is
  `{"ok":true,"hasMore":…,"nextOffset":…,"records":[…]}`; loop while
  `hasMore` is true. On failure: `{"ok":false,"error":{…}}` — always check
  `ok` first.
- `--display-value true|false|all` — display values (`all` = raw + display).
- `-s/--select 'records[0].sys_id' --output raw` — extract one value, handy
  in `$(...)`. A path that doesn't resolve prints `null`/empty.
- `--no-count` faster on big tables · `--timeout` ms (default 30000) ·
  `--view` UI view for field selection.
- `now-sdk explain query-guide` prints the CLI's own bundled reference if its
  version drifts from this doc.

### Encoded query syntax (not SQL)

`^` AND, `^OR` OR, `^ORDERBY<field>` / `^ORDERBYDESC<field>`. Operators:
`=`, `!=`, `LIKE`, `STARTSWITH`, `ENDSWITH`, `ISEMPTY`, `ISNOTEMPTY`, `IN`
(comma-separated), `<`, `>`, `<=`, `>=`. Example:
`active=true^nameLIKEmy_prefix^ORDERBYname`.

"Who am I" has no endpoint; use:

```bash
now-sdk query sys_user -q 'sys_id=javascript:gs.getUserID()' -f user_name,name -o json -a <alias>
```

## Useful tables

| Table | Holds |
| --- | --- |
| `sys_dictionary` | Field metadata (`element`, `column_label`, `internal_type`, `reference`, `mandatory`, `max_length`) |
| `sys_db_object` | Table metadata (label, `super_class`, extendable) |
| `sys_script` | Business Rules (`when`, `condition`, `order`, `collection`, `active`, `filter_condition`, `script`) |
| `sys_security_acl` | ACLs (`type`, `operation`, `admin_overrides`, `condition`, `script`) |
| `sys_script_client` | Client Scripts (`type`, `field`, `table`, `ui_type`, `order`) |
| `sys_ui_policy` | UI Policies; per-field conditions are in `sys_ui_policy_condition` |
| `sys_ui_action` | UI Actions (`table`, `condition`, surfaces, `client` flag) |
| `sys_script_include` | Script Includes |
| `sys_user`, `sys_user_role`, `sys_scope` | Users, roles, application scopes |
| `sys_properties` | System properties (`name`, `value`) |
| `sys_flow_context` | Flow Designer execution contexts (`name`, `state`, `error_message`) |
| `syslog` | System log (`level`, `source`, `message`) |

## Recipes

```bash
# Table columns (before writing a GlideRecord query)
now-sdk query sys_dictionary -q 'name=incident^elementISNOTEMPTY' \
  -f 'element,column_label,internal_type,reference,mandatory' -o json -a <alias>

# Table inheritance
now-sdk query sys_db_object -q 'name=incident' -f 'name,super_class,label' \
  --display-value all -o json -a <alias>

# Business Rule trigger metadata
now-sdk query sys_script -q 'name=<rule name>' \
  -f 'name,when,condition,order,collection,active,filter_condition' -o json -a <alias>

# ACLs on a table
now-sdk query sys_security_acl -q 'name=incident^ORnameSTARTSWITHincident.' \
  -f 'sys_id,name,operation,active' -o json -a <alias>

# Resolve a name to a sys_id
roleId=$(now-sdk query sys_user_role -q 'name=admin' -f sys_id \
  --select 'records[0].sys_id' --output raw -a <alias>)

# Collision check before creating a record: records.length > 0 means it exists
now-sdk query sys_script -q 'name=My Rule^collection=incident' -f sys_id -o json -a <alias>

# System property value
now-sdk query sys_properties -q 'name=glide.sys.date_format' -f 'name,value' -o json -a <alias>

# Recent Flow Designer flow runs/errors (sys_flow_context)
now-sdk query sys_flow_context -q 'state=Error^ORDERBYDESCsys_created_on' \
  -f 'name,state,error_message,sys_created_on' --limit 20 -o json -a <alias>

# Recent error-level system log entries. syslog is huge: always bound by time
# and add --no-count, otherwise the request can time out
now-sdk query syslog -q 'level=2^sys_created_on>javascript:gs.minutesAgoStart(60)^ORDERBYDESCsys_created_on' \
  -f 'sys_created_on,source,message' --limit 20 --no-count -o json -a <alias>
```

## Typical workflow

1. Confirm instance and alias (see "Before querying").
2. Need to know exactly when a Business Rule/ACL/Client Script fires? Query
   its table for trigger metadata.
3. About to write a GlideRecord query? Confirm field internal names via
   `sys_dictionary` — never guess.
4. Anything beyond read-only lookups → hand back to the user.
