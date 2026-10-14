# Forgejo pull request watch and merge verification

Empirical record for the merge watch and the merge path on Forgejo, alongside the existing GitHub and GitLab ones (`docs/gitlab-merge-watch.md`).
Every finding here was verified against a real, disposable Forgejo instance rather than assumed from `tea --help`, because several of them contradict what the help text alone would suggest.
All evidence below was collected on 2026-09-13.
A project opts in with the `forge=forgejo` registry binding, which `bin/fm-project-mode.sh` owns; the watch and merge paths recorded here key off the pull request URL itself, and the binding decides only that a generated ship brief names `tea` rather than `gh-axi`.

## Versions

```
$ tea --version
Version: development	golang: 1.26.4
```

This is a locally built development binary; its exact source revision could not be determined from the binary (no embedded VCS stamp).

```
$ jq --version
jq-1.7

$ bash --version | head -1
GNU bash, version 5.2.21(1)-release (x86_64-pc-linux-gnu)

$ docker --version
Docker version 29.8.0, build 88096ef
```

## The evidence instance

Unlike the GitLab doc's public fixture project, a self-hosted Forgejo instance is not something a reader can be pointed at over the network: that is the entire reason firstmate needs to support it.
The evidence here instead comes from a disposable, throwaway Forgejo 7 instance (`codeberg.org/forgejo/forgejo:7`) run locally in Docker for exactly this verification and destroyed afterward, with one repository (`fixture/fixture-repo`) and several pull requests created through the same `tea` CLI a real task would use.
Nothing here depends on any private or production instance, and none of it is reproducible by a reader without standing up an equivalent throwaway instance themselves.

## Why tea needed real verification before writing any code

`tea --help` documents `--fields`/`-f` and `--output`/`-o` on `tea pulls`, which looks at first glance like `glab mr view <n> -R <url> -F json`.
It is not:

```
$ tea pulls 1 --login fm-fixture --repo fixture/fixture-repo -f state -o simple
  # #1 Add the merged example file (merged)
  @fixture created 2026-09-13 22:08	**main** <- **merged-branch**
  merges
  --------
  http://localhost:3980/fixture/fixture-repo/pulls/1
```

`tea pulls <index>` always renders the same free-text detail view regardless of `-f`/`-o` - the help text's "will show it in detail" is the whole story, and field/output selection is silently ignored.
Only the list form honors them:

```
$ tea pulls list --login fm-fixture --repo fixture/fixture-repo --state all -f index,state -o csv
index,state
2,open
1,merged
```

A list read is bounded by `--limit`, so a watched pull request older than that window would never be seen merged.
The merge watch (`bin/fm-pr-poll.sh`), the terminal-run PR detail (`bin/fm-crew-state.sh`), and teardown's merged-PR proof (`bin/fm-teardown.sh`) therefore read the one pull request by number through `tea api ... /repos/{owner}/{repo}/pulls/{n}`, the same REST record the merge path below reads, and accept only a record whose `merged` field is a boolean (`fm_pr_forgejo_read_record` in `bin/fm-pr-lib.sh`).
**Not verified here:** that by-number read was not re-run against the evidence instance; it relies on the same `tea api` pull request record whose `state`, `merged`, and `head.sha` fields the merge path's read-back below did verify.

## Why the login is resolved fresh, and how

`tea pulls`/`tea api` address a repository by `owner/repo` slug only.
The host comes from a named `tea login` (`tea login add --name <name> --url <host> --token <token>`), selected by `--login <name>`; there is no flag that takes a bare host or full URL the way `glab -R <url>` does:

```
$ tea pulls ls --repo https://example.com/owner/repo -o simple
Error: path segment [1] is empty
```

Three logins were registered on the verification machine, two real ones on a private instance plus the throwaway fixture; those two are on a private network and their identifiers stay out of this record, the same way `docs/gitlab-merge-watch.md` withholds identifiers from its private-instance runs:

```
$ tea login list --output json
[
  {
    "name": "forgejo",
    "url": "<redacted, private instance>",
    "ssh_host": "<redacted, private instance>",
    "user": "",
    "default": "true"
  },
  {
    "name": "forgejo-admin",
    "url": "<redacted, private instance>",
    "ssh_host": "<redacted, private instance>",
    "user": "",
    "default": "false"
  },
  {
    "name": "fm-fixture",
    "url": "http://localhost:3980",
    "ssh_host": "localhost",
    "user": "",
    "default": "false"
  }
]
```

Every Forgejo read parses this list as JSON with `jq`, matches the validated PR host against each login's bare hostname (stripping scheme, userinfo, port, and path), and refuses when zero or more than one login matches, rather than guessing.
It never splits the output line by line, because a login printed as one compact object would then read its name where the url belongs.
`fm_pr_forgejo_login` in `bin/fm-pr-lib.sh` owns that match for every sourcing script; `bin/fm-pr-poll.sh` is a static, standalone watcher body with no sourced dependency, so it keeps its own copy and re-derives the match on every poll instead of trusting a name recorded anywhere durable.

## URL shape

Forgejo (and Gitea) serve pull requests at `<scheme>://<host>[:<port>]/<owner>/<repo>/pulls/<number>` - plural "pulls", no `/-/` route separator, confirmed by the fixture's own PR-creation output:

```
$ tea pulls create --login fm-fixture --repo fixture/fixture-repo --head merged-branch --base main --title "..."
  ...
  http://localhost:3980/fixture/fixture-repo/pulls/1
```

This never collides with GitHub's `.../pull/<n>` (singular), GitLab's `.../-/merge_requests/<n>`, or Gerrit's `.../c/<project>/+/<n>`.

The scheme and port come from the instance's own `ROOT_URL`, and every reporter repeats that URL verbatim: the instance, `tea`, and no-mistakes all printed `http://localhost:3980/fixture/fixture/pulls/<n>` for a Forgejo 16.0.6 instance with `ROOT_URL=http://localhost:3980/` (2026-10-13 trial below).
Firstmate therefore accepts a Forgejo URL with `http` or `https` and an optional explicit port, keeps it exactly as printed as the pull request's identity, and resolves the `tea` login from the bare hostname; GitHub, GitLab, and Gerrit URLs stay `https`-only with no port.
`fm_pr_url_parse` in `bin/fm-pr-lib.sh` owns the accepted shapes, and `tests/fm-pr-check-security.test.sh` pins them, including the refused non-canonical ports.
Before this, Firstmate refused the printed URL outright (`error: invalid PR check request`), so a local instance off port 443 or without TLS could not be armed, watched, or merged.

## End to end: arming and polling a real pull request

```
$ fm-pr-check.sh task-f1 https://localhost/fixture/fixture-repo/pulls/4
armed: state/task-f1.check.sh

$ cat state/task-f1.pr-poll
forgejo
https://localhost/fixture/fixture-repo/pulls/4
localhost
fixture/fixture-repo
4

$ state/task-f1.check.sh
(nothing - PR 4 is still open)
```

PR 4 was then merged directly through the head-bound API call described below, and the same published check was run again with no other change:

```
$ state/task-f1.check.sh
merged
```

This 2026-09-13 run addressed the instance as `https://localhost/...` with no port; login matching only ever compares bare hostnames, so the `localhost:3980` login still served it.
The URL the instance actually prints, `http://localhost:3980/...`, is accepted as-is since the URL-shape change above.

## A missing tea produces no wake, never a false merge

```
$ PATH="$notea" bin/fm-pr-poll.sh --validated forgejo https://localhost/fixture/fixture-repo/pulls/1 localhost fixture/fixture-repo 1
(nothing)

$ PATH="$notea" bin/fm-pr-check.sh task-x https://localhost/fixture/fixture-repo/pulls/1
error: watching a Forgejo pull request requires tea on PATH
```

## Merging: tea's merge subcommand cannot bind a head, but the API can

`tea pulls merge --help` documents no `--sha`/head-commit flag, unlike `gh pr merge --match-head-commit` and `glab mr merge --sha`.
Forgejo/Gitea's underlying REST API does support one, as an undocumented-by-tea `head_commit_id` field on the merge request body (confirmed against the instance's own served Swagger definition, `MergePullRequestOption`), so `bin/fm-pr-merge.sh` calls that API directly through `tea api` instead of `tea pulls merge`:

```
$ tea api --login fm-fixture --repo fixture/fixture-repo -X POST '/repos/{owner}/{repo}/pulls/2/merge' \
    -f Do=merge -f head_commit_id=0000000000000000000000000000000000000000 -i
HTTP/1.1 409 Conflict
{"message":"head out of date", ...}

$ tea api --login fm-fixture --repo fixture/fixture-repo -X POST '/repos/{owner}/{repo}/pulls/2/merge' \
    -f Do=merge -f head_commit_id=<the real head sha> -i
HTTP/1.1 200 OK
```

A stale or wrong head is refused with `409 head out of date`; the exact current head succeeds and actually merges (`state` read back as `closed`, `merged: true`).
This exactly parities `--match-head-commit`/`--sha`: a push landing between the verifying read and the merge call is refused rather than merged un-reviewed.
The `Do` field is the style the caller names (`--merge`, `--rebase`, `--squash`, or `--method <style>`); when none is named it is the repository's own `default_merge_style`, read live from `/repos/{owner}/{repo}` through the same login, and the merge is refused when that value cannot be read.

**`tea api` reports an HTTP-level failure with exit status 0.** Both calls above exited 0; only the JSON body differed.
This was the single most consequential finding of this verification pass: a naive implementation gating success on `tea api`'s exit status would treat every rejected merge as a success.
`bin/fm-pr-merge.sh` therefore never trusts that exit status - it always re-reads the pull request afterward (`forgejo_confirm_merged`) and treats `merged == true` as the only evidence a merge landed.

## Pre-merge conditions

Read live from `tea api ... /repos/{owner}/{repo}/pulls/{n}` (`state`, `mergeable`, `draft`, `head.sha`) and `tea api ... /repos/{owner}/{repo}/commits/{sha}/status?limit=50&page={p}` (the combined commit status).
The combined-status endpoint computes its `state` from one page of the latest status per context, so every page is read until an empty one and the head is green only when each non-empty page reads `success`.
A positive `draft` reading refuses; an instance too old to report the field still refuses a work-in-progress merge server side.
`tea api` passes a query string through unchanged, because it builds the request URL by plain concatenation onto the login's `/api/v1` base.
An already-merged pull request is correctly refused for a second merge attempt:

```
$ fm-pr-merge.sh task-f1 https://localhost/fixture/fixture-repo/pulls/4
error: refusing to merge https://localhost/fixture/fixture-repo/pulls/4
  - state is "closed", not open
  - the combined commit status is "none", not success
```

An open, mergeable pull request with no configured CI reports the combined status as empty with `total_count: 0`:

```
$ curl .../repos/fixture/fixture-repo/commits/<sha>/status
{"state":"","sha":"","total_count":0,"statuses":null,...}
```

This is read as `none`, the same way GitLab's null `head_pipeline` is read as `none` in `docs/gitlab-merge-watch.md`, and refuses the merge rather than treating an absent check as nothing to verify:

```
$ fm-pr-merge.sh task-f2 https://localhost/fixture/fixture-repo/pulls/<open-mergeable-pr>
error: refusing to merge ...
  - the combined commit status is "none", not success
```

The merge gate requires the combined status's `state` field to read exactly `success` on every non-empty page, matching GitHub's and GitLab's own state enums, and refuses on anything else including an unrecognized value.
The next section records that gate against real Forgejo Actions results.

## Real Forgejo Actions results

Verified 2026-10-13/14 against a disposable Forgejo `16.0.6` (`/api/v1/version` -> `16.0.6+gitea-1.22.0`) with Actions enabled, `forgejo/runner` `13.2.0`, and `tea` `0.16.0`, using real pull requests and real Actions runs; the instance was destroyed afterward.
These requests used the `https://localhost/...` spelling because Firstmate did not yet accept the printed `http://localhost:3980/...` URL; the login is matched by bare hostname, so all traffic reached the real instance.

A populated combined status (`tea api '/repos/{owner}/{repo}/commits/<sha>/status?limit=50&page=1'`) for a green run:

```json
{"state":"success","sha":"6cfc8048...","total_count":1,
 "statuses":[{"id":3,"status":"success","target_url":"/fixture/fixture/actions/runs/2/jobs/0",
   "description":"Successful in 21s","context":"CI / build-test (push)","creator":null, ...}],
 "commit_url":"","url":""}
```

- Each status carries its outcome in `status`, not GitHub's `state`; the gate reads only the top-level `state` and each page's array length.
- Contexts are named `<workflow name> / <job> (<event>)`, and `target_url` is relative.
- Per-status values seen were `success`, `failure`, `pending` while running, and `skipped` for an `if:`-false job.
- The combined `state` of `success` plus `skipped` is `success`, a running job gives `pending`, and a failing job gives `failure`.
- `total_count` counts that page only, the default page size is 30, and an empty page reads `{"state":"","sha":"","total_count":0,"statuses":null}`.
- The most recently updated contexts come first, the oldest land on the last page, and each page's `state` covers only that page.

A failure hidden behind 55 newer green contexts sits on page 2, so a single-page read would merge a red pull request:

```
?limit=50&page=1  {"state":"success","n":50}
?limit=50&page=2  {"state":"failure","n":8,"nonsuccess":[{"context":"aa-early-failure","status":"failure"}]}
?limit=50&page=3  {"state":"","n":0}
```

`bin/fm-pr-merge.sh` against pull requests 2 (red run), 3 (still running), 5 (`WIP:` draft), 8 (failure only on page 2), 1 (green), 4 (green with a skipped job), and 6 (57 contexts over two pages):

```
$ fm-pr-merge.sh task-red https://localhost/fixture/fixture/pulls/2
error: refusing to merge https://localhost/fixture/fixture/pulls/2
  - the combined commit status is "failure", not success
$ fm-pr-merge.sh task-slow https://localhost/fixture/fixture/pulls/3
error: refusing to merge https://localhost/fixture/fixture/pulls/3
  - the combined commit status is "pending", not success
$ fm-pr-merge.sh task-draft https://localhost/fixture/fixture/pulls/5
error: refusing to merge https://localhost/fixture/fixture/pulls/5
  - mergeable is "false", not true
  - the pull request is a draft
$ fm-pr-merge.sh task-page-late https://localhost/fixture/fixture/pulls/8
error: refusing to merge https://localhost/fixture/fixture/pulls/8
  - the combined commit status is "failure", not success
$ fm-pr-merge.sh task-green https://localhost/fixture/fixture/pulls/1
verified: https://localhost/fixture/fixture/pulls/1 is open and mergeable, with a successful combined status at head d974aae8...
$ fm-pr-merge.sh task-skip https://localhost/fixture/fixture/pulls/4 -- --squash
verified: ... successful combined status at head 0c1d4dd6...
$ fm-pr-merge.sh task-page-green https://localhost/fixture/fixture/pulls/6
verified: ... successful combined status at head 7e3c778e...
```

Pull request 1 read back as `{"number":1,"state":"closed","merged":true,...}`, and pull request 4 landed as one squash commit on `main`.

The pull request record on Forgejo 16 carries a boolean `draft`, and a `WIP:` title gives `draft: true` and `mergeable: false`.
A merge call on a pull request whose mergeability is not yet settled, or that conflicts, answers `405 {"message":"Please try again later"}` with `tea` exit status 0, which the read-back after every merge call already treats as not merged.
The by-number merge watch reported `merged` after the merge, the crew-state read and teardown's merged-pull-request proof read the same record correctly, and Forgejo served `refs/pull/<n>/head` for every pull request.

Arming a merge watch now also refuses a Forgejo draft, reading that same `draft` field, as it already did for a GitHub draft (`bin/fm-pr-check.sh`); the 2026-10-13 trial showed the earlier arming path accepting a draft (`armed: state/task-draft.check.sh`), while the merge-time refusal above still held.
`tea` `0.16.0`'s `tea pulls <n> --output json` now prints structured JSON including `headSha`, unlike the development build recorded above; Firstmate still reads the `tea api` record, which every release serves the same way.

## Registered `tea login`s used

`fm-fixture` (`http://localhost:3980`) was created solely for this verification and removed afterward, along with the fixture instance itself.
The two pre-existing real logins on the private instance (`forgejo`, `forgejo-admin`) were read only through `tea login list` to prove the host-matching logic against real data; no pull request on either was read, merged, or otherwise touched.
