# Registry Migration — Container Registry Rollover

> Migrating a container registry before the lights go out.

## Problem

The self-hosted registry at `reg.mini.dev` was being shut down. 46 live deployments across the infrastructure pulled images from it. When it went offline, any container restart would fail to pull its image — services would go down one by one.

## What I Built

`registry-rollover.py` — a Python script that inventories every running deployment, identifies the unique images they pull, migrates those images to new registries, and rolls deployments over to the new image references.

### How It Works

1. **Inventory** — scans all Docker Compose stacks across the fleet, finds every image reference pointing at `reg.mini.dev`
2. **Deduplicate** — 46 deployments used only 7 unique images
3. **Mirror** — pulls each image from the old registry and pushes to both targets:
   - **DOCR** (DigitalOcean Container Registry) — immediate target
   - **Harbor** at `registry.weown.tools` — long-term self-hosted registry
4. **Rollover** — updates each deployment's `docker-compose.yml` to reference the new registry, pulls the new image, and restarts the stack
5. **Verify** — confirms each container is running with the new image

### Rollover Results

- 12 of 13 live deployments rolled over successfully
- 1 deployment skipped (required manual intervention due to custom entrypoint)
- All rolled-over services verified healthy

## Architecture

```
registry-rollover.py
         │
         ├── Scan fleet ──► 46 deployments found
         │                   7 unique images
         │
         ├── Mirror images
         │       │
         │       ├──► DOCR (registry.digitalocean.com)
         │       └──► Harbor (registry.weown.tools)
         │
         └── Roll over deployments
                 │
                 ├── Update docker-compose.yml refs
                 ├── docker compose pull
                 └── docker compose up -d
                         │
                    12/13 deployments ✓
```

## Decisions

| Decision | Why |
|---|---|
| Python over Bash | Needed structured data handling for 46 deployments; YAML parsing, deduplication, error tracking |
| Dual-target (DOCR + Harbor) | DOCR for immediate reliability; Harbor for long-term self-hosted control |
| Deduplicate before mirroring | 46 deployments but only 7 unique images — mirror 7, not 46 |
| Skip-and-report over force-all | One deployment had a custom entrypoint; safer to skip and flag than force |

## Outcome

46 deployments inventoried, 7 images migrated, 12/13 live deployments rolled over before the old registry went dark. Zero unplanned downtime.

## Stack

`Python` · `Docker` · `DOCR` · `Harbor` · `Docker Compose`

## Context

Built as Platform Engineer at [WeOwn](https://weown.net) (2026). Source code is proprietary.
