# Upstream PR assessment

Reviewed 20 October 2026. All 46 open PR titles were screened through the GitHub
API. Five relevant candidates received targeted source-diff review, including the
OIDC handler, snapshot listing changes, Windows literal matching, diff endpoint
and health handlers. This is not a full audit of all upstream changes.

**No upstream PR code is merged in Phase 0.** Changes go only to project-owned
GitHub and self-hosted repositories. Do not create PRs, comments or commits in
the upstream project as part of this work.

## Recommended candidates and conditions

| Upstream PR / inspected head | Value | Decision |
| --- | --- | --- |
| [#1411: exact bracket filenames on Windows](https://github.com/garethgeorge/backrest/pull/1411), f1fbdd17f4a5b0c0c7e7cdcca5af6664f5eeb08a | Replaces wildcard widening with literal bracket matching; adds matcher and real-Restic restore tests | Highest-priority separate compatibility fix for the admin backup engine. Reproduce tests on Windows before adoption; employee downloads still use exact dump, never restore |
| [#1373: no-lock snapshot listing](https://github.com/garethgeorge/backrest/pull/1373), af0fecd4be955c71091131b662a93f62b771e614 | Avoids listing locks orphaned by cancellation; covers both listing entry points | Good candidate for the adapter/admin reliability PR. Validate concurrency and Windows behavior; do not generalize no-lock to extraction or maintenance |
| [#1262: OIDC login](https://github.com/garethgeorge/backrest/pull/1262), 7f86917f45b3b955d46d22170faa01d02ac741fa | Uses OIDC discovery, code exchange, S256 PKCE, state and nonce | Reference implementation only; adapt deliberately to the separate identity provider boundary, not a wholesale cherry-pick |
| [#1364: snapshot diff viewer](https://github.com/garethgeorge/backrest/pull/1364), c85c514475d2aca94a8c1e409a5144f77794619c | Useful future "what changed" view | Defer until per-file filtering is proven; never expose whole-repository diff results to employees |
| [#1259: health/readiness endpoints](https://github.com/garethgeorge/backrest/pull/1259), 226eccd2620ed8f3f6ed8cbd473ac08f89e32855 | Useful deployment probes | Adapt the concept with bounded resource usage and sanitized output; original is not sufficient for Flashback readiness |

## Reasons not to import OIDC unchanged

The inspected OIDC identity contains email/name rather than stable Entra tid+oid,
and the callback mints the session subject from email. That conflicts with this
project's explicit identity requirement. Its verifier/PKCE/state/nonce structure
is useful to study, but it does not supply Windows identity mapping or ACL checks.

The fallback redirect derives origin from the request Host and
X-Forwarded-Proto; Flashback requires a fixed configured origin and trusted-proxy
handling. Existing long-lived administrative session semantics are also not the
new revocable session model. The PR includes extensive generated changes and a
Keycloak database dump, so isolate useful code instead of importing test data and
config migrations wholesale. GitHub reported merge conflicts at inspection time.

## Diff viewer and health caveats

The diff handler builds a free-form RunCommand string from supplied snapshot
identifiers, buffers the output and parses human-oriented lines. Flashback needs
validated full snapshot IDs, typed argument construction, bounded output, stable
structured parsing and current authorization for each exposed path. No shell
execution vulnerability is asserted here; the concern is unsuitable input and
authorization boundaries for an employee service.

The proposed readiness handler times out its HTTP wait while its underlying
OpLog query has no cancellation context, which can leave goroutines working after
timeouts. It also returns underlying error text, and a test named as a failure
case actually asserts readiness for an empty healthy store. Adapt with bounded,
coalesced checks, generic public results and genuine timeout/failure tests.
Gateway health must also account for required policy/audit dependencies; one
successful database query does not establish download safety.

## Additional backlog from title screening

[Snapshot-browser single-child expansion #1266](https://github.com/garethgeorge/backrest/pull/1266)
may improve navigation, and
[German translation corrections #1076](https://github.com/garethgeorge/backrest/pull/1076)
may improve usability. These were title-screened only, not code-reviewed.
Generic dependency/update PRs require their own compatibility/security review.
No new notification provider, backup hook or repository administration feature
belongs in the simple end-user portal.

## Adoption procedure

Fetch the candidate into a separate review branch, pin the inspected head,
read every changed production/test file, preserve attribution, check license
compatibility, reproduce the failing case and run the relevant regression suite.
Open a dedicated PR only in the project-owned fork. Re-check the current upstream
state before adoption; these PRs and hashes may have changed.

For the five inspected heads GitHub exposed a successful license/CLA status,
but no green build/test checks in statusCheckRollup. Author-reported validation
is useful context and **is not a test run performed for this fork**.
