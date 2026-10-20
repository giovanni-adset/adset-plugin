---
name: platform-ops
description: Work with Adset's Platform Ops rules through the Adset connector — list, create and change Rule Sets and Rules (mostly ad duplications with conditions such as spend and Triple Whale NC ROAS), run them, review every recommendation, and apply or skip the ones the user picks, after an explicit yes. Use for Platform Ops requests and for rules that select ads by Triple Whale metrics, e.g. "run the Meta Platform Ops rules", "let's create a rule that duplicates ads with spend above €100 and NC ROAS above 0.5", "change rule X" (also in Italian: "esegui le regole Meta di Platform Ops", "modifica la regola X").
---

# Adset Platform Ops

Platform Ops is the same rules engine as Media Buying (product `platform_ops`), used mostly for **duplications** (`duplicate_to`): copying winning ads / Ad Sets into other campaigns or accounts, keeping the post's likes and comments where possible. A run only creates recommendations; nothing is duplicated until someone approves.

## Tools (Adset connector)

- Read: `list_workspaces`, `list_ad_accounts`, `get_entity`, `get_performance`, `rules_list_rule_sets`, `rules_get_rule_set`, `rules_get_rule_options`, `rules_get_run`, `recommendations_list`, `recommendation_get`, `list_changes`, `get_rate_limits` (Meta).
- Write: `rules_save_rule_set`, `rules_save_rule`, `rules_run_rule_set`, `recommendation_act`, `recommendations_act_batch`, `change_set_destination_status`, `revert_change`.

## Running rules and deciding

1. `rules_list_rule_sets` with `product: "platform_ops"`; match the user's words (platform, name); ask if several fit.
2. `rules_run_rule_set`; if still running, `rules_get_run` then `recommendations_list`.
3. Show EVERY open recommendation, numbered: # · source object · destination (account, campaign, Ad Set, new Ad Set) · landing page / Page overrides · reason (rule + conditions) · key data · actions · status. Money is in each row's `currency`, never an assumed one.
4. Duplications can be **approved** or **skipped** (as in Adset; no budget edit). A blocked one may offer `approve_with_post_id` — explain its blocked reason first.
5. Picked items → `recommendation_act` (one) or `recommendations_act_batch` (several): preview → explicit yes → same call with the code. On Meta, before a batch, read `get_rate_limits` for the destination account(s): every copy makes several Meta calls, so with the account paused or its usage at 75% or more, apply fewer at a time and say why. Only skips the user already named exactly need no second yes (a skip duplicates nothing); any approval does.
6. Report per recommendation: what was created (name, id, change id), failures with the reason, still running. Never state a copy's status without reading it (`get_entity`): a copy into an existing Ad Set is created active inside that Ad Set and spends only if the Ad Set and campaign are on. If the user wants it paused, `set_status` after their yes.
7. The Ad Set a duplication created or filled can be turned on with `change_set_destination_status` — only when the user explicitly asks (it starts spending).

## Creating a rule

Example: "duplicate the ads with spend > €100 and Triple Whale NC ROAS > 0.5 in the Scaling CBO campaign (ID 1223453)".

1. **Options.** `rules_get_rule_options`: metrics, operators, actions, attribution, duplication settings. Use only what it lists.
2. **Platform and accounts.** From the request or `list_ad_accounts`; ask if unclear.
3. **Translate the conditions** (conditions in one group are all required):
   - spend > 100 → `{metric: "spend", operator: "gt", value: 100}` (account currency).
   - NC ROAS on Triple Whale > 0.5 → `{metric: "pixel_nc_roas", operator: "gt", value: 0.5}`. `pixel_*` metrics come from the **Rule Set's** Attribution Source, which must be Triple Whale.
   - A campaign is targeted by **name**, not id: read its name (`get_entity` level campaign) and use `campaign_name` `equals` that name; tell the user.
   - Time range: ask if not given (the preview shows it).
4. **Resolve ambiguity before saving.** "…in the Scaling CBO campaign" can mean the ads to look at OR where the copies go — ask. For duplicate_to also settle: destination account, existing Ad Set (`destinationParentId`) or a new one (`adSetMode: "new"`, `destinationCampaignId`, `newAdSetName`), and any landing page / name suffix. Missing required settings → ask, don't guess.
5. **Rule Set.** Reuse an existing Platform Ops Rule Set on the same platform, accounts and Triple Whale attribution, or create one with `rules_save_rule_set` (`product: "platform_ops"`, `attribution: {source: "triple_whale"}`). Changing an existing Rule Set's attribution affects all its rules — say so first. New Rule Sets are manual: they run only when run. Set a schedule only if the user asks for automatic runs.
6. **Preview → yes → save.** `rules_save_rule` without a code returns the preview: show the rule in plain words (when / over / then, metrics source, Rule Set). After the yes, call again with the same fields and the code. Even when the request already names the exact change ("turn rule 317 back on"), show the preview and wait for the user's yes in their next message: a rule change isn't recorded in Adset's Logs, so `revert_change` can't undo it.
7. **Saved ≠ run.** Say the rule is saved and that nothing has been duplicated. Offer to run it now (`rules_run_rule_set`) and go through its recommendations.

## Changing a rule

`rules_get_rule_set` to show the current rules; `rules_save_rule` with `rule_id` and only what changes (`enabled: false` switches it off); `rules_save_rule_set` with `state: "paused"` pauses a whole Rule Set. Always preview → yes → code, also for a change the user named exactly (switching a rule on or off, a new threshold): the save isn't in Adset's Logs and can't be reverted. Deleting rules isn't available here: it is done in Adset's Rules page.

## Ground rules (all Adset skills)

- Work only through the Adset connector. Never invent ids, accounts, campaigns, metrics or results; never build a second recommendation system — rules and recommendations live in Adset.
- Reads need no confirmation. Anything that saves, runs, applies or turns on: show exactly what will happen and get an explicit yes; use only the confirmation code of the preview the user approved.
- Never say a rule was created, run or applied before a tool result says so. Report each object as done, failed (with the reason) or still running.
- Never repeat a write that may already have gone through: check its status first. Reply in the user's language.
