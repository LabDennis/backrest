# Architecture and module boundaries

**Proposed design; no implementation or security certification in Phase 0.**

## Decision: separate companion service

Keep Backrest as the administrator-only backup control plane. Add Flashback as an
independently deployed service and frontend in the fork. Put the Explorer client
in its own repository. This avoids exposing Backrest's broad RPC surface to
employees and limits upstream merge conflicts.

    Explorer client ──HTTPS──> Flashback portal/API <── browser
                                   │
                      authenticated identity + mapped SID
                                   │
                     mTLS authorization request/decision
                                   │
                  Windows agent on the actual file server
                          │                      │
                     SMB share SD        current NTFS state
                                   │
                     short-lived operation-bound decision
                                   │
                    isolated historical-file reader
                                   │
                           Restic repository

Backrest continues scheduling backups, retention and repository verification on a
separate administrative origin. It is not a downstream API used with an admin
token by the employee portal.

## Proposed modules

| Boundary | Responsibility | Must not possess/expose |
| --- | --- | --- |
| flashback gateway, Go companion process | HTTP, session, policy orchestration, authorized catalog queries | Restic passwords, production write handle, admin RPC proxy |
| identity provider adapters | Entra OIDC and LDAP authentication; stable external identity | Authority to grant file access |
| identity mapper | Administrator-approved external identity to AD object/SID mapping | Email-only matching or client-supplied group/SID trust |
| source registry | Stable logical sources, target history, repository/path bindings | End-user-editable targets or credentials |
| Windows authorization agent | Resource-server-relative Windows identity, SMB and NTFS evaluation | Repository keys, restore endpoint, arbitrary supplied UNC probing |
| historical reader worker | Approved snapshot metadata and single regular-file streaming | Production mount, generic command endpoint, user-selected flags |
| catalog store | File/version index, source revisions and migration history | Authorization grants derived from indexed ACLs |
| portal frontend | Simple file/version/download experience | Backrest admin client or secret-bearing config |
| audit service/view | Append events and show authorized audit reports | Public global user activity feed |
| flashback-explorer repository | User-session path resolution and browser launch | Repository knowledge, file authorization or backup credentials |

Suggested future paths in the fork are cmd/flashback, internal/flashback/*
and a separate portal/ frontend. The Windows agent is a separately packaged
executable from the fork. Nothing imports the administrative router into the
companion. The public Go module path stays unchanged.

## Authentication and identity mapping

Entra uses a tenant-specific OIDC authorization-code flow with S256 PKCE. Validate
signature, supported algorithm, issuer, audience, tenant, nonce, state and expiry.
Allow only configured organizational users; cloud guests require explicit policy.
Key user identity by verified (tenant ID, object ID), with provider/issuer context,
not email, display name, UPN alone or arbitrary app-only tokens. Request only
openid/profile initially; Graph permissions are not assumed necessary. Entra
Conditional Access supplies MFA; the application does not implement a substitute.

OIDC callbacks use an exact configured HTTPS origin and redirect URI. Trust proxy
headers only from configured proxies. Store temporary login state and PKCE
verifiers server-side and consume once. OAuth access, ID and refresh tokens stay
server-side; no bearer credentials in browser URLs, localStorage or logs.

LDAP supports LDAPS or mandatory StartTLS with hostname/chain validation; fail
when StartTLS fails and never downgrade. Escape filter values, limit search
bases, refuse anonymous/empty-password binds, and require exactly one directory
identity. Resolve nested membership with cycle, depth and size limits, including
AD-specific primary-group considerations. LDAP groups are identity information,
not a replacement for Windows permission evaluation.

User passwords exist only transiently during bind and are never persisted.
Service-account secrets come from protected secret references/OS facilities,
not checked-in config. Prefer integrated service identities where supported.
Generic LDAP accounts need explicit mapping to an AD principal before accessing
Windows data.

The mapping record links provider identity to an AD objectGUID, current SID and
domain, with a revision and enable/disable state. Missing, ambiguous, deleted,
re-created or disabled identities deny access. Never auto-link by email or trust
a SID supplied by the browser. Mapping changes invalidate sessions and pending
operations.

Use opaque server-side sessions in Secure, HttpOnly, host-only cookies, bounded
idle/absolute lifetimes, rotation on login and revocation on logout. SameSite is
a defense in depth; CSRF tokens plus Origin checks protect state-changing cookie
requests. Rate-limit authentication and expensive operations.

## Windows authorization: critical release gate

Authorization must be evaluated relative to the **actual resource server**, not
the Linux gateway, workstation or backup identity. Run the agent on each approved
Windows file server; use mutually authenticated, explicitly authorized peers.

Prefer a complete supported Windows logon-token context with
AuthzInitializeContextFromToken when available. For a mapped SID without a suitable
token, prototype the resource-server-local Authz/S4U path and verify group and
logon semantics against real SMB opens. Do not silently accept incomplete
AuthzInitializeContextFromSid fallback results. Domain-local groups, server-local
groups, deny-only memberships, nested groups, SID history and logon characteristics
must be covered by tests. Never manually invent an Allow result from LDAP groups.

Read current share security descriptors and current file/directory descriptors
on that server. Evaluate share read rights **and** NTFS read rights, required
metadata rights and directory traversal using Windows APIs and proper generic
access mappings. Account for the user's traversal privileges, not those of the
service. Evaluate actual ACE order/inheritance with AuthzAccessCheck; do not
implement "allow if member of one group."

Unsupported conditional ACEs, central access policies, unavailable required claims,
descriptor/group lookup failures or missing share information fail closed. The
agent must report inability to decide, not a guessed grant. A privileged agent
may retrieve descriptors; its own permissions never satisfy the user's check.

Resolve reparse points against approved roots using native handle-based checks.
Bind the decision to principal, source revision, selected target, relative path,
file identity, operation, nonce and short expiry. Revalidate if any binding
changes. Check immediately before listing sensitive metadata and before starting
every download; no cached positive decision across downloads. A live-state
change between check and use must trigger rejection/recheck where detectable.

Current-state evaluation has propagation and race limits: AD/DC replication and
an already-running transfer cannot promise instantaneous global revocation.
Define freshness policy, check cancellation/revocation during long streams, and
document the residual interval. Do not claim "zero bytes after every ACL change."

**Deleted or unresolvable files are denied by default.** A readable parent or an
old snapshot ACL is insufficient. Renamed-file history requires a verified,
administrator-controlled continuity mapping to a current object. Reused paths
and uncertain object continuity must not automatically reveal older occupants.

## Read-only data and API contract

Public API sketch, version 1:

| Endpoint | Purpose and checks |
| --- | --- |
| GET /api/v1/me | Session identity and own UI capabilities |
| GET /api/v1/projects | Return only authorized project metadata |
| POST /api/v1/files/search | Bounded search; authorize results before metadata/count disclosure |
| GET /api/v1/files/{opaque-id}/versions | Current read check; approved source and catalog binding |
| POST /api/v1/files/{opaque-id}/versions/{version-id}/download | Session, CSRF, current ACL, exact version binding; streamed attachment |
| POST /api/v1/launch-intents | Bounded untrusted routing hint; no metadata or access grant |
| POST /api/v1/launch-intents/redeem | Session/CSRF, atomic single use, mapping and ACL check |
| GET /api/v1/me/downloads | Own audit history only |
| GET /api/v1/admin/downloads | Auditor/admin permission, bounded filters and pagination |

No restore, overwrite, delete, retention, config, repository URL, arbitrary command
or destination parameter exists on the employee surface. Admin/auditor UI is on
a protected administrative origin and uses separate authorization; a role claim
must never replace the file check on employee download routes.

The backend owns opaque file/version IDs. Resolve each to a registered source,
repository identity, **full immutable snapshot ID** and exact stored file path.
Validate source/snapshot membership and regular-file type before extraction.
Neither repository IDs nor client path hints authorize access. Reject directory,
symlink and special-node downloads in the first release: a directory archive can
otherwise contain unauthorized children.

Wrap Restic snapshots/ls/dump behind typed methods. All executable paths,
repository locations, secrets and options are administrator-controlled. Use
argument arrays, bounded subprocess lifetime and output, cancellation, no shell
interpolation and no hook execution. Repository keys never enter portal memory.
The reader's identity has no production write access or production mount.

Prefer storage-enforced read-only repository access for the reader. Validate the
chosen backend and Restic locking behavior; a second Restic password alone does
not create a read-only role. Any required lock-object writes need a separately
documented minimal permission model, or a coordinated read-only replica. Do not
disable locking for extraction while ignoring concurrent prune.

Stream one file with bounded memory, cancellation/backpressure and process cleanup.
Use attachment disposition with safe Unicode filename encoding, nosniff,
no-store, and no buffering of whole CAD files. Emit sanitized errors before
headers when possible; if the process fails midstream, abort and audit failure.
The initial contract does not promise resumable/range requests. Temporary storage,
if later required, is private, bounded and cleaned on cancel/restart.

## End-user and audit experience

The default employee view is deliberately simple:

1. Search within authorized projects, or arrive from Explorer at one file.
2. See filename, understandable project label, version date/time and size.
3. Choose **Herunterladen**. No repository, server, restore destination or backup
   administration knowledge is required.

Keep technical troubleshooting behind a support reference. Distinguish no backup
version, expired handoff and temporary service unavailability without confirming
the existence of an unauthorized file. Keyboard access, clear focus, German
labels, localization and screen-reader names are acceptance criteria.

The admin/auditor download log answers **who downloaded which file, when, and
which historical version**. Record immutable internal principal reference plus
display label, UTC start/end timestamps (rendered in the chosen local timezone),
logical file identity, snapshot/version, result, bytes streamed and correlation
ID. Separate requested, authorized, streaming, server-completed, failed,
cancelled and denied events. "Server-completed" does not prove the browser saved
the file. Never claim a download completed merely because a link was generated.

Ordinary users see only their own events. Auditor access is distinct from backup
administration. Restrict exports, audit audit-log access, configure retention and
protect the append-only sink. No passwords, repository credentials, tokens or raw
auth responses in logs. Store file paths in the restricted audit store only;
general telemetry uses IDs and reason codes. Durable audit-start recording is
required before releasing bytes; sink failure fails the operation closed.
A crash may leave an event incomplete; reconciliation must not invent success.

## Deployment and decisions still to validate

Windows Server 2019 hosts the authorization agent; the gateway/reader may be on
Windows or Linux. TLS for users, mTLS for agents, scoped service accounts,
separate config/data/cache directories and an explicit egress allowlist are
mandatory design constraints. Backup service privileges remain separate.

The first implementation spike must prove resource-server token completeness,
share descriptor access, native reparse handling, and the precise repository
snapshot path layout for UNC versus Linux SMB mounts. Deployment instructions
must use reserved example domains and secret references only.

See [threat model](threat-model.md) and [implementation plan](implementation-plan.md)
for release gates.

## References

- [Microsoft: authorization code flow and PKCE](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-auth-code-flow).
- [Microsoft: stable identity and claim validation](https://learn.microsoft.com/en-us/entra/identity-platform/claims-validation).
- [Microsoft: Authz context construction](https://learn.microsoft.com/en-us/windows/win32/api/authz/nf-authz-authzinitializecontextfromsid).
- [Microsoft: resource-server-relative effective permissions](https://learn.microsoft.com/en-us/troubleshoot/windows-server/windows-security/access-checks-windows-apis-return-incorrect-results).
