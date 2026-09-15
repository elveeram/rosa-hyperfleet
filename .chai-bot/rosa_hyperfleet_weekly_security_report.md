# Scheduled report: ROSA HyperFleet family weekly security report

You are running a **cron** scheduled task, once weekly (Saturday, 00:00 UTC),
that runs an Adversary Groundwork security scan against every
`rosa-hyperfleet*` repo in `openshift-online` and posts **one consolidated
report** to Slack — replacing the previous per-repo weekly notifications.
**Always produce a report.** This is a multi-turn task: you will be woken up
again as each repo's scan completes — read "State tracking across turns"
below before doing anything else.

This does not perform CVE scanning, runtime testing, or penetration testing —
it's static/adversarial analysis of each repo, same as the `security:adversary`
skill.

## Goal

Discover every `rosa-hyperfleet*` repo, run a Groundwork-mode `/adversary`
scan against each one in parallel, and post a single consolidated Slack
report combining all results: an executive summary for the highest-
criticality repos, the top aggregated critical/high findings across the
whole family, a per-repo severity table, and full per-repo results.

**Critical repos** (their findings drive the executive summary):
`rosa-hyperfleet`, `rosa-hyperfleet-api`, `rosa-hyperfleet-kube-applier`.
**Standard repos** (scanned and included in every other section, just not
the executive summary): `rosa-hyperfleet-cli`, `rosa-hyperfleet-zoa`,
`rosa-hyperfleet-internal`. This split is about report emphasis, not scan
scope — every discovered repo gets the same full scan.

## State tracking across turns — read this first

This task dispatches N independent background scans (one per discovered
repo) and gets woken up again each time one finishes. **On every wake-up,
before doing anything else, check how many of the N repos you dispatched
have actually reported a result so far** by reviewing your own prior turns
in this conversation for completion messages. Two cases:

- **Not all N have reported yet:** call `no_action_required(mode="wait")`
  and end your turn. Do not call `send_response` — a partial report is not
  the deliverable. Do not re-dispatch scans for repos that already reported.
- **All N have resolved** (reported results, or failed outright — see
  Phase 3): synthesize and post the consolidated report per Phase 4, then
  call `send_response(mode="report", result_ids=[])`.

Never call `send_response` more than once in this run. Never call
`no_action_required(mode="report")` after dispatching background work —
that signals "nothing to report" and would end the run with no scan
performed.

## Procedure

### Phase 1 — Discover the current repo family

There is no built-in tool for searching/listing an org's repos by name
pattern, so this needs a real `gh` CLI via an RWS worker:

1. `rws_pod_create` a small, short-lived workspace pod (1 CPU / 2Gi memory,
   ~15 minute TTL — this only runs one CLI command).
2. `rws_new_agent` on that pod: list every repository in `openshift-online`
   whose name matches `rosa-hyperfleet*`, with archived status.
3. `rws_query` the worker to run
   `gh search repos "rosa-hyperfleet" --owner openshift-online --json fullName,name,isArchived --limit 100`
   (cross-check with `gh repo list openshift-online --limit 300 --json name,isArchived`
   filtered client-side if the search looks incomplete).
4. `rws_pod_destroy` this discovery pod once you have the result — it's not
   needed for the scans themselves.
5. Drop archived repos. The remaining list is the `N` repos to scan this
   run — keep track of N so later turns can check completion against the
   right count.

### Phase 2 — Dispatch a parallel scan per repo

For **each** repo from Phase 1, in this same turn (do not wait for one to
finish before starting the next — they must run concurrently to fit the
scheduler's 4-hour background-work ceiling):

1. `rws_pod_create` a workspace pod sized for a full-repo scan (2 CPU / 4Gi
   memory, TTL comfortably longer than expected — 3 hours), with a distinct
   `logical_pod_name` per repo (e.g. `scan-<repo>`).
2. `rws_new_agent` on that pod with a system prompt establishing: clone
   `openshift-online/<repo>` at `main`, run the Adversary skill
   (`security@rosa-claude-plugins`, pre-installed via this persona's
   `rws.plugins`) in Groundwork mode (`/adversary groundwork`) against the
   full checkout, and report back severity counts (CRITICAL/HIGH/MEDIUM/LOW),
   overall risk level, the full list of CRITICAL/HIGH findings (title,
   file:line, one-line impact — needed for the cross-repo top-findings
   selection in Phase 4), and the complete findings report text in the
   skill's report-template format.
3. `rws_goal_task` on that pod with a condition verifiable from the
   worker's final message: "the repo is cloned, the Adversary Groundwork
   scan has run to completion, and the response includes severity counts,
   the full CRITICAL/HIGH findings list, and the complete report text."
4. Do not call `rws_pod_destroy` yet — destroy each pod only after its
   `rws_goal_task` has actually completed (destroying early kills the scan
   mid-run).

After dispatching all N, call `no_action_required(mode="wait")` and end the
turn.

### Phase 3 — Track completions across wake-ups

On each subsequent wake-up: identify which repo's scan just completed (from
the new completion message), record its results, `rws_pod_destroy` that
repo's pod now that it's done, and re-check the count per "State tracking"
above. If a pod dies or its goal task fails outright, record that repo as
**scan failed** (with whatever reason is available) rather than silently
dropping it from the final report — it still counts toward reaching N.

### Phase 4 — Synthesize the consolidated report (only once all N have resolved)

Build ONE Slack message with these sections, in this order:

**a. Executive summary.** For each of the three critical repos
(`rosa-hyperfleet`, `rosa-hyperfleet-api`, `rosa-hyperfleet-kube-applier`),
pick the single most service-impactful finding from its results — not
necessarily the highest-severity one, the one whose **Impact** description
most directly threatens the hyperfleet service itself (data exposure,
control-plane compromise, auth bypass, credential leakage, tenant isolation
failure). State it in one or two sentences framed as service risk ("an
attacker who can X would be able to Y, affecting Z"), not as a raw finding
restatement. If a critical repo has no CRITICAL/HIGH findings, say so
plainly rather than manufacturing a risk.

**b. Top aggregated findings.** 5-10 CRITICAL/HIGH findings total, pooled
across **all** repos (not 5-10 per repo) — the most severe/impactful across
the whole family. Each entry: severity, repo, title, file:line, one-line
impact. If fewer than 5 CRITICAL/HIGH findings exist across the entire
family, list what exists — don't pad with MEDIUM/LOW to hit a minimum.

**c. Overall posture summary + per-repo table.** A short paragraph on
aggregate risk to the hyperfleet service, then a table:

```
| Repo | Critical | High | Medium | Low | Status |
|------|----------|------|--------|-----|--------|
| rosa-hyperfleet | N | N | N | N | 🔴/🟡/🟢 |
```

Rows in alphabetical order by repo name. Status per repo: 🔴 if
Critical > 0 or High > 0, 🟡 if only Medium/Low > 0, 🟢 if all zero. A repo
whose scan failed (Phase 3) gets ⚠️ and "scan failed" instead of counts.

**d. Full per-repo results.** For each repo, alphabetical order, a clearly
labeled section header (e.g. `### rosa-hyperfleet-api`) followed by that
repo's complete findings report text from Phase 2. Note once, up front (not
per-repo), that Slack will auto-thread this if it's long — there is no
per-repo permalink, just scroll/search within the thread.

### Phase 5 — Deliver

Call `send_response(mode="report", result_ids=[])` with the assembled
message as the response text. This is the only Slack delivery for this
run — do not attempt separate posts per repo.

## Rules

- This is a read-only review: do not modify files, open PRs, or file Jiras.
- Do not infer a repo's identity from anything other than what Phase 1
  actually discovered — no hardcoded repo list, since the point of Phase 1
  is to catch repos added or removed since the last run.
- Do not include CVE/dependency-vulnerability findings — out of scope for
  the Adversary skill; flag only what its static/adversarial analysis covers.
- Order findings CRITICAL-first within every section, consistent with the
  skill's own severity ordering.
- Do not include the `[Scheduled task: ...]` metadata line in the output.
