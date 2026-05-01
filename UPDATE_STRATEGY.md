# Card Database Update Strategy

A repeatable process for keeping `cards_data.json` accurate as issuers refresh card terms. Card benefits change roughly quarterly (sometimes mid-year for Amex / Chase / Citi). This doc captures the format, the audit checklist, and the cadence.

## Where the catalog lives

The live catalog lives in a **separate public repo**: [WTL7/perksly-data](https://github.com/WTL7/perksly-data). The app source repo (`WTL7/Perksly`) is private; hosting the JSON there would require auth on every fetch, so we split.

Two copies exist for different purposes:

| Where | Purpose |
|---|---|
| `WTL7/perksly-data/cards_data.json` (public, raw URL) | **Source of truth.** App fetches this on launch, throttled to 24h. Update this for live changes. |
| `Perksly/Card_Data/cards_data.json` in the app repo | **Bundled fallback.** Used on first launch + when offline + when remote is unreachable. Update at every App Store release so fresh installs aren't months behind. |

**Live update flow** (changes reach users within 24h, no App Store review):
1. Edit `cards_data.json` in the `perksly-data` repo
2. Bump the `version` field (lex-comparable `YYYY.MM` with leading zeros, e.g. `"2026.05"`)
3. Validate (`python3` script below)
4. `git commit -am "Catalog v2026.05 — <summary>"` then `git push`
5. Users get the new catalog within their next 24h refresh window

**Bundled-copy refresh** (do this whenever you ship an App Store update):
1. Copy the latest JSON from `perksly-data` over the bundled `Perksly/Card_Data/cards_data.json`
2. Build, ship in the App Store release

## TL;DR — monthly update flow

1. Run the **audit checklist** (below) against any card whose terms change.
2. For each affected card, look up current benefits on the issuer's official benefits page (search-result links go stale; primary sources don't).
3. Apply the change as a JSON patch in `perksly-data`, bump `version`, validate enums.
4. Commit + push to `perksly-data` — live within 24h.
5. Periodically (e.g., before each App Store release), refresh the bundled copy in the app repo too.

## Schema reference

The schema is enforced by the SwiftData / JSON decoders. Everything below has been tested against current code.

### Root object

```json
{
  "version": "2026.04",
  "cards": [ ... ]
}
```

`version` — string in `YYYY.MM` form (mandatory leading zeros, e.g. `"2026.04"` not `"2026.4"`). The app picks the newer of bundled vs cached vs remote by **lexicographic compare**, so leading zeros are required for the ordering to be correct. Bump on every meaningful catalog change.

### Card object

```json
{
  "id": "<issuer>_<short_name>",
  "name": "Display Name",
  "issuer": "American Express | Chase | Citi | Capital One | Bank of America | ...",
  "image_asset_name": "card_<id>",
  "benefits": [...],
  "perks": [...]
}
```

`id` convention: lowercase with underscores, prefixed by issuer abbreviation (`amex_`, `chase_`, `citi_`, `capone_`, `bofa_`, `wf_`, `usbank_`, `barclays_`). Keep stable — IDs are referenced by user data.

### Benefit object — for **expiring**, cycle-based credits

```json
{
  "id": "<card_id>_<short_name>",
  "name": "Annual Travel Credit",
  "description": "Concise, end-user readable.",
  "value_amount": 300.0,
  "currency_symbol": "$",
  "reset_frequency": "monthly | quarterly | semi_annual | annual | quadrennial",
  "reset_rule": "calendar_year | card_anniversary | calendar_month | transaction_date",
  "category": "airline | hotel | dining | shopping | transport | other",
  "is_enrollment_required": false,
  "monthly_cap": 25.0,
  "notes": "Optional — quirks, edge cases, conditional triggers."
}
```

- **`value_amount`**: total annual / cycle value. For monthly benefits with a uniform cap, the model derives the per-month figure from `monthly_cap`.
- **`currency_symbol`**: `"$"` for dollar credits. Use a non-`$` unit for count-based benefits (`night`, `companion`, `boarding`, `voucher`, `pass`). The UI auto-formats: `$ → currency style`, others → plain number + unit suffix.
- **`monthly_cap`** (optional): set when each month has the same cap (e.g., `$25/month digital entertainment`).
- **`sub_benefits`** (optional): per-month overrides for benefits whose monthly cap varies (e.g., Amex Plat Uber Cash: $15/mo Jan-Nov, $35 Dec). 12-element array of `{"month":"Jan","value":15}`.
- **`sub_periods`** (optional, informational only): quarterly/semi-annual breakdown for display purposes. Not decoded by Swift currently — the model derives sub-period value from `value_amount / N`.

### Perk object — for **always-on** features

```json
{
  "name": "Centurion Lounge Access",
  "description": "Concise, end-user readable.",
  "icon": "airplane.circle.fill"
}
```

SF Symbol palette in active use:
- `airplane.circle.fill` — lounges, airline status
- `crown.fill` — hotel/loyalty status
- `car.fill` — car rental status, auto rental insurance
- `shield.lefthalf.filled` — trip insurance bundles
- `globe` — no foreign transaction fees
- `phone.circle.fill` — concierge, cell phone protection, travel hotline
- `creditcard` — card features (intro APR, build credit, no AF)
- `gift.fill` — bonus categories, anniversary points, statement credits
- `bag.fill` — purchase / return protection, baggage insurance
- `lock.shield.fill` — fraud / ID theft protection
- `tv` — entertainment perks (Apple TV+, Disney bundle)
- `arrow.uturn.backward.circle.fill` — return protection
- `person.2.fill` — companion certificate perks (when held as a perk)

## Benefit vs. perk classification rules

The most common error is misclassifying. When in doubt, ask: *if the user does nothing, does this still have value next year?*

| Type | → Where | Reason |
|---|---|---|
| Statement credit that resets per cycle (annual / monthly / quarterly) | **Benefit** | Has a deadline |
| Anniversary points / miles auto-deposited to loyalty account | **Perk** | No expiry once deposited |
| Per-stay credit (no annual cap, fires every eligible booking) | **Perk** | No cycle |
| Free-night certificate from anniversary or earned | **Benefit** (`unit: "night"`, `value_amount: 1`) | Expires 6-12 months after issuance |
| Companion certificate (Delta, Citi AA Globe, etc.) | **Benefit** (`unit: "companion"`, `value_amount: 1`) | Expires 1 year after issuance |
| Conditional spend credit ("$X after $Y spend") | **Benefit** if it resets cycle, **Perk** if one-time / 2+ year validity | Track if calendar bound; describe in perk if not |
| Lounge passes with annual cap (4 Admirals Club passes/yr) | **Benefit** (`unit: "pass"`, `value_amount: N`) | Resets per cycle |
| Always-on % discount, status, lounge membership, insurance | **Perk** | No usage cap |
| Conditional welcome offer (only year 1) | **Perk note**, not benefit | Not recurring |

## Non-`$` unit registry

Adding a new unit requires updating two places in `InsightsView.swift`:

```swift
private static func displayNameForUnit(_ unit: String) -> String { ... }
private static func iconForUnit(_ unit: String) -> String { ... }
```

Current registry:
| Unit | Display name | Icon | Example cards |
|---|---|---|---|
| `night` | Free Nights | `bed.double.fill` | IHG Premier, Marriott Boundless / Brilliant / Business, Ritz-Carlton, Hilton Aspire, Hyatt |
| `companion` | Companion Certificates | `person.2.fill` | Delta Reserve / Platinum (personal + business), Citi AA Globe, Atmos Ascent, Alaska Business |
| `boarding` | Upgraded Boardings | `airplane.circle.fill` | Southwest Priority |
| `voucher` | In-flight Vouchers | `ticket.fill` | Southwest Performance Business |
| `pass` | Lounge Passes | `airplane.circle.fill` | Citi AA Globe, Citi Strata Elite |
| `upgrade` | Room Upgrades | `arrow.up.circle.fill` | Ritz-Carlton (3 Club Level upgrades/year) |

Adding a new unit:
1. Add a benefit with the new `currency_symbol` value
2. Add cases for the unit in both helper functions in `InsightsView.swift`
3. Sync to both repo paths
4. Test that the Insights "By Unit" section shows the new row

## Audit checklist (run monthly or on news event)

For each card the audit covers, verify:

### Card-level
- [ ] Annual fee unchanged? (If changed, update description in any benefit that mentions it.)
- [ ] Issuer unchanged? (Some cards switch issuers — e.g., Amazon Business cards moving from Amex to US Bank in Aug 2026.)
- [ ] Card still available to new applicants? (Closed-to-new is okay; existing holders still need accurate data.)

### Benefits
- [ ] Each benefit's `value_amount` matches current issuer terms.
- [ ] `monthly_cap` matches current per-month cap if monthly.
- [ ] `reset_frequency` and `reset_rule` enums valid (one of the lists above).
- [ ] `category` enum valid.
- [ ] `is_enrollment_required` accurate.
- [ ] Description still matches reality (issuer wording change, eligibility changed, etc.).
- [ ] No benefit silently demoted by issuer (e.g., Hilton Business spend-based free night was removed — required moving to perks or removing entirely).

### Perks
- [ ] Status tier still offered (Hilton Diamond vs Gold, Marriott Platinum vs Gold, IHG Platinum vs Silver).
- [ ] Lounge access network unchanged (Priority Pass departures — Hilton Business sunset its Priority Pass, this kind of change is common).
- [ ] Insurance limits unchanged (Trip cancellation $/trip, $/year, etc.).
- [ ] No FTF still applies.

### Format hygiene
- [ ] All four enum sets (frequency, rule, category, currency_symbol) validated by the script below.
- [ ] No invalid `per_stay`, `per_use`, etc. — these silently fall back to `annual / calendar_year` in `Card.init(from:)`.

## Validation script

Drop this in a one-shot Python invocation any time you patch the JSON:

```python
import json
data = json.load(open('cards_data.json'))
valid_freq = {'monthly','quarterly','semi_annual','annual','quadrennial'}
valid_rule = {'calendar_year','card_anniversary','calendar_month','transaction_date'}
valid_cat  = {'airline','hotel','dining','shopping','transport','other'}
known_units = {'$','night','companion','boarding','voucher','pass'}

errors = []
for c in data['cards']:
    if not c.get('id') or not c.get('name') or not c.get('issuer'):
        errors.append(('missing required field', c.get('id')))
    seen_b_ids = set()
    for b in c.get('benefits', []):
        if b['id'] in seen_b_ids:
            errors.append(('duplicate benefit id', c['id'], b['id']))
        seen_b_ids.add(b['id'])
        if b['reset_frequency'] not in valid_freq:
            errors.append((c['id'], b['id'], 'freq', b['reset_frequency']))
        if b['reset_rule'] not in valid_rule:
            errors.append((c['id'], b['id'], 'rule', b['reset_rule']))
        if b.get('category') and b['category'] not in valid_cat:
            errors.append((c['id'], b['id'], 'cat', b['category']))
        if b.get('currency_symbol') not in known_units:
            errors.append((c['id'], b['id'], 'unit', b['currency_symbol']))
    for p in c.get('perks') or []:
        if not p.get('icon'):
            errors.append((c['id'], 'perk', p['name'], 'missing icon'))

if errors:
    print(f'{len(errors)} validation errors:')
    for e in errors: print(' ', e)
else:
    print(f'Clean — {len(data["cards"])} cards.')
```

Run after every edit. New units in `currency_symbol` should be added to `known_units` here AND to the InsightsView mappings.

## Where to edit

Live catalog edits happen in the **`perksly-data` repo only**. There's no path-sync ritual for live updates — the app fetches from the public raw URL.

The bundled fallback inside the app repo (`Perksly/Card_Data/cards_data.json`) only needs to be refreshed at App Store release time. When refreshing it, copy from the public repo to make sure both are aligned:

```sh
# When prepping an App Store release, refresh the bundled copy:
cp /path/to/perksly-data/cards_data.json \
   "/Users/waves/Claude-Workspace/Credit-Card-Tracker/Perksly/Card_Data/cards_data.json"
cp /path/to/perksly-data/cards_data.json \
   "/Users/waves/Documents/GitHub/Credit-Card-Tracker/Perksly/Card_Data/cards_data.json"
```

**Stable IDs are load-bearing.** The app uses each `card.id` and `benefit.id` as a join key when diffing user-saved data against catalog updates. Once a benefit ships in a public release, **never reuse or repurpose its id**. Add new IDs for replacement benefits; let the old ones disappear from the JSON if they're discontinued. The app handles "removed from template" via its dismissed-benefit flow.

## Adding a new card

1. Pick the smallest correct id (`<issuer_abbr>_<short_name>`).
2. Choose `image_asset_name = card_<id>`. Add the image asset to `Assets.xcassets` separately if you want a card image.
3. Build benefits + perks per the schema above.
4. Append to `data['cards']` in the JSON.
5. Validate. Sync. Commit.

## Source priority for terms verification

In order of trust:
1. **Issuer official benefits page** — `americanexpress.com/.../benefits`, `creditcards.chase.com`, `citi.com/credit-cards`, `capitalone.com`, etc. These are authoritative and dated.
2. **Card-specific Amex/Chase benefit detail URLs** — e.g., `global.americanexpress.com/card-benefits/detail/...` — these list current credit values and enrollment URLs.
3. **TPG / NerdWallet / Upgraded Points / AwardWallet / FrequentMiler** — generally fast on issuer changes; double-check value_amount against issuer page before persisting.
4. **Reddit / forums** — useful for spotting changes earliest, but verify against issuer page.

When a number disagrees between sources, trust the issuer page. When the issuer page is silent on a benefit a third-party page mentions, the third-party may be ahead of an officially announced change — note in `notes` that it's pending verification.

## Discontinued-but-active cards

Several cards are closed to new applications but existing holders still carry them and rely on accurate benefit data. These are worth keeping in the database with a perk note that they're closed:

- **Chase Ritz-Carlton Card** — closed years ago; still has $300 airline credit, anniversary free night (85K), 3 Club Level upgrades/year
- **Citi Prestige Card** — closed to new apps in 2021; still has $250 travel credit, 4th-night-free (2/year), 5x dining/air, Priority Pass with guests, $100 GE every 5 years (note: 5 years, not 4)
- **AAdvantage Aviator Red (Barclays)** — migrating to Citi AAdvantage Platinum Select on April 24, 2026
- **Hawaiian Airlines Business (Barclays)** — closed to new apps; existing holders retain 50%-off companion benefit
- **Citi AAdvantage Platinum Select Mastercard** (older non-World-Elite version) — current product is World Elite
- **U.S. Bank Altitude Reserve** — closed to new applicants in 2024 but still active
- **Citi Rewards+** (replaced by Citi Strata) — auto-upgraded in July 2025

When adding these, lead with a perk titled "Closed to New Applicants" so users immediately understand the card's status. Keep all benefit data current — these holders are the primary audience for the app's tracking value.

## Common gotchas seen during initial database build

- **Hilton Business 2024-2025 refresh**: Priority Pass discontinued; the $15K / $60K spend-based free night was removed entirely. Old guides still describe these.
- **Capital One Venture X Feb 2026 lounge changes**: guest fees apply; authorized users no longer get free lounge access. Existing data may overstate guest privileges.
- **Amazon Business cards move to US Bank Aug 2026**: still valid as Amex through that date. Plan to add a duplicate US Bank entry in late summer 2026 and gradually deprecate the Amex one.
- **Aeroplan 2026 changes**: Level Up benefit replaced by tiered SQC system; old guides outdated.
- **Chase Slate**: rebranded from Slate Edge in 2026; Slate Edge no longer accepting new applicants.
- **Citi Rewards+ → Citi Strata** (July 2025): existing Rewards+ holders auto-upgraded; remove old Rewards+ if it appears.
- **Credit values silently increase**: Disney bundle on BCE/BCP went from $84 → $120 (BCP only). Watch monthly Amex benefit dashboards for value bumps.

## Cadence recommendation

| Cadence | What to do |
|---|---|
| **Monthly** | Skim issuer "Card benefits update" feeds (Amex Insider, Chase blog, NerdWallet news, TPG news). Audit any card mentioned in news. |
| **Quarterly** | Full audit on top-10 most-tracked cards. Run validation script. |
| **Semi-annually** | Full audit across all cards. Verify enums, sync, commit. |
| **On news event** | When a major issuer announces refresh (Amex, Chase, Citi, Capital One typically refresh once or twice a year), audit all cards from that issuer within a week. |

## Future improvements to consider

- Per-card `version` and `last_verified_at` fields, so users can see how fresh each card's data is.
- `effective_from` dates per benefit so the app can deprecate old terms automatically.
- `data_source_url` per benefit linking to the issuer's authoritative page — enables quick re-verification.

(The catalog-level top `version` field is already in use — see the schema reference.)
