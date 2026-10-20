# Threat model

**Design baseline, 20 October 2026. All mitigations below are requirements to
implement and test, not claims about functionality already present.**

## Assets, actors and trust boundaries

Protect historical CAD/document bytes, filenames and project membership, current
ACLs, directory identities, repository keys, sessions and download audit records.
Production files must not be modified through the self-service application.

Actors include a legitimate employee, an employee with revoked access, an
unauthenticated requester, a malicious workstation, an auditor, backup and source
administrators, and compromised gateway/worker/agent processes. An administrator
already possessing repository encryption keys can read repository data outside
the portal; this is not a protection the portal can promise.

Untrusted inputs include Explorer paths, DFS hints, file IDs, browser payloads,
HTTP headers and snapshot content. The gateway trusts only configured identity
providers, explicitly registered source/agent identities and administrator-owned
registry records. The browser, gateway, Windows agent, historical reader and
backup administrator are separate security boundaries.

## Threat/control/test matrix

| Threat | Required control | Required evidence |
| --- | --- | --- |
| Another tenant's login or recycled email | Signature/issuer/audience/tenant validation; stable tid+oid; explicit AD mapping | Wrong tenant, missing oid, same email/different identity and app-only token denied |
| LDAP injection or credential downgrade | Escaped filters, bounded search, validated LDAPS/StartTLS, no cleartext fallback | Injection payloads, bad cert, empty bind and StartTLS failure denied |
| Incomplete Windows group context | Resource-server-relative token/Authz context, supported semantics only | Nested, domain-local, server-local and deny-only groups compared to real SMB reads |
| Backup account grants access accidentally | Separate end-user context and intersected SMB/NTFS checks | Privileged agent can inspect descriptors while denied user still gets no bytes |
| Share deny ignored | Both share SD and NTFS SD evaluated for exact target | NTFS allow/share deny and reverse cases both deny |
| Revoked access after backup | Current-state check before each metadata response/download | Revoke group/share/file permission after snapshot; next operation denies |
| Stale mapping or substituted source | Stable source ID, revision, immutable snapshot binding | Swap target, repository, snapshot or mapping between authorization and extraction |
| UNC/DFS hint used for credential relay | Parse and allowlist registry before networking; restrict agent egress | Attacker UNC, IP literal or forged physical hint causes no unapproved SMB contact |
| Path traversal/ADS/device tricks | Windows parser, segment containment, explicit unsupported forms | Dot segments, ADS, device prefix, encoded separators, 8.3 collisions, long paths |
| Symlink/junction escape or race | Native handle-bound identity and containment; revalidate changes | Swap reparse target before/during check; deny outside-root result |
| Deleted file gains access from parent | Default deny if current object cannot be checked | Readable parent and permissive historical ACL do not grant deleted file |
| Recreated or renamed path leaks old occupant | Verified continuity policy; no basename/hash guessing | Same path reused by different file; unverified rename fails closed |
| More permissive DFS replica selected | Trusted target selection, tested equivalence, no "any allow" rule | Stale client referral, failover and conflicting target ACLs |
| Handoff replay/theft/login confusion | Random routing-only intent, short TTL, atomic claim, browser/auth binding | Two simultaneous claims, cross-browser claim, expired intent, different user |
| Session theft/CSRF/XSS | HttpOnly Secure cookies, state/nonce/PKCE, Origin+CSRF, strict CSP | Login CSRF, forged download, logout, untrusted Host/proxy headers |
| File/version ID enumeration | Per-request binding and current ACL before returning metadata | Guessed IDs, search counts and error messages reveal no unauthorized existence |
| Archived directory contains denied child | Single regular-file extraction only initially | Folder/symlink/special-node selection rejected |
| Client command/flag injection | Typed allowlisted adapter, full snapshot IDs, argv execution | Options disguised as paths/IDs cannot change repository or run commands |
| Broad administrative API reachable | Separate process/origin/router and network policy | User session cannot invoke config, restore, RunCommand, forget, prune or peer APIs |
| Credentials or paths in logs | Structured reason codes and restricted audit store | Secret sentinels absent from stdout, error body, proxy logs and browser telemetry |
| Resource exhaustion | Request, rate, concurrency, index, subprocess and stream limits | Large tree, slow reader, repeated intents, disconnect and process failure tests |
| Prune races with extraction | Defined repository coordination or consistent replica | Deleted snapshot aborts explicitly; no silent fallback to another version |
| Audit repudiation/privacy leak | Durable start/end events, append-only sink, role filters, retention | Denied/failure/partial flows recorded; employee cannot query others' history |
| Malicious historical content | Attachment-only delivery, nosniff, no active-content preview | HTML/SVG filenames/content cannot execute as portal origin |
| Installer/update substitution | Signed client/package, protected config, verified update provenance | Tampered installer, path-to-executable quoting, downgrade/uninstall tests |

## Availability and audit failures

An unavailable DC, file server, share descriptor, agent, identity mapping or
authorization context denies data access. The portal may show a generic retry
message with a support ID. Do not keep using a previous grant during an outage.

Audit start must be durable before data is released. Mark completion only after
the reader and server stream complete successfully. On disconnect, cancellation,
crash or incomplete final logging, preserve an incomplete/failed state and reconcile
it without inventing success. Restrict audit search/export by role, escape export
values (including spreadsheet-formula prefixes), and audit report access.

## Explicit limits and release blockers

- Domain replication and live filesystem races limit instantaneous revocation.
  Measure and document freshness and in-flight cancellation behavior.
- Real Windows permission behavior cannot be established by mocked Authz results.
  No production release before the Windows/AD test gate passes.
- Deleted objects and uncertain rename/migration continuity remain denied until a
  separately reviewed policy can establish current authorization.
- Unsupported claims-based ACLs, encryption/resource policies, share types or
  identity topologies fail closed and are documented as unsupported.
- Repository compromise and trusted administrator abuse remain separate risks;
  process isolation reduces exposure but does not make the reader harmless.
- No employee API can promise that a user will not manually overwrite a production
  file after downloading it. The enforced boundary is that this application offers
  no production-write operation.
- No production rollout, tenant registration or infrastructure change is performed
  by this Phase 0 PR.
