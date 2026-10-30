---
name: hotels
description: Use when searching Hotels.com Taiwan (tw.hotels.com) hotels, checking TWD room rates and cancellation terms for given dates and party size (including children's ages), comparing prices of several hotels, or listing the user's saved favourites and trips (no booking).
---

# hotels - Hotels.com Taiwan from a shell (no booking)

`~/.local/bin/hotels` -> `~/Projects/hotels-cli/hotels`. Python 3.8+ stdlib. Every call goes through a background
tab of the user's signed-in Edge on CDP 127.0.0.1:9222 (plain HTTPS is 429-blocked). Never launch Edge; run ONE
command at a time. Start with `hotels doctor`.

- `hotels suggest <text> --json` -> region_id, name, area, type.
- `hotels search <dest|regionId> --checkin YYYY-MM-DD --checkout YYYY-MM-DD [--adults 2] [--children 7,4] [--rooms 1]
  [--sort price|price-desc|recommended] [--page N] [--limit N] --json` -> items[] with id, name, rating, reviews,
  price_per_night, price_total, url, sponsored. Children ages change the result set (measured 368 vs 290 matches).
  Sponsored cards are dropped; price sorts are monotonic with no null prices (measured).
- `hotels hotel <id|url> --checkin --checkout --adults --children [--rooms]` -> check-in/out times, amenities, units[]
  with rate_plans[] (price_per_night, price_total, cancellation, payment, rooms left). Empty units plus
  `unavailable_reason` means the party does not fit.
- `hotels price <id>... --checkin --checkout ...` -> lowest nightly price per hotel.
- `hotels trips` (read-only booked/saved tabs), `hotels saved list|add|rm <id> [--yes]` (dry run without --yes).
  `add` creates an auto trip named after the city; `rm` removes it again.
- `hotels login|whoami|logout`, `hotels raw <path>` (GET only).

Price rules: `price_total` is Hotels.com's own stay total and is the figure to compare hotels by. The API does not
say whether taxes/fees are included; the total is higher than rate x nights (1-night Taipei: 4,351 vs 5,025).
Very low nightly rates can be a shared-dorm bed: confirm with `hotels hotel <id>`.

Unverified: `--stars`, `--free-cancel`, `--breakfast`, `--min/--max`, `--rooms 2`, `--sort rating|distance|stars`,
non-TWD currency, signed-out login flow, past-trips tab, non-empty trips list. Exit 3 = run `hotels login`.
NEVER book, hold, or pay; this CLI has no such command.
