# trip-cli

Agent-native CLI for Trip.com Taiwan (`tw.trip.com`, zh-TW, TWD) hotels. Zero dependencies (Python 3.8+ stdlib).
Search, hotel rooms/rates, price checks, login, orders (read-only) and saved hotels (收藏).
**No booking, no checkout, no payment.**

Every command takes `--json` and returns `{ok, data|error}`. Exit codes: 0 ok, 1 api, 2 usage, 3 auth, 4 net/blocked.

## Transport

Trip.com's hotel services (`/restapi/soa2/...`, JSON POST) answer plain HTTPS normally **when the request carries a
signed-in session cookie** (`cticket`). Without one they return 200 with an empty body (`ResultId 201`), even with
the anti-bot headers the site's JS adds (measured 2026-10-30). So run `trip login` first: it copies the Trip.com
cookies from your own Edge on CDP (`127.0.0.1:9222`, read-only) and checks them against the live account service.

After that, search/hotel/price/trips/saved-list all use plain HTTPS; Edge is not touched. Edge is needed only for
`login`, `saved add/rm` (clicks the hotel page's own 收藏 button in a background tab) and `--via browser`.
Background tabs are created with `Target.createTarget(background=true)`, never focused, and closed afterwards (if it
is the last tab, it is blanked instead, so Edge does not quit). Run one command at a time.

Locale: tw.trip.com pages set `ibulocale`/`ibulanguage`/`cookiePricesDisplayed` on `.trip.com`, which would switch
your hk.trip.com tabs to Taiwan/TWD. The CLI only overrides those in its own HTTPS requests, and the browser path
saves the three cookies before opening a tab and restores them after (verified: still `zh_hk`/`HKD` afterwards).

## Commands

```bash
trip doctor
trip login | whoami | logout          # whoami: masked email, member tier (vipGrade)
trip suggest 心齋橋
trip search 心齋橋 --checkin 2026-12-04 --checkout 2026-12-07 --adults 3 --children 7 \
     [--rooms 1] [--sort recommended|price|price-desc|rating|distance] [--page 1] [--limit 20] \
     [--pool 100] [--verify] [--stars 3,4] [--free-cancel] [--breakfast] [--min N --max N] --json
trip hotel <id|url> --checkin ... --checkout ... --adults 3 --children 7     # alias: rooms
trip price <id> <id> ... --checkin ... --checkout ... --adults 3 --children 7
trip trips [--status All|NotTravel|AwaitPay|AwaitReview]                    # read-only orders
trip saved list
trip saved add <id> [--yes]           # dry run without --yes
trip saved rm  <id> [--yes]
trip raw /robots.txt                  # GET only; refuses booking/order/payment paths
```

`--children` takes ages (`7,4`). A destination is free text resolved through the typeahead (the match is printed,
e.g. 心齋橋 = landmark `13|6790054` in 大阪, city 219) or a numeric Trip.com city id.

## Prices: read this

- `price_total` is Trip.com's stay total for the whole party (all rooms, all nights), labelled **含稅項及費用**
  (taxes and fees included). `price_per_night` = `price_total / nights`. Sorting and comparing use `price_total`.
- `extra_charges_possible: true` means the site adds **可能額外收費**: something may be charged at the property.
- `units` tells what the price buys: `3 張床 × 3晚` is three dorm beds, `2間 × 3晚` two rooms. The site's own
  per-unit price is not used for sorting, because it mixes beds, rooms and apartments.
- `room` is the cheapest room name on the card. Budget rows can be dorm beds (`4床混合宿舍間的雙人床位`).
- Member prices: with a session you get member rates. Same list page, anonymous vs signed in: hotel 132129124
  TWD17,433 vs TWD15,977. Cards with a member rate carry `member_deal: "白銀會員優惠"`.
- Trip.com's 最低價/最高價 order is only sorted inside each of its pages (measured: page 4 held NT$4,430 while page 1
  started at NT$4,497). Price sorts therefore read a fixed pool of the first `--pool` hotels (default 100) in the
  site's price order, re-sort it by total, and serve `--page` windows from that pool. A cheaper hotel can exist past
  the pool; raise `--pool` to look further.
- List prices can be stale. With 2 rooms, 2 of the 4 cheapest list rows were sold out on the hotel's own room list.
  `--verify` re-checks every shown row (one extra call per row) and marks `SOLD OUT` / mismatches. For the real
  1-room case, 10 of 10 matched.
- `hotel`/`price` use only the rates the hotel page shows for your party. The room service also returns rates the page
  hides (`此房間無法容納您搜尋的旅客人數`); these are counted as `hidden_rates`, never used as "cheapest".

## Verified live (2026-10-30)

- `search` 心齋橋 (3 nights), 3 adults + child 7, 1 room: `--sort price` pages 1/2 ascending
  (4,430..5,884 | 5,892..6,832), 0 nulls, 0 overlap; `--sort price-desc` descending (165,319..42,969 | 38,824..25,208),
  0 overlap. Recommended order: 4 sponsored cards dropped per page, 0 overlap.
- Child changes the result: 3 adults 4,602 hotels (cheapest a NT$3,597 dorm, `3 張床`), with child 7: 4,462.
- `hotel 117573109` cheapest 303, NT$4,430 = its search row; `price` on 117573109 / 117611545 / 111110329 =
  4,430 / 4,497 / 12,298, equal to the search rows.
- Taipei 2026-11-20..21, 2 adults: 673 hotels; cheapest 3182366 NT$1,176 in search and hotel (a dorm bed).
- `--rooms 2`, `--stars 4`, `--breakfast --free-cancel`, `--verify`, `--via browser`, `suggest`, `raw`, `doctor`.
- `whoami`: masked email + member tier. `trips`: matches the 我的訂單 page (order, status, total, dates, room).
- `saved`: list 0 -> add 117573109 (verified, 1 saved) -> rm (verified) -> list 0, same as the start.

## Limitations

- Needs a signed-in session for every hotel call (`trip login`). `login` with a signed-out Edge (opens the sign-in
  page and waits) and `--from-clipboard`/`--keyring` were not run live.
- `vipGrade` labels: only grade 1 = 白銀 was seen. `--sort rating|distance`, `--min/--max` and non-TWD currencies
  were not verified live. `--min/--max` filter on `price_per_night` client-side.
- `saved add/rm` depends on the hotel page's 收藏 button (the write call itself was not visible to capture); the
  result is always checked against the server list. A click on a not-yet-hydrated tab did nothing (measured), so
  the CLI waits 5s before clicking, and it never clicks twice.
- Opening a hotel detail page while signed in may add it to the account's 最近瀏覽 (recently viewed).
- Undocumented private API; service ids and fields can change.
