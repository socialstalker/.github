# SocialStalker

A social media archiving toolkit: a browser extension that downloads and unlocks
media on Instagram and Facebook, a backend that records what it did, and a
dashboard for looking at those records.

Three repositories, one system.

| Repository | Stack | Role |
|---|---|---|
| [extension](https://github.com/socialstalker/extension) | TypeScript, React, Vite, Manifest V3 | Runs in the user's browser. Full-size profile pictures, story/post/reel/feed downloads, whole-profile archiving, seen-state blocking. |
| [backend](https://github.com/socialstalker/backend) | NestJS, Drizzle, Postgres, BullMQ, Redis | Ingest API for the extension, admin API for the dashboard, and a worker that normalises events and mirrors media. |
| [dashboard](https://github.com/socialstalker/dashboard) | Next.js 16, React 19, Tailwind | Read-only admin UI over the backend's `/admin` routes. |

## How a request flows

```
                    ┌───────────────────────────────┐
   user's browser   │  extension                    │
                    │  content scripts + MV3 worker │
                    └───────────────┬───────────────┘
                                    │ POST /v1/installs/enroll
                                    │ POST /v1/events
                                    │ POST /v1/heartbeat
                                    ▼
                    ┌───────────────────────────────┐
                    │  backend — ingest API         │
                    │  install token guard          │
                    └───────┬───────────────┬───────┘
                            │               │
                   Postgres │               │ Redis (BullMQ)
                 partitioned│               │
                            ▼               ▼
                    ┌───────────────────────────────┐
                    │  backend — worker             │
                    │  normalise, mirror media,     │
                    │  build archives, geoip        │
                    └───────────────┬───────────────┘
                                    │
                                    ▼
                            object storage (S3)


                    ┌───────────────────────────────┐
   an operator ────▶│  Authentik forward-auth proxy │
                    └───────────────┬───────────────┘
                                    │ identity headers
                                    ▼
                    ┌───────────────────────────────┐
                    │  dashboard (server-side only) │
                    └───────────────┬───────────────┘
                                    │ + x-socialstalker-proxy
                                    ▼
                    ┌───────────────────────────────┐
                    │  backend — /admin             │
                    │  audited against the operator │
                    └───────────────────────────────┘
```

Two things about that lower path are deliberate. The browser never reaches the
admin API — every dashboard request originates server-side, so no credential
capable of reaching `/admin` is ever handed to a client. And the operator's
identity is forwarded rather than replaced, so the audit log names the person who
clicked instead of naming the dashboard.

## Operations

Deployed on **Dokploy**: the backend image runs twice, once as the API and once
as the worker, with different start commands. Postgres and Redis are
Dokploy-managed resources in the same project.

Secrets live in **Infisical** and are fetched at boot by a machine identity —
nothing reads a `.env` in production. Dokploy's own env tabs carry only the
per-application operational keys and the identity coordinates.

Errors from the extension go to **Sentry**, with platform credentials scrubbed by
an allowlist before anything leaves the browser.
