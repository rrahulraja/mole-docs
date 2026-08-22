# Backend logging

Structured logging via **pino** (`backend/src/lib/logger.ts`).

## Usage

```ts
import { logger, log } from '../lib/logger';

logger.info({ bookingId }, 'payout accrued');
logger.error({ err }, 'payout failed');

// Component child logger — tags every line with { component }.
const walletLog = log('wallet');
walletLog.warn({ userId }, 'insufficient balance');
```

Inside a request handler prefer the request-scoped logger — it carries the
request id so all lines for one request correlate:

```ts
req.log.info({ educatorId }, 'accepted booking');
```

**Pass structured fields as the first arg (an object), the message second.** Don't
string-interpolate values into the message — `logger.info({ userId }, 'x')`, not
`logger.info(\`user \${userId}\`)`. Structured fields are queryable downstream.

## Levels

`LOG_LEVEL` env (default `info`): `fatal|error|warn|info|debug|trace|silent`.
Set `LOG_LEVEL=debug` locally for verbose output.

## Format

- **development** — pretty, colorized (via `pino-pretty`).
- **production / test** — one JSON object per line, ready to ship to Loki /
  SigNoz / CloudWatch / any log pipeline. Base fields `service` + `env` are on
  every line.

## HTTP requests

`pino-http` (wired in `app.ts`) logs one line per request with method, url,
status and duration. Each request gets an `x-request-id` (generated if the client
didn't send one, echoed back in the response header) exposed as `req.id` and
baked into `req.log`. `/health` is excluded to cut noise. 5xx logs at `error`,
4xx at `warn`.

## Redaction

The logger redacts secrets globally — auth headers, cookies, `set-cookie`, and
any `password` / `otp` / `accessToken` / `refreshToken` / `tempToken` field
(top-level or one deep) become `[REDACTED]`. This is the single guarantee that no
log leaks a credential; add new sensitive keys to the `redact.paths` list in
`logger.ts`.

## Migration

Existing `console.*` calls are being migrated to `logger` incrementally. Wired so
far: the request pipeline + error handler (`app.ts`), server startup
(`index.ts`), and the cron runner (`crons/index.ts`). When you touch a file with
`console.log/error/warn`, swap it for `logger` (or a `log('component')` child).

## Observability (future)

JSON logs are the first pillar. For traces + metrics, the intended path is
**OpenTelemetry** instrumentation exporting to an OSS backend (SigNoz or the
Grafana LGTM stack), with **GlitchTip / Sentry** for error tracking across
backend + web + mobile. Not wired yet — see the PR discussion.
