# MiMo Quota Monitor · Usage

## Check five things before installing

**1. Your DesireCore client is 10.0.156 or later.** This Agent relies on platform fixes shipped in that version: completion contracts for scheduled runs, stability of the built-in browser over long runs, and schedules keeping their tools after being paused and resumed.

**2. Your MiMo account is in the account store.** In DesireCore, open Resources → Compute → Accounts and add an account with "Site / service" set to `platform.xiaomimimo.com` (that is how the Agent recognizes a MiMo account; without it the Agent reports that the account store has none). Note the account's id (for example `mimo-main`); that id is how you tell the Agent which account to check. DesireCore stores the password encrypted, and the Agent fills the login form through placeholders without ever seeing it in plain text.

**3. Run the first check in a conversation and complete one SMS verification yourself.** Tell the Agent "Check my MiMo quota with mimo-main". It creates a persistent browser environment for that account, opens the login page, fills in the account and password, stops at the SMS verification page and tells you. In the browser panel, click 「发送验证码」 (Send code) and enter the code, then click "Resume Agent" in the panel and **reply to the Agent**; it carries on and reads the quota. Do reply once you are done: the Agent keeps the browser environment while it waits, and any scheduled check in the meantime is recorded as "in use". The login is kept in that environment, so you normally do not log in again; when cookies expire it first tries to restore the session without the form and only asks you if that fails.

**4. One browser environment per account.** Each account is kept separate, and the mapping lives in `mimo-quota/accounts.json` in the work directory, created during the first check in a conversation. Do not let two accounts share a browser environment — Xiaomi's login cookies overwrite each other.

**5. It is read-only and runs no commands.** It reads only four query endpoints (plan detail, usage, balance and login status). It never purchases, renews, cancels, changes the plan, resets an API key or logs out. The whole check uses only the built-in browser and file tools and runs no commands; if you see it running a command in its run records, the model has gone off track.

## How to use it

- "Check my MiMo quota", "Check once with mimo-main" — run one check now and report
- "Check mimo-main every hour" — have it create a scheduled job
- "What did the last check find?" — read the latest result without opening the browser

## Setting up scheduled checks

First get one check working in a conversation as described in item 3, then tell the Agent "Check mimo-main every hour". It creates a schedule for that account with a completion contract that requires each run to write `mimo-quota/mimo-main/latest.json`. Completion contracts can currently only be set through the Agent, so let it create the schedule rather than creating one by hand in the UI.

Run the first check and create the schedule in this Agent's own conversation rather than in a team or a topic with its own work directory: scheduled checks use the Agent's default work directory, and a different directory will not find the browser environment mapping.

With the completion contract in place, if the model ends its turn before finishing, the platform continues the same session and asks it to finish; if the file is still missing, the run is recorded as failed and you are notified, so a missed check never goes unnoticed.

Scheduled checks never create browser environments, and once a previous run already needed a manual login they do not keep filling in the login form (repeated attempts tend to trigger Xiaomi's risk controls). If the login expires and cannot be restored, the Agent notifies you; ask it to check again in a conversation and complete the verification as in item 3.

Approvals:
- Reading the endpoints uses browser navigation and page-text reads, which need no approval.
- Creating a browser environment and writing the result files request approval. Under the default AI approval mode, a conversation first leaves you a short window (about 30 seconds by default) to decide yourself, after which AI approval decides; in scheduled runs AI approval decides directly.

Model: the default is smart routing at the flagship tier. Multi-step browser work is demanding; in real runs lightweight models tended to stop halfway, lose track of context or wander into unrelated tasks, so lowering the tier is not recommended.

## What you get

A `mimo-quota/` directory in the work directory:

```
mimo-quota/
├── accounts.json            which browser environment belongs to which account
└── mimo-main/
    ├── latest.json          the most recent result
    └── history/             every result, one file per check
```

Each result records the plan tier, total quota, used, remaining, remaining percentage, reset time (both the original UTC value and Beijing time), the change since the last successful check, the data source (API or page), data quality, and the alerts that apply to this check.

## When it notifies you

Every scheduled check leaves a one-line run receipt in this Agent's conversation. On top of that, the Agent sends you a message when one of the situations below appears. **It notifies you once per situation while that situation lasts**: if the quota stays at 15%, you hear about it only when it first drops to 20% or below, a single failed check in between does not repeat the notice, and a quota reset back to full does not trigger a false alarm.

- Remaining quota drops to 20% or below
- The quota resets within 3 days
- Remaining quota dropped by 10 points or more between two consecutive successful checks no more than 24 hours apart (unusual consumption)
- The figures fail validation
- Problems that need you: the account is missing from the account store, the account has no browser environment yet, the login expired and could not be restored, a human verification appeared, the page changed, and similar
- Problems that may clear up on their own (browser temporarily unavailable, network failure, browser environment in use, session interrupted, unknown operation outcome) persisted for two checks in a row

## What it will not do

- Spend money, change the plan or account settings, or call anything outside the query allow-list
- Bypass sliders, image captchas or SMS verification — it stops and asks you
- Store or output passwords, cookies, verification codes or full API keys; phone numbers keep only the last 4 digits
- Invent numbers: when it cannot read a value it records the check as blocked and says where it got stuck
- Switch to another account because one account cannot log in
- Run commands or launch other programs

## Known limitations

- The first login in a new browser environment needs your SMS verification
- Scheduled checks need DesireCore running; nothing is checked while the computer sleeps or the client is closed
- When the page or the endpoints change, it records "page changed" and notifies you instead of guessing
- If two checks of the same account overlap (for example a manual test run colliding with a scheduled one), the later one is recorded as "in use" and the next run normally proceeds; you are notified only if this happens two checks in a row
- During a long network outage, or when the built-in browser cannot open the page, the check is recorded as "browser temporarily unavailable", and you are notified only if it has not recovered two checks in a row. If the outage lasts long enough, the AI approval needed to write the result files can fail too, and that check is recorded as failed

## Privacy

Account data stays inside DesireCore on your machine: the password in encrypted storage, the login in the local browser environment, and the results in your work directory. This Agent never sends any of it anywhere other than the MiMo open platform itself.
