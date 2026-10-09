# MCP Filesystems (Experimental Extension)

> **Status:** Experimental. This work is for prototyping and feedback only, and is not an accepted or official MCP extension.

This repository holds the working documents and reference implementations for **Resource operations**: a common interface for operating on MCP Resources beyond a single read (`create`, `update`, `delete`, a metadata read (`stat`), scoped listing, search, and glob matching), with optimistic concurrency control on writes. It is maintained by the Files Working Group, which replaces the File Uploads and Filesystems Working Groups.

**Charter:** [Files Working Group charter](https://github.com/modelcontextprotocol/modelcontextprotocol/pull/3389) (in review): mission, scope, leadership, and active work items.
**Discord:** `#filesystem-wg` on the [MCP Discord](https://modelcontextprotocol.io/community/communication#discord).
**Open work:** [pull requests](https://github.com/modelcontextprotocol/experimental-ext-filesystems/pulls) and [issues](https://github.com/modelcontextprotocol/experimental-ext-filesystems/issues). Proposals, findings, and implementations are welcome.

## Repository Contents

| Document | Description |
| :--- | :--- |
| [Open Questions](docs/open-questions.md) | Questions the group needs to settle, starting with the charter's |
| [Decision Log](docs/decisions.md) | ADR-lite record of the group's decisions, and the vehicle for proposing changes |
| [Related Work](docs/related-work.md) | SEPs, extensions, and discussions this work builds on |
| [Implementations](docs/implementations.md) | Servers, SDKs, and hosts that implement Resource operations |
| [`python/`](python/) | Python reference implementation |
| [`conformance/`](conformance/) | Conformance scenarios for each new method and schema element |

Nothing under `docs/` is part of a specification. A specification will live under `specification/` once the group has accepted text to put there.

## Implementations

| Language | Directory | Package | Status |
| :--- | :--- | :--- | :--- |
| Python | `python/sdk/` | `mcp-ext-filesystems` | Planned |

The charter also lists a TypeScript SDK reference implementation ([@ochafik](https://github.com/ochafik)).

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for how to participate.

## License

Apache License 2.0. See [LICENSE](LICENSE).
