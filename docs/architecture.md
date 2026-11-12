# Architecture note - Git Repository Statistics

## The shape

One Go service (`cmd/app`) with the project logic in packages under `internal/`.
A request to analyse a repository moves like this:

    POST /repositories {path, merge_policy}
      -> handler validates the body, assigns id N, stores state "analysing", returns 202
      -> a goroutine starts ONE `git log --numstat` process on the path
      -> the parser reads its output as a stream, one commit at a time
      -> each commit becomes a compact record kept in memory under id N
      -> state becomes "done", `commits` and `analysed_in_ms` are filled in
    GET /repositories/N/contributors | activity | churn
      -> apply identity rules and the from/to/path filters to the stored records
      -> aggregate, and return JSON that names the merge policy used

Planned packages: `internal/gitlog` (run git, parse its output), `internal/store`
(repositories and their records, guarded by a mutex), `internal/report`
(contributors, activity, churn, identity folding), `internal/api` (handlers and
the error shape). `internal/version` comes from the template.

## The data

Per analysed repository I keep a list of commit records: hash, author name,
author email, timestamp, whether it is a merge, and per file the lines added and
removed. Everything lives in memory; **nothing survives a restart**, and
`POST /reset` simply empties the store. Identity rules (`PUT /identities`) are a
single global setting, also in memory. Reports are computed from the records on
each request.

## Three decisions

1. **Shell out to `git` instead of using a Go Git library.** One streamed
   `git log` process is far cheaper than one process per commit, and the
   output is stable and documented. I rejected go-git: it is a large
   dependency and I expect it to be slower on a 10,000-commit history. Cost:
   the runtime image must contain the `git` binary, so I added it to the final
   Docker stage with `apk add`.
2. **The merge policy is fixed when the analysis starts, per repository id.**
   `count` keeps merge commits and diffs them against their first parent, so
   a merge is credited with everything it brought in; `ignore` drops merges, so
   the work is credited to the people who wrote the original commits. I rejected
   computing both and choosing at query time: the API takes the policy in the
   POST body, and two separate analyses make the manual comparison test direct.
3. **Store per-commit records and aggregate at query time, not at ingest.**
   The `from`, `to` and `path` filters and the identity rules can change after
   the analysis, so pre-aggregated totals would be wrong or would force a
   re-run. I rejected aggregating while parsing: it is cheaper in memory but
   cannot answer a filtered question. Cost: memory grows with commits times
   files touched; I have not measured it yet.

## What is not built yet

Almost everything specific to the project. Done: the template's four contract
routes (with handler tests), `--version`, and CI. Not done:

- the git parser and any of the `/repositories` routes;
- both merge policies, and the stated policy in contributor responses;
- activity by day/week/month, and churn;
- identity folding and `PUT /identities`, including how I will decide that two
  addresses are one person;
- the `from`, `to` and `path` filters;
- validation, the 400/404/409/422 error cases, and tests for the project logic;
- a run on a repository of at least 10,000 commits, and its timing. I have not
  measured where the time goes, so decision 1 is currently an expectation, not
  a result.
