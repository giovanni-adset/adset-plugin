---
name: ads-management
description: Day-to-day ad operations through the Adset connector across Meta, Google and AppLovin — find campaigns, ad sets and ads by name, status, URL, text or performance, change landing pages, budgets, status, names, bid strategy, attribution or ad set targeting (age, gender, locations, custom audiences included / excluded), duplicate ads, create text variants, and combine these with launches and rules, always showing the exact list of objects and getting an explicit yes before any change. Use for spot changes, read-only overviews ("give me an overview", "what's open?") and cross-cutting requests, e.g. "change the landing page of these three ads", "find the active ads with the old domain", "duplicate these ads", "create a variant with a new primary text", "exclude the Customers audience from this ad set", "set the age to 25-54 and add Germany" (also in Italian: "fai il punto", "trova le ads attive con il vecchio dominio", "cambia il landing page di queste ads", "duplica queste ads").
---

# Adset Ads Management

The operating layer for spot work on the ad accounts a workspace manages in Adset. Use the specialised skills when the request is theirs:

| Request | Follow |
|---|---|
| Launch ads from Launcher tasks | the adset:launcher skill |
| Run Media Buying rules, decide on their recommendations | the adset:media-buying skill |
| Platform Ops rules, or selecting ads by Triple Whale metrics (NC ROAS…) | the adset:platform-ops skill |
| Everything else on campaigns, ad sets and ads | this skill |

## Tools (Adset connector)

- Read: `list_workspaces`, `list_ad_accounts`, `get_performance`, `get_entity`, `list_ads`, `list_changes`, `get_creation_options`, `get_rate_limits` (Meta), `list_custom_audiences` and `search_locations` (Meta).
- Write: `set_landing_page`, `update_budget`, `set_status`, `rename`, `set_bid_strategy`, `set_attribution`, `set_targeting` (Meta), `bulk_update` (several objects or changes in one call), `duplicate`, `bulk_duplicate` + `bulk_duplicate_status` (several copies in one call), `create_ad_variant`, `delete_entity`, `revert_change`, and the create tools (`meta_create_*`, `google_create_*`, `applovin_create_*`).

## Workflow for every request

1. **Understand** the action and its target. Ask only for what you can't read from Adset.
2. **Identify** the account (`list_ad_accounts`), then the campaigns / ad sets / ads involved.
3. **Read** them: `list_ads` (status, URL, texts), `get_performance` (spend, ROAS… as the platform reports them), `get_entity`. Reads that don't depend on each other go in parallel, in the same turn.
4. **Check ownership:** only accounts of this workspace; Adset refuses the rest — never work around it.
5. **Ask** about anything missing or ambiguous (which ads, which URL, which campaign).
6. **Show the plan:** a numbered list of every object with its current value and the new one. Many objects → let the user narrow or remove some first.
7. **Get an explicit yes** for exactly that list.
8. **Execute.** One change: the matching tool. Two or more objects, or several changes (status, budget, name, landing page, targeting, bid strategy, attribution) of one object: `bulk_update` — first with `dry_run: true`, which reads every object and returns current → new, problems and Meta's rate-limit state in one call (use that list as the plan of step 6, no separate `get_entity` / `get_rate_limits`), then once more without `dry_run` right after the yes (the dry run doesn't hold it). Stop and ask if the same platform error repeats.
9. **Verify** from the result: every change tool returns the object as it is afterwards (`after`), so no extra `get_entity` is needed unless something looks wrong or the result says it is still running.
10. **Report** per object: done (with change id), failed (reason), not attempted. Changes are undone with `revert_change`.

## Meta's limits (never block the account)

Meta refuses calls, and can lock an ad set's budget or cut the whole ad account off, when changes come too fast. Before a series of Meta changes (several objects, or more than one edit of the same object) read the account's limits — `bulk_update`'s dry run includes them; otherwise call `get_rate_limits` — and follow the `advice`:
- `pausedUntil` set → change nothing on that account before that time; tell the user when it ends.
- Budget: at most 4 changes per hour on one ad set or campaign (a change and its undo count as two). `budgetChanges` shows how many are used and when the next is allowed; plan budget edits so none needs a 5th.
- One object: Meta accepts one change of a campaign / ad set / ad every 30 s; Adset makes a quicker one wait (or stops and says until when). Leave at least 30 s between changes of the same object, and change other objects meanwhile.
- A result with `rateLimitWait` waited for Meta's limit before it was sent: tell the user it waited and why, never "no waits".
- `highestPct` of 75% or more → only the changes that are needed, one at a time, no batches.
- A refusal that names a time ("try again after HH:MM") means nothing was applied, whether Meta refused it or it was never sent: report it with the time and the change id if any, and don't retry before that time, not even once.

## Speed

- Use what you already know (workspace id, ad account id, ids from earlier results): don't call `list_workspaces` / `list_ad_accounts` / `get_entity` again for them.
- Several objects or changes → `bulk_update` (dry run, then one real call): the objects are changed side by side on the server instead of one tool call each.
- Several changes of the same Meta object (e.g. budget + targeting + bid strategy of an ad set, or name + status of an ad) → ONE `bulk_update` item with all of them: Adset sends them to Meta in one request, while separate tools would each wait ~30 s for Meta's per-object limit.
- Two or more copies → ONE `bulk_duplicate` (dry run, then one real call), never one `duplicate` per copy: separate calls run one after another (~20 s per Meta ad set copy of 5 ads, ~50 s with 10 ads), while `bulk_duplicate` makes the copies side by side and sets each copy's name, budget and targeting as it makes it — no `bulk_update` afterwards.

## Selecting objects by criteria

- Name, status, landing page, text: `list_ads` filters (`name_contains`, `url_contains`, `text_contains`, `status`, `campaign_ids`).
- Platform numbers (spend, ROAS, CPA as Meta/Google report them): `get_performance` at the right level and period, then filter and show the numbers you used.
- Triple Whale numbers (NC ROAS, pixel ROAS): only the rules engine reads them. Use a Platform Ops rule — an existing one or a new manual one (adset:platform-ops) — run it, and act on its recommendations: they ARE the selection, and approving them performs the action.

## Specific operations

- **Landing page.** Meta: `set_landing_page` builds a new creative, so the ad loses its likes and comments and goes back to review — tell the user; if social proof matters, offer `create_ad_variant` or `duplicate` with the new URL instead. Undoing it (`revert_change`) points the ad back at its previous creative, so the original post and its likes return. Google: changed in place. AppLovin: on the campaign or the Creative Set (level ad_set).
- **Targeting** (Meta ad sets, `set_targeting`): age (`age_min` / `age_max`, 65 = 65+), `gender`, `locations` and `excluded_locations` (countries as 2-letter codes, regions and cities by the keys `search_locations` returns), `custom_audiences` and `excluded_custom_audiences` (ids from `list_custom_audiences`; `add` / `remove` to change part of the list, `set` for the whole list), `advantage_audience`. Only what you pass changes. Show current → new from `get_entity` (its `targeting`), run `dry_run: true`, then for real after the yes; read the ad set back. Tell the user a targeting change on an active ad set restarts its learning phase, and that with Advantage+ audience on, max age, gender and included audiences are only suggestions. Google and AppLovin targeting can't be changed from here yet.
- **Bid strategy** (`set_bid_strategy`): Meta on the object that holds the budget (campaign with campaign budget, otherwise the ad set). **Attribution** (`set_attribution`): Meta refuses changing it on an existing ad set, so a paused copy with the new attribution is created — say so before.
- **Duplicate** (`duplicate` for one copy, `bulk_duplicate` for two or more): same or other ad set / campaign / account, optional new landing page or Facebook Page; copies stay paused. `bulk_duplicate` items can list the same source several times, each with its own `new_name` (or `name_suffix`), `daily_budget`, `attribution` and — Meta ad set copies — `targeting` (as `set_targeting`). Show its dry run as the plan (source → copy name, budget, targeting) and get one yes. It answers within ~45 s: if `done` is false, tell the user how many are copied and follow with `bulk_duplicate_status` (`batch_id`) until done — never send the same copies again. Report each copy (change id, new id, name) and any `failed` / `renameOrTargetingFailed` with its reason. A Meta ad set copy gets the source's countries and ages (not its gender, interests or audiences — those come from the workspace defaults), so set the targeting the copy needs.
- **Variant** (`create_ad_variant`): new texts and/or URL, same images and videos, same Ad Set, paused. Run with `dry_run: true`, show before → after, create after the yes.
- **Budgets and status**: daily budgets in the account's currency; never turn anything on unless the user explicitly asks.
- **Delete** only on an explicit request (`delete_entity` with `confirm: true`); it can't be undone — offer pausing first.
- **Create** campaigns / ad sets / ads: `get_creation_options` first, `dry_run: true` first on Meta and Google, everything created paused. Never create a campaign the user didn't ask for.

## Ground rules (all Adset skills)

- Work only through the Adset connector. Never invent ids, accounts, metrics or results.
- Reads need no confirmation. Anything that creates, changes, duplicates or deletes: show the exact objects and values and get an explicit yes for that set.
- Never say something is done before a tool result says so; report each object separately.
- Never repeat a write that may already have gone through: check first (`list_changes`, `get_entity`).
- If something isn't supported, say so plainly; never simulate it. Reply in the user's language.
