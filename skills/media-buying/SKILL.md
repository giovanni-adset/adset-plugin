---
name: media-buying
description: Run Adset's Media Buying rules (budget and performance rules on Meta, Google and AppLovin) and handle their recommendations through the Adset connector — show every recommendation in a table, then approve, change the budget of, pause or skip only the ones the user picks, after an explicit yes. Use when the user asks to run a Media Buying rule or Rule Set, see or decide on recommendations, or approve / edit / ignore some of them, e.g. "run the Meta Media Buying rule", "apply the first and the third", "set the second one's budget to €250" (also in Italian: "esegui la regola Meta di Media Buying", "applica la prima e la terza").
---

# Adset Media Buying

A Rule Set is a group of Rules; running it reads the ad accounts and produces **recommendations** (increase/decrease/set a budget, turn on/off). Nothing changes on a platform until someone approves a recommendation. Adset's Dashboard and Logs are the same ones you act on here.

## Tools (Adset connector)

- Read: `list_workspaces`, `rules_list_rule_sets`, `rules_get_rule_set`, `rules_get_run`, `recommendations_list`, `recommendation_get`, `list_changes`, `get_rate_limits` (Meta).
- Write: `rules_run_rule_set`, `recommendation_act` (one), `recommendations_act_batch` (several), `revert_change`; for creating or changing rules, `rules_get_rule_options`, `rules_save_rule_set`, `rules_save_rule`.

All rule tools take `product: "media_buying"` where asked.

## Procedure

1. **Workspace** from the conversation, else `list_workspaces` (ask if several).
2. **Rule Set.** `rules_list_rule_sets` and match the user's words (platform, name). One match → use it; several → list them (name, platform, accounts, last run) and ask.
3. **Run.** `rules_run_rule_set`. It refuses a Rule Set that is already running (say so and offer to wait). Still running after the wait → `rules_get_run`, then `recommendations_list` with the Rule Set.
4. **Table.** Amounts are in each row's `currency` (never assume €). Show EVERY open recommendation, numbered, with: # · object (level and name) · type · current → proposed · reason (rule + conditions met) · key data (the metrics the rule used) · possible actions · status. More than 50 → page with `offset`. Blocked ones show their blocked reason. A `previousOutcome` means the same proposal was already decided before and the rule proposes it again because its conditions still match.
5. **Wait.** Do not act until the user says what to do. Never "approve all" on your own.
6. **Translate the instruction** into exact recommendation ids and actions, then show that list back:
   - "accept all the scaling ones" → every row of type Increase budget (say which ones).
   - "only the ones with NC ROAS > 1" → filter on the data shown (`pixelNcRoas` = Pixel NC ROAS from the Attribution Source). A row without that number is left out and listed as such.
   - "set the second one's budget to €250" → row 2, `edit_budget` with `new_daily_budget: 250` (the account's currency).
   - "ignore / reject" → `skip`: Adset has no separate reject; skip removes it from the queue, changes nothing, and the next run proposes it again if still true.
   - "pause it instead" → `turn_off`.
   - Ambiguous ("the good ones", a row number that doesn't exist) → ask.
7. **Preview and yes.** One recommendation: `recommendation_act` without a code → preview. Several: `recommendations_act_batch` with the exact items → one preview for the set (and what can't be done). Show it; get an explicit yes for exactly that. Exception: skips the user already named exactly ("skip 2 and 3", "skip all three") change nothing on the platform and need no second yes — but if the set mixes skips with approvals or budget edits, the whole set needs the yes.
8. **Apply.** On Meta, before applying budget changes or turning things off on several objects, call `get_rate_limits` for the ad account(s) (with `object_ids` = the Ad Sets / campaigns involved): Meta allows 4 budget changes per hour on one Ad Set or campaign, and a paused account or a budget at its limit means those items will be refused — say which ones and when they can go, and leave them out of the batch. Then call the same tool again with the same items and the confirmation code — several picked recommendations always in ONE `recommendations_act_batch` call: Adset applies them side by side (different objects at once, one object's in order), much faster than one `recommendation_act` each. A changed recommendation or a different amount invalidates the code: preview again.
9. **Report** per recommendation: applied (with change id), failed (Adset's reason), still applying (check with `recommendation_get`). The rest stays pending in the Dashboard. A failure for a Meta rate limit names a time: nothing was applied; don't retry it before that time.
10. **Undo** on request: `revert_change` with the change id; `list_changes` with `only_claude: false` shows the Logs, rule changes included.

## Creating or changing a Media Buying rule

`rules_get_rule_options` → build the Rule Set / Rule → `rules_save_rule_set` / `rules_save_rule` return a preview and a code → save after the yes. Even when the request already names the exact change ("turn rule 317 back on"), show the preview and wait for the user's yes in their next message: a rule change isn't recorded in Adset's Logs, so `revert_change` can't undo it. New Rule Sets are manual unless the user explicitly asks for a schedule. The adset:platform-ops skill describes the same steps in detail.

## Ground rules (all Adset skills)

- Work only through the Adset connector. Never invent ids, accounts, campaigns, metrics or results.
- Reads need no confirmation. Anything that changes or applies: show exactly what will happen and get an explicit yes for that exact set; use only the confirmation code of the preview the user approved.
- Never say something is done before a tool result says so. Report each object as done, failed (with the reason) or still running.
- Never repeat a write that may already have gone through: check its status first.
- If an action isn't offered for a recommendation, say so; never simulate it. Reply in the user's language.
