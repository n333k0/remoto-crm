# Vercel deployment

The CRM uses separate Vercel projects for the web app, API, and research agent.

## Projects

Create these projects from the same repository:

| Project | Root directory | Runtime |
| --- | --- | --- |
| `remoto-crm-app` | `apps/app` | Next.js |
| `remoto-crm-api` | `apps/api` | Node.js serverless function |
| `remoto-crm-agent` | `apps/agent` | eve deployment |

The Next.js app proxies `/api/*` to the API project, so browsers use the web app origin for auth and tRPC. The API project uses the committed Vercel Build Output configuration produced by `bun run build:vercel`.

The local API command remains Bun-based. Vercel uses `apps/api/api/index.ts`, which initializes Nest once per warm function and passes requests to the Express adapter. It does not run `src/main.ts` or open a long-lived server.

## API environment

Set these on the API project:

- `DATABASE_URL`: pooled Postgres URL for runtime queries
- `DIRECT_DATABASE_URL`: direct, unpooled Postgres URL used by build-time migrations
- `BETTER_AUTH_SECRET`: the same strong secret used by the app project
- `ALLOWED_SIGN_IN`: the permitted email domains and addresses
- `GOOGLE_CLIENT_ID`
- `GOOGLE_CLIENT_SECRET`
- `API_URL`: the public API project URL
- `APP_URL`: the public web app URL
- `CRON_SECRET`: at least 16 characters, for the Google sync cron
- `REDIS_URL`: recommended for shared production cache
- `BLOB_READ_WRITE_TOKEN`: optional, for mirrored images
- `AGENT_URL`: the public agent project URL, when the bridge is enabled
- `AGENT_BRIDGE_SECRET`: the same value used by the app and agent, when the bridge is enabled

`POSTGRES_URL_NON_POOLING` and `DATABASE_URL_UNPOOLED` are accepted as fallbacks for `DIRECT_DATABASE_URL` by the API build script.

## App environment

Set these on the web app project:

- `API_URL`: the public API project URL; this is inlined into the Next build and used for server-side proxying
- `APP_URL`: the public web app URL
- `DATABASE_URL`: the same database used by the API, because server components read sessions directly
- `BETTER_AUTH_SECRET`: exactly the same value used by the API
- `AGENT_URL` and `AGENT_BRIDGE_SECRET` when the agent tab is enabled
- `AUTH_COOKIE_DOMAIN` only when using separate subdomains of one parent domain

The app project does not need the Google client secret because Google OAuth is owned by the API, but keeping the variables synchronized is acceptable if the platform requires a shared environment set.

## Database migrations

The API build applies `prisma migrate deploy` when `VERCEL=1` and a direct database URL is available. Use a direct, unpooled URL for `DIRECT_DATABASE_URL`; never use a pooler for migrations. Do not run `db:migrate`, `db:push`, `db:reset`, or `db:seed` against production from a laptop.

## Google OAuth

For a production API URL of `https://api.example.com`, add this exact callback to the Google OAuth web client:

```text
https://api.example.com/api/auth/callback/google
```

Replace `api.example.com` with the actual API project domain. If the app uses the Vercel default domain, use that exact domain. Add every production, preview, or custom API domain that will actually be used. Gmail API and Google Calendar API must be enabled, and the OAuth consent screen must permit the configured sign-in accounts.

## Verification

From the repository root:

```sh
bun install --frozen-lockfile
bun run build
bun run check-types
bun run lint
```

To validate the API Vercel artifact without deploying:

```sh
bun run --cwd apps/api build:vercel
```

This writes only ignored `.vercel/output` artifacts. It does not deploy or push anything.
