# MCP Filesystems Python SDK

Python reference implementation of the experimental MCP Resource operations extension, built on the [MCP Python SDK](https://github.com/modelcontextprotocol/python-sdk) v2.

**Status:** Planned. This package is a placeholder until the group has a proposal to implement; see [open questions](../../docs/open-questions.md).

## Development

```bash
uv venv && uv pip install -e . pytest ruff mypy
ruff format --check . && ruff check .
mypy src/mcp_ext_filesystems
pytest
```

## License

Apache License 2.0. See [LICENSE](../../LICENSE).
