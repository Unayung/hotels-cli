# hotels-cli

Agent-native CLI for Hotels.com Taiwan (`tw.hotels.com`, zh-TW, TWD). Zero dependencies (Python 3.8+ stdlib).
Search, hotel rooms/rates, price checks, login, trips, and favourites. **No booking, no checkout, no payment.**

Every command takes `--json` and returns `{ok, data|error}`. Exit codes: 0 ok, 1 api, 2 usage, 3 auth, 4 net/blocked.

## Transport

Hotels.com talks to one GraphQL endpoint (`POST https://tw.hotels.com/graphql`, Apollo persisted queries).
Every call runs inside a background tab of your own signed-in Edge on CDP (`--cdp`, default
`http://127.0.0.1:9222`; setup below). Tabs are created with `Target.createTarget(background=true)`, never
focused, and closed afterwards. The CLI never launches a browser. Run one command at a time (several parallel
background tabs made Edge close itself during testing).

**Why a real browser** (measured 2026-10-30): Akamai Bot Manager (`_abck`, `bm_*` cookies) scores each client and
Expedia's `botOrNot` answers suspicious ones with a 429 "身分驗證" page carrying a DataDome captcha. A new client
gets an invalid `_abck`; only a browser that runs Akamai's sensor script gets it validated. Python's HTTPS is
challenged on the first request. [curl_cffi](https://github.com/lexiforest/curl_cffi) (Chrome TLS fingerprint, no
JS) passed about 40 requests, then this IP + that fingerprint got the captcha on every request, with or without
cookies, and it got worse, not better, after 30 idle minutes. Lightpanda (runs JS) passed one page, but Akamai's
script throws in it and `_abck` stays invalid. Edge on the same IP and session kept working throughout. It is a
bot classifier, not a rate limit, so the fix is a real browser, not waiting or rotating clients.

`--via http` still exists for a handful of plain-HTTPS requests (it uses curl_cffi when installed; `pip install
curl_cffi`). Expect it to be cut off after a short grace, and never send the browser's `datadome` / `_abck`
cookies from it: that got Edge challenged too until it passed the check again.

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
must show the member name. That check, `whoami` and every read run in the Edge tab, so they see Edge's own
hotels.com sign-in (sign in once in that Edge profile); the saved header is what `--via http` sends. Exit 3 means
there is no valid session: sign in to hotels.com in that Edge and log in again.

**Edge on CDP 9222 is needed for every command** (setup: [Browser on CDP 9222](#browser-on-cdp-9222-setup)).

## Browser on CDP 9222 (setup)

The commands listed above as needing Edge drive a background tab in your own browser through the Chrome DevTools
Protocol, so their requests carry a real browser's fingerprint and sign-in. The CLI never launches a browser;
start one yourself with remote debugging on:

```bash
# Linux (Edge; google-chrome / chromium take the same flags)
microsoft-edge-stable --remote-debugging-port=9222 --user-data-dir="$HOME/.config/microsoft-edge-debug"

# macOS
"/Applications/Microsoft Edge.app/Contents/MacOS/Microsoft Edge" --remote-debugging-port=9222 \
  --user-data-dir="$HOME/Library/Application Support/Microsoft Edge Debug"
```

- `--user-data-dir` must point at a separate profile: Chromium-based browsers ignore `--remote-debugging-port` on
  the default profile (Chrome 136+). Sign in to tw.hotels.com once inside this profile; it stays signed in across
  restarts. If that profile is already open without the flag, quit the browser first: launching again only opens
  a window in the running instance.
- Check it: `curl -s http://127.0.0.1:9222/json/version` prints the browser version, and `hotels doctor` shows
  `cdp ok`. Another port or host: `--cdp http://127.0.0.1:9333` or `HOTELS_CDP=http://127.0.0.1:9333`.
- `--remote-allow-origins` is not needed: the CLI connects without an `Origin` header (verified 2026-10-30), and
  leaving it out keeps web pages from connecting. The port listens on 127.0.0.1 only; keep it that way, because
  anything that can reach it controls the browser and every session in it.
- The CLI only creates background tabs (`Target.createTarget(background=true)`), never focuses or touches yours,
  closes its own when done, and blanks rather than closes the last tab so the browser does not quit. Run one
  command at a time.

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

- Needs Edge on CDP 9222 and a hotels.com sign-in in it. `--via http` is a short-grace opt-in (see Transport).
- Not verified live: `--stars`, `--free-cancel`, `--breakfast`, `--min/--max`, `--rooms 2`, `--sort rating|distance|stars`,
  currencies other than TWD, `login --from-browser` from a signed-out browser, the 已過去 (past) trips tab.
- `trips`: this account has no bookings, so only the empty state was seen; a non-empty list returns raw page text.
- `saved list` only reports empty vs non-empty (raw text). `saved add` puts the hotel in an auto-created trip named
  after the destination; `rm` removes it from that trip.
- `hotel` shows the distance-to-landmark text as the area; the API gives no street address or star class.
- `search` `free_cancellation` was null in every row observed; use `hotel` for cancellation.
- Persisted-query hashes can rotate when Hotels.com ships a new build.
