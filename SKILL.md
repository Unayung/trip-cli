---
name: trip
description: Use when searching Trip.com Taiwan (tw.trip.com) hotels, checking TWD room rates, totals incl. taxes and cancellation terms for given dates and party size (including children's ages), comparing prices of several hotels, or listing the user's Trip.com orders and saved hotels (no booking).
---

# trip - Trip.com Taiwan hotels from a shell (no booking)

`~/.local/bin/trip` -> `~/Projects/trip-cli/trip`. Python 3.8+ stdlib. Hotel calls go over plain HTTPS with the
saved signed-in session (without one Trip.com returns empty replies). Edge on CDP 127.0.0.1:9222 is only used by
`login`, `saved add/rm` and `--via browser`; never launch Edge. Run ONE command at a time. Start with `trip doctor`;
exit 3 = run `trip login`.

- `trip suggest <text> --json` -> name, kind (city/landmark/station/hotel), filter_id, city_id.
- `trip search <dest|cityId> --checkin YYYY-MM-DD --checkout YYYY-MM-DD [--adults 2] [--children 7,4] [--rooms 1]
  [--sort recommended|price|price-desc|rating|distance] [--page N] [--limit N] [--verify] --json` -> items[] with id,
  name, stars, rating, reviews, price_total, price_per_night, units, room, free_cancellation, breakfast,
  member_deal, extra_charges_possible, distance, url. Sponsored cards dropped; price sorts monotonic, 0 nulls, 0
  page overlap (measured). Children change results (4,602 vs 4,462 hotels).
- `trip hotel <id|url> --checkin --checkout --adults --children [--rooms]` -> address, stars, rating, check-in/out,
  amenities, saved, rooms[].rates[] (price_total, price_per_night, breakfast, cancellation, free_cancel_until,
  payment, max_guests, rooms_left) and `cheapest`.
- `trip price <id>... --checkin --checkout ...` -> lowest bookable total per hotel.
- `trip trips` (read-only orders), `trip saved list|add|rm <id> [--yes]` (dry run without --yes; verified on the
  server list after every write).
- `trip login|whoami|logout`, `trip raw <path>` (GET only).

Price rules: compare by `price_total` = stay total for the whole party, 含稅項及費用 (site label).
`extra_charges_possible` = 可能額外收費 at the property. `units` like `3 張床 × 3晚` means dorm beds; check `room`.
Price sorts rank the first `--pool` (default 100) hotels of Trip.com's own price order, re-sorted, because the site's
order is not monotonic across its pages. List prices can be stale: use `--verify` or `trip price` before deciding
(2-room search: 2 of 4 cheapest rows were sold out on the hotel page; 1-room: 10/10 matched).

Unverified: `--sort rating|distance`, `--min/--max`, non-TWD currency, signed-out login flow, keyring/clipboard login,
member tiers other than 白銀. NEVER book, hold, or pay; this CLI has no such command.
