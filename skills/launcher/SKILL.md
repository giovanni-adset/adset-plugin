---
name: launcher
description: Launch Adset Launcher tasks (ClickUp, Notion or Google Sheets) as ads on Meta, Google or AppLovin through the Adset connector — check the tasks, ask only what is missing, show the full summary, launch after an explicit yes, report per task. Use when the user asks to launch, relaunch or check the launch of tasks or task ids, e.g. "launch these tasks on Meta", "relaunch z935c9kv8b", "launch the ready tasks", "how did the launch go?" (also in Italian: "lancia questi task su Meta", "lancia i task pronti").
---

# Adset Launcher

Launch ads from Adset tasks exactly as the Launcher page's Launch Wizard does: same checks, same templates and defaults, same jobs and Logs. Adset does the work; you collect the decisions, show them, and get the user's yes.

## Tools (Adset connector)

- Read: `list_workspaces`, `launch_list_ready_tasks`, `launch_status`, `launch_list_sessions`.
- Write: `launch_start`, `launch_answer`, `launch_confirm`, `launch_cancel`, `launch_retry`, `launch_mark_launched`.

## Procedure

1. **Workspace.** Use the one already in the conversation; else `list_workspaces` and, if there are several and the request doesn't say which, ask once.
2. **Tasks.** Take every task id or link the user gave (up to 50, all from one source). If they say "the ready ones" (optionally "the ready ones with X in the name"), don't list them first: call `launch_start` with `ready_tasks: true` (and `ready_search`) — its answer lists the tasks, and the user can leave some out (`exclude_tasks`). Use `launch_list_ready_tasks` only when the user just wants to see what is ready.
3. **Start.** Call `launch_start` with the tasks and, as `answers`, every setting the user already stated (platform, account, campaign, Ad Set mode, budget, countries…): each one given here is a round of questions saved (keys in [references/answers.md](references/answers.md)). Pass names as the user said them ("Hike UGC", "Scaling CBO"): Adset matches them against the real options and asks if several match. Never pass an id you did not get from Adset.
4. **Task check.** Report what Adset found: tasks not found, already launched (needs `relaunch`), being launched right now (left out). Ask about relaunching only when Adset asks.
5. **Questions.** Ask exactly the questions Adset returns — at most 4 at a time, as multiple-choice with the recommended option first (use the client's choice widget when there is one). Do not ask anything Adset didn't ask: the rest comes from the task, the Launch Template, the last launch or the default. Send the answers with `launch_answer`. If Adset reports `answerErrors`, explain and ask again.
6. **Summary.** When the status is `ready_to_confirm`, show the whole summary: the tasks that will launch, every setting with where it comes from (your answer / template / last launch / default) and any problem. Ask for an explicit yes.
7. **Changes.** If the user changes anything, send it with `launch_answer` (changing a setting drops the answers that depended on it) and show the new summary: it comes with a new code, and the old one no longer works.
8. **Launch.** Only after the yes, call `launch_confirm` with the code of the summary the user approved. Calling it again never launches twice: it reports the launch already started.
9. **Results.** `launch_confirm` waits up to ~45 s and usually returns the final results. Report per task and platform: launched (ads created, campaign / Ad Set), failed (Adset's reason), or still running — for those call `launch_status` with `wait_seconds: 45` (one call waits for them; don't poll in a loop). New Ad Sets / Ad Groups are created paused, as in the Launcher.
10. **After.** For tasks that launched, ask whether to move them to Launched in ClickUp / Notion / Sheets; only on a yes call `launch_mark_launched`. For a failed task, explain the cause; once it is fixed, `launch_retry` re-runs that job with the same settings (ask first). A task whose launch "completed with problems" can't be retried: relaunching it creates its ads again.

## Speed

- Skip what you already know: the workspace id from earlier in the conversation, task ids the user gave. No `list_workspaces` / `launch_list_ready_tasks` call you don't need.
- Collect everything the user said into `answers` of the first `launch_start`; ask all open questions of a round together (up to 4).
- The confirmation is one yes on the whole summary; don't ask again per task or platform.

## Decisions

- **Amounts** are in the ad account's currency, daily. If the user's currency differs from the account's (the summary shows it), ask before using the number.
- **Missing or not allowed:** if Adset says a platform isn't connected, an account isn't enabled for the workspace, or the user's role is read-only, stop and say so — never look for another way.
- **Several platforms at once** are fine (`platforms: ["meta","google"]`): each gets its own questions and its own part of the summary.
- **A conversation can be resumed:** `launch_list_sessions` lists the user's launches of the last 14 days; `launch_answer` with no answers shows where one stands.
- **Cancel** with `launch_cancel` when the user drops a launch that wasn't confirmed.

## Ground rules (all Adset skills)

- Work only through the Adset connector. Never invent ids, accounts, campaigns, templates, metrics or results.
- Reads need no confirmation. Anything that launches, creates, changes or applies: show exactly what will happen and get an explicit yes for that exact set; use only the confirmation code of the preview the user approved.
- Never say something is done before a tool result says so. Report each object as done, failed (with the reason) or still running.
- Never repeat a write that may already have gone through (a timeout, "in progress"): check its status first.
- Every change is in Adset's Logs, "by <name> (via Claude)". Reply in the user's language.
