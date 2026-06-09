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

## Block-test result (2026-06-09): GitHub runners are BLOCKED ❌

The first dispatch returned **HTTP 403 / 602-byte JSON bot-block page** for both files —
Akamai blocks GitHub's datacenter IPs the same way it blocks the Pi. **The GitHub Actions path
does not work.** The daily schedule is disabled (manual dispatch kept to re-test if CME ever
changes policy).

### Fallback: residential runner
The `curl` commands in `.github/workflows/mirror.yml` are runner-agnostic. To make this work,
run them from a **residential IP** (a laptop on a home/office network, or a residential proxy),
then `git push` the `data/` files to this repo — the Pi reads them via raw.githubusercontent
exactly as designed. First confirm your residential IP isn't blocked too: open
`https://www.cmegroup.com/delivery_reports/deliverable-commodities-under-registration.xls`
in a browser — if it downloads a real `.xls`, the laptop path will work; if it 403s, your whole
network IP is flagged and you'll need a residential proxy or a different network.

## How the Pi consumes it

`ag_movement_monitor/amm/fetchers/cme_delivery.py` reads `cme_mirror_url` (if set), downloads
`dcur.xls` from this repo, parses it locally, and feeds the Illinois→CBOT CME-certs panel.
Direct cmegroup.com stays as a fallback for if/when the Pi is ever unblocked.
