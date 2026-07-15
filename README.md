# Entihack (prototype)

A local defensive-security toolkit: honeypot capture + heuristic attacker
profiling + IP intel + a deception "maze" (tarpit) + a dashboard, with an
optional AI plain-language summary of each captured session.

## Easiest way to run this (no terminal typing required)

- **Mac:** double-click `Start_Entihack.command`.
  *(First time only: macOS blocks newly-downloaded scripts from running by
  double-click — if nothing happens, right-click the file, choose "Open,"
  then confirm "Open" in the popup. After that, double-clicking works normally.)*
- **Windows:** double-click `Start_Entihack.bat`.
  *(Requires Python installed from [python.org](https://python.org) first —
  a one-time install like any other program. If double-clicking does
  nothing, that's what's missing.)*
- **Linux:** double-click `Start_Entihack_Linux.sh` and choose "Run" or "Run
  in Terminal" if your file manager asks. If double-clicking just opens it
  as text instead, right-click it → Properties → Permissions → enable
  "Allow executing file as program." Always works as a fallback: open a
  terminal in this folder and run `bash Start_Entihack_Linux.sh`.
- **Chromebook (ChromeOS):** turn on Linux first — Settings → Advanced →
  Developer → "Linux development environment." Then copy this folder into
  the "Linux files" location in the Files app, open the Terminal app Linux
  setup added, `cd` into the folder, and run `bash Start_Entihack_Linux.sh`.

Either one installs what's needed, starts everything (honeypot, tarpit,
live API, dashboard), and opens the dashboard in your browser
automatically. To stop everything, double-click `Stop_Entihack.command`
(Mac), `Stop_Entihack.bat` (Windows), or `Stop_Entihack_Linux.sh`
(Linux/Chromebook).

**Setting your API key without a terminal:** the launcher creates
`api_key.txt` for you automatically from the template. Open it in any plain
text editor, type your key on its own line below the `#PUT YOUR KEY HERE`
comment, and save. **The key can be literally any text you want** — there's
no required format, length, or character set.

Everything below this section is the manual/terminal-based version of the
same steps.

## Running this on Google Cloud Platform (GCP) instead of your own machine

1. Create a free-tier-eligible VM: **Compute Engine → Create Instance** —
   an `e2-micro` instance qualifies for GCP's always-free tier in certain
   regions (verify current terms at
   [cloud.google.com/free](https://cloud.google.com/free)).
2. Open the firewall for the ports you need: **VPC network → Firewall →
   Create firewall rule** — allow TCP on 5050 (the API), and optionally
   2222/8080/8888 for the honeypot itself.
3. SSH into the instance (browser-based SSH button, no separate client
   needed), then:
   ```bash
   git clone <this repo>
   cd entihack
   bash Start_Entihack_Linux.sh
   ```
4. Use the VM's external IP as the `LIVE_URL` in `entihack_viewer.html`'s
   "Connect to a live server" tab — e.g. `http://<external-ip>:5050`.

Revisit the security notes further down before doing this for real —
running `api.py` reachable from the public internet makes the HTTPS and
rate-limiting gaps mentioned there actually matter, not just in theory.

## What this actually is

This is a **passive** capture and deception toolkit, not an offensive tool.
It listens, logs, and (in the tarpit) wastes an automated scanner's time
with fake pages. It never sends anything to an attacker's machine beyond a
generic fake banner/page, never executes anything on their system, and
never "hacks back." Actively attacking or sending commands to someone
else's machine, even someone attacking you, is unauthorized computer access
in essentially every jurisdiction (e.g. the CFAA in the US).

## Components

| File | What it does |
|---|---|
| `storage.py` | SQLite storage shared by everything |
| `honeypot.py` | Fake SSH banner + fake HTTP login page; logs every connection |
| `tarpit.py` | Flask app serving an endless fake directory maze, deliberately slow |
| `profiler.py` | Rule-based heuristic labeling — always a guess, never a verdict |
| `intel.py` | IP geolocation (ip-api.com) + WHOIS (ipwhois) lookups |
| `dashboard.py` | Web UI at http://localhost:5000 to browse captures + the blocklist |
| `ai_explainer.py` | Optional: calls the Anthropic API to summarize a session in plain English |
| `blocker.py` | Auto-blocks repeat offenders — refuses their connections, never contacts them |
| `export.py` | Exports every captured attack + the blocklist into a portable JSON/CSV report |
| `entihack_viewer.html` | Standalone app — file-upload mode, or live mode with a real Block button |
| `api.py` | The real backend for live mode: JSON API + authenticated block/unblock endpoints |
| `Start_Entihack.command` / `.bat` | Double-click launchers (Mac / Windows) |
| `Start_Entihack_Linux.sh` / `Stop_Entihack_Linux.sh` | Linux and ChromeOS launcher/stopper |
| `Stop_Entihack.command` / `.bat` | Double-click to stop everything |
| `api_key.txt.example` | Template for your API key |
| `abuseipdb_key.txt.example` | Template for your optional AbuseIPDB key |
| `rubiscout.py` | Standalone tool: analyzes a suspicious email via the real Rubiscout API |
| `rubiscout_key.txt.example` | Template for your optional Rubiscout key |

## Setup

```bash
cd entihack
pip install -r requirements.txt --break-system-packages
python3 storage.py            # creates data/entihack.db
```

## Running it

```bash
python3 honeypot.py --ssh-port 2222 --http-port 8080
python3 tarpit.py --port 8888
python3 dashboard.py          # then visit http://localhost:5000
```

Test it against yourself first:
```bash
curl http://localhost:8080/
curl http://localhost:8888/wp-admin
```

For the AI summary feature:
```bash
export ANTHROPIC_API_KEY=sk-ant-...
python3 ai_explainer.py 1
```

## Live mode — threat level, IP, and a real Block button

`entihack_viewer.html` has two tabs: **Load an export file** and **Connect
to a live server**, which talks directly to `api.py` over HTTP and updates
every 5 seconds.

```bash
export ENTIHACK_API_KEY="something-long-and-random"
python3 api.py --port 5050 --allow-origin "https://your-viewer-site.example"
```

Or save the key in a file instead:
```bash
cp api_key.txt.example api_key.txt
```
Replace the `#PUT YOUR KEY HERE` line with your actual key. Lines starting
with `#` are ignored. `.gitignore` already excludes the real `api_key.txt`
and `data/entihack.db` so you don't accidentally commit secrets or capture
data.

In the viewer, click **Connect to a live server**, enter your server's
address and port, paste the API key, and hit Connect. Live rows show a
computed **threat level** and a **Block now** button that calls
`POST /api/block` on your server — refusing that IP's connections from then
on, never contacting them.

This needs a real backend, not static hosting: blocking an IP means running
code with access to your firewall/app state, which a page on Netlify or
GitHub Pages can't do by itself.

**Security note:** the API key gates every block/unblock call — without
it, `POST /api/block` returns 401. Set `--allow-origin` to your actual site
rather than `*` once running somewhere real.

## Documenting and viewing attacks (export + viewer)

```bash
python3 export.py
```

Each entry documents one captured attack: source IP, timestamp, service,
path/target, credentials tried, the heuristic attack-type label, location
estimate, and block status.

Open `entihack_viewer.html` in any browser and drag the exported `.json`
file onto it — a self-contained local page, nothing uploaded anywhere.

## "Stop the hacker" (auto-blocking — multi-tier, rate-based)

Blocking is based on **speed of repeat attempts, checked against a table of
windows from 10 seconds up to 24 hours**. The table (`TIERS` at the top of
`blocker.py`) is checked shortest-window-first; an IP is blocked the moment
ANY tier trips:

| Window | Threshold | Implied rate | Catches |
|---|---|---|---|
| 10 seconds | 3 events | ~1080/hr | Obvious automated bursts |
| 30 seconds | 4 events | ~480/hr | Slightly throttled bots |
| 1 minute | 5 events | ~300/hr | |
| 2 minutes | 6 events | ~180/hr | |
| 5 minutes | 8 events | ~96/hr | |
| 1 hour | 12 events | ~12/hr | Persistent moderate-pace probing |
| 5 hours | 20 events | ~4/hr | |
| 12 hours | 25 events | ~2/hr | Deliberately-paced, slow evasion scanning |
| 24 hours | 40 events | ~1.7/hr | Very slow, multi-day-style evasion |

The threshold rises with window size, but the implied rate falls — a burst
is far more suspicious than the same count spread across hours. Edit the
`TIERS` list directly to add, remove, or retune any row.

Once blocked, Entihack refuses all further connections from that IP:

- **App-level (always on):** connection closed immediately, no banner, no
  fake page, no delay. No special privileges needed.
- **OS-level (optional, `--enable-os-block`, Linux + root):** also adds an
  `iptables -A INPUT -s <ip> -j DROP` rule.

This never sends anything to the source IP — it only changes how your own
server responds to their inbound traffic.

```bash
python3 blocker.py list
python3 blocker.py block 1.2.3.4 --os-level
python3 blocker.py unblock 1.2.3.4
```

## Worldwide report count (AbuseIPDB) vs. this installation's own totals

- **Per-IP worldwide reports (AbuseIPDB):** how many other systems around
  the world have reported a specific IP — real data from
  [AbuseIPDB](https://www.abuseipdb.com).
- **This installation's totals (`/api/stats`):** attacks/blocks *this
  specific install* has seen — local-only, labeled as such. There's no
  live, cross-installation "everyone running Entihack" number.

```bash
cp abuseipdb_key.txt.example abuseipdb_key.txt
```
Register a free account at abuseipdb.com, get your key, add it below the
placeholder comment. Without a key, this feature is skipped everywhere and
says so plainly.

## Legal notes (not legal advice — verify for your situation)

- Running this on infrastructure you own/control and logging incoming
  connections is standard, legal defensive practice in the US and most
  jurisdictions.
- If exposed to the public internet, you're logging real people's IP
  addresses, which may count as personal data under laws like GDPR — check
  applicable data-protection obligations before running this at scale.

## License

This project is licensed under the **Creative Commons Attribution-NonCommercial 4.0 International (CC BY-NC 4.0)** License.

### 🟢 You are free to:
* **Share** — Copy and redistribute the material in any medium or format.
* **Adapt** — Remix, transform, and build upon the material.

### 🔴 Under the following terms:
* **Attribution** — You must give appropriate credit, provide a link to the license, and indicate if changes were made.
* **Non-Commercial** — You may not use the material for commercial purposes.

See the full [LICENSE](LICENSE) file for details.

You should credit this like this:
"This project uses code from OshJ11872's EntiHack-Downloads-Check-The-README , which is licensed under CC BY-NC 4.0."
