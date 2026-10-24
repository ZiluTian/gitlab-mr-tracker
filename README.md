# mr-ci

A small, dependency-free CLI that **live-tracks a GitLab MR's CI pipeline and
comments in your terminal** — and can reply to comments. Works with any GitLab
instance (self-hosted or gitlab.com). No credentials are stored: the access
token is read from the environment or a flag at run time.

![status](https://img.shields.io/badge/python-3.6%2B-blue) ![deps](https://img.shields.io/badge/deps-none-brightgreen)

## Features

- **Live CI dashboard** — auto-refreshes (default every 30s) and redraws in
  place: MR title/state, source→target, head SHA, pipeline #, every job with a
  color-coded status and duration, and a `x/y done` progress line. It follows
  the MR's *current* head, so a new push automatically tracks the new pipeline.
- **Comment awareness** — always shows the latest comment, and highlights
  comments that arrive while you watch (author, time, snippet, and a deep link
  to the exact note). Your own comments don't trigger alerts.
- **Reply to comments** — post a top-level comment or a threaded reply, with
  `@username` mentions. `--dry-run` previews the request.
- **Follows to completion** — when the pipeline reaches a final state it prints
  a result banner (with failed-job trace links) and exits with a meaningful
  code. `--watch` keeps it alive afterwards to keep monitoring comments.
- **Zero dependencies** — pure Python standard library. Handles self-signed /
  private instance certificates automatically.

## Install

```bash
# from the repo
chmod +x mr-ci
# optionally put it on your PATH
install -d ~/.local/bin && cp mr-ci ~/.local/bin/   # or: ln -s "$PWD/mr-ci" ~/.local/bin/
```

## Configure

Nothing to install, no config file, no stored secrets. Provide:

| what     | how                                                        |
|----------|------------------------------------------------------------|
| token    | `export GITLAB_TOKEN=glpat-...`   or   `--token`          |
| instance | `export GITLAB_INSTANCE=https://gitlab.example.com` (default `https://gitlab.com`) |
| project  | `export GITLAB_PROJECT=group/project` (or numeric id)     or   `--project` |

> Use a **personal access token** with at least the `read_api` scope
> (add `read_user` if you want the "new comments by others" filtering to
> recognize you). To enable `--reply`, also grant `api`.

## Usage

```bash
mr-ci 123                 # live-track MR !123 (uses GITLAB_PROJECT) until it finishes
mr-ci 123 --once          # single snapshot, then exit
mr-ci 123 --watch         # keep watching comments after CI finishes
mr-ci feature/branch      # track by source branch (newest open MR on that branch)
mr-ci 123 --interval 15   # faster refresh
mr-ci 123 --plain         # don't clear the screen (log/pipe friendly)
```

### Reply to comments

```bash
mr-ci 123 --reply "lgtm" --to latest        # thread a reply into the newest comment
mr-ci 123 --reply "thanks" --to 4567        # reply to a specific note id
mr-ci 123 --reply "@dev can you review?"    # top-level comment with a mention
mr-ci 123 --reply "msg" --to latest --dry-run   # preview without posting
```

Note ids are the `#note_<id>` anchors shown in the dashboard.

## Exit codes

| code | meaning                                   |
|------|-------------------------------------------|
| 0    | pipeline succeeded (or `--once`/`--help`) |
| 1    | pipeline failed / canceled                |
| 130  | interrupted (Ctrl-C)                      |
| 2    | usage / config error                      |

## How it works

- Resolves the MR (by IID, or the newest open MR for a branch), then the newest
  pipeline for the MR's current head SHA, then its jobs — each poll.
- Polls MR notes; new non-system notes authored by someone other than you are
  surfaced in a "NEW this session" section.
- Threaded replies are posted to `POST /merge_requests/:id/discussions/:did/notes`
  (the thread id is looked up via the discussions API); top-level comments use
  `POST /merge_requests/:id/notes`.

## Privacy

The only secret involved is your GitLab token, and it never touches disk — it's
read from `GITLAB_TOKEN` (or `--token`) when the command runs. There is no
config file, no keychain, nothing to commit.
