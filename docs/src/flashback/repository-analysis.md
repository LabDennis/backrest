# Repository analysis

Reviewed fork: `LabDennis/backrest`, commit
[`148ba12c12a7a4fef6d3c6c8e675ca2c5e7d3635`](https://github.com/LabDennis/backrest/commit/148ba12c12a7a4fef6d3c6c8e675ca2c5e7d3635).
Analysis date: 20 October 2026. This is a structural and targeted security-boundary
review, not a full security audit or certification of every source file.

## Structure and stack

| Location | Responsibility and observations |
| --- | --- |
| `cmd/backrest` | Go application startup, HTTP routing, environment and platform/tray integration |
| `cmd/docker-entrypoint` | Container entry point |
| `internal/api` | Connect RPC handlers, download handler, authentication and multihost synchronization |
| `internal/auth` | Configured username/password authentication, bcrypt hashes and JWTs |
| `internal/config`, `proto/v1/config.proto` | Configuration, validation, migrations and persistent repository GUIDs |
| `internal/orchestrator` | Scheduling, repository wrappers and backup/restore/maintenance tasks |
| `internal/oplog`, `kvstore`, `logstore` | Operation history, SQLite-backed stores and logs |
| `internal/hook`, `metric`, `eventemitter` | Hooks, metrics and event delivery |
| `internal/queue`, `protoutil`, `platformutil`, `ioutil` | Scheduling and platform/support utilities |
| `internal/cryptoutil`, `resticinstaller` | Peer cryptography and Restic binary provisioning/verification |
| `pkg/restic` | Restic CLI wrapper, outputs, execution and streaming dump |
| `proto`, `gen/go`, `webui/gen/ts` | Protobuf contracts and generated Go/TypeScript clients |
| `webui/src` | React 19, TypeScript 6, Chakra UI 3, Vite 8, Connect-Web and Paraglide localization |
| `webui/src/test`, `webui/e2e` | Vitest/Testing Library and Playwright coverage |
| `test`, `internal/testutil` | Go end-to-end tests and test helpers |
| `docs` | VitePress documentation with a pnpm lockfile |
| `build`, `.goreleaser.yaml`, Dockerfiles | Inno Setup, macOS bundles and cross-platform packaging |
| `.github/workflows`, `scripts` | Tests, documentation, releases and maintenance |
| `.devcontainer`, `.vscode`, Nix files | Developer tooling and reproducible environment definitions |

The module declares **Go 1.26**. The README and CI setup still mention earlier Go
versions. Do not treat the README's minimum version as authoritative over `go.mod`.
The documentation workflow also used `npm ci` and a nonexistent
`docs/package-lock.json` despite the checked-in `docs/pnpm-lock.yaml`; this PR
aligns that workflow with pnpm to validate these documents.

No `AGENTS.md` or `CLAUDE.md` was found in the cloned repository. New production
modules should follow Go package conventions and existing TypeScript component
patterns; generated protobuf files must be regenerated from their source.

## Existing authentication and authorization

[`internal/auth/auth.go`](https://github.com/LabDennis/backrest/blob/148ba12c12a7a4fef6d3c6c8e675ca2c5e7d3635/internal/auth/auth.go)
authenticates configured users with bcrypt and issues seven-day HS256 JWTs keyed
by the configured username. The
[`middleware`](https://github.com/LabDennis/backrest/blob/148ba12c12a7a4fef6d3c6c8e675ca2c5e7d3635/internal/auth/middleware.go)
accepts Basic authentication or bearer tokens, and can pass requests through when
authentication is disabled.

[`webui/src/api/client.ts`](https://github.com/LabDennis/backrest/blob/148ba12c12a7a4fef6d3c6c8e675ca2c5e7d3635/webui/src/api/client.ts)
keeps the administrative JWT in localStorage. The regular user model contains no
enterprise SID mapping or per-file Windows authorization policy. Multihost peer
permissions belong to replication and must not be confused with employee ACLs.
No Entra OIDC, LDAP login or end-user NTFS/SMB permission evaluation was found in
the inspected authentication and request paths.

## Snapshot browse, download and restore trace

1. The React `SnapshotBrowser` calls the generated Connect client.
2. `cmd/backrest/backrest.go:newRootMux` routes the Backrest RPC service through
   the shared authentication middleware.
3. `internal/api/backresthandler.go` uses the orchestrator/repository wrappers for
   snapshot listing, file listing, restore scheduling and download URL creation.
4. `internal/orchestrator/repo/repo.go` delegates to `pkg/restic`.
5. `pkg/restic` launches Restic processes; `Dump` streams stdout to an `io.Writer`.

The
[`Backrest service`](https://github.com/LabDennis/backrest/blob/148ba12c12a7a4fef6d3c6c8e675ca2c5e7d3635/proto/v1/service.proto)
also exposes configuration, restore, retention-related tasks and free-form
`RunCommand`. The existing restore handler avoids an already-existing target,
but it still schedules filesystem writes. Hiding UI buttons cannot turn this
service into the proposed employee API.

[`GetDownloadURL` and signing](https://github.com/LabDennis/backrest/blob/148ba12c12a7a4fef6d3c6c8e675ca2c5e7d3635/internal/api/signing.go)
create a signed URL with a 24-hour expiry and an operation/path payload.
[`/download/`](https://github.com/LabDennis/backrest/blob/148ba12c12a7a4fef6d3c6c8e675ca2c5e7d3635/cmd/backrest/backrest.go)
is routed outside the regular login middleware. The
[`download handler`](https://github.com/LabDennis/backrest/blob/148ba12c12a7a4fef6d3c6c8e675ca2c5e7d3635/internal/api/downloadhandler.go)
checks that signature and operation, then dumps snapshot data or packages an
existing restore directory. It does not revalidate the requesting employee's
current NTFS and share rights. This is an administrative download facility, not
a suitable self-service authorization contract.

The snapshot indexing task records snapshot metadata as operations and marks
disappearing snapshots forgotten. It is not a persistent per-file historical
search index. File discovery and indexing must be designed separately.

## Stable versus internal interfaces

The [existing API documentation](../docs/api.md) marks only documented APIs as
stable: backup triggering and operation retrieval, with an explicit warning that
operation shapes may change. The remaining Connect methods and protobuf schemas
are implementation interfaces. A `pkg/` location alone does not establish a
long-term compatibility guarantee for the Restic wrapper.

Use the Restic CLI through a narrow, tested adapter for snapshot enumeration,
file metadata and single-file extraction. Reuse `pkg/restic` where its behavior
fits, behind an interface that exposes only `Snapshots`, `ListDirectory` and
`DumpFile`. No generic flags, repository URL, command or environment may come from
an end user. Pin and contract-test the Restic version and JSON format.

Do not give the portal a Backrest admin token, proxy its API, read its entire
secret-bearing config, or couple the new catalog to the private operation schema.

## Maintenance implications

- Add a companion process and separate frontend rather than rewriting upstream
  routing, authentication and config migration paths.
- Keep new catalog data and migrations in a separate database.
- Pin source-to-snapshot mappings independently of display names and drive letters.
- Preserve upstream history, license, module paths and repository formats.
- Keep Windows native API code behind Windows build tags and small interfaces.
- Require real Windows Server/AD/DFS tests; Linux unit tests cannot establish ACL
  correctness.
- Restic retention and concurrent prune can invalidate catalog entries or running
  reads. Coordinate access or fail explicitly; never silently select another file.
- Fork CI/release configuration needs an explicit audit before shipping artifacts:
  upstream image names, publishing credentials and release destinations are not
  automatically appropriate for Flashback.

## Verification boundary

The source tree, manifests, router, auth, download, restore, indexing, client,
packaging and workflow paths above were inspected. Phase 0 changes documentation
and its build workflow only. Runtime tests of unchanged Backrest and enterprise
integration tests are not evidence claimed by this document.
