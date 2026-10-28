# sn-skills

A plugin that bundles my ServiceNow agent skills in one installable package.

| Skill | Purpose | Upstream |
|---|---|---|
| `servicenow-fluent` | Develop apps with Fluent (`.now.ts`) using `now-sdk explain` | [repo](https://github.com/AlexAlvarez092/servicenow-fluent) |
| `servicenow-query` | Read-only Table API queries on a live instance via `now-sdk query` | [repo](https://github.com/AlexAlvarez092/servicenow-query) |
| `servicenow-docs` | Search and read the official ServiceNow product docs | [repo](https://github.com/AlexAlvarez092/servicenow-docs) |

## Install

```bash
copilot plugin install AlexAlvarez092/sn-skills
```

Restart the CLI afterwards. See each upstream repo for requirements (Node.js 20+, `@servicenow/sdk`, instance authentication for `servicenow-query`).

## Maintenance

The skills are copies of the upstream repositories, which remain the source of truth. To sync, replace `skills/<name>/SKILL.md` with the upstream `SKILL.md` and commit.

## License

MIT
