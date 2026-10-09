# Related Work

SEPs, extensions, and discussions that Resource operations build on or must stay consistent with. Add entries by pull request; keep each to a line or two.

## SEPs named in the charter

| SEP | Status | Relevance |
| :--- | :--- | :--- |
| [SEP-2571](https://github.com/modelcontextprotocol/modelcontextprotocol/pull/2571) | Closed 2026-09-22 | Proposed `resources/create` and `resources/delete`. The charter takes its create and delete operations as input to workstream 2. |
| [SEP-2631](https://github.com/modelcontextprotocol/modelcontextprotocol/pull/2631) | Draft | File objects and out-of-band transfer. Input to the content transfer and file input workstreams. |
| [SEP-2532](https://github.com/modelcontextprotocol/modelcontextprotocol/pull/2532) | Open | `resources/stream`. To be reconciled with the transfer work. |
| [SEP-2356](https://github.com/modelcontextprotocol/modelcontextprotocol/pull/2356) | Closed 2026-06-26 | File input model. Input to the file inputs workstream. |

## Extensions and groups

- [Skills extension](https://github.com/modelcontextprotocol/ext-skills): reads skill files through Resources and defines an optional directory read. The charter asks this group to coordinate on any Resource operation that affects skills.
- [Interceptors extension](https://github.com/modelcontextprotocol/experimental-ext-interceptors): repository layout this one follows for per-language reference implementations.
