# perksly-data

Public credit-card benefit catalog consumed by the [Perksly](https://github.com/WTL7/Perksly) iOS app.

The app fetches `cards_data.json` from this repo's raw URL on launch (throttled to once per 24h) and falls back to the bundled copy when offline.

## Raw URL

```
https://raw.githubusercontent.com/WTL7/perksly-data/main/cards_data.json
```

## Schema

See `cards_data.json` for the full structure. Top-level fields:

- `version` — string in `YYYY.MM` form (use leading zeros, e.g. `"2026.04"`). Bumped on every meaningful change. The app picks the newer of bundled vs cached vs remote by lexicographic compare.
- `cards` — array of card templates, each with stable `id`, `benefits`, optional `perks`.

## Updating

1. Edit `cards_data.json`
2. Bump the `version` field
3. `git commit -am "Catalog v2026.05"` then `git push`
4. App users get the new catalog within their next 24h refresh window

**Stable IDs are load-bearing.** Once a `card.id` or `benefit.id` ships, never reuse or repurpose it — the app uses these as join keys when diffing user-saved data against catalog updates.
