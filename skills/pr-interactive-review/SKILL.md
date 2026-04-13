---
name: pr-interactive-review
description: Interactive code review walkthrough for a GitHub PR. Takes a PR link as input, fetches the branch into a git worktree, and walks the reviewer through the changes step by step — explaining context, showing code with diffs, surfacing decisions and risks, and collecting review comments. At the end, offers to post the review to GitHub. Use when conducting a thorough code review or exploring PR changes. TRIGGER when user says "review this PR", "walk me through this PR", "review PR https://...", or provides a GitHub PR URL.
user-invocable: true
argument-hint: <GitHub PR URL>
allowed-tools: [Bash, Read, Write, Grep]
---

# PR Interactive Review

Lead an interactive code review of a GitHub PR. Fetch the code into an isolated worktree, walk the reviewer through the changes step by step, collect review comments, and offer to post them to GitHub when done.

This is a conversation, not a document. Present one section at a time. Wait for the reviewer to respond before moving on.

## Process

1. Set up the workspace
2. Explore and understand the changes
3. Walk through interactively
4. Present review and post

## Step 1: Set Up Workspace

The PR link is the required input. Parse it to extract owner, repo, and PR number.

1. **Fetch PR metadata**: `gh pr view <url> --json title,body,baseRefName,headRefName,author,commits,comments,reviews`
2. **Fetch branches**: `git fetch origin <headRefName> <baseRefName>`
3. **Create worktree inside the project**: `git worktree add .git/pr-reviews/pr-<pr-number> origin/<headRefName>`. Placing the worktree inside `.git/` keeps it invisible to `git status` — no untracked files, no gitignore changes, nothing modified in the user's working tree. Still within the repo directory scope, so no permission prompts.
4. **All subsequent file reads and git commands operate from the worktree path.** Never touch the user's working tree.

Generate a short, meaningful feature name from the PR title/content (e.g., `board-summary-enrichment`, `kafka-retry-logic`). Used for the review comments file.

## Step 2: Explore and Understand

Work entirely from the worktree:

1. **Get the diff**: `git diff origin/<baseRefName>...origin/<headRefName>` from the worktree.
2. **Get commit history**: `git log --oneline origin/<baseRefName>..origin/<headRefName>`.
3. **Read full source files**: For every changed file, read the full file in the worktree — not just the diff. Understand how changes fit into existing code, what calls what, how data flows.
4. **Map the logical flow**: Trace how the change works end-to-end. Identify entry points, the path through the system, and where new code connects to existing code.
5. **Read PR description and existing comments**: The author's PR description and any existing review comments provide context. Use them.
6. **Classify changed files**:
   - **Core flow**: Files in the main logical flow — full walkthrough.
   - **Tests**: Summarized (see Special Changes below).
   - **Configuration**: Summarized (see Special Changes below).
   - **Dependencies**: Summarized (see Special Changes below).
   - **Mechanical**: Renames, formatting — mention briefly or skip.

## Step 3: Interactive Walkthrough

Present the review as a series of conversational steps. After each step, **stop and wait** for the reviewer to respond. Explicitly invite questions or signal they can say "next" / "continue".

Scale to the PR size. A small PR doesn't need every section.

### Walkthrough Steps

**Step A — Context & Scope**

Open with:
- Who authored the PR
- What feature or behavior is being introduced/changed, and how it fits in the product (infer from PR description, commits, and code)
- If relevant, include a high-level ASCII diagram showing how the change fits into the system — data flow, service interactions, or component relationships. Keep it simple. Example:

  ```
  Client ──> Board App Service ──> Board Properties Service (new) ──> Summarization API
                                          │
                                          ▼
                                   Cache (HybridCache)
  ```

- What this PR specifically covers and what it doesn't
- Any notable points from the PR description

End with: _"Before I walk through the code — any questions about the scope or context?"_

**Step B — Code Flow Walkthrough**

Walk through core flow files in **logical execution order** — trace how a request, action, or event flows through the system.

If the PR has multiple distinct flows (e.g., a write path and a read path), walk through each separately. Name each flow clearly.

For each step in the flow:
- Name the file and relevant function/class with `file:line` references
- Show the relevant code — see "Always show the code" guideline below
- Explain what it does in the flow — skip what's obvious
- Highlight decisions or patterns the author chose: _"The author went with X here — likely because Y"_
- Flag anything risky, unusual, or worth discussing: _"Worth noting — this doesn't handle the case where..."_

Present one logical step at a time. After each step, pause: _"Questions about this part? Or should I continue to [next part]?"_

**Step C — Special Changes**

Summarize non-flow changes. These change types are summarized, not walked through line-by-line. Present them together as one conversational step. Only include categories that have changes.

- **Tests**:
  - *New tests*: What behavior do they cover? What scenarios or edge cases?
  - *Updated tests*: Why? Behavior change, regression fix, or refactor alignment?
  - *Removed tests*: Why? Lost coverage (red flag) or expected (deleted feature)?
  - Flag: tests removed without replacement, coverage gaps in critical paths, tests that pass trivially.

- **Configuration** (CI/CD, Dockerfiles, infra-as-code, service manifests, env configs):
  - *Why* the config change was made (the diff shows "what")
  - *Risk*: Does it affect production? Could it cause outages?
  - *Scope*: All environments or specific ones?

- **Dependencies** (for each new or changed dependency):
  - *Why* is it needed?
  - *Failure handling*: What happens when it's down?
  - *Retries*: Implemented? If not — explicit decision or oversight?
  - *Performance*: On the critical path? Latency impact?

- **Mechanical**: Mention in one line or skip.

**Step D — Decisions & Risk Areas**

Surface the key decisions and potential risks:
- Design choices the author made and whether they seem sound
- Edge cases or failure modes that may not be covered
- Performance implications
- Security considerations if relevant

Frame as discussion: _"The author chose X — does that align with how we'd want this to work?"_

**Step E — Wrap Up**

Summarize:
- Key findings from the walkthrough
- Concerns and action items raised
- Open questions

Then proceed to Step 4.

## Step 4: Present Review and Post

When the walkthrough is complete, present the full review to the reviewer. This is posted on their behalf — they must see and approve everything before it goes out.

1. **Print the full review contents** — everything from the review comments file:

   **a.** **General PR comments** — the top-level `body`. These are comments not tied to a specific diff line. If there are none, say so.

   **b.** **Inline comments** — every line comment with its file, line number, and comment text. Number them so the reviewer can reference specific ones (e.g., "rephrase #3", "drop #5").

2. **Ask the reviewer**: _"Want to adjust, add, or remove anything?"_ If the reviewer wants changes — rephrase, remove, add, adjust tone — apply them, present the updated version, and ask again. Keep iterating until the reviewer is satisfied.

3. **When the reviewer is done**, ask: _"Ready to post? Comment, Approve, or Request Changes?"_

4. **Post to GitHub** — use the GitHub API directly, not `gh pr review`. The `gh pr review` CLI only supports a single body-level comment — it cannot post inline comments on specific code lines.

   Use `gh api` with `--input -` and pipe a JSON payload:

   ```bash
   cat <<'PAYLOAD' | gh api repos/{owner}/{repo}/pulls/{pr_number}/reviews -X POST --input -
   {
     "commit_id": "<head_commit_sha>",
     "event": "COMMENT",
     "body": "PR-level comment text",
     "comments": [
       {
         "path": "src/some/file.ts",
         "line": 42,
         "body": "Single-line inline comment"
       },
       {
         "path": "src/other/file.ts",
         "start_line": 10,
         "line": 15,
         "body": "Multi-line inline comment spanning lines 10-15"
       }
     ]
   }
   PAYLOAD
   ```

   **Key rules:**
   - Inline comments (`comments` array) can only reference files that are part of the PR diff. The `line` number refers to the line in the new version of the file.
   - Comments on files not in the diff must go in the top-level `body` field as PR-level text. Include file links manually using `[path:line](https://github.com/{owner}/{repo}/blob/<sha>/path#Lstart-Lend)`.
   - Never use `--raw-field` for the comments array — `gh` treats it as a string, not JSON. Always use `--input -` with a piped JSON document.
   - For multi-line inline comments, use `start_line` (first line) and `line` (last line).
   - The `event` field maps to review type: `"COMMENT"`, `"APPROVE"`, or `"REQUEST_CHANGES"`.

5. **Show the review link** — after posting, print the GitHub review URL so the reviewer can see it on GitHub.
6. **Clean up**: `git worktree remove --force .git/pr-reviews/pr-<pr-number>`. The comments file inside the worktree is disposable — the review now lives in the conversation and on GitHub.

## Review Comments File

**Never add review comments into source code files.**

The comments file is a temporary scratch pad that lives in the worktree and is deleted with it at cleanup. Store it at: `.git/pr-reviews/pr-<pr-number>/codereview-comments-{feature-name}.md`

As the review progresses, append every comment, issue, suggestion, and question. When an entry references code, include a `file:line` link to the approximate location.

Format:

```markdown
# PR Review: {feature name}
**PR**: {pr url}
**Author**: {author}

## Line Comments (posted on specific code lines)
- `src/services/board-service.ts:42` — Bulk API call missing — N individual HTTP calls per batch, performance concern
- `src/handlers/enrichment.ts:87` — No error handling for external service timeout
- `src/cache/hybrid-cache.ts:15` — Consider caching at service layer for flexible TTLs

## General Comments (posted as PR-level review)
- What's the expected latency for the external summarization call? Is it on the critical path?
- Test coverage solid for happy path but missing edge cases for empty boards
```

## Guidelines

- **Always show the code.** Every time you reference a code change, include a fenced diff code block with the relevant lines. Use `diff` syntax — `+` for additions, `-` for removals. Never just describe a change in words without showing it. The reviewer is here to look at code.

  Example — instead of _"Added an item-type filter using the keyword sub-field"_, show:

  `boards-search-service.ts:562-564`
  ```diff
  + if (query.itemType) {
  +   filters.push({ term: { "item_type.keyword": query.itemType } });
  + }
  ```

- **This is a conversation.** Present one step, wait for response, continue. Never dump the entire walkthrough at once.
- **Follow the code flow, not the file list.** Trace execution order through the system.
- **Present the author's intent, then your analysis.** First explain what the author likely intended, then surface concerns or questions.
- **Surface risks, not just descriptions.** Focus on what could go wrong, what's missing, what's worth discussing.
- **Read the room.** If the reviewer engages deeply on one step, stay there. If they say "looks good, next" — keep moving.
- **Scale to the PR.** A small PR might only need steps A, B, and a brief wrap-up. Don't force structure onto simple changes.
- **Everything stays in the worktree.** Never modify files in the user's working tree.

## Review Comment Tone & Style

The review comments posted to GitHub should read like they come from a friendly, experienced colleague — not a linting tool or an audit report.

### Tone
- **Suggestive, not directive.** Use "might be worth", "could be nice to", "worth considering" — not "you should", "we need to", "you must". The reviewer is offering perspective, not giving orders.
- **Friendly but critical.** Be pleasant and respectful, but don't soften findings into nothing. If something is a real concern, say so clearly — just frame it as a suggestion rather than a command.
- **Assume good intent.** The author made choices for reasons. Acknowledge that before suggesting alternatives.

### Brevity
- **1-3 sentences per comment.** Make the point and stop. If it takes a paragraph to explain, it's too verbose for a PR comment — have the conversation live instead.
- **Lead with the concern, follow with the suggestion.** Don't explain the entire background. The author is an engineer — they'll understand the implication.
- **No essays.** A review comment isn't a design document. If a topic needs deep discussion, flag it briefly and suggest a sync.

### Examples

Too long and directive:
> The `updateBoardMetadataBulk` calls here and at line 272 are fire-and-forget (not awaited, errors swallowed by `.catch()`). We should `await` these writes instead. The performance difference of awaiting is negligible compared to the LLM inference time, and we've seen stability issues with services choking on many floating promises. Un-awaited writes also risk silent data loss (especially in short-lived processes) and race conditions in the read-then-write logic of `updateBoardMetadataBulk`.

Good:
> These writes (here and line 272) are fire-and-forget. Might be worth awaiting them — the overhead is negligible compared to the LLM call, and we've seen floating promises cause stability issues in other services.

