# Flashback: enterprise file-version downloads

**Status: Phase 0 design proposal, 20 October 2026. No enterprise functionality is implemented by this change.**

Flashback is the proposed name for a self-service companion to Backrest and Restic.
Employees will browse and download historical versions of files they can currently
read on Windows SMB servers. They will have no production restore, overwrite,
delete, repository administration or arbitrary command capability.

The initial target is Windows Server 2019, AD DS, Entra ID, Windows 10/11 and CAD
project directories containing DWG, DXF, PDF and Office files. The portal and
repository reader may run on Linux; Windows remains authoritative for Windows
permissions.

| Priority | Capability | Delivery prerequisite |
| --- | --- | --- |
| Critical | Current NTFS and SMB permissions, SID mapping | Real Windows/AD validation before any employee download |
| High | Entra ID and LDAP/LDAPS login | Provider validation, separate identity mapping and authorization |
| High | Portal, search, versions and streamed downloads | Authorized metadata filtering and per-download checks |
| Medium | Explorer context menu and intelligent path resolution | Versioned backend contract and deployed secure portal |

## Design documents

- [Repository analysis](repository-analysis.md): observed behavior, source references and upstream risks.
- [Architecture and module boundaries](architecture.md): process isolation, identities and read-only contracts.
- [Windows paths and browser handoff](windows-paths.md): mapped drives, DFS, migrations and ambiguity handling.
- [Threat model](threat-model.md): trust boundaries, threats and release gates.
- [Implementation plan and acceptance tests](implementation-plan.md): phased PRs and test evidence required.
- [Upstream PR assessment](upstream-prs.md): candidate changes and adoption conditions.

The Windows context-menu component has a separate repository:
**flashback-explorer**.
It owns the Windows executable, Explorer registration, installer and client tests.
The backend owns the handoff protocol, source registry and authorization decisions.

## Naming and compatibility

Use **Flashback** for the planned employee portal and Explorer client, with
“Powered by Backrest / Restic” attribution. The existing Backrest administrative
application, Go module path, config format, repository GUIDs, service names,
environment variables and Restic format remain compatible. A whole-product rename
requires a separate migration PR covering installers, paths, links, update feeds,
documentation and distribution notices.

The repository retains its existing `LICENSE` and attribution. This proposal does
not assert that the Flashback name is available for commercial distribution.

## Phase 0 scope

This PR records design decisions and acceptance criteria before production code.
Planned interfaces and settings below are **not available commands or deployable
configuration**. Passing the documentation build does not validate Windows ACLs,
DFS, Entra, LDAP, large-file downloads or the proposed security boundary.
