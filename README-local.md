# BBGO on `u-mac-10` — local runbook

This file is **specific to this host**, not upstream documentation. It records how
BBGO is installed here, how to rebuild it after an update, and the handful of
traps this machine has (a tiny `/tmp`, no `make`, no `sudo`, 2 GB of RAM).

Upstream docs: `README.md`, `CLAUDE.md`, `doc/`.

---

## 1. Where everything lives

| Path | What it is |
|---|---|
| `~/projects/bbgo` | Git checkout. Source of truth. Tracks `github.com/novaua/bbgo`. |
| `~/.local/bin/bbgo` | Installed binary (full build, dashboard embedded, ~114 MB). |
| `~/bbgo/` | **Runtime directory.** Everything the bot reads at run time. |
| `~/bbgo/bbgo.yaml` | Config: sessions + strategies. |
| `~/bbgo/.env.local` | API keys and DB settings. Mode `600`. |
| `~/bbgo/bbgo.sqlite3` | SQLite database (trades, orders, klines). |
| `~/.config/systemd/user/bbgo.service` | The unit. Runs with `WorkingDirectory=~/bbgo`. |
| `~/.local/node` | Node 22.14.0 (user-level; matches `.nvmrc`). |
| `~/.local/bin/yarn` | yarn 1.22.22, used only for building the dashboard. |
| `~/.bbgo_profile` | Build env to `source` before rebuilding (§3). Not in git. |
| `~/go/pkg/mod` | Go module cache (~1.7 GB). |
| `~/go/tmp` | Go build temp dir. **Mandatory** — see §3. |

`~/projects/bbgo` is the *source*; `~/bbgo` is the *running deployment*. They are
deliberately separate so `git pull` / `git clean` can never eat your config or
your database.

---

## 2. Day-to-day service commands

```bash
export PATH=$HOME/.local/bin:$PATH     # needed in every non-login shell

systemctl --user status bbgo           # is it up? how much RAM?
systemctl --user start bbgo
systemctl --user stop bbgo
systemctl --user restart bbgo          # after any config or binary change

journalctl --user -u bbgo -f           # follow logs
journalctl --user -u bbgo -n 100 --no-pager
```

It is a **user** unit, so `sudo` is never involved. It starts at boot because
lingering is on for this account — verify with:

```bash
loginctl show-user vgerman -p Linger   # want: Linger=yes
systemctl --user is-enabled bbgo       # want: enabled
```

If the box reboots and the bot does not come back, `Linger` is the first thing to
check. Re-enable with `loginctl enable-linger vgerman` (no sudo needed).

---

## 3. Rebuilding after an update

### The three environment quirks this machine forces

1. **`/tmp` is an 853 MB tmpfs** (half of RAM). Go puts its build work directory
   there by default, and a BBGO build needs several GB — you get
   `No space left on device` or a misleading `BFD assertion fail` from `as`.
   **Always** point Go's temp dir at the real disk:

   ```bash
   export GOTMPDIR=$HOME/go/tmp TMPDIR=$HOME/go/tmp
   mkdir -p $GOTMPDIR
   ```

   Do **not** try to fix this by enlarging the tmpfs. A tmpfs is RAM, not
   disk: `size=2G` on a 1.7 GB box does not add capacity, it just turns a clean
   "No space left on device" into swap thrash and OOM kills. Putting `/tmp` on
   the disk (77 GB free) is the sound long-term fix, but it is *not* done here —
   hence the env var instead.

2. **`make` is not installed**, so the `Makefile` targets (`make bbgo`,
   `make bbgo-slim`, `make static`) are unavailable. Use the raw commands below;
   they are exactly what those targets run.

3. **Installed Go is 1.26.8 but `go.mod` requires 1.27.3.** `GOTOOLCHAIN=auto`
   (the default here) silently downloads the newer toolchain on first build and
   caches it. That is expected — the first build after a Go bump needs network.

Also useful: keep parallelism at 2 so the build cannot blow past 2 GB of RAM
(`GOMAXPROCS=2 GOFLAGS=-p=2`).

### Source `~/.bbgo_profile` instead of typing all that

All of the above lives in one file kept **outside** the repo:

```bash
source ~/.bbgo_profile      # PATH, GOTMPDIR, TMPDIR, GOMAXPROCS, GOFLAGS
```

It is deliberately **not** wired into `~/.bashrc`. `TMPDIR` is inherited by every
program a shell launches, so redirecting it belongs to a build session, not to
your whole login environment. Sourcing it twice is safe — `PATH` is only
extended when the entry is missing.

Recreate it with this if it ever goes missing:

```bash
case ":$PATH:" in *":$HOME/.local/bin:"*) ;; *) export PATH="$HOME/.local/bin:$PATH" ;; esac
export GOTMPDIR="$HOME/go/tmp"
export TMPDIR="$HOME/go/tmp"
mkdir -p "$GOTMPDIR"
export GOMAXPROCS=2
export GOFLAGS=-p=2
```

Verify with `go env GOTMPDIR` — it must print `/home/vgerman/go/tmp`.

### A. Normal update (Go code only, dashboard unchanged)

This is the common case for a code-only update. It reuses the existing generated
`pkg/server/assets.go`, so it never needs Node. Budget several minutes: measured
here, even a single small `cmd` package took 5m52s wall against 31s CPU, so the
time goes to linking and I/O, not compilation.

```bash
source ~/.bbgo_profile

cd ~/projects/bbgo
git checkout main && git pull          # §9 — the checkout lives on main-local
git checkout main-local && git rebase main

go build -tags web,release -o build/bbgo/bbgo ./cmd/bbgo
install -m 755 build/bbgo/bbgo ~/.local/bin/bbgo

systemctl --user restart bbgo
bbgo version
```

`-tags web,release`: `web` embeds the dashboard, `release` stamps the version.

### B. Frontend changed (anything under `apps/frontend/`)

The dashboard is baked into the binary at build time, so a Go rebuild alone
**will not** pick up UI changes. Regenerate the assets first:

```bash
source ~/.bbgo_profile

cd ~/projects/bbgo

# stop first: the full build peaks around 1.4 GB on a 2 GB box
systemctl --user stop bbgo

cd apps/frontend
yarn install          # only needed if package.json / yarn.lock changed
yarn export           # next build + next export  →  apps/frontend/out
cd ~/projects/bbgo

go run ./utils/embed -package server -tag web \
    -output pkg/server/assets.go apps/frontend/out

go build -tags web,release -o build/bbgo/bbgo ./cmd/bbgo
install -m 755 build/bbgo/bbgo ~/.local/bin/bbgo

systemctl --user start bbgo
```

### C. If `pkg/server/assets.go` is missing

It is **gitignored** (`.gitignore:39`), so a fresh clone or a `git clean -xdf`
deletes it, and `go build -tags web` then fails with an undefined `FS`.
Recover with path **B** (or at minimum the `yarn export` + `go run ./utils/embed`
pair). Do not hand-write it.

The same is true of `apps/frontend/{out,node_modules,.next}` — all gitignored,
all disposable, all regenerable.

### D. Slim build (no dashboard)

Half the size and no Node needed, but **replaces** the installed binary and
removes the dashboard. Only do this deliberately:

```bash
go build -tags release -o build/bbgo/bbgo-slim ./cmd/bbgo
```

---

## 4. Exchange setup: preview vs real trading

The bot reads `~/bbgo/bbgo.yaml` and `~/bbgo/.env.local` from the service's
`WorkingDirectory`, which is why both live in `~/bbgo/`.

### Current state: preview mode

`bbgo.yaml` currently has `publicOnly: true` on the binance session. That means
**public market data only** — no balances, no orders, no trading. It exists so
the dashboard works without credentials.

### Verifying credentials before you start the service

This is the safe way to test keys — it queries the account once and exits:

```bash
export PATH=$HOME/.local/bin:$PATH
cd ~/bbgo
bbgo account --session binance
bbgo balances --session binance
```

A 401 / `API-key format invalid` means the keys in `.env.local` are still
placeholders or wrong.

### Switching to real trading

1. Put real keys in `~/bbgo/.env.local` (keep mode `600`):

   ```bash
   nano ~/bbgo/.env.local
   # BINANCE_API_KEY=...
   # BINANCE_API_SECRET=...
   chmod 600 ~/bbgo/.env.local
   ```

   Variable names must match the session's `envVarPrefix`, upper-cased:
   `envVarPrefix: binance` → `BINANCE_API_KEY` / `BINANCE_API_SECRET`.

2. **Remove the `publicOnly: true` line** from `~/bbgo/bbgo.yaml`. Leaving it in
   is the most common surprise: the dashboard comes up but balances and order
   history stay empty forever, because there is no user data stream.

3. Restart and confirm the session is healthy:

   ```bash
   systemctl --user restart bbgo
   journalctl --user -u bbgo -f     # want "querying account balances ... " with no 401
   ```

At this point the bot is connected and **idle** — it has no strategies yet, so it
cannot place an order. That is the right place to stand before adding one.

### Adding a strategy (this is where it starts trading for real)

```yaml
sessions:
  binance:
    exchange: binance
    envVarPrefix: binance

exchangeStrategies:
- on: binance          # must match a session name above
  grid:
    symbol: BTCUSDT
    quantity: 0.001
    gridNumber: 20
    profitSpread: 1000.0
    upperPrice: 30_000.0
    lowerPrice: 28_000.0
```

Copy parameters from a template under `~/projects/bbgo/config/` (e.g.
`grid.yaml`, `xmaker-btcusdt.yaml`) rather than inventing them — `grid2`,
`xmaker`, `bollmaker` and `supertrend` are the common starting points. The values
above are illustrative, not advice.

Rules that matter:

- `on:` must name a session defined in the same file.
- Once the keys are valid and a strategy is present, **it trades with real money
  on restart.** There is no dry-run.
- The bot does **not** reload `bbgo.yaml`; always `systemctl --user restart bbgo`.
- Remove the strategy block to go back to idle. You do not need to remove keys.

---

## 5. Dashboard

The unit runs:

```
ExecStart=%h/.local/bin/bbgo run --enable-webserver --webserver-bind 127.0.0.1:8080
```

`%h` is `/home/vgerman`. It is bound to **loopback on purpose**: `pkg/server` has
no authentication at all — no login, no token on the trading routes — so anyone
who can reach the port can see your account and place orders.

Reach it over the SSH tunnel:

```bash
ssh -N -L 8080:127.0.0.1:8080 vgerman@u-mac-10
# then browse http://127.0.0.1:8080
```

**The dashboard only appears after the exchange bootstrap succeeds.** If the keys
are bad, bbgo retries the account query during startup and never reaches the
point where the HTTP server binds — the tunnel then reports
`channel 2: open failed: connect failed: Connection refused`. That error means
"nothing listening", not "tunnel broken". Diagnose with:

```bash
ss -ltn | grep 8080
journalctl --user -u bbgo -n 20 --no-pager | grep -E "401|failed to query"
```

To expose it on the LAN instead (only on a trusted network):

```bash
sed -i 's|--webserver-bind 127.0.0.1:8080|--webserver-bind 0.0.0.0:8080|' \
  ~/.config/systemd/user/bbgo.service
systemctl --user daemon-reload && systemctl --user restart bbgo
```

---

## 6. Database and backups

SQLite, chosen because a MySQL server would not fit on this box. The DSN lives in
`.env.local`:

```
DB_DRIVER=sqlite3
DB_DSN=/home/vgerman/bbgo/bbgo.sqlite3
```

Schema migrations run automatically on startup, so a new binary may upgrade the
database the first time it runs. Back up before upgrading:

```bash
systemctl --user stop bbgo
cp ~/bbgo/bbgo.sqlite3 ~/bbgo/bbgo.sqlite3.$(date +%F)
systemctl --user start bbgo
```

Stop before copying — copying a live SQLite file can capture a torn page. Also
back up `bbgo.yaml` and `.env.local` (they are small and not in git).

---

## 7. Living with 2 GB of RAM

The unit caps the bot so a runaway strategy cannot take the machine down:

```
MemoryHigh=900M
MemoryMax=1400M
```

Check actual use:

```bash
systemctl --user show bbgo -p MemoryCurrent -p MemoryHigh -p MemoryMax
free -m
```

Typical: ~20 MB idle with the dashboard, a few hundred MB once strategies and
private streams are running. Swap is 8 GB, so brief spikes are survivable but
slow.

**Do not run a full build while the bot is trading** — a `next build` plus link
will push the machine into swap and can starve the bot. Stop it first (§3B).

---

## 8. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `No space left on device` or `as: BFD assertion fail` during build | Go wrote its work dir to the 853 MB `/tmp` tmpfs | `export GOTMPDIR=$HOME/go/tmp TMPDIR=$HOME/go/tmp` (§3) |
| `make: command not found` | `make` is not installed | Use the raw `go build` / `go run ./utils/embed` commands (§3) |
| `go.mod requires go >= 1.27.3` | host Go is 1.26.8 | Nothing — `GOTOOLCHAIN=auto` fetches it. Needs network. |
| `undefined: FS` when building with `-tags web` | `pkg/server/assets.go` missing (gitignored) | Regenerate via §3B |
| Tunnel says `Connection refused` | nothing listening on 8080 | Session bootstrap failed (bad keys) or service stopped — check `ss -ltn \| grep 8080` and the journal |
| Dashboard up but no balances | still `publicOnly: true` | Remove that line, restart (§4) |
| `401 ... API-key format invalid` | placeholder/wrong keys | Fix `.env.local`, verify with `bbgo account` |
| Service not running after reboot | lingering off | `loginctl enable-linger vgerman` |
| Config change had no effect | bbgo does not hot-reload | `systemctl --user restart bbgo` |
| `git pull` says "no tracking information" | checkout is on `main-local` | `git checkout main && git pull`, then rebase (§9) |
| `bind: address already in use` | stale bbgo or another process on 8080 | `ss -ltnp \| grep 8080`, `pkill -x bbgo` |

Logs are the first stop for anything runtime-related:
`journalctl --user -u bbgo -n 200 --no-pager` (add `-f` to follow).

Keep the journal from growing without bound:

```bash
journalctl --user --vacuum-size=200M
```

---

## 9. Updating from upstream

This checkout sits on the local branch **`main-local`**, which is `main` plus the
commits holding this runbook. `main` is the branch that tracks `origin/main`, so a
bare `git pull` while `main-local` is checked out fails with "no tracking
information". Pull explicitly, then replay the runbook on top:

```bash
cd ~/projects/bbgo
git status                       # should be clean; build outputs are all gitignored

git checkout main && git pull    # main tracks origin/main
git checkout main-local
git rebase main                  # replay the runbook commit on top of upstream

# then §3A, or §3B if apps/frontend changed
```

A rebase conflict can only occur in `README-local.md`; resolve it, then:

```bash
git add README-local.md && git rebase --continue
```

### The remote is SSH

`origin` is `git@github.com:novaua/bbgo.git`. It was switched from the HTTPS URL
the repo was originally cloned with, because there is no credential helper here:
over HTTPS a push would demand a personal access token, while
`~/.ssh/id_ed25519` already authenticates as `novaua`.

```bash
git remote -v                     # expect git@github.com:novaua/bbgo.git
git push -u origin main-local     # publish this branch
```

Cloning over HTTPS still works for *reads* — the repository is public — so only
pushing depends on the key.

### About this file

`README-local.md` exists only on `main-local`, so it never lands in `origin/main`.
It is committed, so it does appear in `git status` — that is intended. To drop it
back out of version control:

```bash
git rm --cached README-local.md
echo 'README-local.md' >> .git/info/exclude
```

### Reclaiming disk

Only if you need it — each of these costs a slow re-download:

```bash
du -sh ~/go/pkg/mod ~/projects/bbgo/apps/frontend/node_modules
go clean -modcache                                # ~1.7 GB
rm -rf ~/projects/bbgo/apps/frontend/node_modules # ~473 MB
```

---

## 10. Go-live checklist

```bash
export PATH=$HOME/.local/bin:$PATH
cd ~/bbgo

# 1. keys present and valid?
bbgo account --session binance

# 2. publicOnly removed from bbgo.yaml?
grep -n publicOnly bbgo.yaml        # want: no match

# 3. strategy added, on: matches a session?
grep -n -A2 exchangeStrategies bbgo.yaml

# 4. start and watch
systemctl --user restart bbgo
journalctl --user -u bbgo -f

# 5. confirm the dashboard responds
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8080/
```
