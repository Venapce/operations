# GitHub commit security review

Clone a repository onto the plugin host, have a Claude agent read the last N
commits **in that checkout**, and write one **Stage** row per security finding.

Nothing is promoted automatically. Every row lands at `disposition: pending` with
its evidence and its provenance attached, and a person decides from the venapce
panel whether it becomes a Finding, an Issue, or nothing at all.

## What it does

```
          ┌─ Clone the repository ─────┐
Start ──▶ │  (git, on the plugin host) │──▶ Wait ──▶ Review the commits ──▶ findings? ──▶ normalize ──▶ map ──▶ Stage
 seed     └─ List the recent commits ──┘   for all     with Claude          │                                  (1 row each)
 params      (GitHub's own record)                     (schema-bound)       └─ clean ──▶ record the review, write nothing
```

The clone and the commit list do not depend on each other, so they run in
parallel and join at a Wait for All. The agent's workspace is
`{{$.clone.path}}` — whatever directory the clone node actually produced — so the
review reads the real tree rather than a payload passed between nodes.

The agent answers against a JSON Schema, so the rest of the flow reads fields
instead of parsing prose. It must also return `reviewedCommits`: every commit it
read, with the paths each one touched. That list is the evidence the review
happened, and it is what makes a `clean` verdict worth anything.

## Parameters

| Param | Default | What it is |
|---|---|---|
| `owner` | — (required) | The GitHub organization or owner. **Must match** the organization on the account profile you bind the GitHub nodes to. |
| `repo` | — (required) | The repository: a bare name, `owner/name`, or any GitHub URL (ssh remotes and links to a file inside the repo included). An owner found in the value wins over `owner`. |
| `commits` | `3` | How many commits back to read. The clone goes one deeper, so the oldest reviewed commit still has a parent to diff against. |
| `ref` | `""` | Branch, tag or commit. Empty means the default branch. |
| `severityFloor` | `low` | Findings below this severity never reach Stage. |

The params are substituted into the flow at install time. After that they live on
the run's **context**, under `$.scan` — so you can point the same installed flow
at another repository by editing the context, without reinstalling. The seed node
takes whatever the context already carries in preference to its own defaults,
which is what makes that work.

## Before it will run

1. **Connect a GitHub account** in FloMorphic → Connect, and bind a profile whose
   **organization is your `owner`** to both GitHub nodes. The package ships no
   profile ids — they are per-install — so a freshly installed flow has those nodes
   unbound, and a run fails with *"no organization / owner set"* until you bind
   them in the drawer. That is also the readiness check the operation page shows as
   `unknown`. The clone and
   commit-list actions take a repository name only; the owner comes from that
   profile, never from the flow.
2. **Bind a Claude profile** whose **workspace root contains the GitHub plugin's
   clone root** — `GITHUB_OC_CLONE_ROOT`, or `./workspace` beside that plugin.
   The agent refuses a workspace outside its root before it starts, so a root
   that does not contain the clone fails every run.
3. **Both plugins must run on the same host**, or share that directory. The clone
   is a path on a disk, not something passed between nodes.
4. **The Cast / Mapping builtin must be running.** The mapping node is a Cast, and
   a Cast whose plugin is down fails with *"the plugin is not running"* — including
   when the process is up but was started with a stale credential, which looks
   identical from the flow's side. Its log line `Subscribed Action :
   inflow.cpu.…-cast.run` with no following authorization error is the
   confirmation. Leave its **Store** empty: the keys it maps are Stage's columns,
   not a document-store schema.

> **Re-installing the flow clears those bindings.** `POST
> /operations/:id/flows/:key/install` re-imports the package file over the bound
> flow, and the package carries no profile ids — so upgrading the operation, or
> re-installing it for any reason, leaves the GitHub and Claude nodes unbound and
> the next run fails with *"no organization / owner set"*. Re-bind them in the
> drawer after every install.

Keep the Claude root at the clone root rather than widening it. The root is
organisation, not a sandbox — Claude Code resolves its project from the git root
above `cwd` — so a wide root means the agent can read whatever else lives in that
tree, including other plugins' `.env.inflow`.

## What a Stage row carries

| Column | Value |
|---|---|
| `title` | `[severity] finding title — owner/repo@abc1234` |
| `summary` | What is wrong and what it lets an attacker do |
| `source` | `github` — where the data came from |
| `origin` | `flow:github-commit-security-review` — the process that produced the row |
| `disposition` | `pending` |
| `tags` | `github`, `code-review`, `claude-agent`, `repo:<name>`, `severity:<sev>`, `category:<cat>` |
| `ref` | Provenance: the agent's model and session id, the repo, the commit and GitHub's record of it, the fingerprint, the workspace |
| `data` | The evidence: file, line, snippet, remediation, confidence, category |
| `meta` | Enrichment: `falsePositiveRisk`, severity rank, the floor applied, and the review's cost and turn count |

`ref.fingerprint` is `github:<owner>/<repo>:<file>:<line>:<slug of title>`. It is
unique **within** a run — the normalize step drops a second finding carrying the
same one — but it is only as stable **across** runs as the model's own wording:
two runs over the same commits reported the same issue at line 47 and then line
48, with slightly different titles, which are two different fingerprints. Treat it
as a strong hint for recognising a repeat, not as a primary key. A stricter key
would have to be built from the file and the commit alone, at the cost of
collapsing two genuinely different findings in one file.

**It does not deduplicate across runs.** `db.stages.upsert` updates in place only
when it is given an `id`, and the venapce plugin exposes no read action a flow
could use to look a fingerprint up first — so running this flow twice over the
same commits stages the same finding twice. The fingerprint is what makes those
repeats *recognisable* (in a saved view, in a query, or by whoever triages them);
closing the loop needs either a read action on the venapce plugin or a promotion
flow that collapses them.

## False positives

The agent is told to be strict, and that a human triages what it reports. Where
it is unsure it must either leave the finding out or mark
`meta.falsePositiveRisk: high` and say in `data.evidence` why the thing may be
benign. Filter or sort on that field in the panel before promoting anything.

The severity floor is the blunter control: raise it to `medium` and info/low
observations never reach the inbox at all.

**The verdict varies between runs on borderline findings.** Over three runs of the
same three commits — each reading all three in full — two reported one
low-severity issue (its own `falsePositiveRisk: high`) and one called the same
commits clean. That is the honest behaviour of a judgement call near the
threshold, not a bug — and it is the reason `confidence` and `falsePositiveRisk`
are on every row and a person promotes. Widening the window makes it steadier:
eight commits produced three findings, including a medium one.

If you need repeatability more than breadth, raise the floor so only findings the
model is confident about survive, and schedule the flow over a fixed window rather
than "the last N" so two runs read the same commits.
