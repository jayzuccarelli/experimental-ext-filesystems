# Scenarios: write conflicts

Implementation-independent checks for the conflict behaviour in the [Resource operations strawman](../docs/proposals/resource-operations.md). A harness runs them against any server that declares `io.modelcontextprotocol/resource-operations`, using one or two client connections.

Conventions:

- `<U>` is a URI the server accepts for writes and that does not exist at the start. The harness gets it from configuration.
- `<V…>` are versions captured from earlier responses. Versions are opaque: compare them only for equality, never parse them.
- "Precondition failed" means the error code assigned to precondition failures (provisionally `-32010`).
- Each step lists the request and the checks on its response. A step fails the scenario if any check fails.

## RO-CONFLICT-1: create-if-absent

| # | Connection | Request | Expect |
| :--- | :--- | :--- | :--- |
| 1 | A | `resources/create {uri: <U>, contents: {uri: <U>, text: "a"}}` | Result. `version` is a non-empty string; save as `<V1>`. `changes` contains `{uri: <U>, effect: "created", version: <V1>}`. |
| 2 | B | `resources/create {uri: <U>, contents: {uri: <U>, text: "b"}}` | Error. Code is precondition failed. `data.uri` = `<U>`, `data.reason` = `"alreadyExists"`, `data.currentVersion` = `<V1>`. |
| 3 | A | `resources/stat {uri: <U>}` | Result. `version` = `<V1>`. The second create changed nothing. |

## RO-CONFLICT-2: lost update

The memory-server case from [use cases](../docs/use-cases.md): two sessions read the same version and both write it back.

| # | Connection | Request | Expect |
| :--- | :--- | :--- | :--- |
| 1 | A | `resources/create {uri: <U>, contents: {uri: <U>, text: "0"}}` | Result. |
| 2 | A | `resources/stat {uri: <U>}` | Result. Save `version` as `<V1>`. |
| 3 | B | `resources/stat {uri: <U>}` | Result. `version` = `<V1>`. |
| 4 | A | `resources/update {uri: <U>, contents: {uri: <U>, text: "0A"}, ifMatch: <V1>}` | Result. Save `version` as `<V2>`; `<V2>` != `<V1>`. `changes` contains `{uri: <U>, effect: "updated", previousVersion: <V1>, version: <V2>}`. |
| 5 | B | `resources/update {uri: <U>, contents: {uri: <U>, text: "0B"}, ifMatch: <V1>}` | Error. Code is precondition failed. `data.reason` = `"versionMismatch"`, `data.currentVersion` = `<V2>`. |
| 6 | B | `resources/stat {uri: <U>}` | Result. `version` = `<V2>`. The rejected write changed nothing. |
| 7 | B | `resources/update {uri: <U>, contents: {uri: <U>, text: "0AB"}, ifMatch: <V2>}` | Result. `changes` contains `previousVersion: <V2>`. |
| 8 | B | `resources/delete {uri: <U>, ifMatch: <V1>}` | Error. Code is precondition failed. `data.reason` = `"versionMismatch"`. |
| 9 | A | `resources/stat {uri: <U>}` | Result. The resource still exists. |

Optional, if the server reads back what it stores: after step 7, `resources/read {uri: <U>}` returns text `"0AB"`.
