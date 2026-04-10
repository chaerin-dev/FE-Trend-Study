---
name: chrome-tabs-to-pr
description: Use when the user asks to turn currently/recently open Chrome tabs into a dated FE-Trend-Study markdown file and PR (e.g. "크롬 탭 정리해서 PR", "열린 탭 요약해서 문서 만들어", "오늘 본 아티클 PR 만들어"). Detects open Chrome tabs, filters articles, summarizes each in Korean, writes YYYY/MMDD.md in the repo's existing format, and opens a PR.
---

# chrome-tabs-to-pr

Turn the user's Chrome tabs (article-ish URLs) into a dated study log markdown file + PR, following this repo's existing conventions.

This is a **shared team skill** for FE-Trend-Study contributors. It must work for any team member, not just one person.

## When to use

Trigger when the user asks anything like:
- "크롬 탭에 모아둔 아티클 정리해줘"
- "열린 탭들 요약해서 PR 만들어"
- "오늘 본 글들로 MMDD.md 만들어줘"

## Workflow

### 1. Identify the contributor

You need a display name + GitHub handle for the presenter row. Resolve in this order:

1. If the user gave a name in the message, use it.
2. Run `git config user.name` and `gh api user --jq .login` to get local git identity.
3. Check existing files like `2026/*.md` to see how this person has previously formatted their own row (some use `홍길동 (@handle)`, some just `@handle`).
4. If still ambiguous, ask the user once: "발표자 이름 / GitHub 핸들 알려줘" — then remember it for the rest of the session.

**Never hardcode a specific person.** Different team members will run this skill.

### 2. Collect tabs

> Platform note: the commands below are macOS. On Linux/Windows the AppleScript step won't work — fall straight to the session-file fallback (paths differ: Linux `~/.config/google-chrome/`, Windows `%LOCALAPPDATA%\Google\Chrome\User Data\`). If you can't access Chrome at all, ask the user to paste URLs.

**Try AppleScript first** (macOS, targets frontmost Chrome app):

```bash
osascript -e 'tell application "Google Chrome"
set output to ""
set winIndex to 0
repeat with w in windows
set winIndex to winIndex + 1
set tabIndex to 0
repeat with t in tabs of w
set tabIndex to tabIndex + 1
set output to output & winIndex & "." & tabIndex & " | " & (title of t) & " | " & (URL of t) & "\n"
end repeat
end repeat
return output
end tell'
```

**Caveat**: If chrome-devtools MCP launched its own Chrome (profile at `~/.cache/chrome-devtools-mcp/chrome-profile`), AppleScript may return only those tabs (typically `localhost:...`). If the result looks suspiciously small or all-localhost, fall back.

**Fallback — Chrome session files** (macOS path; adjust per OS):

```bash
ls -lt "$HOME/Library/Application Support/Google/Chrome/"*/Sessions/Session_* 2>/dev/null | head
```

Pick the `Session_*` file with the most recent mtime (today's date). Use `Session_*`, **NOT** `Tabs_*` — `Tabs_*` contains closed-tab history and will return stale URLs the user already forgot about.

Extract URLs:

```bash
strings "<path-to-Session_file>" | grep -Eo 'https?://[^ ]+' | sort -u
```

If multiple profiles have same-day sessions, list candidate profiles (with their newest mtime) and ask the user which to use. Don't guess — different people use different profiles.

### 3. Filter

Drop these:
- `localhost`, `127.0.0.1`, `*.local`
- Search result pages (`google.com/search`, `search.naver.com`, `www.bing.com/search`)
- Auth/redirect URLs (`accounts.google.com/...signin`, `linkedin.com/safety/go`, `*/auth/login`)
- Ad/tracking (`doubleclick.net`, `googlesyndication.com`, `googlevideo.com`, `safeframe.googlesyndication.com`)
- Bare domain roots (e.g. `https://github.com` or `https://github.com/` with no path)
- Duplicates (normalize: strip query strings + trailing slash for dedupe key, keep the longest original URL)
- Personal/private apps unless explicitly requested: Gmail, Meet, Drive, Docs edit URLs, Notion, Slack, Jira, etc.

Keep: article blogs, GitHub repo pages, documentation sites, conference/spec pages, YouTube watch pages **only if the user asked for videos**.

### 4. Confirm with user

Show the filtered URL list (numbered) and ask "이거 맞아? 빼거나 추가할 거 있어?". The session-file fallback is noisy and will include stuff the user closed earlier — confirmation prevents wasted WebFetch calls and wrong-content PRs.

### 5. Summarize

For each URL, in **parallel** (one message, multiple WebFetch calls):

- **GitHub repos**: WebFetch with prompt `"Describe this repo's purpose in 1-2 Korean sentences."` (or `gh api repos/{owner}/{repo}` if you only need the description field)
- **Articles**: WebFetch with prompt `"Give me the Korean article title and a 1-2 sentence Korean summary of the main point."`

If a fetch fails (paywall, auth, redirect), note it and ask the user whether to skip or paste the content manually.

### 6. Write the markdown file

- **Path**: `YYYY/MMDD.md` where date = today (`date +%Y/%m%d`). Year directory must already exist; if not, create it.
- **Format**: match existing files exactly. Read the most recent `YYYY/*.md` first to confirm the current format hasn't drifted.

Template (verify against latest existing file before using):

```markdown
| **발표자**                                        | **날짜** |
| ------------------------------------------------- | -------- |
| [{이름} (@{handle})](https://github.com/{handle}) | YYYYMMDD |

<br />
<br />

## 내용

- **{한국어 제목}**: {1-2문장 한국어 요약}.
  - [{표시 제목} | {출처}]({URL})
- ...
```

Rules:
- Title bolded, colon, then summary ending with a period
- Sub-bullet is a markdown link `[제목 | 출처](url)`
- Multiple sources for one topic → multiple sub-bullets under the same main bullet
- Korean throughout (matches existing files)
- Presenter row uses the identity resolved in Step 1 — **do not hardcode a name**

### 7. Branch, commit, PR

Inspect existing branches/PRs for the team's naming convention before assuming:

```bash
gh pr list --state all --limit 10
git branch -a | head
```

The current convention (verify it still holds): branch `feature/MMDD`, commit + PR title `Create MMDD.md`, base branch `main`.

```bash
git checkout -b feature/MMDD main
# write file
git add YYYY/MMDD.md
git commit -m "Create MMDD.md"
git push -u origin feature/MMDD
```

Then `gh pr create --base main` with this body format (verify against `gh pr view <recent>`):

```markdown
## Summary
- YYYY년 M월 D일 FE 트렌드 스터디 아티클 정리
- {테마 요약} 등 N개 주제

## Articles
- {항목1 짧은 제목}
- {항목2 짧은 제목}
- ...

🤖 Generated with [Claude Code](https://claude.com/claude-code)
```

If the day's file already exists (someone already created `MMDD.md`), don't clobber it — ask whether to append, replace, or use a different filename.

## Known gotchas

- **`Tabs_*` vs `Session_*`**: `Tabs_*` mixes closed-tab history. Only read `Session_*`.
- **MCP-spawned Chrome**: If chrome-devtools MCP is running, AppleScript may target *its* Chrome instance (often just `localhost:...`). Treat suspiciously small results as a signal to fall back to session files.
- **Multiple Chrome profiles**: `Default`, `Profile 1`..`Profile N`. The user's real working profile is whichever has the newest `Session_*` for today — but on shared/multi-account machines, ask.
- **Identity is per-user**: Resolve from git/gh, not from prior conversations or hardcoded values.
- **Format drift**: Always read the most recent `YYYY/*.md` and most recent PR before writing — conventions may have evolved since this skill was written.
- **OS portability**: AppleScript and the session-file path are macOS-specific. Adapt or fall back gracefully on other platforms.
