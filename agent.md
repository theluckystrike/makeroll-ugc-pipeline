# MakeRoll UGC pipeline: setup and safe use, for AI agents

> One command, `./ugc run`, turns a written two-character script into a finished 9:16 animated
> reel: voice, lip-synced mouths, gestures, captions, a banner, a "Made with AI" chip and a 25 fps
> master. An automatic quality gate then scores the reel and passes it at 95/100. It runs on the
> user's own computer (macOS 13+ or Ubuntu 22.04+). The paid AI calls (voice and images) go
> through the user's own treg.to account, billed to their prepaid balance. The kit ships a starter
> show whose cast is already built, so its first real run pays only for the voice, about $0.07.

The kit is included with MakeRoll Lifetime. A member key (`mrk_` plus 40 letters and digits)
authorises the download. The human gets it from the MakeRoll bot on Telegram (`/pipeline`).

Human guide: https://makeroll.com/ugc-pipeline/
This page: https://makeroll.com/ugc-pipeline/agent.md

## 1. Set up (one line)

    curl -fsSL https://makeroll.com/ugc-pipeline/install.sh | sh -s -- mrk_THEKEY

The key can also come from the environment: `MAKEROLL_KEY=mrk_THEKEY`, and then
`curl -fsSL https://makeroll.com/ugc-pipeline/install.sh | sh`.

What it does: it fetches the file list, then downloads every file and checks each one's sha256.
It resumes partial downloads. It unpacks the kit into `~/makeroll-ugc-pipeline` (set
`MAKEROLL_DIR` for another folder) and saves the key to `~/.makeroll/key` with mode 600. It then
runs the kit's `./install.sh --yes`, which installs Python packages into `./.venv`, plus ffmpeg,
fonts and the treg CLI. That step prints every command first, and sudo asks for the password on
the terminal. Last, it installs this skill to `~/.claude/skills/makeroll-ugc/SKILL.md` and
`~/.agents/skills/makeroll-ugc/SKILL.md`. Running it again is safe: verified files are skipped.

Installer exit codes:

| Code | Meaning | What to do |
|---|---|---|
| 0 | Installed. The output may still list a `[todo]`, usually `treg login`. | Continue with section 2. |
| 1 | The computer lacks a tool, disk space, or a usable folder. | The message names it. Fix it and rerun. |
| 2 | There is no key, or it is not in the `mrk_...` format. | Ask the human for the key from the bot's `/pipeline` message. |
| 3 | The key was refused: 401 (unknown or revoked) or 403 (lifetime access not active). | Tell the human. Do not retry with guessed keys. |
| 4 | Rate limited (429): more than 40 file downloads with this key in 24 hours. | Wait for the time printed, then rerun. Finished files are kept. |
| 5 | A download failed, or a file did not match its sha256. Nothing was unpacked. | Rerun once. If it repeats, the human writes to support@makeroll.com. |

Without the one-liner: the bot also sends the core zip `makeroll-ugc-pipeline-v2.zip`. Unzip
it, `cd makeroll-ugc-pipeline`, then run `MAKEROLL_KEY=mrk_THEKEY ./install.sh`. The kit's
installer downloads the starter-show parts through the API (section 7) and verifies them.

## 2. The one human step: treg

The pipeline spends through the human's own treg.to account. Ask the human to run:

    treg login

This opens the browser. With no browser, use `treg login --email you@example.com`. Then check
three things. You can run these commands yourself:

    treg --version     # treg 0.22.0 or newer
    treg org ls        # one row marked * (the active team)
    treg balance       # credit left; new accounts usually get $1.00

If the login said `Active org: None`, the human creates a team. They can sign in at https://treg.to
and name it, or run `treg org create "My Studio"`. That command prints a team token, so do not run
it in a shared log. If `treg: command not found`, run `export PATH="$HOME/.local/bin:$PATH"`.
Topping up is the human's decision: `treg topup 10` is the minimum, and the human pays on a Stripe page.

## 3. Verify (free)

    cd ~/makeroll-ugc-pipeline && ./ugc doctor && ./ugc run --dry-run

- `./ugc doctor` must end with `verdict OK` and exit 0. A fresh kit lists its stage outputs as
  `SKIP not built yet`, which is expected.
- `./ugc run --dry-run` executes nothing and spends nothing. On the starter show, it must print
  `estimated spend this run   $0.0725  to  $0.0725` (voice only) and `verdict OK`. A dry run
  exits 0 even when preflight would refuse the real run, so read the verdict. `verdict FAIL`
  with `NOTE: preflight would REFUSE this run (exit 4)` means fix the FAIL rows first.

## 4. Run safely

1. Run the dry run (section 3).
2. **Show the human the estimated spend and wait for a yes.** A `$0.0000` estimate needs no approval.
3. `./ugc run`. It takes 3 to 8 minutes. Preflight runs again before anything paid is submitted.
4. Report the exit code, the `TOTAL nn.nn/100` line and the video path.

Hard rules:

- Never use `--force` (rebuilds paid stages: about $0.65 on the starter show) or `--allow-orphans`
  (skips the unfinished-paid-job check) without the human's explicit approval for that run.
- Never raise `budget.max_run_usd` in `production/project.json` (the starter cap is $1.00) or set
  `UGC_MAX_RUN_USD` without approval. Preflight refuses any run whose worst case is above the cap.
- Paid work is never retried automatically. Do not rerun in a loop after a failure. Fix the cause,
  then run `./ugc run --from <stage>`.
- `treg login`, `treg topup` and `treg org use` belong to the human.

Exit codes of `./ugc run` (and `./ugc doctor`):

| Code | Meaning | What to do |
|---|---|---|
| 0 | The video passed the 95/100 gate | Report the score and `final/one_prompt_master.mp4`. |
| 1 | A stage failed, or wrote an empty or broken output | Read the output above the error, fix it, then `./ugc run --from <stage>`. |
| 2 | The video scored below 95 | The score table names the weak dimension. The video is still in `final/`. See the README, Troubleshooting item 8. |
| 3 | No quality report was produced | `./ugc run --from score` (free). |
| 4 | Preflight refused to start. Nothing was spent. | Read the FAIL row. For `treg`, run `treg login`. For `balance`, the human tops up. For `orphans`, wait a few minutes and rerun. For `budget`, ask the human. For `inputs`, rerun `./install.sh`. |
| 5 | Another run holds the lock | Wait for it. If no `ugc` process is running, the next run reclaims the lock. |
| 6 | `doctor` found a broken environment or file | Fix the FAIL rows. `./install.sh` repairs missing packages. |

## 5. Outputs

| What | Where |
|---|---|
| The video | `final/one_prompt_master.mp4` (`paths.master` in `production/project.json`) |
| The score | `final/quality_report.json`; the run also prints the table |
| Frame contact sheets | `./ugc strips` writes them to `final/strips/` |
| Each run's score, estimate and true spend | `./ugc log` |
| Balance and spend per endpoint | `./ugc cost` |

## 6. A new episode or show

    ./ugc new my-episode           # a copy of the starter show at ~/ugc-shows/my-episode
    cd ~/ugc-shows/my-episode
    # edit production/beats.json: one {"beat", "speaker", "line"} per line (GREG or NADIA)
    python3 bin/plan_voice.py --write
    ./ugc run --dry-run            # show the estimate, get a yes
    ./ugc run

- With the same cast, only the voice is paid: about $0.07 to $0.10 per episode.
- `./ugc new my-show --blank` starts a new cast. It is a scaffold, not a runnable show: it
  prints a checklist of about 41 values (rig stills, mouth and eye boxes, voices, shots) that a
  person fills in, and preflight refuses the run (`project` FAIL, exit 4) until they are done.
  Its first episode then costs about $0.61.
- `--dir PATH` or `$UGC_SHOWS_DIR` puts shows somewhere else.

Costs per stage and the price table: https://makeroll.com/ugc-pipeline/ and `TREG_SETUP.md` in the
kit. treg prices were checked on 2026-10-01; the dry run always uses current ones.

## 7. API (member key required)

Send the key in either header: `Authorization: Bearer mrk_...` or `X-MakeRoll-Key: mrk_...`.

    MAKEROLL_KEY="$(cat ~/.makeroll/key)"

    # the file list: version, and per file its name, bytes, sha256, kind (core|starter) and url
    curl -fsS https://makeroll.com/api/pipeline/v1/manifest \
      -H "Authorization: Bearer $MAKEROLL_KEY"

    # one file (zip). The response carries Content-Length and X-Content-SHA256.
    curl -fSo makeroll-ugc-pipeline-v2.zip \
      https://makeroll.com/api/pipeline/v1/files/makeroll-ugc-pipeline-v2.zip \
      -H "Authorization: Bearer $MAKEROLL_KEY"

Errors come back as JSON `{"error": "...", "message": "..."}`:

| HTTP | Meaning |
|---|---|
| 401 | The key is unknown or was revoked |
| 403 | The key is valid, but lifetime access is not active |
| 404 | No such file |
| 429 | More than 40 file downloads with this key in a rolling 24 hours; see `Retry-After` |

Always compare each file's sha256 with the manifest before you unzip it. The installer does this
for you.

## 8. Key safety

- The key downloads a paid product. Never paste it into a public place: no issue, commit, log,
  screenshot, shared chat or pastebin. Keep it in `~/.makeroll/key` (mode 600) or in an
  environment variable, and pass it in headers only, never in a URL.
- If it leaks, the human sends `/pipeline newkey` to the MakeRoll bot. That revokes the old key at
  once and issues a new one. Then rerun the installer with the new key, or write it to `~/.makeroll/key`.
- The treg sign-in in `~/.treg/config.json` can spend the human's balance. Never print or copy it.

## 9. Troubleshooting

| You see | Do this |
|---|---|
| `treg` FAIL `not on PATH` or `not signed in` | `export PATH="$HOME/.local/bin:$PATH"`, then the human runs `treg login`. |
| The treg installer says it needs Python 3.12 or 3.13 | Rerun `./install.sh --yes`. It installs `uv` first, which fetches that Python privately. |
| `Active org: None`, or `no valid active team` | The human creates or picks a team (section 2). |
| FAIL `balance`, exit 4 | The balance is below the estimate x 1.15. The human tops up (`treg topup 10`). Nothing was spent. |
| FAIL `orphans`, exit 4 | An earlier paid job is still running at the vendor. Wait a few minutes and rerun. Check with `treg calls --limit 20`. |
| FAIL `inputs`, exit 4 | The starter pack is missing. Run `./install.sh` (with the key in `~/.makeroll/key`). |
| `py:numpy` / `py:scipy` / `py:PIL` not importable | Start the pipeline with `./ugc`, not `python3`. Rerun `./install.sh` if `.venv` is missing. |
| Linux: `render_text: no bold caption font found` | `sudo apt-get install -y fonts-liberation fonts-dejavu-core` (the human runs sudo). |
| Score below 95, exit 2 | The table names the weak dimension. See the README, Troubleshooting item 8. |
| Anything else | `README.md` in the kit lists every error found in testing. Write to support@makeroll.com for setup help. |
