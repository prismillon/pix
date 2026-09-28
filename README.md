# pix

Self-hosted Instagram embed fixer that serves **https://px.prr.sh/** — the host Discord links get rewritten
to so posts unfurl with real images/videos instead of a login wall.

Deployed on the `neptune` Talos cluster; the manifests live in [prismillon/ops](https://github.com/prismillon/ops)
under `apps/pix/`.

## Where the code comes from

Nothing is vendored in this repo. The workflow checks out the upstream project at a **pinned commit** and builds the
image from it, so the source of truth stays upstream and every rebuild is reproducible:

- Upstream: [Lainmode/InstagramEmbed-vxinstagram](https://github.com/Lainmode/InstagramEmbed-vxinstagram) — this is
  the software behind `vxinstagram.com`, MIT licensed, by Lainmode.
- Scraping backend: [ahmedrangel/snapsave-media-downloader](https://github.com/ahmedrangel/snapsave-media-downloader),
  bundled into the image and run as a child process on port 3200. No API keys, no third-party embed host.
- Pinned at `UPSTREAM_SHA` in `.github/workflows/build.yml`.

To update: review the upstream diff, bump `UPSTREAM_SHA`, push. To move to our own fork later, point
`UPSTREAM_REPOSITORY` at it — no other change needed.

## Image

`ghcr.io/prismillon/pix:latest` (and a `:<upstream sha>` tag per build), published with the workflow's own
`GITHUB_TOKEN`, so this repo carries no secrets. Built for `linux/amd64`, weekly on a schedule.

Ops pins the image by digest; Renovate opens the bump PRs like it does for every other app in that repo.

## Runtime

Single container, serves `:8080` (`ASPNETCORE_URLS`). Routes: `/p/{id}`, `/reel/{id}`, `/stories/...`, media
proxying under `/offload/{id}`, plus `/oembed`. Readiness/Liveness probe `/`.

Why self-hosted: the public instances of this class of service (vxinstagram, xnstagram, kkinstagram, ddinstagram)
are shared-capacity and keep dying or running out of quota — one verified instance was answering crawlers with
`Error: Out of requests`. The container is cheap and removes that dependency.
