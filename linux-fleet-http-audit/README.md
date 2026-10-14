# Linux fleet HTTP audit

## Mission

Ask a whole Linux fleet **"who is serving HTTP, where, and under which names?"** and get one evidence row per node in Stage — the starting material for an exposure review: which hosts expose a web server at all, whether it runs on the host or in Docker, and which hostnames it answers to.

## Flows

### Step 1 — Linux fleet — HTTP / nginx + served hostnames audit (`http-audit`, entry)

Three fleet-wide osquery sweeps run in parallel (TCP listeners + owning process, web-server processes, Docker containers + published ports), a JS node folds them into a per-node inventory, a rule branches on whether anything is serving at all, four more probes fan out once per node that is (nginx config files, `server_name` via augeas, TLS cert CN + SANs, OS + `/etc/hosts` names), and a final JS node assembles a report that `db.stages.upsert` writes as one Stage row per host — tagged `linux / http-audit / nginx`, with the full evidence in `data`. Details: [docs/http-audit.md](docs/http-audit.md).

## What it needs

- The **venapce** plugin installed on FloMorphic and an osctrl it can reach (your own, or a managed space from Settings → FloMorphic).
- Linux nodes enrolled and carrying the tags you set in the **Fleet tags** parameter.

## After install

Set `env` and `fleetTags` to the environment and tags that select *your* Linux nodes, then press **Run** on step 1. Open Stage afterwards: one new row per host that served HTTP.

Recipe write-up, topology diagram and the debugging story: <https://github.com/FloMorphic/flow-cookbook/tree/main/linux-fleet-http-audit>.
