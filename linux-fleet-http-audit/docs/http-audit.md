# Flow: Linux fleet — HTTP / nginx + served hostnames audit

**Role:** entry (step 1). Starts from an empty context plus the package params; writes one Stage row per host that serves HTTP.

## Topology

```
Start
  ├─▶ Fleet: TCP listeners + owning process      osquery.queryByTags   key: listeners
  ├─▶ Fleet: web-server processes on host        osquery.queryByTags   key: web_processes
  └─▶ Fleet: docker containers + published ports osquery.queryByTags   key: docker_web
          ↓ promissall
      Build per-node inventory                   js                    key: inventory
          ↓
      Any node serving HTTP?                     rule   with_http | no_http
          ├─ no_http ──▶ Nothing found — note it js
          └─ with_http ─▶ four probes, scope $.inventory.http_targets[*]
                            nginx config · server_name (augeas) · TLS CN+SANs · /etc/hosts
                          ↓ promissall
                          Assemble audit report  js                    key: report
                          ↓
                          Record one stage row per node   db.stages.upsert   scope $.report.nodes[*]
```

## What it reads

- `${param.env}` — the osctrl environment every query targets.
- `${param.fleetTags}` — the tags the three sweeps select nodes by.

## What it writes

One Stage row per host with `title: HTTP/nginx audit — <hostname>`, tags `linux / http-audit / nginx`, `disposition` = where nginx runs (host / docker / none) and the full evidence in `data` (listeners, processes, containers, hostname bags with their sources).

## After import, review

- The three sweep nodes: confirm the tags select your Linux nodes.
- The plugin nodes: pick the venapce settings profile if your install requires one.
