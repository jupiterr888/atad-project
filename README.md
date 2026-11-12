# Git Repository Statistics

A web service that reads a local Git repository and reports what happened in
it: who contributed and how much, how activity changed over time, and which
files keep being rewritten. It is aimed at someone who has inherited or joined a
project and wants a factual picture of its history without reading `git log`.

- **Student:** Novacescu Alexandra, group DMWD
- **Project:** 10 - Git Repository Statistics
- **Language:** Go

## What it does

You point the service at a checkout (`POST /repositories` with a `path`) and it
walks the history once, in the background. When it is done you can ask for
per-contributor totals, commits per day/week/month, and a ranking of the most
frequently changed files. Every contributor report states the **merge policy**
that produced it, because a merge commit can make one person appear to have
written a feature that a colleague wrote. Analysis only reads the repository
and never modifies it.

It does not clone remote repositories into anything permanent, does not look at
file contents beyond the added/removed line counts Git reports, and does not
try to judge anyone's work: the numbers describe commits and lines, nothing
more.

**State: early.** At this milestone the service answers the four common contract
routes (`/health`, `/version`, `/`, `POST /reset`) and the project routes are
designed but not yet implemented. See `docs/architecture.md` for what exists and
what does not.

## How to run it

```
docker run --rm -p 8080:8080 ghcr.io/jupiterr888/atad-project:latest
```

To build and run without Docker:

```
go run ./cmd/app
go run ./cmd/app --version
```

## What you should see

With the service running:

```
$ curl http://localhost:8080/health
{"status":"ok","uptime_seconds":3}

$ curl http://localhost:8080/version
{"app":"atad-project","version":"0.0.0-dev","commit":"<sha>","built_at":"<time>"}
```

`go run ./cmd/app --version` prints the same JSON on one line and exits 0.
`go test ./...` should pass.