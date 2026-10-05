# Mole Docs

Documentation for **Mole** — a live 1:1 tutoring marketplace (students book
real-time video sessions with educators). This repo holds the docs that used to
live under `docs/` and `backend/docs/` in the [main monorepo](https://github.com/rrahulraja/mole).

> Code, per-package `README.md` files, and `CLAUDE.md` stay in the main repo.
> Only prose documentation lives here.

## Infrastructure

- [LiveKit self-hosting](infrastructure/livekit-self-hosting.md) — features, cost
  (500 sessions/mo), where to host cheaper than AWS, and the SFU + egress deploy guide.

## Deployment

- [AWS deploy](deployment/aws-deploy.md)
- [AWS EC2 runbook](deployment/aws-ec2-runbook.md)
- [AWS ECS runbook](deployment/aws-ecs-runbook.md)
- [Free deploy](deployment/free-deploy.md)
- [Local run — pm2 + mobile](deployment/local-run-pm2-and-mobile.md)
- [Local testing](deployment/local-testing.md)
- [Integrations setup](deployment/integrations-setup.md)
- [Integrations E2E](deployment/integrations-e2e.md)
- [Google Play Console setup](deployment/google-play-console-setup.md) — step-by-step new account → production
- [Google Play release plan](deployment/google-play-release-plan.md) — personal account → internal testing → org transfer; versioning + release notes

## Backend

- [Business flows](backend/business-flows.md)
- [Schema](backend/schema.md)
- [Requirements](backend/requirements.md)
- [E2E architecture](backend/e2e-architecture.md)
- [Deployment guide](backend/deployment-guide.md)
- [Crons](backend/crons.md)
- [FCM testing](backend/fcm-testing.md)
- [Backend logging](backend-logging.md)

## Mobile

- [Mobile OTA updates](mobile-ota-updates.md)

## Releases

- [Android v1.0.0](releases/android-v1.0.0.md) — release notes (draft)
- [Play Store listing](releases/play-store/listing.md) — name, descriptions, icon, feature graphic, screenshot plan, store settings

## Test cases

- [Overview](test-cases/README.md) · [backend API](test-cases/backend-api.md) ·
  [web](test-cases/web.md) · [mobile](test-cases/mobile.md) ·
  [admin](test-cases/admin.md) · [E2E](test-cases/e2e.md)

## Design docs (superpowers)

Plans and specs for shipped epics — see [`superpowers/plans`](superpowers/plans)
and [`superpowers/specs`](superpowers/specs): session quiz, live-now booking,
slot picker, pricing, languages + location.
