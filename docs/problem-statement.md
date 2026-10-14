# Problem Statement

MCP Resources are read-oriented. A client can list Resources and read one by URI, but there is no common way to create, update, or delete one, to read only its metadata, or to list, search, or glob within a scope. Servers that need these operations expose them as tools, each with its own names, arguments, and error shapes, so clients cannot treat them uniformly.

Writes add a second problem: concurrency. Without a shared mechanism, two clients that read a Resource and write it back can silently overwrite each other's changes (last writer wins), and "create only if it does not exist" cannot be expressed.

The [charter](https://modelcontextprotocol.io/community/working-groups/filesystems) scopes one Extensions Track SEP for this: create, update, delete, and `stat`; optimistic concurrency control on writes, including create-if-absent; and defined interactions with `notifications/resources/updated`, `ttlMs`, `cacheScope`, and `lastModified`.

Moving content that does not fit inline in JSON-RPC, and marking tool or elicitation inputs as files, belong to the [File Uploads Working Group](https://modelcontextprotocol.io/community/working-groups/file-uploads). Resource operations need to stay compatible with that work, in particular on where an upload flow ends and a resource write begins.
