# hotels-cli

Agent-native CLI for Hotels.com Taiwan (`tw.hotels.com`, zh-TW, TWD). Zero dependencies (Python 3.8+ stdlib).
Search, hotel rooms/rates, price checks, login, trips, and favourites. **No booking, no checkout, no payment.**

Every command takes `--json` and returns `{ok, data|error}`. Exit codes: 0 ok, 1 api, 2 usage, 3 auth, 4 net/blocked.

## Transport

Hotels.com talks to one GraphQL endpoint (`POST https://tw.hotels.com/graphql`, Apollo persisted queries).
It sits behind DataDome. Measured 2026-10-30: Python's own HTTPS gets a 429 captcha page ("身分驗證") on the
very first request, whatever the cookies; this is fingerprinting, not a rate limit. With
[curl_cffi](https://github.com/lexiforest/curl_cffi) installed, the CLI impersonates Chrome's TLS fingerprint and
everything except `trips` / `saved` works over plain HTTPS without a browser, within a small request budget
(see Limitations):

```bash
pip install curl_cffi            # or: sudo pacman -S python-curl_cffi (Arch) - for the python3 that runs `hotels`
hotels doctor                    # "http ok ... via curl_cffi (Chrome TLS)"
```

It never sends the browser's `datadome` cookie: that cookie is bound to the browser's fingerprint, and sending it
from curl_cffi brings the captcha back (measured; it can also get that cookie flagged in the browser, after which
the browser has to pass the captcha once on hotels.com).

Without curl_cffi, and always for `trips` / `saved` (they read and click the live page), calls run inside a
background tab of your own signed-in Edge on CDP (`--cdp`, default `http://127.0.0.1:9222`). Tabs are created with
`Target.createTarget(background=true)`, never focused, and closed afterwards. Edge must already be running with
remote debugging on; the CLI never launches a browser. Run one command at a time (several parallel background
tabs made Edge close itself during testing).

## Login

Hotels.com signs in with a password (+ verification code), so the CLI never takes credentials. It logs in with
the `Cookie` request header of a browser session you already have:

1. In your browser: DevTools > Network > click any request to `tw.hotels.com` > Request Headers.
2. Copy the value of `Cookie:` (with or without the leading `Cookie: `).
3. Run `hotels login`. It reads the clipboard, or stdin when piped.

```bash
hotels login                    # Cookie header from stdin (piped) or the clipboard;
                                #   nothing usable there = exit 2 with these steps
hotels login --from-clipboard   # clipboard only
hotels login --cookie-stdin     # paste (hidden) or pipe it
hotels login --keyring          # store it in the OS keyring instead of the file
hotels login --from-browser     # opt-in: copy the session from Edge on CDP 9222; if signed out, opens
                                #   the sign-in page in a background tab (--wait 300)
hotels whoami | logout
```

The header must contain `EG_SESSIONTOKEN`. It is saved to `~/.config/hotels/state.json` (0600, `HOTELS_HOME`
moves it) or the OS keyring with `--keyring`; `HOTELS_COOKIE` overrides both. Before saving, the live homepage
must show the member name. With curl_cffi that check, `whoami` and all reads use the pasted session itself.
Without curl_cffi they run in an Edge tab and so see Edge's own session instead. Exit 3 means there is no valid
session (or it expired): copy a fresh Cookie header and log in again.

**Edge on CDP 9222 is needed** only without curl_cffi, for `trips` / `saved`, and for `login --from-browser`.

## Commands

```bash
hotels doctor
hotels suggest 心齋橋
hotels search 心齋橋 --checkin 2026-12-04 --checkout 2026-12-07 --adults 3 --children 7 \
      [--rooms 1] [--sort price|price-desc|recommended] [--page 1] [--limit 20] --json
hotels hotel <id|url> --checkin ... --checkout ... --adults 3 --children 7     # alias: rooms
hotels price <id> <id> ... --checkin ... --checkout ... --adults 3 --children 7
hotels login | whoami | logout
hotels trips                       # read-only: booked / saved tabs of /trips
hotels saved list
hotels saved add <id> [--yes]      # dry run without --yes
hotels saved rm  <id> [--yes]
hotels raw /robots.txt             # GET only
```

`--children` takes ages (`7,4`). A destination is free text (resolved through the typeahead, matched region is
printed) or a numeric region id.

## Prices: read this

- `price_per_night` is Hotels.com's lead (cheapest bookable unit) nightly rate; `price_total` is its own stay total
  (`總價`) for the same unit. Total is not simply rate x nights.
- The API does not say whether taxes/fees are included. Example: a 1-night Taipei stay showed NT$4,351/night vs
  NT$5,025 total. Compare hotels by `price_total`.
- Budget rows can be a shared-dorm bed rate (e.g. id 134096514 at NT$304/night). Check with `hotels hotel <id>`.
- Search, `hotel` and `price` agree for the same dates and party (id 59441016: 1,764 / 7,183 in search and hotel;
  `price` gives 1,764; id 134457477: 1,827 / 8,872 in both).
- Search drops sponsored cards (only `LodgingCard` rows are kept) and dedupes by id, so price sorts are monotonic.

## Verified live (2026-10-30)

- `search` 心齋橋 (3 nights), 3 adults + 1 child age 7: `--sort price` pages 1/2 ascending, 40 rows, no null
  prices, 0 overlap; `--sort price-desc` descending, no nulls. Without the child: 368 matches, with the child: 290.
- `hotel`: rooms with beds, occupancy, rate, total, cancellation ("可全額退款 (11 月 4 日 (三) 前)"), rooms left,
  check-in/out times.
- `price` for 3 hotels; `suggest 心齋橋` (region 6223824); `raw /robots.txt`; `doctor`; `whoami`; `trips`.
- `saved add/rm/list`: add created an auto trip "大阪" holding the hotel; rm removed it and the saved list went
  back to empty (`尚無儲存項目`).
- Sanity search 台北 2026-11-20..21, 2 adults: 385 matches.

## Limitations

- Without curl_cffi everything needs Edge on CDP; with it, only `trips` / `saved` do.
- **curl_cffi has a budget.** Measured 2026-10-30: about 40 requests in 15 minutes passed, then every request
  carrying the signed-in cookies got 403 / the 429 captcha page, while an anonymous homepage request still got
  200. Treat it like a small allowance: one command at a time, no loops, and stop on exit 4 instead of retrying.
- Not verified live: `--stars`, `--free-cancel`, `--breakfast`, `--min/--max`, `--rooms 2`, `--sort rating|distance|stars`,
  currencies other than TWD, `login --from-browser` from a signed-out browser, the 已過去 (past) trips tab.
- `trips`: this account has no bookings, so only the empty state was seen; a non-empty list returns raw page text.
- `saved list` only reports empty vs non-empty (raw text). `saved add` puts the hotel in an auto-created trip named
  after the destination; `rm` removes it from that trip.
- `hotel` shows the distance-to-landmark text as the area; the API gives no street address or star class.
- `search` `free_cancellation` was null in every row observed; use `hotel` for cancellation.
- Persisted-query hashes can rotate when Hotels.com ships a new build.
