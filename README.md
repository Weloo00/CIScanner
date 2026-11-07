# 🛰️ CIScanner — Recon & Vulnerability Scanning on GitHub Actions

Automate reconnaissance and vulnerability scanning straight from GitHub Actions — no VPS, no local install.

Two workflows ship here:

| Workflow | What it does |
|---|---|
| **Subdomain Monitor** (`subdomain-monitor.yml`) | subfinder → **httpx** → diff against the last run → Discord report + committed state |
| **Vulnerability Scan** (`vulnscan.yml`) | subfinder → **httpx** (live hosts only) → nuclei → Discord + artifacts |

**Why httpx matters:** subfinder returns names, not servers. Half of them are dead, parked, or
wildcard DNS. Feeding those straight into nuclei wastes the whole run on DNS errors. httpx
verifies which hosts actually serve HTTP(S), records status codes, titles, tech stack, CDN and
IPs, and nuclei then scans **only the live ones**.

---

## 🚀 Quick start

1. **Enable Actions** on the repository (Actions tab → *I understand my workflows, go ahead*).
2. **Add the webhook secret** — Settings → Secrets and variables → Actions → *New repository secret*:

   ```
   Name:  DISCORD_WEBHOOK_URL
   Value: https://discord.com/api/webhooks/xxxx/yyyy
   ```

   The legacy name `DISCORD_WEBHOOK` is also accepted.

3. *(Optional)* set a default target so scheduled runs don't scan `example.com`:
   Settings → Secrets and variables → Actions → **Variables** → *New repository variable*:

   ```
   Name:  SCAN_DOMAIN
   Value: yourdomain.com
   ```

4. **Run it** — Actions → *Subdomain Monitor* → **Run workflow** → enter a domain.

---

## ⚙️ Workflow inputs

### Subdomain Monitor

| Input | Default | Meaning |
|---|---|---|
| `domain` | `example.com` | Target (falls back to the `SCAN_DOMAIN` variable) |
| `notify_when_unchanged` | `true` | Post even when nothing changed |

- Cron: `0 */6 * * *` (every 6 hours).
- State is stored in `state/<domain>.subdomains.txt` and `state/<domain>.live.txt`, committed back
  with `[skip ci]`, so the next run can diff.

### Vulnerability Scan

| Input | Default | Meaning |
|---|---|---|
| `domain` | `example.com` | Target |
| `severity` | `critical,high,medium` | nuclei severity filter |
| `rate_limit` | `5` | requests/second per host |
| `threads` | `5` | parallel requests |

- Cron: `0 6 * * 1` (Mondays 06:00 UTC).

---

## 🔒 Built-in rate limiting

Both workflows pass `-rl 5 -t 5` to httpx and `-rl 5 -c 5` to nuclei by default — 5 requests per
second and 5 parallel requests. Raise them only for hosts you own; many bug-bounty programs cap
automated traffic at exactly this level and treat breaches as a policy violation.

---

## 📦 Artifacts

Every run uploads its output (30-day retention):

**Subdomain Monitor**
- `subdomains.txt` — everything subfinder found
- `live-urls.txt` — httpx-verified live URLs
- `live.tsv` — status code, tech stack, URL
- `new-subdomains.txt`, `removed-subdomains.txt`, `new-live.txt` — the diff

**Vulnerability Scan**
- `subdomains.txt`, `live.txt`, `live-urls.txt`
- `nuclei.txt` — severity, name, host, matched URL
- `nuclei.jsonl` — full nuclei records

Each run also writes a Markdown summary to the Actions **Job Summary** page.

---

## 🧩 Tools

| Tool | Role |
|---|---|
| [subfinder](https://github.com/projectdiscovery/subfinder) | passive subdomain enumeration |
| [httpx](https://github.com/projectdiscovery/httpx) | live-host verification, status/tech/CDN/IP fingerprinting |
| [nuclei](https://github.com/projectdiscovery/nuclei) | template-based vulnerability scanning |
| [jq](https://jqlang.github.io/jq/) | safe JSON payload building for Discord |

---

## 🛠️ Repo layout

```
.github/workflows/subdomain-monitor.yml   # active
.github/workflows/vulnscan.yml            # active
state/<domain>.subdomains.txt             # last-seen subdomains (auto-committed)
state/<domain>.live.txt                   # last-seen live URLs   (auto-committed)
```

> Earlier versions of this repo shipped the workflows as `*.yml.txt`, which GitHub ignores.
> They are now real `.yml` files, so they actually run.

---

## ⚠️ Notes

- GitHub-hosted runners are shared IPs. Some targets will rate-limit or block them; that is
  expected and is not a finding.
- `nuclei` runs with `-disable-update-check` for speed. Templates still auto-install on first use.
- Keep `permissions:` minimal — the scan workflow only needs `contents: read`; only the monitor
  needs `contents: write` (to commit state).

---

## 🧠 Roadmap

- [x] HTTPx live-host verification
- [ ] CVE severity scoring / EPSS enrichment in Discord notifications
- [ ] Automatic PR generation for new high-severity findings
- [ ] Slack / Telegram notification backends

---

## 🏁 License

MIT — see [LICENSE](LICENSE).
