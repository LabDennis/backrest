# Windows path resolution and browser handoff

**Phase 4A contract proposal; implement only after identity and ACL gates pass.**

The context menu text is **Ältere Dateiversionen herunterladen**.
The dedicated **flashback-explorer** repository
owns the lightweight Windows client and its installer.

## Client workflow

1. Explorer starts a signed executable with one quoted selected path in the
   interactive user's session. Do not use cmd.exe, PowerShell evaluation, an
   elevated helper or a Windows service to resolve the user's drive mapping.
2. Classify the input as absolute local, mapped-drive, direct UNC, DFS candidate or
   unsupported device/relative path. A drive letter is never a durable identity.
3. For mapped network paths call WNetGetUniversalNameW with a bounded buffer resize
   loop. Preserve the logical UNC result before attempting physical resolution.
   Capture both logical path and diagnostic resolution status; do not transmit the
   original drive letter as an identifier.
4. For UNC/DFS, preserve the logical namespace. Probe the matching DFS root/link
   prefixes with NetDfsGetClientInfo, not an assumption that the API resolves any
   full leaf path. Use cached active-target information when available, release
   native buffers and bound all work. Do not mutate referrals or drive mappings.
5. A physical hint includes the matched DFS-link prefix, target server/share and
   remaining relative suffix. Never just replace the domain name with a server
   name. Mark absent, stale or multiple active referral results as uncertain.
6. Handle applicable symlinks/junctions with native APIs while retaining the
   original logical path. A final path is only a hint. Unsupported local devices,
   links outside known network roots, loops or failed resolution produce a useful
   error. Plain local disks are not mapped to server backups by resemblance.
7. POST a bounded UTF-8 JSON payload over HTTPS to the configured portal origin.
   The client validates server TLS and never follows a redirect carrying path
   information to an unrelated origin.
8. Open the default browser on the fixed portal origin with an opaque launch
   intent in the URL fragment. The browser removes it from visible history
   immediately and submits it in a POST body. It carries no login or download
   authority.
9. The browser authenticates if necessary. The backend then maps the path, checks
   the user's current rights and shows the file's historical versions.
10. Every selected download repeats current authorization independently.

No backup password, repository name, access token or production-write operation
is present in this client. The native Windows 11 top-level context menu is a
separate investigation; a registered command may appear under **Weitere Optionen
anzeigen**. State that limitation in the installer and rollout documentation.

## Request example

Illustrative contract, not a working endpoint:

    {
      "protocolVersion": 1,
      "logicalPath": "\\\\example.test\\projects\\design\\Bridge\\Plan A.dwg",
      "physicalHint": {
        "path": "\\\\files-a.example.test\\design$\\Bridge\\Plan A.dwg",
        "dfsLink": "\\\\example.test\\projects\\design"
      },
      "resolutionStatus": "dfs-referral-observed"
    }

Do not accept a client project ID, repository ID, SID, group list or an
authorization flag. Treat the physical hint as untrusted diagnostic input and
verify target membership server-side. An omitted physical hint does not invalidate
an otherwise unambiguous registered logical path.

## Registry model

| Record | Stable identity and stored facts |
| --- | --- |
| Source | Random immutable source ID, logical root, state and policy revision |
| Logical alias | Explicit normalized UNC/DFS root, source ID, validity and provenance |
| Physical target | Agent identity, approved server/share/root, source ID and validity |
| Backup binding | Source ID, repository immutable identity, exact snapshot path prefix, applicable snapshot set/epoch |
| File/version | Source ID, relative segments, full snapshot ID, exact stored path and node metadata |
| Continuity mapping | Explicit historical-to-current object relationship, evidence, revision and approval |
| Migration record | Old/new roots and target epochs, verified continuity and audit event |

One DFS source can have several configured physical targets and several backup
bindings. A server migration retains its source ID and old snapshot bindings;
it does not rewrite snapshot paths. A display name or Backrest plan name is not
an identity. Secondary repositories need verified source/snapshot provenance.

Use snapshot identity and controlled binding epochs, not client timestamps alone.
Conflicting epochs and duplicate catalog entries cannot silently prefer one
repository. Expose independently verified versions or reject ambiguity.

## Backend normalization and matching

Use one Windows-specific parser on every backend platform; Linux filepath
semantics are insufficient. Normalize delimiters and the supported extended UNC
prefix only after parsing. Reject malformed UTF-16/UTF-8, NUL/control characters,
relative/drive-relative paths, device namespaces, ADS suffixes, dot/dot-dot
segments and ambiguous trailing dots/spaces. Avoid repeated URL decoding and
unbounded normalization. Do not trim a valid filename into a different one.

Preserve original Unicode spelling and stored Restic path bytes. Use documented
Windows comparison semantics for matching roots; do not assume Unicode NFC,
locale-sensitive lowercasing or DNS canonicalization proves file identity.
Case-sensitive directories, normalization collisions and 8.3 aliases require
verified native identity or a safe unsupported/ambiguous result. Accept long
supported Windows paths without silent truncation; enforce explicit transport
and parser limits.

Match on complete path segments. A registered root ending in "Project" must not
match "Project-old". Overlapping nested roots require explicit registry
precedence/ownership validated at configuration time; reject unresolved overlap.
Do not guess a repository from the basename or choose the first matching prefix.

The logical path is the association key. DNS aliases are administrator-registered
equivalences, not reverse-DNS discoveries. A physical path alone may map to one
source only when an approved binding proves that association; multiple logical
sources sharing a target are ambiguous. Only contact configured agents/servers,
after syntactic and registry validation. Never probe an attacker-specified SMB
host with service credentials.

## DFS failover and migrations

A client referral can be stale or attacker-controlled. A trusted agent verifies
current namespace/link membership and the selected target. Never authorize by
finding *any* permissive target. For a supported replicated namespace, the
deployment must define a trusted current-target selection policy and demonstrate
ACL/identity equivalence across eligible targets. Conflicting ACLs, unproven
equivalence, changed file identity or an indeterminate target deny access.

If the client cannot resolve a referral, continue using the registered logical
path. If the server cannot establish current share and NTFS rights on a trusted
current target, report temporary unavailability and release no metadata/bytes.
Historical associations survive failover; a previous positive ACL check does not.

During server migration, add and validate the new target, retain old backup
prefixes, prove current-object continuity, then activate a new registry revision.
In-flight requests with older target/revision bindings re-resolve and reauthorize.
Never automatically reassign a source to an unregistered host after a DNS change.

## Handoff lifecycle and authentication order

The proposed handoff is a **routing intent, not an access token**. To support both
LDAP and Entra without storing passwords or long-lived credentials in the desktop
client, creating an intent does not authenticate the client and does not reveal
whether its path exists, maps to a repository or is readable.

Create at most a small bounded record containing untrusted path hints, a random
256-bit opaque value (store its hash), creation time and absolute expiry. Use a
60-second initial TTL, request-size caps, per-origin/IP rate limits and global
storage caps. An accepted syntax-only request always receives the same response
shape. Do no SMB/network probing at this stage.

The browser POSTs the fragment to claim it atomically into a short-lived,
browser-bound pending transaction, removing the fragment first. GET/prefetch
does not consume it. A CSRF-protected claim establishes the browser binding.
Authentication may take longer than 60 seconds, so the server-side pending
transaction has a separate bounded login lifetime (proposed five minutes), with
no authority or metadata access. On successful login, bind it to that principal
and consume it once using an atomic state transition shared across replicas.

Only then normalize/map the source, verify the optional physical hint and evaluate
current authorization. Return an opaque file view ID, never internal repository
paths. Replay, expiry, concurrent redemption, failed auth or missing binding yields
a generic handoff error and a new right-click action is required. Stealing an
intent cannot grant its creator's permissions; the redeemer must independently
qualify for the file. It can still cause denial of service, which is rate-limited
and audited.

This deliberately moves authoritative mapping and rights checks until **after
browser authentication**. It avoids turning the anonymous desktop handoff into a
file-existence oracle. If a future authenticated desktop flow is added, bind its
intent to the verified principal and require the same principal in the browser.

No OAuth/session/download credential is put in a URL. Set no-store,
Referrer-Policy: no-referrer and a strict CSP on handoff pages; include no
third-party scripts. Redact fragments/intent values from telemetry. Accept no
client-controlled return URL; server chooses a same-origin file view.

## Safe errors and diagnostics

| Condition | User-facing behavior |
| --- | --- |
| Unresolved local/mapped path | "Der Netzwerkpfad konnte nicht ermittelt werden." |
| Unknown or ambiguous source | "Diese Datei kann keiner eindeutigen Sicherungsquelle zugeordnet werden." |
| Missing permission or unverifiable current file | "Für diese Datei ist kein Zugriff auf frühere Versionen möglich." |
| Server/share/ACL service unavailable | "Die Zugriffsprüfung ist vorübergehend nicht verfügbar. Bitte erneut versuchen." |
| No authorized historical version | "Für diese Datei sind derzeit keine früheren Versionen verfügbar." |
| Expired or used handoff | "Der Aufruf ist abgelaufen. Bitte erneut im Explorer öffnen." |

Return a support correlation ID. Restricted audit events can distinguish actual
reasons; general logs contain reason codes, source IDs and operation IDs, not
passwords, tokens, raw directory responses or arbitrary supplied paths.

## References

- [WNetGetUniversalNameW](https://learn.microsoft.com/en-us/windows/win32/api/winnetwk/nf-winnetwk-wnetgetuniversalnamew)
  resolves a drive-based resource and reports required buffer sizes.
- [NetDfsGetClientInfo](https://learn.microsoft.com/en-us/windows/win32/api/lmdfs/nf-lmdfs-netdfsgetclientinfo)
  reads cached root/link information; it is not proof of current authorization.
- [Restic restoration and dump](https://restic.readthedocs.io/en/stable/050_restore.html)
  informs the extraction adapter; stored snapshot paths must be observed in tests.
