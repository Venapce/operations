# Venapce Operations

Installable **operation packages** for the [Venapce](https://inflowenger.com) panel — expert-authored work you install and run against your own estate.

An *operation* is the unit you install: one or more [FloMorphic](https://inflowenger.com/flomorphic) workflows that together carry out one **mission**, plus everything your estate has to provide for them to run. Install one and it lands on the Operations page with its mission write-up, its flows in step order, and a readiness check per requirement. Run it and it writes rows into the **Stage → Findings → Issues** pipeline, where a person triages them.

These are not snippets to read and adapt — they are versioned packages with a manifest, operator params, declared requirements and an upgrade path. (For raw flows to learn from, see the [FloMorphic flow cookbook](https://github.com/FloMorphic/flow-cookbook).)

---

## Catalog

| Operation | Mission | Estate |
| --- | --- | --- |
| [`linux-fleet-http-audit/`](linux-fleet-http-audit/) | Who is serving HTTP on the fleet, where (host or Docker), and under which hostnames — three fleet-wide osquery sweeps, per-node probes for nginx config, TLS names and `/etc/hosts`, one evidence row per web server. | A few servers · Fleet · Small business |
| [`github-commit-security-review/`](github-commit-security-review/) | Clone a repository, have a Claude agent read the last N commits **in that checkout**, and stage one row per security finding — schema-bound, deduplicated, left at `pending` for a human to promote. | A few servers · Small business · Fleet |

[`catalog.json`](catalog.json) is the machine-readable index of the same list.

## Install one

In the panel: **Operations → Import**, then one of

- **this repository's URL** — `https://github.com/Venapce/operations` — to read `catalog.json` and pick from the list;
- **one package's folder URL** — e.g. `https://github.com/Venapce/operations/tree/main/linux-fleet-http-audit`;
- **a local folder**, picked in the file dialog (clone this repo first);
- **a pasted manifest**, which installs without the README, so the operation page has no mission write-up until you re-import from the folder.

Before anything reaches your server the import shows you what will and will not run: each requirement in `requires` as a readiness check against what the panel already knows about your osctrl and FloMorphic plugins, and every param with its default for you to set. An import of the same `id` at a higher `version` is an upgrade, and **Update** re-reads a package from wherever it came from.

### What a package needs from you

Each package's README has the authoritative list. In general:

- **osctrl** — nodes enrolled and tagged, for the operations that sweep a fleet.
- **FloMorphic plugins** — the named plugins installed, with the actions the flows call. Some need a host-side binary (`git`, Claude Code) or a connected account.
- **Settings profiles** — the exports carry **no credentials**. LLM and account nodes reference a profile *by id*, left blank on purpose, so a package can be published without leaking a key. Select your own profiles after import; the readiness list names each one.
- **Params** — the knobs that adapt a package to one organization (osctrl tags, environment name, repository owner, thresholds). Substituted into the flows at install time.

## Author one

One folder = one operation. The layout:

```
my-operation/
  operation.json     the manifest
  README.md          the mission, and one section per flow  (REQUIRED — the import refuses a package without it)
  flows/             FloMorphic workflow exports (flomorphic.kind = workflow)
  docs/              optional deeper write-ups a flow's `doc` points at
```

`operation.json` is validated against [`venapce-operation.schema.json`](https://github.com/Venapce/venapce-wapp/blob/main/public/schemas/venapce-operation.schema.json); point `$schema` at it for editor completion. Required: `schema`, `id`, `name`, `version`, `flows`. Worth filling in every time:

| Field | Why |
| --- | --- |
| `id` | Stable slug, unique in this catalog. An install is keyed by `<catalog>/<id>`, so never reuse or rename it. |
| `version` | Semver. A higher version of the same `id` is an upgrade. |
| `description` | One paragraph: what it finds out and where it writes it. |
| `flows[].role` | `entry` originates rows; `on-row` is picked from a row's **Run flow**; `helper` is called by another flow. |
| `flows[].step` | Position in the mission, so the page presents a multi-step operation in order. |
| `params` | Anything organization-specific. Mark secrets `secret: true`. |
| `requires` | `osctrl`, `plugins`, `settings`, other operations — each becomes a readiness check, so say what a flow actually calls. |
| `provides` | The tags the rows carry, so the panel can link to what the operation leaves behind. |
| `scale` | Which estate it's meant for — a catalog filter, not a gate. |

The panel can do the packing for you: **Operations → New from FloMorphic** picks flows from your canvas, infers the step shape, scaffolds the manifest and a README outline, and will either install it locally or **download the folder as a zip, ready to drop into this repository**.

Checklist before opening a PR:

- [ ] The README explains the mission as a whole, then each flow in step order — what it reads, what it writes, what to check afterwards.
- [ ] No credentials, tokens, account ids, internal hostnames or customer data in the flow exports or sample output.
- [ ] Every settings profile id left blank; every plugin and action the flows call declared in `requires`.
- [ ] Imported from a clean checkout into a panel at the `venapce.min` version, and run end to end.
- [ ] Added to [`catalog.json`](catalog.json) with a matching `id`, `path` and `version`.

## Contributing

Issues and pull requests welcome — a new operation, a fix to an existing one, or a sharper write-up. Bump `version` in both `operation.json` and `catalog.json` for any change to a package's flows or params.

## License

Each package carries its own `license` field; those published here are Apache-2.0 unless the package says otherwise.
