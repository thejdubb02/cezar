# How we run Cezar (WSG)

This repo is our fork of open-mercato/cezar. Everything outside `wsg/` is the
upstream app. This folder describes how we run it. The scripts and systemd units
live in the devbox repo (`devbox/tools/cezar_*.py`, `devbox/deploy/cezar/`),
because that is where the rest of the fleet tooling lives.

## Where it runs

- Cezar runs on **cezar1** (194.163.45.146, tailnet 100.93.124.8), cockpit on
  `127.0.0.1:4321` there. The devbox reaches it through an ssh tunnel on its own
  `127.0.0.1:4321`. cezar.willhitestrategy.org is the web view.
- Claude builds, Codex reviews (`build-review`). When Claude's weekly quota is
  running ahead of pace, Codex builds and reviews alone (`codex-build-review`).

## The nightly backlog run (since 2026-11-11)

Justin's settings, decided 2026-11-11:

| Setting | Value | Where it lives |
|---|---|---|
| Work hours | 10pm to 7am Pacific, every day | `devbox/deploy/cezar/cezar-dispatch.timer` (hourly hand-offs 22:00 to 06:00) |
| Repo priority | Relay, Sentinel, Dealophant, Meridian | `devbox/deploy/cezar/backlog.json`, in list order |
| Daily cap | 6 new issues a night | `daily_cap` in `backlog.json` |

What happens each night, all on cezar1:

1. **21:50 Pacific, the picker** (`devbox/tools/cezar_pick.py`, timer
   `cezar-pick.timer`). For each repo in priority order it tops the repo up to
   its `per_day` count of `cezar:queued` issues, until 6 new labels are placed.
   Inside a repo it takes `priority:urgent`, then `priority:high`, then
   unlabelled, then `priority:low`, oldest first. It skips anything labelled
   `needs-justin`, bot alert issues (`gitty`, `stew`, `report`), questions, and
   anything already with Cezar. A repo that already has its `per_day` queued
   (Relay had 21 on 2026-11-11) gets nothing new until that drains.
2. **Every hour 22:00 to 06:00 Pacific, the dispatcher**
   (`devbox/tools/cezar_dispatch.py`, timer `cezar-dispatch.timer`). It hands
   queued issues to Cezar by swapping `cezar:queued` for `cezar:go` (or
   `cezar:go-codex` when Claude's quota is ahead of pace), up to 6 per hour, one
   run in flight per repo. Before a hand-off it runs the repo's own test gate on
   main; a red main gets a `cezar:base-red` issue first and the queue waits.
3. **Cezar builds.** Each repo has two automations watching for those labels.
   A run builds in a worktree, runs the repo's tests (`.ai/cezar/test-cmd`), and
   gets a Codex review. Nothing is pushed or deployed by the run.
4. **Landing happens in the repo's own window.** When Justin opens a repo's
   window, the session briefing says "Cezar finished N run(s) waiting to land"
   (`agent-infra/tools/hooks/session_brief.py`). That window lands them with the
   `cezar-session` skill and closes the issues. The only exception is
   `cezar:urgent` on Relay, which `cezar_lander.py` lands on its own.

## Changing it

- **Priority or per-repo amount:** edit `devbox/deploy/cezar/backlog.json`,
  commit, `git pull` in `/root/platform/devbox` on cezar1. Read every night, no
  restart needed.
- **Daily cap:** `daily_cap` in the same file.
- **Hours:** `OnCalendar` in `devbox/deploy/cezar/cezar-dispatch.timer` and
  `devbox/deploy/cezar/cezar1/cezar-pick.timer`, then copy to
  `/etc/systemd/system/` on cezar1 and `systemctl daemon-reload`.
- **Adding a repo:** it must be a Cezar project on cezar1 with a working
  `.ai/cezar/test-cmd` and the two label automations (copy them from meridian
  with `cez automation create`). Then add it to `backlog.json`, to the
  dispatcher's repo list in `devbox/deploy/cezar/cezar1/cezar-dispatch.service`,
  and to `REPOS` in `devbox/tools/cezar_check.py` (the health check that turns
  Gatus `cezar-devbox` red when a label has no automation).
- **Pausing everything:** `systemctl disable --now cezar-pick.timer
  cezar-dispatch.timer` on cezar1.

Preview tonight's picks without labelling anything:
`python3 /root/platform/devbox/tools/cezar_pick.py --dry-run` on cezar1.

## Quota protection

- Overnight only, so runs do not compete with daytime windows.
- 6 new issues a night at most, one run per repo at a time.
- The dispatcher reads the Claude usage meter and switches to Codex-only builds
  when the week is running ahead of pace.

## Not in the nightly run yet

- **Client sites** (`wsg-client-projects`, `site-easy-breezy`): not set up on
  cezar1. Client site issues often need facts only the client has, so they stay
  with their own windows for now.
- Mark's estate and personal repos are never picked.

## Sentinel on cezar1

Set up 2026-11-11: venv `/opt/venvs/sentinel` from `requirements.txt`, and
`test-cmd` runs the suite without a database (`--ignore=tests/test_waymark_api.py`,
which needs Postgres; CI covers it with a Postgres service).
