# Runbook 2 — Curating the body of work

> **Produced:** a curated public profile — the repos that represent the work indexed and described, stale/tutorial repos archived (never deleted), and a coherent one-screen index written. **Repo:** n/a (this is a meta runbook operating across an account's repos).
> **What the review cycle caught:** the pin/index steps had to stay honest about what was actually done versus deferred — the final report distinguishes "set" from "left as a manual step", rather than claiming completion it couldn't verify.

**For:** an AI agent with `gh` CLI (or a GitHub API token) authenticated as the account owner.
**Goal:** a curated body of work. Someone spending 60 seconds on the profile should see a coherent engineering story, not tutorial clutter — a profile index, a handful of representative pins, and descriptions on the repos that matter.

The discipline here is editorial, not promotional: deciding what work represents you, removing noise without destroying history, and writing an index that reads as one body of work.

## Guardrails (non-negotiable)

1. **Some repos are off-limits** — if a repo has been deliberately deferred (a pending decision, or it holds material that can't be public), do not touch it in any way: no archiving, no visibility change, no edits. List it for a human decision; don't act.
2. Archive only — **never delete** any repo. Archiving is reversible; deletion is not.
3. Operate only on the intended account. Nothing in any org.
4. Verify identity first: `gh auth status` must show the intended account. Stop if it shows anything else.
5. If a repo doesn't match a classification rule below, leave it alone and list it in the final report as "needs a human decision".

## Step 1 — Full inventory

Get the complete list first — never operate on a partial view:

```bash
gh repo list <account> --limit 100 --json name,description,isFork,isArchived,pushedAt,primaryLanguage \
  > repo-inventory.json
```

## Step 2 — Classify

Sort every repo into one of three buckets: **KEEP**, **ARCHIVE**, or **needs a human decision**.

**KEEP (do not archive)** — the repos that represent the work:

- the foundational repos (the agent runtime and the eval harness)
- any repo with real, non-tutorial work — give it a description if it's missing one
- the profile-index repo (created in Step 4)

**ARCHIVE** — noise that dilutes the story. Archive if ANY of:

- fork of a tutorial/course repo (`isFork: true` and clearly not a meaningful contribution)
- name matches course-code or throwaway patterns (`*-test`, `*Test`, single throwaway words, obvious course prefixes)
- last push before a cutoff date AND no description AND no README beyond boilerplate

If none apply and it isn't a keeper → the "needs a human decision" bucket. When unsure, don't archive.

## Step 3 — Execute archiving

```bash
for repo in <list of matched stale repos>; do
  gh repo archive "<account>/$repo" --yes
done
```

Add descriptions and topics to keepers that lack them:

```bash
gh repo edit <account>/<repo> --description "<clear one-line description>" --add-topic <topic> --add-topic <topic>
```

A keeper without a description is indistinguishable from clutter at a glance — that's the whole reason to describe it.

## Step 4 — Profile index

Create the profile-index repo (public) containing a single `README.md`. Rules for the content:

- **Lead with what you build**, in plain language — the unglamorous parts that make the systems work, not adjectives.
- **Describe any non-public or employer work generically** — capability and scale, never names ("a large microservice fleet", "an internal code-intelligence service") — and keep it that way.
- **Link the representative repos** with a one-line "why this matters" each, so the index reads as a curated set, not a dump.
- Keep it to one screen. The index is a table of contents for a body of work, not a biography.

```bash
git init -b main
# write README.md per the rules above
git add README.md && git commit -m "Profile index"
gh repo create <account>/<account> --public --source=. --push
```

## Step 5 — Pins

Pin a small, ordered set — the representative repos first. Fewer, stronger pins beat a wall of them.

There is no REST endpoint for pins. Two options:

1. GraphQL (try first): query the repo node IDs, then `mutation { changeUserProfilePinnedItems(input: {itemIds: [...]}) }` — if the schema rejects it (this mutation has limited availability), fall back to 2.
2. Browser: "Customize your pins" → select the repos in order. If running without browser control, output this as a short manual step for a human, and say so honestly in the report — do not claim the pins are set when they are not.

## Step 6 — Report

Produce a summary containing: repos archived (count + names), repos left untouched pending a human decision, keeper descriptions/topics set, profile-index URL, pin status (done or manual step remaining), and confirmation that any off-limits repo was not modified. Be precise about what was actually done versus deferred.

## Acceptance checklist

- [ ] `gh auth status` = intended account before any mutation
- [ ] All known-stale repos archived; zero deletions
- [ ] Uncertain repos reported, not archived
- [ ] Profile-index repo live with a one-screen README, no employer names
- [ ] Pins set (or honestly flagged as the one manual step)
- [ ] Any deferred/off-limits repo byte-for-byte untouched
