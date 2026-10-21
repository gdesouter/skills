---
name: community-pulse-builder
description: Set up your own community-listening skill. Asks which products, keywords, channels and tracker file to use, creates the tracker note, generates a tailored "name-pulse" monitoring skill, runs a baseline sweep and can schedule daily refreshes.
---

# Community Pulse Builder

This skill **builds a monitoring skill**. It doesn't do the monitoring itself. Run it once for each thing you want to watch, such as a product, product family, campaign or competitor set. You end up with three things:

1. **A tracker note.** A markdown file in a folder on your computer, with tables for posts, events, press, official posts and GitHub, plus a monthly assessment and a run log.
2. **A monitoring skill** called `<slug>-pulse`. It's generated from the template at the bottom of this file, with your settings filled into its **Config** section, and it's offered to you on a skill card to save.
3. **Optionally, a scheduled daily run** that refreshes the tracker and messages you a summary.

The method was first proven on a UiPath product launch. It covers LinkedIn, YouTube, X, Discourse forums, GitHub, the VS Code Marketplace and Open VSX, Japanese dev sites, and press. Each browser workaround in the template exists because the simple approach failed. Keep them.

---

## Phase 0: Prerequisites (check before asking anything)

Tell the user what's required, then verify each item. Don't start the interview until all of these pass, or the user knowingly accepts a gap.

| Need | Why | How to verify |
|---|---|---|
| Claude desktop app, with this task **linked to their computer** | Tracker file edits and scheduled runs happen on their computer. | `get_device_info` succeeds. |
| A connected folder for the tracker (and one for any sync copy) | That's where the file lives. | `connectedFolders`. If it's missing, use `device_request_folder_access`. |
| `device_bash` available | Edits are made in place with python, never by retyping the file. | Run `echo ok` with it. |
| **Claude in Chrome** extension connected | Every social read goes through their signed-in Chrome. The cloud container can't reach LinkedIn, X, GitHub search, forums or marketplaces. | `tabs_context_mcp` with `createIfEmpty:true`. |
| Signed in to LinkedIn, X and GitHub in that Chrome (only for the channels they pick) | Search results need a login. | Covered in Phase 3. |

Known limits to state up front:
- **Reddit is blocked** for browser automation, WebFetch and curl. Don't try to get around the block. Reddit is covered only if a Reddit connector is added; otherwise users can paste threads in.
- **Scheduled runs need the computer awake**, with the desktop app and Chrome open. A locked screen is fine. Sleep or shutdown means that day's run is logged as FAILED, and the next run catches up.
- **Platform terms.** Reading LinkedIn and X through a signed-in browser is low volume, but it is still automation on a personal account. Keep one daily run for each monitor, and don't stack many monitors on the same account at the same hour.

---

## Phase 1: Interview

Use `AskUserQuestion` for choices and plain chat for free text. Ask in rounds, and don't dump every question at once. Pre-fill smart defaults and let users correct them.

**Round A: What to watch** (free text in chat)
1. **Monitor name** and slug (e.g. "Studio Web pulse" → `studio-web-pulse`). The slug can't contain "claude" or "anthropic".
2. **Products** being tracked, the value each gets in the `Product` column, and **aliases and retired names** (e.g. a launch name that was later renamed, which the community keeps using).
3. **Search terms**:
   - exact phrases;
   - hashtags;
   - the company handle or name to pair with ambiguous words;
   - **known noise** (e.g. bare "Orchestrator" or "Studio" without "UiPath" pulls in unrelated tools).
4. **Official accounts and employee markers.** These decide what goes in the "Official" table, e.g. "at <Company>" in a headline, or the company's YouTube and X handles.
5. **Adjacent and competitor frames** to log whenever they come up (other tools people compare yours to).

**Round B: Channels** (`AskUserQuestion`, multiSelect). Offer:
- LinkedIn
- YouTube
- X
- Discourse forums (ask for the base URLs: your own product forum, plus any partner or adjacent community forums)
- GitHub (ask for the repo search phrase, an optional code-search file pattern, orgs to exclude, and repos to track stars on)
- VS Code Marketplace and Open VSX (ask for publisher and extension IDs)
- Japan (Qiita, Zenn, note.com)
- Press and blogs (WebSearch)
- Other: free text. Add it as a custom channel, using the closest pattern from the template.

**Round C: Files and cadence** (mix of both)
1. **Tracker path.** If they already have a file, migrate its content into the new tables; never discard it.
2. **Sync copies** (e.g. a OneDrive shortcut to a SharePoint folder), or none. The tracker is always the master, and copies are overwritten from it.
3. **Cadence:** daily, weekdays, weekly or manual. Ask for a local time and time zone. Suggest late morning, when they're likely online.
4. **Notifications:** push, email, both, or none.

**Round D: Messaging rules** (free text)
1. **Flag list.** Wording or framing that counts as drift whenever someone uses it: wrong names, retired names, banned descriptors, wrong comparisons. For each item, ask what the correct wording is.
2. **Don't-flag list.** Wording that's fine even if it sounds off-message. This is an explicit list so the monitor doesn't over-index on it. For example, a team may decide a descriptor the community uses is fine, even if it isn't in official copy.
3. **Always-flag topics**, beyond wording: paid posts, pricing questions, production-proof asks, naming fatigue, hosted sessions carrying wrong copy.
4. **Report style:** audience and tone. Default to short bullets with owners named and the biggest risks first.

**Calibration before confirming.** Run 2–3 of their search terms live in Chrome (LinkedIn past-week, X live, YouTube) and show what comes back. Point out noise and suggest exclusions or pairings. This is the single biggest quality lever.

Then **show a config summary** that mirrors the template's Config section, and get a clear yes before building.

---

## Phase 2: Build the tracker

1. If the file exists: back it up to the device VM home (`cp <file> "$HOME/<slug>_backup_<date>.md"`). Migrate existing entries into Table 1 to 5 rows, keeping links and dates. If you can't parse an entry cleanly, put it in "Unsorted legacy items" instead of guessing.
2. If it doesn't exist, write the **Tracker template** below, filled with their names. Use a python write on the device; never retype a file you've read.
3. Drop the tables for channels that aren't enabled. For example, with no GitHub there's no Table 5 and no GitHub counters. With no extensions or GitHub, remove the adoption counters section completely.
4. Set `Last successful run` to `never`. The first run is then a full sweep.

---

## Phase 3: Preflight the real environment

Run the generated skill's Step 0 checks for real:
- folders mounted;
- the master is writable;
- Chrome is connected;
- each enabled login works: LinkedIn returns posts, X shows articles, GitHub code search shows a file count.

Fix the gaps with the user now. A scheduled run can't fix them by itself.

---

## Phase 4: Generate the monitoring skill

1. Take the **Monitor skill template** below. Replace every `{{…}}` with the confirmed config. **Delete the channel subsections that aren't enabled**, and their rows in the tables and counters.
2. Make sure nothing unresolved is left: no `{{` remains, every enabled channel has search terms, and the paths are absolute and match `connectedFolders`.
3. Save a copy to the scratchpad. Then call `propose_skills` with `kind: "new"`, the slug as the name, a one-line description (e.g. "Refresh the <Monitor> community tracker: scan channels, update tables, snapshot and monthly assessment, sync copies"), and the full SKILL.md.
4. Tell the user to **save the card**. Scheduled runs call the skill by name.

---

## Phase 5: Baseline run

Offer to run the first full sweep now (default: the last 30 days). If they agree, follow the generated skill text directly, since it may not be saved yet.

The baseline is the real test. Check that:
- links resolve to canonical URLs;
- dates are exact (UTC);
- every row names an owner;
- the table validator passes;
- the sync copy matches the master (`cmp`).

Report what the run found, and what it couldn't reach.

---

## Phase 6: Schedule (optional)

Create a scheduled task with `create_trigger`:
- `name`: "<Monitor name>".
- `cron_expression`: `CRON_TZ=<their IANA tz> <min> <hour> * * <days>`. Move off :00 and :30 by a few minutes.
- `requires_local_device: true`. The folders attach from the conversation, so only list extra folders in `folders`.
- `notifications`: as chosen.
- `initiation: "human_request"`.
- `prompt`: the **Schedule prompt template** below, filled in.

Then confirm in one sentence which approval setting the task got. If its runs will ask for approval, tell them they can switch it to "Automatically approve" in the task's settings, if their organization allows it.

---

## Phase 7: Hand-off

Give them short bullets:
- where the tracker lives and its sync targets;
- the skill name, and that they need to save its card;
- the schedule and what it needs (computer awake, desktop app and Chrome open, logged in);
- how to change things:
  - to add a keyword or a rule, edit the skill's Config, either by asking Claude to update the skill or by re-running this builder;
  - to monitor another product, run this builder again.

---

## Rules that apply to every generated monitor

- **The master is the source of truth.** Copies are overwritten from it and never edited directly.
- **Public posts only.** Record what's on the post: name, public headline or role, text, engagement. Don't compile anything else about people.
- **Signed-in GitHub search can see private repos** in the user's orgs. Always exclude the company's own orgs, and never list private or internal repos.
- **Don't get around blocked sites.** If a site is blocked, write it in Coverage notes and move on.
- **The Run log is honest.** `Last successful run` only moves after the scan, the update, validation, sync and `cmp` all pass.

---

## Tracker template

~~~markdown
---
updated: {{TODAY}}
tags: [community, {{SLUG}}]
---

# {{MONITOR_NAME}}: Community Posts

**Last checked:** {{TODAY}}. Every engagement and adoption figure in this note is as of this date.
**Last successful run:** never. Scheduled refreshes scan from this timestamp, minus a 2-day buffer.
**Scope:** {{PRODUCTS_SHORT}} · Channels: {{CHANNELS_SHORT}}

## How to use this note
- New items go at the top of each table (newest first). Every row names the **owner** of the post.
- Engagement key: 👍 reactions/likes · 💬 comments · 🔁 reposts · 👁 views · ⭐ stars · 🍴 forks.
- Reach: 🔥 ≥100 reactions or ≥1,000 views · ▲ ≥30 reactions or ≥300 views · · below that.
- Sentiment: 🟢 positive · 🟡 neutral, mixed or wrong framing · 🔴 negative.
- Affiliation: MVP, Community, Partner, Employee, Paid (#ad), Media, Analyst, Investor.
- After every check: update Last checked, refresh the counters and add a history row, rewrite the Snapshot, update the current month in Patterns, add a Run log row, and sync the copies.

## Developer adoption counters
| Metric | Value | Notes |
|---|---|---|

**History**
| Checked | {{COUNTER_COLUMNS}} |
|---|---|

## Snapshot: {{TODAY}}
- (Rewritten on every run.)

## 1. Community posts & videos
| Date | Owner | Role / org | Channel | Affil. | Product | Summary | Engagement | Reach | Sent. | Watch-outs / signal | Link |
|---|---|---|---|---|---|---|---|---|---|---|---|

## 2. Community events & workshops
| Date | Owner / host | Organiser | Title | Format / time | Notes | Link |
|---|---|---|---|---|---|---|

## 3. Press, analyst & investor commentary
| Date | Owner / outlet | Type | Summary | Sent. | Watch-outs / signal | Link |
|---|---|---|---|---|---|---|

## 4. Official & employee posts
| Date | Owner | Role / account | Summary | Engagement | Link |
|---|---|---|---|---|---|

## 5. GitHub community repos
| Created | Owner | Repo | What it is | ⭐ / 🍴 | Last push | Link |
|---|---|---|---|---|---|---|

## Patterns & signals
| Month | Community items | Top organic post | Top video | Sentiment 🟢 / 🟡 / 🔴 | Dominant themes | Key risk | Trend vs prior month |
|---|---|---|---|---|---|---|---|

## Coverage notes & gaps
- Reddit: blocked for automation. It's unverified unless a connector is added or threads are pasted.

## Run log
| Run (UTC) | Status | Scan window | New items | Notes |
|---|---|---|---|---|
~~~

---

## Schedule prompt template

~~~text
Run the {{MONITOR_NAME}} community refresh using the `{{SLUG}}` skill. Invoke it with the Skill tool and follow it exactly.

- Master: "{{MASTER_FILENAME}}" in {{MASTER_FOLDER}}. The master is always the source of truth.
- Sync copies: {{SYNC_TARGETS_OR_NONE}}. Overwriting them is intended.
- Scan window: from the note's "Last successful run" (UTC) minus 2 days, up to now. If it says "never", or the gap is more than 14 days, do a full sweep.

This run is unattended:
1. Run the skill's preflight in full. If any check fails, add a FAILED Run log row if the master is writable, never move Last successful run, report exactly what's missing with SendUserMessage, and stop.
2. Back up, scan every enabled channel, update the note, validate the tables, set Last successful run, add an OK row, sync, and check with cmp. If the sync fails, roll back Last successful run and mark the row PARTIAL.
3. Report with SendUserMessage: status and scan window; new items (owner, what, engagement, flag); counter deltas; the top 3 risks, with drift first; and anything that couldn't be checked. Then attach the file with SendUserFile.

Apply the skill's Messaging rules exactly, including its don't-flag list. Close every Chrome tab you open.
~~~

---

## Monitor skill template

Fill this in and propose it as the new skill. Everything under `## Config` is the user's. Everything below it is the proven procedure, so keep it unless a channel is disabled.

~~~markdown
---
name: "{{SLUG}}"
description: "Refresh the {{MONITOR_NAME}} community tracker: scan {{CHANNELS_SHORT}} for posts about {{PRODUCTS_SHORT}}, update tables, counters, snapshot and monthly assessment, then sync copies."
---

# {{MONITOR_NAME}}

Community-listening tracker for **{{PRODUCTS_SHORT}}**, owned by {{OWNER_NAME}}. Each run does six things:
1. Find new posts, starting from the last successful run.
2. Refresh engagement on existing rows.
3. Refresh the adoption counters.
4. Rewrite the snapshot and the monthly assessment.
5. Record the run.
6. Sync the copies.

## Config (edit here to change what's monitored)

**Products** (with the `Product` column value for each):
{{PRODUCTS_LIST}}

**Aliases and retired names:** {{ALIASES}}

**Search terms:**
- Exact phrases: {{PHRASES}}
- Hashtags: {{HASHTAGS}}
- Pair these ambiguous words with {{COMPANY_NAME}}: {{AMBIGUOUS}}
- Known noise to exclude: {{NOISE}}

**Official accounts** (these go in Table 4): {{OFFICIAL_ACCOUNTS}}
**Employee markers:** {{EMPLOYEE_MARKERS}}
**Adjacent and competitor frames to log:** {{COMPETITORS}}

**Channels enabled:** {{CHANNELS_LIST}}
- Discourse forums: {{FORUM_BASE_URLS}}
- YouTube channels to check by name: {{YT_CHANNELS}}
- GitHub:
  - repo search phrase: {{GH_REPO_PHRASE}}
  - code-search pattern: {{GH_CODE_PATTERN}}
  - orgs to exclude: {{GH_EXCLUDE_ORGS}}
  - repos to track: {{GH_TRACKED_REPOS}}
- Extensions (Marketplace and Open VSX IDs): {{EXTENSION_IDS}}
- Japan terms: {{JP_TERMS}}

**Files:**
| Role | Path on {{OWNER_NAME}}'s computer |
|---|---|
| **Master** | `{{MASTER_PATH}}` |
| Sync copies | {{SYNC_PATHS_OR_NONE}} |

**Messaging rules.** Flag these in the Watch-outs column, and set sentiment to 🟡 when the post repeats them:
{{FLAG_LIST_WITH_CORRECT_WORDING}}

**Always flag:** {{ALWAYS_FLAG_TOPICS}}. Hosted or official sessions carrying wrong copy are the highest priority.

**Don't flag** (acceptable wording: no drift flag, no sentiment downgrade, no risk callout): {{DONT_FLAG_LIST}}

**Report style:** {{REPORT_STYLE}}

---

## Step 0: Preflight (always first; stop if anything fails)

1. **Device:** `get_device_info` must succeed. A locked screen is fine; asleep or off is not.
2. **Folders:** the master folder (and each sync folder) must be in `connectedFolders`. If one is missing, call `device_request_folder_access`. Confirm with `ls "$HOME/mnt/"`.
3. **Master:** it must be readable and writable with `device_bash`. Record its mtime.
4. **Chrome:** `tabs_context_mcp` with `createIfEmpty:true` returns a tab.
5. **Logins** (enabled channels only):
   - LinkedIn content search returns posts, not a login wall.
   - X live search shows articles.
   - GitHub code search shows a file count.
6. **Known-unavailable channels:** Reddit is blocked for automation. Note it and don't retry it, unless a Reddit connector is present. The cloud container can't reach the social sites, so do every channel read through Chrome, and use WebSearch only for press.
7. **Run record:** read `Last successful run` and the Run log. Mention recent FAILED runs in the report.
8. **Backup:** `cp` the master to `$HOME/{{SLUG}}_backup_<YYYYMMDD>.md`, in the device VM home, outside the synced folder.

If preflight fails during an unattended run: add a FAILED Run log row if the master is writable, never move `Last successful run`, report what's missing, and stop.

## Step 1: Current state and scan window

- Read the header, Snapshot, scorecard, Coverage notes and Run log.
- **Window:** from `Last successful run` minus 2 days (LinkedIn's indexing and engagement lag), up to now.
  - If the value is `never`, or the gap is over 14 days, do a full sweep with past-month filters.
  - Record the window in the Run log.
- Parse tables by splitting on unescaped `|`. Editors re-pad columns, so match headers with regex, not exact strings.

## Step 2: Channel scan (Chrome)

For each item, capture:
- **owner** (always a person, or the account if no person is shown);
- role or org;
- affiliation;
- exact UTC date;
- a 1–2 sentence summary;
- engagement;
- reach;
- sentiment;
- watch-outs;
- the canonical link.

Only add items inside the window, and dedupe by link.

### Chrome gotchas (keep them; each one cost real debugging time)

- **JS fails on pages with video or embed iframes.** The errors are "Cannot access a chrome-extension:// URL of different extension" or "Detached".
  - Run JS **immediately after `navigate`, in the same `browser_batch`**, before any scrolling.
  - For long jobs, start an in-page `setInterval` or async IIFE that writes results into `document.title`, then read the title with `tabs_context_mcp`.
  - `find`, `get_page_text` and `read_page` still work when JS doesn't.
- **Tool output is capped at about 1.5k characters.** Print compact lines and slice the output.
- **Output containing `=`, `&`, `?` or `%` can be blocked.** Replace those characters before printing.
- **For JSON APIs, navigate to the site's `robots.txt`** (same origin), then `fetch` from there. That avoids CORS and heavy pages.
- **Close every tab you open.** Prefer navigating the group tab over creating new ones.

### LinkedIn (signed in)

- **Queries:** each phrase and hashtag in Config, on `/search/results/content/?keywords=…`.
  - Use `datePosted` `past-24h`, `past-week` or `past-month`: the smallest that covers the window.
  - Also run the main phrase with `sortBy="date_posted"`.
- **Loading more:** real wheel scrolls only. Use `computer scroll` at (550, 500), 10 ticks, then wait 5 s, and repeat. `window.scrollBy` and `&page=` don't load more. Each query caps at about 15 results.
- **Post URLs aren't in the DOM.** Use the copy-link hook:
  1. On a neutral search (`keywords=qqqzzzxxx999nothing`), run JS immediately. Override `navigator.clipboard.writeText` to push into `window.__copied`. Then start a `setInterval` that clicks each `button[aria-label*="control menu for post by"]`, then its "Copy link to post" item, and appends `name=code` to `document.title`.
  2. Call `history.pushState` to the real search URL and dispatch `popstate`. It's an SPA navigation, so the hook survives.
  3. Wheel-scroll, then read the title.
- **Resolve** each `https://lnkd.in/p/<code>` by navigating to it and reading the canonical URL from `tabs_context_mcp`. Do 4 per batch. Store the URL without `utm` parameters.
- **Exact date** from the 19-digit activity or ugcPost ID: `datetime.utcfromtimestamp((id >> 22) / 1000)`.
- **Engagement:** use `find` for "N reactions", "X and N others reacted" (that's N+1), "N comments" and "N reposts". Read the top comments with `get_page_text`; comments often carry the real signal.
- **Watch for** `#ad` (paid) posts, employee headlines (see the Config markers) and MVPs.

### YouTube

- **Search:** `results?search_query=<term>&sp=CAISAggC` (this week, sorted by date), for each term. Include Shorts. Get the IDs with `find`.
- **Stats:** from `youtube.com/robots.txt`, run a background IIFE that fetches `watch?v=<id>` and extracts:
  - views with `/"viewCount":"(\d+)"/`
  - likes with `/like this video along with ([\d,]+) other/`
  - comments with `/"commentCount":\{"simpleText":"([^"]+)"/`
- **Check the named channels** in Config.

### X (signed in)

- **Queries,** with `f=live`: the phrases paired with the company name, `<company> <product> since:<window start>`, and the hashtags ORed together. Bare product words are often noisy (see Config).
- **Extract from each `article`:** `time[datetime]`, the status link (the `a` that wraps the time), `[data-testid=User-Name]`, `[data-testid=tweetText]`, and the `[role=group]` aria-label (replies, reposts, likes, views). Loop with `scrollBy`, writing to the title.
- **Investors and finance accounts** go in Table 3. Company accounts go in Table 4.

### Discourse forums (each base URL in Config)

- **Search:** `<base>/search.json?q=<term>%20after%3A<window start>`. Read it with `get_page_text`, or with JS right after navigating.
- **Stats:** from `<base>/robots.txt`, fetch `/t/<id>.json` for each tracked topic and read `posts_count`, `views` and `like_count`.
- **Log** bug and friction threads, challenges and events. A first mention on a competitor's own forum is a headline signal.

### GitHub (signed in)

- **Repos:** fetch `api.github.com/search/repositories?per_page=100&sort=updated&q=<phrase> in:name,description,readme`. From the results, get:
  - the total;
  - distinct owners;
  - created per month;
  - top repos by stars;
  - repos new since the last run.
- **Code:** use `github.com/search?q=<pattern> NOT org:<each excluded org>&type=code`, and read the "N files" heading. **Never list private or internal repos.** Exclude the company's orgs every time.
- **Tracked repos:** stars and forks from `api.github.com/repos/<owner>/<repo>`. For external issues, filter on `author_association` not in MEMBER, OWNER or COLLABORATOR.

### VS Code Marketplace and Open VSX

- **Marketplace:**
  1. From `marketplace.visualstudio.com/robots.txt`, POST to `/_apis/public/gallery/extensionquery`.
     - Header: `Accept: application/json;api-version=7.2-preview.1`.
     - Body: `{filters:[{criteria:[{filterType:8,value:'Microsoft.VisualStudio.Code'},{filterType:10,value:'<publisher or term>'}],pageSize:50}],flags:914}`.
  2. Read `statistics`: `install`, `updateCount`, `averagerating` and `ratingcount`.
  3. Get reviews from `/_apis/public/gallery/publishers/<pub>/extensions/<ext>/reviews?count=20`.
- **Open VSX:** `open-vsx.org/api/-/search?query=<term>&size=50` gives `downloadCount`. Reviews are at `/api/<pub>/<ext>/reviews`.
  - Open VSX counts every download, so compare it with Marketplace installs plus updates.

### Japan

- **Qiita:** `qiita.com/api/v2/items?per_page=50&query=<term> created:>=<window start>`. Read `created_at`, `user.id`, `user.organization` (it shows employees), `likes_count`, `stocks_count`, `title` and `id`.
- **Zenn:** `zenn.dev/api/search?q=<term>&order=latest&source=articles`.
- **note.com:** `note.com/api/v3/searches?context=note&q=<term>&size=30&start=0&sort=new`, reading `data.notes.contents`.

### Press, analysts, blogs, podcasts

- **WebSearch:** the product phrases, plus the company's event and earnings coverage, Medium, dev.to, Substack and Hashnode.
- **Note whether the coverage actually names the products.** A big event write-up that ignores them is a signal in itself.

### Reddit

- **Use Reddit only through a Reddit connector.** Otherwise record "blocked, unverified" in Coverage notes.

**Parallelism:** Chrome is one shared browser, so do the Chrome work in order yourself. Subagents may run WebSearch for press at the same time, but never give Chrome work to them.

## Step 3: Classify and write rows

- **Where rows go:**
  - Table 1: community, MVP, partner and practitioner posts.
  - Table 2: events.
  - Table 3: media, analysts and investors.
  - Table 4: official and employee posts.
  - Table 5: external GitHub repos.
- Every table is newest first.
- **Owner** is in bold. Mark it "likely" if you inferred it from a handle. Never use a literal `|` in a cell; use `·` instead.
- **Reach and sentiment** follow the keys in the note's "How to use" section.
- **Watch-outs:** apply the Config **Messaging rules** exactly, including the don't-flag list.

## Step 4: Refresh (python read-modify-write on the device; never retype the file)

1. **Engagement:** refresh every measurable row, and recompute reach.
2. **Counters:** update the values, and append a History row.
3. **Header:**
   - Set a single `Last checked: <today>`.
   - Update `updated:` in the frontmatter.
   - Don't add per-table dates.
4. **Snapshot:** rewrite it in full as `## Snapshot: <today>`. Cover:
   - volume by month;
   - counters;
   - top organic posts, with paid posts listed separately;
   - top videos;
   - new formats;
   - events;
   - themes;
   - the watch list.
5. **Patterns & signals:**
   - Update the current month's scorecard row. Compute it from Table 1 by month.
   - Update the current month's section: Assessment, What's working, What to watch.
   - On the first run of a new month, freeze the previous month.
   - Keep the sections newest first.
6. **Coverage notes:** list what was checked, what was blocked and what isn't visible yet.
7. **Validate:** every row must have the same number of unescaped `|` as its header. Fix any mismatches before syncing.

## Step 5: Run record and sync

- **When a run counts as successful:** only when the scan, update, validation, sync and `cmp` have all passed.
- **Before the final copy:**
  1. Set `Last successful run` to this run's start time (UTC).
  2. Prepend an `OK` row to the Run log: run time, status, scan window, new-item counts, notes. Keep 60 rows.
  3. Re-validate the tables.
- **Sync:** `cp` the master to each sync path, then `cmp`.
  - If either fails, restore the previous `Last successful run`, mark the row `PARTIAL (sync failed)`, save, and report.
- **Chat copy:** stage the master, copy it to `/mnt/user-data/outputs/`, and send it with `SendUserFile`.
- **Close** every Chrome tab you opened.

## Step 6: Report (short bullets, owners named)

- Scan window and status.
- New items: owner, what, engagement and flag.
- Counter changes since the previous history row.
- Top 3 actions or risks, with drift first.
- What couldn't be checked, and why.

## Style

- {{REPORT_STYLE}}
- Always name the person who owns a post.
- Public information only.
~~~
