# Open Questions

Questions the group needs to settle for Resource operations. The first three come from the [charter](https://github.com/modelcontextprotocol/modelcontextprotocol/pull/3389); add others by pull request. When a question is settled, record the outcome in the [decision log](decisions.md) and link it here.

| # | Question | Source | Status |
| :--- | :--- | :--- | :--- |
| 1 | Is a transferred file addressed as a Resource URI? The answer decides whether a `files/*` method family exists alongside `resources/*`, and binds all three workstreams. | Charter, cross-cutting concerns | Open |
| 2 | Which optimistic concurrency control mechanism do writes use, including create-if-absent? | Charter, workstream 2 | Open |
| 3 | How do writes interact with `notifications/resources/updated`, `ttlMs`, `cacheScope`, and `lastModified`? | Charter, workstream 2 | Open |
| 4 | Which typed errors do Resource operations return? | Charter, workstream 2 | Open |
| 5 | Does the work fit one Extensions Track SEP, or several (for example writes separate from listing and search)? | Charter, workstream 2 | Open |
| 6 | How do Resource operations relate to the optional directory read in the Skills extension? | Charter, related groups | Open |
