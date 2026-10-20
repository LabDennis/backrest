# Implementation plan and acceptance tests

**Incremental delivery plan. None of these runtime tests has been run for an
implementation in Phase 0.** Each implementation PR must include its own observed
test results and unresolved limitations. All commits and PRs target project-owned
repositories only; upstream is a read-only reference.

## PR sequence and gates

| Phase / proposed PR | Smallest maintainable result | Dependency and acceptance gate |
| --- | --- | --- |
| 0 — this design | Repository analysis, boundaries, threat model, source/path contract, upstream assessment | Documentation builds, links resolve, no private deployment data |
| Foundation | Separate gateway/reader/agent interfaces; versioned registry and catalog schema; migration/rollback tests | No public download routes; default-deny authorization stub |
| 1A — identity/session | Tenant-restricted Entra OIDC+PKCE, opaque sessions, provider configuration | Negative issuer/audience/nonce/state/tenant tests; revoked session denied |
| 1B — LDAP and SID mapping | Validated LDAPS/StartTLS, nested identity groups, explicit hybrid mapping | Ambiguous/missing/recreated identity, TLS and injection tests |
| 2A — Windows authorization spike | Real resource-server Authz context; share and NTFS read/traverse checks | Lab comparison to actual SMB reads on Server 2019; unsupported cases deny |
| 2B — agent hardening | mTLS, target allowlist, handle containment, current-state decisions and audit | Reparse races, share denies, expired decisions, revoked permissions |
| 5A — historical reader/catalog | Reuse Restic snapshots/ls/dump; indexed versions and registered backup roots | Exact single-file extraction; no production writes; prune/cancel tests |
| 3A — simple employee portal | Authorized project/search/version/download flow and own download history | End-to-end identity + ACL + stream; no administrative API access |
| 3B / 6 — audit view | Restricted admin/auditor log of who/when/file/version/result | Role isolation; durable starts; complete/partial/failed semantics |
| 4A — path registry/handoff | Stable logical-source matching, DFS targets, migration epochs, single-use intents | All path/handoff cases below pass in API tests |
| 4 — explorer repository | Signed launcher, Windows context command, silent installer/uninstaller, GPO guidance | Windows 10/11 normal-user sessions and end-to-end portal test |
| 5B — backup consistency | Secondary repositories, verification, retention coordination and VSS investigation | File-server-local VSS lab; CAD application consistency documented |
| 6 / 7 — release hardening | Security regression suite, quotas, redacted logs, complete deployment docs | Windows and Linux deployment rehearsals; upgrade/rollback proof |

The non-numeric delivery order is intentional: a portal cannot safely download
without an authorization agent and extraction adapter. Security controls ship
with each phase, not as a late patch. Unfinished Windows authorization blocks
public downloads even if login and UI are ready.

No major refactoring of Backrest is required by the proposal. If implementation
uncovers a required upstream-core change, document the concrete dependency and
ship it as its own reviewable PR.

## Automated acceptance matrix

U = portable unit/contract/fuzz test; W = actual Windows/AD/SMB/DFS lab test;
E = browser/API end-to-end test; R = real disposable Restic repository.

| ID | Scenario | Required result | Layer |
| --- | --- | --- | --- |
| P01 | X: maps to files-a; Z: maps to files-b | Each resolves its own logical UNC; never collide by suffix | U,W |
| P02 | Alice and Bob both use X: for different shares | Resolve in each interactive session; no global drive cache | W,E |
| P03 | Direct UNC and registered server alias | Same source only through explicit verified alias | U,W |
| P04 | DFS namespace with two targets | Logical source remains stable; physical hint grants nothing | U,W |
| P05 | DFS failover while a handoff/download is pending | Re-resolve/recheck current target; same historical association | W,E |
| P06 | Old client referral is absent or stale | Logical mapping may proceed; server ACL still required | U,W |
| P07 | DFS target ACLs differ | Reject unsupported equivalence/target uncertainty; never any-allow | W |
| P08 | Server migration with old and new backup prefixes | Old versions retained under same source; new ACL authority verified | U,W,R |
| P09 | UNC alias not registered; DNS changes | No guessed source or unapproved network contact | U,W |
| P10 | Nested project roots, Project versus Project-old | Segment matching; explicit nested ownership; ambiguity rejected | U |
| P11 | Two equally valid backup mappings | Useful ambiguous-source error; no first-match fallback | U,E |
| P12 | Unicode, combining forms, German/CJK names and long paths | Preserve exact file identity; no truncation/normalization collision | U,W,R,E |
| P13 | Extended UNC prefix and mixed separators | Supported forms normalize once; device namespaces rejected | U,W |
| P14 | ADS, dot-dot, control bytes, trailing dots/spaces, relative drive path | Safe rejection; no backup/SMB access | U,E |
| P15 | Junction/symlink inside root, outside root, loop, target swap | Verified in-root case only; reject escape, loop and race | W |
| P16 | 8.3 alias or case-sensitive directory collision | Verify native identity or reject ambiguity | W |
| P17 | Missing, renamed or deleted file | Deny unverifiable current object; no parent/snapshot-ACL fallback | U,W,E |
| P18 | New file reuses old path | No automatic older-occupant history without continuity evidence | U,W,R |
| P19 | Share offline or DC/agent unreachable | Fail closed, bounded timeout, useful retry message | W,E |
| A01 | Entra same email but different tid/oid | Different identity; no inherited mapping | U,E |
| A02 | Wrong issuer/audience/tenant, expired token, state/nonce replay | Login refused; no session issued | U,E |
| A03 | LDAPS bad cert, StartTLS failure, empty password, filter injection | Reject; no plaintext fallback or ambiguous bind | U,W |
| A04 | Nested AD and server-local groups; explicit/inherited denies | Match actual user SMB read result | W |
| A05 | NTFS allow/share deny and share allow/NTFS deny | Both deny | W |
| A06 | Directory traversal denied; user's traversal privilege differs | Match supported Windows semantics, not service privileges | W |
| A07 | Rights revoked after snapshot or after version listing | Next metadata/download request denies | W,E,R |
| A08 | Agent runs with backup privileges while employee lacks access | No employee grant or byte disclosure | W,E |
| A09 | Missing/duplicate SID mapping; deleted/disabled AD identity | No mapping guess and no file access | U,W,E |
| A10 | Conditional ACE/claims unsupported or group enumeration incomplete | Explicit unsupported/deny result | W |
| H01 | Expired, replayed or concurrently redeemed handoff | At most one successful claim; no credential behavior | U,E |
| H02 | Login outlives initial intent TTL | Only bounded claimed transaction survives; no auth bypass | U,E |
| H03 | Stolen intent used by another user | Redeemer's own current rights required | E,W |
| H04 | Browser prefetch, third-party URL, forged origin/return URL | GET cannot consume; no cross-origin redirect/data leak | U,E |
| H05 | Unknown path posted anonymously | Same syntax-only result shape; no existence or mapping oracle | U,E |
| D01 | Guessed file/version ID or snapshot from different source | Deny before metadata/bytes | U,E,R |
| D02 | Directory, symlink or special snapshot node selected | Single-file API rejects it | U,R |
| D03 | Bracket filenames and misleading siblings | Extract exactly chosen regular file; no wildcard widening | W,R |
| D04 | Multi-GB DWG, slow receiver and concurrent requests | Bounded memory/process count and working cancellation | E,R |
| D05 | Snapshot removed/pruned or worker fails during stream | Abort, record partial failure; no substituted version | E,R |
| D06 | Injection in path, snapshot ID, arguments or download filename | Fixed command/argv, safe headers, no secret leakage | U,E |
| D07 | Employee calls restore/config/RunCommand/forget/prune | Route unavailable or explicitly denied; no production modification | E |
| D08 | Before/after hashes and ACLs of production fixture | Download operations leave fixture unchanged | W,E |
| L01 | Download start, completion, cancellation, denial and crash | Accurate event states, user/time/file/version/bytes recorded | U,E |
| L02 | Audit sink unavailable | No bytes released without durable start event | U,E |
| L03 | Employee requests another user's audit events or export | Deny; auditor queries are themselves audited | U,E |
| L04 | Secrets/tokens/real internal names planted in error inputs | No leak to generic logs, UI or published fixtures | U,E |
| UI01 | New end user opens from Explorer without repository knowledge | File versions appear after login; clear download action | E,W |
| UI02 | Keyboard/screen reader and localized dates/errors | Accessible names/focus; understandable German defaults | E |
| I01 | Silent install/upgrade/uninstall through GPO | Safe command quoting; no admin required during normal use | W |
| I02 | Windows 11 classic versus native menu placement | Actual supported placement documented; no unverified native claim | W |

Fuzz the path parser and registry matcher with arbitrary Unicode, separators and
prefix collisions. Use real APIs in Windows integration tests, not mocks alone.
Never run destructive fixtures against production shares or repositories.

## Backup and deployment work

Backrest/Restic retains schedules, deduplication, retention and repository
verification. Add bounded historical indexing and invalidation, including external
snapshots and secondary repositories, without treating the catalog as an ACL
source. Preserve old bindings during migration and test rollback against a backup
of the independent Flashback database.

Restic documents Windows filesystem snapshots via VSS. Determine whether the
backup process runs locally on the file server and which writers/volumes are
covered. Ordinary reads over SMB do not establish VSS or application-consistent
CAD backups. See [Restic backup documentation](https://restic.readthedocs.io/en/stable/040_backup.html).
Record tested consistency levels rather than promising application consistency.

Deployment deliverables must include Windows Server 2019 and Linux gateway/reader
instructions, Entra app registration without unnecessary Graph scopes, LDAPS/CA
configuration, hybrid mapping examples, agent enrollment/certificate rotation,
source registration, reverse-proxy examples, GPO deployment, signed Explorer
installer, retention/audit policy, upgrades and rollback. All examples use reserved
domains such as example.test and never actual internal infrastructure.

## CI and evidence

Phase 0 validates the VitePress build and documentation links; runtime capabilities
remain pending. Later CI includes Go unit/static analysis on Linux and Windows,
frontend type checks and browser tests, Restic contract tests and a separate
protected Windows/AD/DFS integration runner. Public/untrusted PRs must never get
enterprise credentials or access to that runner.

For each implementation PR record exact commit/tool versions, commands, platform,
passed/failed/skipped cases and reasons. Model-generated suggestions and existing
upstream PR descriptions are review inputs, not test evidence. Release requires
the mandatory Windows lab suite, not just a green portable unit suite.
