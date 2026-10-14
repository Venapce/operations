# Commit security review — how the flow is built

One pass over the twelve nodes, in the order the run visits them, with the reason
each one exists. Read this before editing the flow: three of the shapes here look
like they could be simplified and cannot.

## 1. Seed the scan parameters — `js`, key `scan`

Holds the package's five variables with their defaults, then lets the context
override them:

```js
for (let k in given) { if (v !== null && v !== undefined && v !== '') { scan[k] = v } }
```

That direction matters. The install-time `${param.x}` substitution bakes the
operator's answers into *this* node, but a run reads its variables from the
**context** — so an operator can point the same installed flow at another
repository by editing `$.scan` alone, and the defaults are only a fallback.

It also derives what the rest of the flow needs rather than making later nodes
recompute it: `depth` (= `commits + 1`, so the oldest reviewed commit still has a
parent to diff against), `destination` (a path under the clone root, namespaced by
owner so two repos of the same name cannot collide) and `target` (`owner/repo`).

And it normalizes `repo`, because the GitHub actions want a bare repository name
and an operator pastes a URL. `parseRepo` accepts a bare name, `owner/name`, an
https URL, an scp-style ssh remote (`git@github.com:owner/name.git`) and even a
link to a file inside the repository, reducing all of them to an owner and a name;
the original value is kept on `scan.repoInput`. An owner found inside the value
overrides the `owner` param, since it is the more specific statement of intent.
This was added after the first operator to install the package pasted a URL — the
clone then asked for a repository it could not find, which is a confusing way to
learn the field wanted one word.

## 2 & 3. Clone the repository · List the recent commits — `plugin`, parallel

Neither reads the other's output, so they are two edges off the seed node rather
than a chain. `github.repo.clone` runs git on the plugin's host and reports
`path`, `commit` and `branch`; `github.activity.list` asks GitHub for its own
record of the commits — author, date, URL — which a local clone cannot prove.

Neither node names an owner. Both take a repository name only; the owner comes
from the bound account profile. That is why `owner` is a param *and* a profile
setting, and why they have to agree.

The commit list arrives wrapped — `$.commits.data.commits[]`. The normalize step
does not depend on that shape (see below), so a gateway that nests it differently
will not break the flow.

## 4. Wait for the checkout and the log — `promissall`

Edges into a node do not merge: without this barrier the review node would have
two inbound edges and run **twice**, in parallel, each copy clone-unaware of the
other. The barrier is the only thing in FloMorphic that joins branches.

## 5. Review the commits with Claude — `plugin`, key `review`

```
workspace: {{$.clone.path}}
```

The whole point of cloning first. The agent works in the directory the clone node
actually produced, so it reads the real tree with real git history — `git log`,
`git show`, `Grep` over the surrounding files.

Four choices worth keeping:

- **`structured: true` with a schema.** The answer lands on `$.review.output` as
  fields, so everything downstream reads values instead of parsing prose.
- **`reviewedCommits` is `required` in that schema**, and each entry must carry
  the paths the commit touched. This is the fix for the failure mode this node had
  on its first live run: given only "review the last 3 commits", the agent ran one
  `git show HEAD~2 HEAD~1 HEAD | head -600`, thought briefly, and returned
  `clean` in 8 seconds. Requiring it to enumerate what it read — and telling it in
  the prompt to read each commit separately and never through `head` — turned that
  into ten turns, three commits read in full, and a finding with evidence. The
  schema forces the *work*, not the conclusion.
- **A read-only tool set** (`Read`, `Grep`, `Glob`, `Bash`) with `Write`/`Edit`
  explicitly blocked, and `permissionMode: bypassPermissions`. Nothing can be
  asked of a human mid-flow, and there is nothing to approve: the worst the agent
  can do is read a throwaway clone.
- **`askHuman: false`.** The node's human port exists for interactive flows; a
  scheduled security sweep must not park waiting for someone.

Its `Failed` and `Limit reached` ports both go to one note node. The importer
warns that two branches arriving at one node make it run twice — true in general,
but these ports are mutually exclusive, so at most one edge is ever followed.

## 6. Did the review find anything? — `rule`

A clean review is an outcome, not a failure. Without this branch a run with no
findings would fall into the writer and stage nothing while looking like it
half-worked. `found` / `clean`, exhaustive on both paths.

## 7. Normalize the findings — `js`, key `findings`

Stage's tidy-up step — the part that makes the inbox worth reading:

- severity coerced into one vocabulary (`info`…`critical`), anything unknown → `info`;
- the **severity floor** applied, so below-threshold noise never reaches the table;
- a stable `fingerprint` per finding, and a second finding with the same one dropped;
- GitHub's commit record joined onto each finding by sha;
- findings sorted most severe first.

The commit join walks `$.commits` generically — it collects any object carrying a
`sha`, at any depth — rather than reaching into `data.commits`. That is deliberate:
the gateway's envelope is not part of this flow's contract.

It then prepares what the mapping node lifts: `stageTitle`, `stageRef`,
`stageData` and `stageMeta` per finding. A Cast cannot compute, so anything that
needs computing is done here and the mapping below stays a mapping.

## 8. Map each finding onto the Stage schema — `cast`, scope `$.findings[*]`

**The mapping node.** One Stage row per finding, and one row of the drawer per
Stage column:

| Column | Mode | Value |
|---|---|---|
| `title` | jsonpath | `{{$this.stageTitle}}` |
| `summary` | jsonpath | `{{$this.summary}}` |
| `source` | static | `github` |
| `origin` | static | `flow:github-commit-security-review` |
| `disposition` | static | `pending` |
| `ref` | jsonpath | `{{$this.stageRef}}` |
| `data` | jsonpath | `{{$this.stageData}}` |
| `meta` | jsonpath | `{{$this.stageMeta}}` |

When the Stage schema changes, this is the only node to edit — and it is edited
in the drawer, not in code.

A Cast lifts a path or writes a literal and **computes nothing**. That is the
division of labour with node 7: everything needing computation — the clamped
title, the fingerprint, the model list, the three JSONB objects — is prepared
there as `stageTitle` / `stageRef` / `stageData` / `stageMeta`, and this node only
maps. A whole-token value (`{{$this.stageRef}}`) is lifted **typed**, so `ref`
arrives as an object, not as a string of JSON.

Scope `$.findings[*]` runs it once per finding, and its `key` writes **into that
element** (`$.findings[0].stageRow`) — not into a top-level array. That
per-element write is what pairs it with the per-element write in node 9. A Cast
has no output ports of its own, which is why it is allowed a many-scope at all.

Two details the live data forced on node 7, which this node depends on:

- The title is clamped to 96 characters before `[severity]` and `repo@sha` are
  added, because a model writes a whole sentence as a title and Stage titles are
  read in a list. The model's own wording stays in full on `data.title`.
- `ref.models` comes from the usage table, not from `review.model`. That field is
  whichever model the *last* message used, and Claude Code bills small side-calls
  to a cheaper one — so a row reviewed by Sonnet would otherwise claim it was
  reviewed by Haiku. Provenance in a security table has to be right.

The node needs **no document store**: its keys are Stage's columns, not a store
schema, so leave Store empty in the drawer.

## 9. Stage each finding — `plugin` `db.stages.upsert`, scope `$.findings[*]`

Reads `{{$this.stageRow.*}}` — the element this pass is standing on. `ref`, `data`
and `meta` are passed as whole-object tokens: the venapce plugin inlines the JSON
at that path and binds it as `jsonb`. `tags` has to be one row per tag (the plugin
does not parse a JSON array out of a single row), so the tag vocabulary is written
there as rows, two of them per-finding tokens.

Every row is written with `disposition: pending`. This flow never promotes
anything — a person does that from the panel.

## 10 & 11. Record a clean review · Note the unfinished review — `js`

Both exist so a run that produced no table rows still says why on its context:
`reviewSummary` for an honest clean verdict, `reviewProblem` for an agent that hit
its turn or budget ceiling or failed outright, with the session id to resume from.

## What a run leaves behind

```
$.scan           the variables, resolved
$.clone          path, commit, branch, permission
$.commits        GitHub's commit record
$.review         state, output (verdict / findings / reviewedCommits), turns, costUsd, usage
$.findings[]     normalized, each with .stageRow (mapped) and .stageWrite (the row as the DB returned it)
$.reviewSummary  only on the clean path
$.reviewProblem  only on the failed / limit path
```

A context is reused across runs and nodes never clear each other's keys, so a
`reviewSummary` from an earlier clean run is still there after a later run that
found something. Read `$.review.output.verdict` for what *this* run concluded, not
the presence of a key — or start each run on a fresh context.
