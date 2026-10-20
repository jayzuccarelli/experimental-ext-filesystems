# MCP Filesystems: Problem Statements

## Context

Agent platforms present services to models as filesystems. Claude Code, Gemini CLI and Codex CLI all give their model file operations ([talk, slide 8](https://hosted-files.sched.co/agntconmcpconeu26/eb/The%20Unix%20Philosophy%20for%20AI%20Agents%20-%20Filesystems%20as%20the%20Context%20Primitive%20-%202026-09-16.pdf#page=8); [Claude Code tools](https://code.claude.com/docs/en/tools-reference); [Gemini CLI file system tools](https://github.com/google-gemini/gemini-cli/blob/main/docs/tools/file-system.md)), and models are trained on them ([talk, slide 7](https://hosted-files.sched.co/agntconmcpconeu26/eb/The%20Unix%20Philosophy%20for%20AI%20Agents%20-%20Filesystems%20as%20the%20Context%20Primitive%20-%202026-09-16.pdf#page=7)). MCP Resources addresses content by URI. The specification defines `file://` as identifying "resources that behave like a filesystem" ([Resources: Common URI Schemes](https://modelcontextprotocol.io/specification/draft/server/resources#common-uri-schemes)).

Currently, Resources support read and list ([talk, slide 6](https://hosted-files.sched.co/agntconmcpconeu26/eb/The%20Unix%20Philosophy%20for%20AI%20Agents%20-%20Filesystems%20as%20the%20Context%20Primitive%20-%202026-09-16.pdf#page=6); [Resources: Protocol Messages](https://modelcontextprotocol.io/specification/draft/server/resources#protocol-messages)). Anything more rich is often defined as per-server tools, with each source naming its own operations, e.g. `search_files`, `query` or `list_objects` ([talk, slide 5](https://hosted-files.sched.co/agntconmcpconeu26/eb/The%20Unix%20Philosophy%20for%20AI%20Agents%20-%20Filesystems%20as%20the%20Context%20Primitive%20-%202026-09-16.pdf#page=5)). Each harness then learns each source, so integration work grows as M × N. One contract implemented by every source reduces it to M + N ([talk, slide 10](https://hosted-files.sched.co/agntconmcpconeu26/eb/The%20Unix%20Philosophy%20for%20AI%20Agents%20-%20Filesystems%20as%20the%20Context%20Primitive%20-%202026-09-16.pdf#page=10)).

The reference filesystem server in the MCP servers repository exposes its filesystem operations as tools. `@modelcontextprotocol/server-filesystem` offers `list_directory` and `directory_tree` for traversal, `search_files` for glob matching, `read_multiple_files` for bulk reads, `read_text_file` with `head` and `tail` for partial reads, `get_file_info` for metadata, and `write_file`, `create_directory` and `move_file` for changes ([Filesystem MCP Server, Tools](https://github.com/modelcontextprotocol/servers/tree/main/src/filesystem#tools)). Each of these maps to one of the problem statements below. Furthermore, `write_file` overwrites an existing file with no precondition, so two agents writing the same file hit the lost update described in Problem Statement 5. A client that wants the same operations from a ticketing system or an object store learns a different set of tool names for each source.

The Filesystems WG charter makes Resources bidirectional: create, update, delete, `stat`, optimistic concurrency control, and change notification and caching ([Filesystems charter, In Scope](https://modelcontextprotocol.io/community/working-groups/filesystems#in-scope)). The work also needs a common interface across sources, including efficient discovery and retrieval. The statements below assume content stays addressed as Resources.

### Running example

A support lead asks why customers could not log in this week. Three agents share one workspace:

- Agent 1 groups the week's login tickets into themes.
- Agent 2 counts errors in the application log.
- Agent 3 combines both into a report, using the lead's saved preferences.

| URI | Holds | Lifetime | Access |
| :---- | :---- | :---- | :---- |
| `file:///tickets/4821.md` | One ticket from the ticketing system | Source-managed | Read-only |
| `file:///logs/app.log` | Application log, several GB | Source-managed | Read-only |
| `file:///thread/themes.md` | Ticket themes from Agent 1 | Conversation | Read/write |
| `file:///task/errors.csv` | Error counts from Agent 2 | Unit of work | Read/write |
| `file:///memory/preferences.md` | The lead's report preferences | Long-lived | Append-only |
| `file:///output/report.md` | Report from Agent 3 | Long-lived | Read/write |

Lifetimes are split between the private working state, which lasts for the task, and shared coordination state, which lasts for the conversation ([talk, slide 4](https://hosted-files.sched.co/agntconmcpconeu26/eb/The%20Unix%20Philosophy%20for%20AI%20Agents%20-%20Filesystems%20as%20the%20Context%20Primitive%20-%202026-09-16.pdf#page=4)).

> [!NOTE]
> The layout is a host convention. How a host materialises resources into a filesystem for the model is a host concern ([Filesystems charter, Out of Scope](https://modelcontextprotocol.io/community/working-groups/filesystems#out-of-scope)).

## Problem Statement 1: Scoped listing and directory traversal

Agent 3 needs to find what Agents 1 and 2 wrote under `file:///thread` and `file:///task`. Currently, Agent 3 lists every resource on the server and filters client-side, or calls a server-specific tool.

**Current state**

- `resources/list` is flat and paginated by cursor. It takes no URI, prefix or depth ([Resources: Listing Resources](https://modelcontextprotocol.io/specification/draft/server/resources#listing-resources); [Pagination](https://modelcontextprotocol.io/specification/draft/server/utilities/pagination)).
- Servers MAY mark directories with `inode/directory` ([Resources: Common URI Schemes](https://modelcontextprotocol.io/specification/draft/server/resources#common-uri-schemes)), and `resources/read` on a directory MAY return several contents ([Resources: Reading Resources](https://modelcontextprotocol.io/specification/draft/server/resources#reading-resources)). The specification does not define whether those are children, descendants or bodies.
- The Skills extension defines `resources/directory/read`, which returns direct children with cursor pagination ([Skills: Reading Directories](https://modelcontextprotocol.io/extensions/skills/overview#reading-directories)). It is declared inside the Skills extension through `directoryRead` ([Skills: Capabilities](https://modelcontextprotocol.io/extensions/skills/overview#capabilities)), so other servers have no core method to declare.

**A proposal must**

1. Define a one-level listing below a URI, analogous to `readdir`, returning each entry's type, `size`, `mimeType`, `lastModified` and revision (Problem Statement 5).
2. State its relationship to `resources/list` and to the Skills `resources/directory/read`. The Filesystems charter asks the WG to coordinate with Skills on future resource operations that affect skills ([Filesystems charter, Related Groups](https://modelcontextprotocol.io/community/working-groups/filesystems#related-groups)).
3. Define URI hierarchy: parent resolution, trailing slashes, and which schemes it applies to. The Skills extension already requires directory URIs without a trailing slash ([Skills: Reading Directories](https://modelcontextprotocol.io/extensions/skills/overview#reading-directories)).
4. State the consistency a client can expect across pages. Currently, paginated lists carry no cross-page consistency guarantee ([Caching: Interaction with Pagination](https://modelcontextprotocol.io/specification/draft/server/utilities/caching#interaction-with-pagination)).

**Open questions**

- `resources/list` MAY vary by the authorisation presented on the request ([Resources: Capabilities](https://modelcontextprotocol.io/specification/draft/server/resources#capabilities)). Does a directory listing include children the caller cannot read?

## Problem Statement 2: Server-side search and glob matching

Agent 1 wants this week's tickets that mention "login", and Agent 2 wants the lines in `app.log` that contain `ERROR`. Without matching at the source, the client reads every ticket and the whole log to match locally. The talk notes that mounting a source through to the client makes search inefficient ([talk, slide 6](https://hosted-files.sched.co/agntconmcpconeu26/eb/The%20Unix%20Philosophy%20for%20AI%20Agents%20-%20Filesystems%20as%20the%20Context%20Primitive%20-%202026-09-16.pdf#page=6)).

**Current state**

- Resources defines no search or pattern-matching operation ([Resources: Protocol Messages](https://modelcontextprotocol.io/specification/draft/server/resources#protocol-messages)).
- Resource templates describe parameterised URIs with RFC 6570, and the completion API can fill in their arguments ([Resources: Resource Templates](https://modelcontextprotocol.io/specification/draft/server/resources#resource-templates)). Templates do not enumerate the resources that match them. Server-side glob and find are needed beyond what templates provide.
- The harnesses give the model glob and grep, e.g. Claude Code's `Glob` and `Grep` and Gemini CLI's `glob` and `grep_search` ([Claude Code tools](https://code.claude.com/docs/en/tools-reference); [Gemini CLI file system tools](https://github.com/google-gemini/gemini-cli/blob/main/docs/tools/file-system.md)). Coding agents use them to retrieve just in time ([talk, slide 8](https://hosted-files.sched.co/agntconmcpconeu26/eb/The%20Unix%20Philosophy%20for%20AI%20Agents%20-%20Filesystems%20as%20the%20Context%20Primitive%20-%202026-09-16.pdf#page=8)).

**A proposal must**

1. Define glob matching under a URI prefix, e.g. `file:///tickets/**/*.md`, returning entries in the shape outlined in Problem Statement 1.
2. Define content search under a URI prefix, executed at the source so only matches travel ([talk, slide 7](https://hosted-files.sched.co/agntconmcpconeu26/eb/The%20Unix%20Philosophy%20for%20AI%20Agents%20-%20Filesystems%20as%20the%20Context%20Primitive%20-%202026-09-16.pdf#page=7)). Each match carries its URI and a position, e.g. a line number.
3. State the query syntax, e.g. a literal string or a named regular expression dialect, and how a client learns which syntaxes a server supports.
4. Paginate and bound the results.

**Open questions**

- A ticketing system has its own query syntax. Does the contract expose source-native queries, or only a common subset?
- Is ranked or semantic search in scope, or only matching?

## Problem Statement 3: Bulk and partial reads

Agent 1 reads 40 tickets with 40 `resources/read` calls. Agent 2 needs the last hour of a log several GB in size, and `resources/read` returns the whole resource ([Resources: Reading Resources](https://modelcontextprotocol.io/specification/draft/server/resources#reading-resources)).

**Current state**

- `resources/read` takes one `uri` and returns `text` or a base64 `blob`. It has no offset, length or range ([Resources: Reading Resources](https://modelcontextprotocol.io/specification/draft/server/resources#reading-resources); [Resources: Resource Contents](https://modelcontextprotocol.io/specification/draft/server/resources#resource-contents)).
- SEP-2532 proposes `resources/stream` for raw binary over HTTP, and the Filesystems charter lists it as a proposal to reconcile ([Filesystems charter, In Scope](https://modelcontextprotocol.io/community/working-groups/filesystems#in-scope); [SEP-2532](https://github.com/modelcontextprotocol/modelcontextprotocol/pull/2532)). Range support is an open question in SEP-2532.
- The file tools in the three harnesses above do not operate on byte offsets ([talk, slide 6](https://hosted-files.sched.co/agntconmcpconeu26/eb/The%20Unix%20Philosophy%20for%20AI%20Agents%20-%20Filesystems%20as%20the%20Context%20Primitive%20-%202026-09-16.pdf#page=6)). E.g. Claude Code's `Read` and Gemini CLI's `read_file` take a line `offset` and `limit` ([Claude Code: Read tool behavior](https://code.claude.com/docs/en/tools-reference#read-tool-behavior); [Gemini CLI: read_file](https://github.com/google-gemini/gemini-cli/blob/main/docs/tools/file-system.md)).

**A proposal must**

1. Define a bulk read of several URIs in one call, with a typed error per URI (Problem Statement 4).
2. Decide whether partial reads belong here or with large-content transfer, e.g. SEP-2532. Discovery, search, metadata and small inline writes can progress independently of large-content transfer. The options are no partial reads, byte ranges with HTTP `Range` semantics ([RFC 9110, section 14](https://www.rfc-editor.org/rfc/rfc9110#section-14)), line ranges for text, or a continuation cursor.
3. Report total size and whether content remains.
4. Pin a sequence of partial reads to one revision (Problem Statement 5).

**Open questions**

- Do the content transfer size limits apply to a bulk read as a whole, or per URI?

## Problem Statement 4: Writes, metadata and typed errors

Agent 1 writes `/thread/themes.md` for Agent 3 to read. A client has no standard way to create, update or delete a resource, or to check existence without fetching the body. Each source names its own write operation, e.g. `put`, `update`, `commit` or `publish` ([talk, slide 3](https://hosted-files.sched.co/agntconmcpconeu26/eb/The%20Unix%20Philosophy%20for%20AI%20Agents%20-%20Filesystems%20as%20the%20Context%20Primitive%20-%202026-09-16.pdf#page=3)).

**Current state**

- Resources are read-only from the client ([Resources: Protocol Messages](https://modelcontextprotocol.io/specification/draft/server/resources#protocol-messages); [SEP-2571, Summary](https://github.com/modelcontextprotocol/modelcontextprotocol/pull/2571)).
- SEP-2571 proposed `resources/create`, which returns a server-assigned URI, and `resources/delete`, scoped to the creator ([SEP-2571](https://github.com/modelcontextprotocol/modelcontextprotocol/pull/2571)). It closed on 2026-09-22, and taking it up with its author is the Filesystems WG's first work item ([Filesystems charter, Active Work Items](https://modelcontextprotocol.io/community/working-groups/filesystems#active-work-items)).
- SEP-2631 proposed `files/authorizeUpload` and `files/authorizeDownload`, and stated that file URIs resolve through file transfer methods and not `resources/read` ([SEP-2631, Summary](https://github.com/modelcontextprotocol/modelcontextprotocol/pull/2631)). It closed on 2026-10-14 and continues as the File Uploads experimental extension ([experimental-ext-file-uploads PR #1](https://github.com/modelcontextprotocol/experimental-ext-file-uploads/pull/1)).
- Existence is checked by `resources/read`, which returns `-32602` when the resource is missing ([Resources: Error Handling](https://modelcontextprotocol.io/specification/draft/server/resources#error-handling)). `size` and `lastModified` appear on listing entries only, and both are optional ([Resources: Resource](https://modelcontextprotocol.io/specification/draft/server/resources#resource); [Resources: Annotations](https://modelcontextprotocol.io/specification/draft/server/resources#annotations)).
- The only defined error codes are not found (`-32602`) and internal error (`-32603`) ([Resources: Error Handling](https://modelcontextprotocol.io/specification/draft/server/resources#error-handling)).

**A proposal must**

1. Define create, update and delete on a URI, with capability negotiation ([Filesystems charter, In Scope](https://modelcontextprotocol.io/community/working-groups/filesystems#in-scope)).
2. Support client-named URIs, server-assigned URIs, or both. Agent 3 expects Agent 1's output at a known path, and SEP-2571's job submission wants a server-assigned reference ([SEP-2571](https://github.com/modelcontextprotocol/modelcontextprotocol/pull/2571)).
3. Define `stat`, returning existence, type, size, `lastModified` and revision without the body ([Filesystems charter, In Scope](https://modelcontextprotocol.io/community/working-groups/filesystems#in-scope); [talk, slide 16](https://hosted-files.sched.co/agntconmcpconeu26/eb/The%20Unix%20Philosophy%20for%20AI%20Agents%20-%20Filesystems%20as%20the%20Context%20Primitive%20-%202026-09-16.pdf#page=16)).
4. Let a server mark a URI or subtree read-only, writable or append-only, e.g. `/tickets` and `/memory`.
5. Define typed errors the agent can act on, e.g. not found, permission denied, read-only, conflict and quota exceeded ([talk, slide 9](https://hosted-files.sched.co/agntconmcpconeu26/eb/The%20Unix%20Philosophy%20for%20AI%20Agents%20-%20Filesystems%20as%20the%20Context%20Primitive%20-%202026-09-16.pdf#page=9)). RFC 9457 Problem Details is a candidate shape for the error `data` ([RFC 9457](https://www.rfc-editor.org/rfc/rfc9457)).
6. Share one transfer mechanism with uploads and downloads for content too large to send inline. The Filesystems charter asks for coordination with the File Uploads WG on where an upload flow ends and a resource write begins ([Filesystems charter, Related Groups](https://modelcontextprotocol.io/community/working-groups/filesystems#related-groups)), and the File Uploads extension draft leaves open whether Resources fit its flow ([experimental-ext-file-uploads PR #1](https://github.com/modelcontextprotocol/experimental-ext-file-uploads/pull/1)).
7. Guide server authors on choosing a resource write over a tool ([Filesystems charter, In Scope](https://modelcontextprotocol.io/community/working-groups/filesystems#in-scope)).

Authorisation policy stays out of scope beyond stating where MCP authorisation applies ([Filesystems charter, Out of Scope](https://modelcontextprotocol.io/community/working-groups/filesystems#out-of-scope)).

**Open questions**

- Does delete recurse on a directory? Is directory creation explicit or implied by a write?
- Are rename, move and append separate operations? Append-only `/memory` matters for prompt injection: a subagent that reads untrusted ticket text can add a record, and it cannot alter existing ones.
- Should tools accept Resource inputs as well as file inputs? The File Uploads extension draft leaves open whether Resources fit its flow ([experimental-ext-file-uploads PR #1](https://github.com/modelcontextprotocol/experimental-ext-file-uploads/pull/1)). If they do, Agent 3 could pass `/output/report.md` to a tool by URI. How does that reference interact with the revision in Problem Statement 5?

## Problem Statement 5: Optimistic concurrency control

Two writers to one URI cannot detect each other. Agent 3 reads `/thread/themes.md` at revision 1, Agent 1 writes revision 2, and Agent 3 then overwrites it. The server accepts both writes and Agent 1's update is lost. Two subagents that each scan half of `app.log` and both create `/task/errors.csv` hit the same problem.

**Current state**

- `Resource` carries no revision identifier ([Resources: Resource](https://modelcontextprotocol.io/specification/draft/server/resources#resource)).
- `lastModified` is an optional ISO 8601 annotation and its precision is unspecified ([Resources: Annotations](https://modelcontextprotocol.io/specification/draft/server/resources#annotations)), so two writes within one second can share a value.
- There is no create-if-absent ([Resources: Protocol Messages](https://modelcontextprotocol.io/specification/draft/server/resources#protocol-messages)).
- The Skills extension publishes a SHA-256 digest per file ([Skills: Listing Skills](https://modelcontextprotocol.io/extensions/skills/overview#listing-skills)), which is a precedent for per-resource content identifiers.

**A proposal must**

1. Define an opaque revision identifier, compared byte-for-byte like a strong HTTP `ETag` ([RFC 9110, section 8.8.3](https://www.rfc-editor.org/rfc/rfc9110#section-8.8.3)), and return it from `stat`, listings, reads and writes ([talk, slide 9](https://hosted-files.sched.co/agntconmcpconeu26/eb/The%20Unix%20Philosophy%20for%20AI%20Agents%20-%20Filesystems%20as%20the%20Context%20Primitive%20-%202026-09-16.pdf#page=9)).
2. Define conditional update and delete (`If-Match`) and create-if-absent (`If-None-Match: *`) ([RFC 9110, section 13](https://www.rfc-editor.org/rfc/rfc9110#section-13); [Filesystems charter, In Scope](https://modelcontextprotocol.io/community/working-groups/filesystems#in-scope)). The talk proposes adopting HTTP semantics where HTTP already carries them ([talk, slide 6](https://hosted-files.sched.co/agntconmcpconeu26/eb/The%20Unix%20Philosophy%20for%20AI%20Agents%20-%20Filesystems%20as%20the%20Context%20Primitive%20-%202026-09-16.pdf#page=6)).
3. Return the current revision with the precondition-failed error, so the agent can re-read and retry.
4. State whether a server that supports writes MUST support preconditions.
5. Allow reads pinned to an exact revision.

**Open questions**

- How does a server over an unversioned source produce a revision without a full read on every `stat`?
- Writing `/task/errors.csv` and `/thread/themes.md` together is the dual-write problem. The author recommends explicitly scoping out multi-URI atomicity and pessimistic locking.

## Problem Statement 6: Change notification and caching

The caching rules cover list and read results only ([Caching: Cacheable Results](https://modelcontextprotocol.io/specification/draft/server/utilities/caching#cacheable-results)). Writes need rules for the writer's cache, other clients' caches and subscribers ([Filesystems charter, In Scope](https://modelcontextprotocol.io/community/working-groups/filesystems#in-scope); [talk, slide 16](https://hosted-files.sched.co/agntconmcpconeu26/eb/The%20Unix%20Philosophy%20for%20AI%20Agents%20-%20Filesystems%20as%20the%20Context%20Primitive%20-%202026-09-16.pdf#page=16)). Agent 3, waiting for `/thread/themes.md`, can subscribe to a URI that does not exist yet, or poll.

**Current state**

- List and read results carry `ttlMs`, a freshness hint like `Cache-Control: max-age`, and `cacheScope`, which is `"public"` or `"private"` ([Caching: Time-to-Live Field](https://modelcontextprotocol.io/specification/draft/server/utilities/caching#time-to-live-ttl-field); [Caching: Cache Scope Field](https://modelcontextprotocol.io/specification/draft/server/utilities/caching#cache-scope-field)).
- `subscriptions/listen` delivers `notifications/resources/updated` per URI ([Resources: Subscriptions](https://modelcontextprotocol.io/specification/draft/server/resources#subscriptions)), and `notifications/resources/list_changed` covers the whole list ([Resources: List Changed Notification](https://modelcontextprotocol.io/specification/draft/server/resources#list-changed-notification)). A notification invalidates a fresh cache entry ([Caching: Interaction with Notifications](https://modelcontextprotocol.io/specification/draft/server/utilities/caching#interaction-with-notifications)).
- Polling clients MUST apply jitter and backoff ([Caching: Freshness Calculation](https://modelcontextprotocol.io/specification/draft/server/utilities/caching#freshness-calculation)).

**A proposal must**

1. State whether a write invalidates the writer's cached read and its cached parent listing.
2. State whether writes trigger `updated` for other subscribers, and whether create and delete trigger `list_changed`.
3. Decide whether a client can subscribe to a directory.
4. Return `ttlMs`, `cacheScope` and the new revision on write responses.
5. Define `cacheScope` for writable resources. A `"public"` response MAY be served by a shared gateway to any user ([Caching: Cache Scope Field](https://modelcontextprotocol.io/specification/draft/server/utilities/caching#cache-scope-field)), so a `"public"` entry for `/task/errors.csv` would serve one principal's working state to another.
6. Require `lastModified` to change on write.

**Open questions**

- Does a client need ordering between its write response and the resulting notification?

## Related proposals

| Proposal | Status at time of writing | Statement |
| :---- | :---- | :---- |
| [SEP-2571](https://github.com/modelcontextprotocol/modelcontextprotocol/pull/2571), Resource Submission for Agent Coordination | Closed 2026-09-22, Filesystems WG work item | Problem Statement 4 |
| [SEP-2631](https://github.com/modelcontextprotocol/modelcontextprotocol/pull/2631), File Objects and Transfer | Closed 2026-10-14, continued as the File Uploads extension ([PR #1](https://github.com/modelcontextprotocol/experimental-ext-file-uploads/pull/1)) | Problem Statement 4, cross-cutting decision |
| [SEP-2532](https://github.com/modelcontextprotocol/modelcontextprotocol/pull/2532), Resource Streaming for Binary Content Delivery | Open, listed in the Filesystems charter | Problem Statement 3 |
| [SEP-2356](https://github.com/modelcontextprotocol/modelcontextprotocol/pull/2356), Declarative file inputs | Closed 2026-06-26, File Uploads WG | Problem Statement 4 |
| [SEP-2640](https://modelcontextprotocol.io/extensions/skills/overview), Skills extension | Final | Problem Statement 1, Problem Statement 5 |

Statuses are from the pull requests, the [Filesystems charter](https://modelcontextprotocol.io/community/working-groups/filesystems) and the [Skills overview](https://modelcontextprotocol.io/extensions/skills/overview).

## Sources

- [The Unix Philosophy for AI Agents: Filesystems as the Context Primitive](https://hosted-files.sched.co/agntconmcpconeu26/eb/The%20Unix%20Philosophy%20for%20AI%20Agents%20-%20Filesystems%20as%20the%20Context%20Primitive%20-%202026-09-16.pdf), AGNTCon + MCPCon EU, 17 September 2026
- [Filesystems WG charter](https://modelcontextprotocol.io/community/working-groups/filesystems), `docs/community/working-groups/filesystems.mdx`
- [File Uploads extension draft](https://github.com/modelcontextprotocol/experimental-ext-file-uploads/pull/1), experimental-ext-file-uploads PR #1
- MCP draft specification: [Resources](https://modelcontextprotocol.io/specification/draft/server/resources), [Caching](https://modelcontextprotocol.io/specification/draft/server/utilities/caching), [Pagination](https://modelcontextprotocol.io/specification/draft/server/utilities/pagination)
- [Skills extension overview](https://modelcontextprotocol.io/extensions/skills/overview)
- [SEP-2571](https://github.com/modelcontextprotocol/modelcontextprotocol/pull/2571), [SEP-2631](https://github.com/modelcontextprotocol/modelcontextprotocol/pull/2631), [SEP-2532](https://github.com/modelcontextprotocol/modelcontextprotocol/pull/2532), [SEP-2356](https://github.com/modelcontextprotocol/modelcontextprotocol/pull/2356)
- [Claude Code tools reference](https://code.claude.com/docs/en/tools-reference), [Gemini CLI file system tools](https://github.com/google-gemini/gemini-cli/blob/main/docs/tools/file-system.md)
- [RFC 9110, HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110), [RFC 9457, Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc9457)
