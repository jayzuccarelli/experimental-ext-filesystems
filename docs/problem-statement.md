# Problem Statement

MCP Resources are read-oriented. A client can list Resources and read one by URI, but there is no common way to create, update, or delete one, to read only its metadata, or to list, search, or glob within a scope. Servers that need these operations expose them as tools, each with its own names, arguments, and error shapes, so clients cannot treat them uniformly.

Writes add a second problem: concurrency. Without a shared mechanism, two clients that read a Resource and write it back can silently overwrite each other's changes (last writer wins), and "create only if it does not exist" cannot be expressed.

The [charter](https://github.com/modelcontextprotocol/modelcontextprotocol/pull/3389) puts this in workstream 2, Resource operations: create, update, delete, and `stat`; scoped listing, directory traversal, server-side search and glob matching, and bulk reads; typed errors; optimistic concurrency control on writes, including create-if-absent; and defined interactions with `notifications/resources/updated`, `ttlMs`, `cacheScope`, and `lastModified`.

Moving content that does not fit inline in JSON-RPC (content transfer) and marking tool or elicitation inputs as files (file inputs) are the charter's other two workstreams. Resource operations need to stay compatible with both, and in particular with the open question of whether a transferred file is addressed as a Resource URI.
