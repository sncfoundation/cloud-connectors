<p align="center">
  <img src="https://sncfoundation.github.io/logos/cloud.svg" width="96" alt="Cloud connectors logo">
</p>

<h1 align="center">Cloud connectors</h1>

<p align="center"><b>Run containers on managed cloud backends</b><br>
A <a href="https://sncfoundation.github.io">Sheet-Native Computing Foundation</a> project &#183; analog of <b>Cluster API</b></p>

---

**Status:** 📝 Unsaved Draft &#183; this project is a proposal. Design notes and contributions are welcome.

## About

Run containers on managed cloud backends. Part of the spreadsheet-native stack — the Sheet stays the source of truth, and it reconciles.

`connector` reads the desired **Deployments** from the Sheeternetes apiserver (a Google Apps Script web app in front of your Google Sheet) and reconciles them onto a pluggable cloud backend. Two backends ship today:

- **local-docker** — runs each deployment's image `replicas` times via `docker run` (fully working).
- **digitalocean** — reconciles onto [DigitalOcean App Platform](https://docs.digitalocean.com/reference/api/api-reference/#tag/Apps): one App per deployment, `replicas` mapped to the service `instance_count`.

## Sheet-Ops across clouds

The model is the same one Sheeternetes uses on-prem, stretched across cloud providers:

```
Google Sheet  ->  apiserver (Apps Script /exec)  ->  connector  ->  { local-docker | digitalocean | ... }
 (desired state)      (GET ?kind=deployments)         (reconcile)        (actual state)
```

You edit rows in the **Deployments** tab; `connector` computes a plan (CREATE / KEEP / RECREATE / DELETE) by diffing desired state against what the backend actually reports, then applies it. Each backend is just a pair of shell functions — a `_state` reader and an `_apply` executor — so adding a cloud is a small, self-contained change.

> **Caveat — serverless backends self-manage lifecycle.** On a managed/serverless backend (like DO App Platform) the platform owns process supervision, restarts and autoscaling once the desired spec is applied. The connector reconciles the **spec** (image, replica/instance count), not individual running processes. It also does **not** auto-delete managed cloud apps: a `DELETE` on the `digitalocean` backend is reported but not executed, to avoid a spreadsheet edit tearing down live cloud infrastructure by surprise. Remove those apps deliberately via the DO API or console.

## Usage

Requires `bash`, `curl`, `jq`. The `local-docker` backend also needs `docker`; the `digitalocean` backend needs a `DO_TOKEN`.

Configure the apiserver via environment variables, or a `.skctl.env` file next to the script (see `.skctl.env.example`):

```bash
export WEBAPP_URL="https://script.google.com/macros/s/XXXX/exec"
export TOKEN="CHANGE_ME_super_secret"   # defaults to CHANGE_ME_super_secret
```

Commands:

```bash
./connector list                 # show desired deployments from the apiserver
./connector plan  local-docker   # print what it would do on that backend
./connector apply local-docker   # reconcile onto that backend
./connector plan  digitalocean   # DO_TOKEN required for apply, not for plan
```

For the DigitalOcean backend, export a [DO API token](https://cloud.digitalocean.com/account/api/tokens):

```bash
export DO_TOKEN="dop_v1_..."
./connector apply digitalocean
```

### Testing offline

If the real apiserver isn't reachable, stub the GET with `SHEET_JSON` — either inline JSON or a path to a file. When set, the apiserver is not contacted:

```bash
SHEET_JSON='{"items":[{"name":"hello-web","image":"nginx:1.25","replicas":2}]}' \
  ./connector plan local-docker
```

## Get involved

- 📋 Tracking issue &amp; design: [sncfoundation/sheeternetes#15](https://github.com/sncfoundation/sheeternetes/issues/15)
- 🗺️ [SNCF Landscape](https://sncfoundation.github.io/landscape.html)
- 🧩 [All projects](https://sncfoundation.github.io/projects.html)
- ⚖️ [Governance &amp; how to contribute](https://github.com/sncfoundation/governance)
- 🎓 [Get certified (CSFE)](https://sncfoundation.github.io/certification.html)

## Status legend

Everything starts as an **Unsaved Draft**. It reconciles up the tiers from there — see the
[maturity model](https://sncfoundation.github.io/foundation.html#maturity).

---

<sub>Licensed under Apache-2.0. The SNCF does not recommend running production on a spreadsheet. If you do, please film it.</sub>
