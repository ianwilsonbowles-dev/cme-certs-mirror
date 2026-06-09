# cme-certs-mirror

A one-job GitHub Action that mirrors **CME Group's public grain delivery reports** into this
repo's `data/` folder, so a residential host that CME's Akamai WAF blocks (here: a Raspberry Pi)
can read them from `raw.githubusercontent.com` instead.

Mirrored daily (Mon–Fri 19:30 UTC) + on-demand:

| file | source | cadence |
|------|--------|---------|
| `data/dcur.xls` | [Deliverable Commodities Under Registration](https://www.cmegroup.com/delivery_reports/deliverable-commodities-under-registration.xls) | daily |
| `data/stocks-of-grain.xlsx` | [Weekly Stocks of Grain](https://www.cmegroup.com/delivery_reports/stocks-of-grain-updated-tuesday.xlsx) | weekly (Tue noon CT) |

DCUR = shipping certificates + warehouse receipts outstanding by zone — the certs leg of the
Illinois→CBOT delivery-economics drilldown in `ag_movement_monitor`.

## Setup (one time)

```bash
cd ~/Desktop/cme_certs_mirror
git init -b main
git add -A && git commit -m "CME delivery report mirror + Action"
gh repo create cme-certs-mirror --public --source=. --push     # or create on github.com and push
```

The repo must be **public** so `raw.githubusercontent.com` serves the files without auth (the
mirrored data is itself public CME data — no secrets here).

## The block-test (do this first)

Actions → **Mirror CME grain delivery reports** → **Run workflow**.

- **Green run that commits `data/dcur.xls`** → Akamai does *not* block GitHub runners; the
  workaround works. Note your raw base URL:
  `https://raw.githubusercontent.com/<you>/cme-certs-mirror/main/data`
  and drop it in `~/.claude/secrets/cme_mirror_url` on the Pi.
- **Red run with a `BLOCKED` error annotation** → Akamai blocks GitHub's datacenter range too;
  fall back to running the same fetch from a laptop/residential IP (the curl commands in
  `.github/workflows/mirror.yml` are runner-agnostic).

## How the Pi consumes it

`ag_movement_monitor/amm/fetchers/cme_delivery.py` reads `cme_mirror_url` (if set), downloads
`dcur.xls` from this repo, parses it locally, and feeds the Illinois→CBOT CME-certs panel.
Direct cmegroup.com stays as a fallback for if/when the Pi is ever unblocked.
