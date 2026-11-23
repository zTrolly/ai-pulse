# AI Pulse — Phase 1 Feasibility Report

Status: research only, no app code. Compiled 2026-11-23 (America/Sao_Paulo) from official docs fetched that day. Every claim links the doc it came from. "unknown" means the official docs I read do not say. Read against `SPEC.md` (its non-negotiables win: no backend, no fake metrics, no private APIs, secrets only in Keychain, mocks only in previews/tests, MIT + public repo).

## TL;DR

- **Dynamic Island without a backend: feasible, with limits.** A Live Activity can be updated only by the app (foreground, a background task, or a `LiveActivityIntent` button the user taps) or by APNs push from a server. No server and no Push capability on a free account, so freshness is opportunistic in the background, exact in the foreground. It lasts up to 8 h in the Island (12 h on the Lock Screen) and can't be restarted from the background without a user action.
- **Free Apple account: buildable and installable on Breno's iPhone.** App Groups, Background Modes and Keychain Sharing are allowed. Push Notifications are not. Profiles expire every 7 days (rebuild and reinstall). Live Activities and widgets need no listed capability, but I found no doc that confirms they work under a Personal Team, so check that on the device first.
- **Providers:** no consumer plan (ChatGPT Plus/Pro, Claude Pro/Max, Gemini/Google AI Pro, SuperGrok, Cursor individual) has a documented usage API. GitHub Copilot is the exception: individually billed plans have a usage report. Developer API usage exists for OpenAI, Anthropic, xAI, OpenRouter, Mistral (Enterprise only) and Cursor (Enterprise teams only), almost always behind an org-admin or management credential.
- **MVP provider: OpenRouter.** It is the only provider with a documented **read-only** management key plus a documented credit limit, remaining amount and reset time (`GET /api/v1/key`, `GET /api/v1/credits`). That lets the app show a real percentage without a high-privilege secret.

## 1. Dynamic Island / Live Activity feasibility

| Fact | What the docs say | Source |
|---|---|---|
| Max duration | Active up to **8 hours**, then the system ends it and removes it from the Dynamic Island. It stays on the Lock Screen up to 4 more hours (**12 h max**). | [Displaying live data with Live Activities](https://developer.apple.com/documentation/activitykit/displaying-live-data-with-live-activities) |
| Network | A Live Activity "can't access the network". It runs in its own sandbox. | same |
| Payload size | Static + dynamic data ≤ **4 KB** | same |
| Start | App generally must be in the **foreground** to start. Exception: an App Intent conforming to `LiveActivityIntent` can start one while the app is in the background, e.g. from a Control Center control. | same; [LiveActivityIntent](https://developer.apple.com/documentation/appintents/liveactivityintent) |
| Update from app | `update(_:)` works "while your app is in the foreground or while it's in the background — for example, by using BackgroundTasks". Updates to an ended activity are ignored. | [Activity.update(_:)](https://developer.apple.com/documentation/activitykit/activity/update(_:)) |
| Update via push | APNs push from **your server** (token or broadcast channel). Needs the Push Notifications capability and a remote notification server. Push budget is per hour. Priority 5 doesn't count toward the budget. `NSSupportsLiveActivitiesFrequentUpdates` raises it. | [Starting and updating Live Activities with ActivityKit push notifications](https://developer.apple.com/documentation/activitykit/starting-and-updating-live-activities-with-activitykit-push-notifications) |
| Interactive buttons | Buttons/toggles in a Live Activity run App Intents. A `LiveActivityIntent` runs **in the app's process**. Buttons are inactive on a locked device until it's unlocked. | [Adding interactivity to widgets and Live Activities](https://developer.apple.com/documentation/widgetkit/adding-interactivity-to-widgets-and-live-activities) |
| Staleness | Optional `staleDate`. At that time, `activityState` becomes `.stale` and `isStale` becomes true. | [ActivityContent.staleDate](https://developer.apple.com/documentation/activitykit/activitycontent/staledate) |
| End / dismissal | Default dismissal keeps it up to 4 h after end. `.immediate` or `.after(_:)` are available within that 4 h window. | [Displaying live data…](https://developer.apple.com/documentation/activitykit/displaying-live-data-with-live-activities) |
| BGAppRefreshTask | The system picks when to run it. It gets **up to 30 s** of runtime. Needs the `fetch` background mode and `BGTaskSchedulerPermittedIdentifiers`. `earliestBeginDate` is only a lower bound, not a guarantee. | [Choosing background strategies](https://developer.apple.com/documentation/backgroundtasks/choosing-background-strategies-for-your-app), [BGAppRefreshTask](https://developer.apple.com/documentation/backgroundtasks/bgapprefreshtask), [earliestBeginDate](https://developer.apple.com/documentation/backgroundtasks/bgtaskrequest/earliestbegindate), [Using background tasks](https://developer.apple.com/documentation/uikit/using-background-tasks-to-update-your-app) |
| BGTaskScheduler numeric budget | **unknown**. No refresh count is published. | same |
| Background push | Needs a server. More than 3 per hour gets rate limited. | [Choosing background strategies](https://developer.apple.com/documentation/backgroundtasks/choosing-background-strategies-for-your-app) |
| WidgetKit reload budget | Per widget, per 24 h: **40–70 refreshes** for a frequently viewed widget (about every 15–60 min). Entries ≥ ~5 min apart. Reloads aren't counted while the app is in the foreground or when the widget runs an App Intent. | [Keeping a widget up to date](https://developer.apple.com/documentation/widgetkit/keeping-a-widget-up-to-date) |
| Widget networking | Allowed during snapshot or timeline generation, with limited time. Background URLSession is recommended. | [Making network requests in a widget extension](https://developer.apple.com/documentation/widgetkit/making-network-requests-in-a-widget-extension) |

**Without a backend, can a Live Activity stay fresh? Partly, and the app has to say so honestly.**

1. **Exact while the app is in use:** fetch, then `update(_:)` on launch, on becoming active, and on pull-to-refresh.
2. **On demand from the Island:** a "Refresh" button backed by a `LiveActivityIntent`. It runs in the app's process, fetches, and updates. The user has to tap it, and the device must be unlocked.
3. **Best-effort in the background:** `BGAppRefreshTask` (≥ 15 min `earliestBeginDate` per `PERF_BUDGET.md`) fetches and updates. The system decides if and when it runs, so the app can't promise a cadence.
4. **Honest staleness:** set `staleDate = lastFetch + N`. Render a "stale" state and the "updated X ago" time from the cached `fetchedAt`. Never interpolate or guess values.
5. **Not possible:** push updates (needs a server, and Push isn't available on a free account) or keeping the activity past 8 h. After 8 h, restarting needs the app in the foreground or a user-run `LiveActivityIntent` (Control Center control, Shortcut, Action Button). Scheduling or chaining activities to keep the Island permanently alive would break the SPEC's "no hacks to keep the Island alive" rule, so it's out.

## 2. Free Apple account (Personal Team)

| Item | Status | Source |
|---|---|---|
| App Groups | ✅ allowed ("Apple Developer" column) | [Supported capabilities (iOS)](https://developer.apple.com/help/account/reference/supported-capabilities-ios) |
| Background Modes (`fetch`) | ✅ allowed | same |
| Keychain Sharing | ✅ allowed | same |
| Data Protection | ✅ allowed | same |
| Push Notifications | ❌ not available, so no ActivityKit push and no background push | same |
| iCloud (CloudKit/KVS), Sign in with Apple, Time Sensitive Notifications, Siri, Associated Domains | ❌ not available (none needed by SPEC) | same |
| WidgetKit extension | Not a listed capability (no entitlement). Whether the extension counts toward the 3-apps limit: **unknown** | same; [Developer account overview](https://developer.apple.com/help/account/basics/about-your-developer-account/) |
| Live Activities | Needs only Info.plist `NSSupportsLiveActivities`, no listed capability. Explicit confirmation for a Personal Team: **unknown, verify on device** | [NSSupportsLiveActivities](https://developer.apple.com/documentation/bundleresources/information-property-list/nssupportsliveactivities) |
| Profile expiry | Provisioning profiles expire **7 days** from issuance. Rebuild and reinstall. | [Developer account overview](https://developer.apple.com/help/account/basics/about-your-developer-account/) |
| Limits | Up to **10 App IDs** (expire after 7 days), **3 devices** (expire after 7 days), **3 apps per device** | same |
| Distribution | Free accounts can't distribute apps | [Supported capabilities (iOS)](https://developer.apple.com/help/account/reference/supported-capabilities-ios) |

Plainly: the app, widget extension, App Group, Keychain and BGAppRefresh are buildable for free. Anything push-based isn't. The app plus the widget extension use 2 App IDs. Users of the public repo build it themselves in Xcode, because a free account can't distribute.

## 3. Provider capability matrix

Legend: ✅ documented · ❌ documented as unavailable, or no endpoint found in official docs · ❓ unknown.

| Provider | Usage via official API? | Endpoint(s) | API vs consumer coverage | Credential & privilege | Limits / reset windows exposed | Usage-endpoint rate limit | On-device iOS? | Relevant terms/guidance |
|---|---|---|---|---|---|---|---|---|
| **OpenAI API** | ✅ [Usage](https://platform.openai.com/docs/api-reference/usage) | `GET /v1/organization/usage/{completions,embeddings,images,…}`, `GET /v1/organization/costs` | API only; buckets `1m/1h/1d` | **Admin API key** (org). Admin keys can't call non-admin endpoints. No scopes, only optional expiry ≤ 1 yr. An admin key can create other admin keys ([create admin key](https://developers.openai.com/api/reference/resources/admin/subresources/organization/subresources/admin_api_keys/methods/create), [Admin APIs](https://developers.openai.com/api/docs/guides/admin-apis)). The "Usage" RBAC permission is Org-owner only ([RBAC](https://developers.openai.com/api/docs/guides/rbac)). | Monthly spend limits/alerts via Admin API ([spend limits](https://developers.openai.com/api/docs/guides/spend-limits)). `x-ratelimit-*` headers appear on model responses only ([rate limits](https://developers.openai.com/api/docs/guides/rate-limits)). | ❓ | Technically yes (HTTPS). High risk. | OpenAI: "Keep the administrative credential in the trusted service" ([API keys](https://developers.openai.com/api/docs/guides/api-keys)) |
| **ChatGPT Plus/Pro** | ❌ | none found | Consumer. Limits "may vary" ([What is ChatGPT Plus](https://help.openai.com/en/articles/6950777-what-is-chatgpt-plus)) | n/a | message caps not published as numbers | n/a | n/a | "API usage is separate" (same) |
| **Anthropic API** | ✅ [Usage & Cost Analytics (beta)](https://platform.claude.com/docs/en/manage-claude/usage-cost-analytics-api), [legacy Usage & Cost](https://platform.claude.com/docs/en/manage-claude/usage-cost-api) | `GET /v1/organizations/analytics/{usage,cost}`; legacy `/v1/organizations/usage_report/messages`, `/cost_report`; `/v1/organizations/rate_limits` ([Rate Limits API](https://platform.claude.com/docs/en/manage-claude/rate-limits-api)) | API org usage. The legacy usage endpoint **excludes** Claude Code, Cowork and claude.ai chats. The cost endpoint covers usage-based charges across products. | Admin key `sk-ant-admin01-` (admin role only, **no scopes**), an `org:admin` OAuth token, or a **personal/service-account key not scoped to a workspace** (acts with the owner's permissions) ([Admin API](https://platform.claude.com/docs/en/manage-claude/administration-api), [Admin keys](https://platform.claude.com/docs/en/manage-claude/admin-api-keys), [Auth](https://platform.claude.com/docs/en/manage-claude/authentication)). Not for individual Console accounts or the claude.ai Team plan. | Rate limits readable. Analytics data refreshes within ~4 h (up to 24 h). Legacy data within ~5 min. | Legacy: "polling once per minute for sustained use" | Technically yes. High risk with an admin key. | "store it in a secrets manager, rotate"; prefer WIF/short-lived tokens (same) |
| **Claude Pro/Max (+ Claude Code on plan)** | ❌ | none found | 5-hour session + weekly limits, shared across Claude and Claude Code ([Claude Code with Pro/Max](https://support.claude.com/en/articles/11145838-using-claude-code-with-your-pro-or-max-plan)) | n/a | windows named, quotas not numeric | n/a | n/a | — |
| **Claude Code (org)** | ✅ [Claude Code Analytics API](https://platform.claude.com/docs/en/manage-claude/claude-code-analytics-api) | `GET /v1/organizations/usage_report/claude_code` | Console orgs; data ≥ 1 h old | Admin API credentials | not real-time | ❓ | yes, admin risk | same |
| **Gemini API** | ❌ no usage endpoint in Gemini API docs. Usage is shown in the AI Studio dashboard ([Billing](https://ai.google.dev/gemini-api/docs/billing)) | — | API | API key (per project) | RPM/TPM/RPD per project. RPD resets at midnight Pacific. Tier spend caps ([Rate limits](https://ai.google.dev/gemini-api/docs/rate-limits), [Billing](https://ai.google.dev/gemini-api/docs/billing)) | n/a | n/a | Google Cloud Monitoring/Billing APIs not evaluated: ❓ |
| **Gemini Apps / Google AI Pro** | ❌ | none found | Compute-based limits, refresh every 5 h up to a weekly cap. Only multipliers published ([Gemini Apps limits](https://support.google.com/gemini/answer/16275805)) | n/a | no numbers | n/a | n/a | "limits may change" (same) |
| **GitHub Copilot** | ✅ [Billing usage](https://docs.github.com/en/rest/billing/usage) | `GET /users/{username}/settings/billing/{ai_credit/usage, premium_request/usage, usage, usage/summary}` | **Individual plans only** ("only applicable if the user has purchased their own Copilot plan"). Org/enterprise via org endpoints (admin) | **Fine-grained PAT, "Plan" user permission (read)**, read-only | Gross, discount and net quantities per model/SKU. Allowance denominator not a field: ❓. Legacy premium requests reset on the 1st at 00:00 UTC ([Copilot requests](https://docs.github.com/en/copilot/concepts/billing/copilot-requests)) | 5,000 req/h per user ([REST rate limits](https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api)) | yes, low risk | Org metrics need `manage_billing:copilot` ([Copilot metrics](https://docs.github.com/en/rest/copilot/copilot-metrics)) |
| **Cursor** | ✅ Enterprise teams only ([APIs overview](https://cursor.com/docs/api)) | `POST /teams/spend`, `/teams/filtered-usage-events`, `/teams/daily-usage-data`, `/v2/teams/...` ([Admin API](https://cursor.com/docs/account/teams/admin-api)) | Team spend, including included usage. Individual Pro: ❌ none found | Team API key, Basic auth. `read:*` or `admin:*` scope | Per-user limits, billing-cycle spend | Default 20/min. Usage events 60/min. Poll ≤ 1/h. | yes, team-wide data | — |
| **xAI API** | ✅ [Management API: billing](https://docs.x.ai/developers/rest-api-reference/management/billing) | `POST /v1/billing/teams/{team_id}/usage`; prepaid balance; postpaid spending limits | API team usage | **Management key** (separate from the API key) at `management-api.x.ai`. Can create/delete API keys and **top up prepaid credit** ([guide](https://docs.x.ai/developers/management-api-guide)). Read-only variant: ❓ | spending limits, prepaid balance | ❓ | yes, very high risk | — |
| **Grok / SuperGrok** | ❌ none found | — | One weekly pool shown as % in Settings → Usage ([Grok FAQ](https://docs.x.ai/grok/faq), [Usage limits](https://docs.x.ai/x-subscription/usage-limits)) | n/a | weekly reset (in-app only) | n/a | n/a | — |
| **Mistral** | ✅ **Enterprise only, not self-serve** ([Usage metrics](https://docs.mistral.ai/admin/admin-api/usage-metrics), [Admin API overview](https://docs.mistral.ai/admin/admin-api/overview)) | `GET /v1/admin/usage`, `POST /v1/admin/usage/query`, `GET /v1/admin/spend-limit` ([Admin billing](https://docs.mistral.ai/api/endpoint/beta/admin/billing)) | API + le Chat / Vibe analytics | Admin API key. Standard keys don't work. | monthly spend limit | ❓ | yes, admin risk | Admin API is "Preview" |
| **OpenRouter** | ✅ [Limits](https://openrouter.ai/docs/api-reference/limits), [Get current key](https://openrouter.ai/docs/api-reference/api-keys/get-current-key), [Credits](https://openrouter.ai/docs/api/api-reference/credits/get-remaining-credits) | `GET /api/v1/key` (limit, `limit_remaining`, `limit_reset`, `usage_daily/weekly/monthly`, `effective_budget.resets_at`); `GET /api/v1/credits` (`total_credits`, `total_usage`, management key) | API (credits) | Inference key works for `/key`. **Read-only management key** ("Use read-only keys for monitoring") can't call completions ([Management keys](https://openrouter.ai/docs/guides/overview/auth/management-api-keys)) | Credit limit + reset interval + `resets_at`, free-model daily counter | Per-IP burst limits on `/key` and `/credits` (numbers not published). Reads don't count toward the key-management limit. | yes, **lowest risk** | Recommend expiration on management keys (same) |

## 4. On-device credential risk

Keys live only in the app's Keychain, are never logged, and are never put in the App Group, widgets or Live Activity state. If the stolen secret is the usage credential itself:

| Provider | Credential | Blast radius if leaked | Least-privilege option |
|---|---|---|---|
| OpenAI | Admin API key | Full org admin: users, projects, API keys, rate limits, spend limits, **can mint more admin keys**. No scopes. | Set `expires_in_seconds` (≤ 1 yr). No read-only admin key documented. Org-owner only. |
| Anthropic | Admin key / unscoped personal key | Admin key: full Admin API (members, keys, workspaces, rate limits). Personal key: owner's permissions, **plus** inference. | Key expiration. A personal key held by a lower-role member: whether the usage endpoints accept non-admin roles is ❓. Enterprise: `read:analytics`-only key ([Admin keys](https://platform.claude.com/docs/en/manage-claude/admin-api-keys)). |
| GitHub Copilot | Fine-grained PAT, Plan:read | Read the user's billing/usage. No write. | Already least-privilege. Set token expiry. |
| Cursor | Team API key `read:*` | Read team members, usage and spend for the whole team | Use `read:*`, never `admin:*` |
| xAI | Management key | Create/delete API keys, change billing, **top up prepaid credit** (spends money) | Read-only key not documented: ❓ |
| Mistral | Admin key (Enterprise) | Spend/rate limits, roles | ❓ |
| OpenRouter | Read-only management key | Read keys list, usage, credits, analytics. Writes return 403. **No inference, no spend.** | ✅ documented read-only + expiration |
| OpenRouter (alt) | Inference key with its own credit limit | Can spend up to that key's limit | Per-key `limit` + expiry |

Keychain: use `kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly` so BGAppRefresh can read keys while locked, without iCloud or backup migration ([doc](https://developer.apple.com/documentation/security/ksecattraccessibleafterfirstunlockthisdeviceonly), [Restricting keychain item accessibility](https://developer.apple.com/documentation/security/restricting-keychain-item-accessibility)). Keep secrets in the app's **private** keychain group. The widget extension never needs them, because it only reads the cache. UI rule (SPEC): label admin/management keys "High privilege", show what they can do, and require explicit confirmation before saving.

## 5. MVP recommendation: OpenRouter

The SPEC's named list doesn't include OpenRouter, but the brief does. One provider end to end:

- Only provider with a **documented read-only** credential that can't spend money or run inference.
- Exposes a **real denominator and reset** (`limit`, `limit_remaining`, `limit_reset`, `resets_at`, `total_credits`), so the Island can show an honest % with no invented quota.
- Self-serve for individuals. No Enterprise or org-admin gating.
- Simple JSON GET, easy to test against recorded fixtures (tests only).

Runner-up: **GitHub Copilot (individual)**. Read-only fine-grained PAT, but no allowance denominator in the response, so it shows usage counts, not %. Avoid as MVP: OpenAI, Anthropic, xAI (admin/management keys on device), Cursor and Mistral (Enterprise only), and Gemini (no usage endpoint).

## 6. Architecture sketch (no backend)

- **Targets:** `AIPulse` (app), `AIPulseWidgets` (WidgetKit extension: widgets + Live Activity UI), shared `Core` package.
- **Secrets:** app-private Keychain (`AfterFirstUnlockThisDeviceOnly`). No Keychain sharing unless an extension ever needs to fetch, and the MVP doesn't.
- **App Group `group.<bundle>.aipulse`:** small Codable `WidgetSnapshot` (provider, metric, value, denominator?, resetsAt?, fetchedAt, status). Contains no secrets. SwiftData stays app-side for history.
- **UsageSyncManager (app process only):** triggers are launch, become-active, pull-to-refresh, `BGAppRefreshTask` (≥ 15 min), and `LiveActivityIntent` "Refresh". One in-flight request per provider, 5 min minimum interval (except manual refresh), honor 429/`Retry-After`. After a fetch: write SwiftData, write the App Group snapshot, `WidgetCenter.reloadTimelines` only if data changed, and `activity.update` only if the content state changed (with `staleDate`).
- **Live Activity:** one activity at a time, started from the foreground or a `LiveActivityIntent`. The content state is a few numbers (well under 4 KB). Show the "updated X ago" time from `fetchedAt`. When it's stale, show stale. After 8 h it ends; the app offers "Restart" on next open.
- **Widgets:** timeline built from the App Group snapshot only. No network in the extension for the MVP.

## Top risks / blockers

1. **No push, so Live Activity freshness in the background is opportunistic** (BGAppRefresh has no guaranteed cadence), and the 8 h cap ends the Island; restart needs a user action.
2. **Consumer subscriptions have no official usage API** (ChatGPT, Claude, Gemini Apps, SuperGrok, Cursor individual). These must show "unsupported" or manual tracking only.
3. **Major API providers need high-privilege keys on device** (OpenAI admin, Anthropic admin, xAI management), and Cursor and Mistral are Enterprise-only. Free-account friction: 7-day reinstall, 3 apps/device, 10 App IDs per 7 days, and Live Activity on a Personal Team still unverified on device.
