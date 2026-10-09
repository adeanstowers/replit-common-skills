---
name: Replit Local Development
description: Sets up this replit so that you can work on it locally and update through git
---

---
name: local-linux-development
description: Add a reproducible local Linux development environment to an existing Replit app using Docker Compose, an optional VS Code Dev Container, isolated local data, and a simple guarded test login when hosted authentication cannot run locally. Use when asked to develop a Replit project locally, collaborate through Git, or reproduce this local-development setup in another repository. Preserve hosted behavior and existing Git remotes.
---

# Local Linux Development for an Existing Replit App

## Goal

Let the user clone an existing app, edit it with any Linux editor, run it locally,
and exchange changes with Replit through Git. Docker Compose is the baseline;
VS Code Dev Containers are optional. Do not require an identity server merely
to test locally. Do not change how the hosted app authenticates or stores data.

This is an adaptation procedure, not a project scaffold. Inspect the target
repository and use its actual services, package names, schema tools, and ports.
Do not copy another app's database name, role model, provider subject, or URLs.

## 1. Inspect and resolve consequential choices

- Read repository instructions, current Git status, package manifests,
  lockfiles, workspace configuration, service definitions, and existing local
  tooling. Preserve unrelated changes.
- Identify frontend/backend entry points, routing/base paths, database engine,
  migration commands, authentication, sessions, authorization, and external APIs.
- Check the existing Git remote without exposing embedded credentials. Keep the
  current provider and remote; don't create a new repository or assume visibility.
- Check supported runtime/package-manager versions and native dependency
  architectures. Select explicit compatible image versions, not `latest`.
- Follow applicable platform, package, authentication, web-app, and secrets
  guidance available in the target environment.
- If local authentication is unresolved, consult current provider documentation.
  Prefer supported localhost OIDC with a separate development registration.
  Do not claim a provider cannot support localhost just because no documented
  setup was found.
- If supported local OIDC is unavailable or disproportionate, propose a fixed
  synthetic test identity with a helper-generated sign-in link. Obtain approval
  unless the user already requested this fallback. Do not introduce Keycloak or
  another identity service without a clear need.
- Ask only about choices that materially change the result. Do not ask for
  secrets in chat; use the environment's supported secrets workflow.

## 2. Add local infrastructure without changing hosted services

Adapt these deliverables to the repository:

| File | Purpose |
| --- | --- |
| `Dockerfile` | Development image with compatible pinned runtime and package manager |
| `compose.yaml` | Local app services and isolated data service(s) |
| `.devcontainer/devcontainer.json` | Optional editor attachment using the same setup |
| `.env.example` | Safe placeholders and explanation of optional local credentials |
| `.dockerignore`, `.gitignore` | Exclude local dependencies, credentials, and private data |
| `scripts/local-dev.sh` | Guarded setup/start/link/check/logs/stop commands |
| Local watcher, if needed | Reliable Linux source reload without hosted-workflow changes |
| Local authentication modules/tests | Only when the approved fallback is needed |
| `docs/local-development.md` | Setup, routine commands, limits, and recovery |
| `docs/git-collaboration.md` | Branch/review/handoff workflow |

Container requirements:

- Publish the browser-facing service only on `127.0.0.1`; proxy API requests
  internally. Do not publish the database or internal API unless explicitly
  needed, and then bind only to loopback.
- Use a dedicated local database name/user and named data volume. Use the
  existing engine where practical; do not replace the hosted database.
- Give the database a private internal network. Allow application egress only
  where needed for dependency downloads or intentional external API calls.
- Never copy hosted database contents or credentials into the setup by default.
  Trust authentication, if used for disposable PostgreSQL, must stay confined
  to the private local network with no published database port.
- Bind-mount source and isolate container dependencies in named volumes,
  including every workspace package with its own dependency directory.
- Install from the committed lockfile. Don't silently regenerate it to make
  a container build pass.
- Run app processes as a non-root user matching the Linux host UID/GID.
  A short root initialization step can prepare dependency-volume ownership;
  don't recursively chown the source checkout.
- Reuse the helper-built image for Dev Containers. Avoid editor startup
  rebuilding it with default IDs that differ from the host user's IDs.
- Preserve native architecture support where possible. If the lockfile only
  supports x64, state that constraint and any emulation requirement explicitly;
  don't silently force amd64 for every project.
- Preserve hosted ports, base-path routing, artifact configuration, and run
  commands. Gate local-only Vite proxy/plugin changes behind an explicit flag.
- Use finite health-check timeouts and service health dependencies.

Git and image exclusions must cover:

```gitignore
# Adapt to existing rules; preserve a committable example environment file.
.env
.env.*
!.env.example
/.pnpm-store/
node_modules/
exports/
local-data/
```

Also exclude actual database dump locations, credentials, and temporary outputs.
Check nested dependency directories and Docker build-context exclusions.
Don't ignore legitimate fixtures or migrations with indiscriminate patterns.
Ignore rules don't protect files already tracked: review staged changes.

**Known correction from the working setup:** pnpm can create a root-local
`.pnpm-store` even when dependency directories are managed separately. Include
`/.pnpm-store/` from the start and confirm it with `git check-ignore`.

## 3. Guard setup and database operations

- Make the helper resolve and enter the repository root.
- Refuse production mode, hosted execution markers, and unexpected inherited
  database URLs before schema or seed commands run.
- Construct a fixed local database target inside Compose rather than inheriting
  the Replit connection string. Validate the effective container environment
  again immediately before touching the schema.
- Never print effective configuration containing secrets while validating it.
- Separate explicit `setup` from routine `up`. Setup builds the image, installs
  locked dependencies, waits for the database, builds required CLI entry points,
  applies the project's schema workflow, and optionally seeds local test access.
- Preserve migration review prompts. Don't use force/push-reset shortcuts.
- `up` starts services, waits for health, and prints the browser URL and, when
  applicable, a newly issued login link.
- `down` retains data volumes. Document destructive resets separately with a
  clear warning; don't reset data as a routine fix.
- Confirm permission requirements and explain that Docker access is powerful.
  The helper should run as the normal Linux user rather than under sudo.

## 4. Implement the approved test-login fallback

Skip this section if real supported local authentication works.

### Fail closed

- Require explicit local-auth enablement AND development mode. Reject partial
  or malformed configuration, hosted execution markers, a non-loopback browser
  origin, and database targets outside the dedicated local database.
- Run configuration validation before opening database connections or accepting
  requests. Keep hosted middleware's identity assurance intact.
- Hosting-marker detection must distinguish actual hosting metadata from an
  intentionally optional API credential; an API key alone does not prove that
  a process is hosted. Never expose credential values.
- Validate the exact browser-facing Host and Origin. Reject unexpected
  forwarding headers. Configure internal proxies consistently; don't weaken
  origin validation to make a health check succeed.
- Return 404 for local-only login endpoints in hosted mode. Reject synthetic
  sessions there even if they somehow appear in its session store.

### Fixed identity, normal authorization

- Use one clearly synthetic subject and an `example.test` email. Do not accept
  caller-selected users, emails, roles, or redirect destinations.
- Label local mode visibly. Don't present a synthetic email as OIDC-verified.
- Mark session provenance separately (for example, `local-test`). The narrowly
  scoped local identity exception must not become a global authentication bypass.
- Continue to use database grants, subject binding, normal role checks, live
  revocation, and existing self/last-admin safeguards.
- Seed the synthetic user and initial grant atomically under the appropriate
  database lock. Keep a durable initialization marker independent of the grant.
  Rerunning setup must not recreate revoked grants or undo demotions.
- Don't silently reseed during application startup or token redemption.
- Keep required external services honest: report missing configuration rather
  than fabricating billing, customer, or other domain data.

### Short-lived sign-in links

- Prefer a helper-issued link over a publicly reachable "mint login" endpoint.
  A Linux implementation can use a same-user Unix socket with mode 0600,
  owner/type checks, a restrictive bind-time umask, and safe stale-socket handling.
- Generate cryptographically random tokens, single use, five-minute expiry,
  bounded outstanding-token storage, and no persistence across process restarts.
- Put the token in the URL fragment, not a query parameter or path.
- Serve a no-store page with restrictive CSP and referrer policy. Clear the
  fragment from browser history before sending the token in a same-origin POST.
- Require the exact expected JSON body and Origin; reject identity overrides.
  Rotate the session on successful login and issue an HTTP-only, SameSite cookie.
  A non-Secure cookie is acceptable only in guarded local HTTP mode.
- Give sessions a per-process epoch so hot reloads/restarts invalidate them.
  Support logout and let the helper issue a fresh link on demand.
- Print links only on explicit helper/start commands, not ordinary application
  logs. Never put tokens, cookies, or session data into docs or test reports.
- Document private local endpoints and follow the project's API-contract
  conventions without exposing them as normal hosted login options.

## 5. Verify proportionally and report limits

Run the checks available in the current environment:

1. Shell/JSON/YAML syntax, generated API consistency where applicable, type
   checks, builds, and whitespace checks.
2. Unit tests for configuration guards, exact hosts/origins, token replay/expiry,
   session provenance/epoch, seed rollback/idempotence, and revoked-grant safety.
3. When Docker is available: validate Compose, build from a clean dependency
   volume, initialize a fresh database, start services, check proxy/health
   behavior and source reload, and verify optional editor attachment.
4. Local integration smoke: sign in, inspect status, perform permitted CRUD,
   reject hostile origins/hosts, test viewer denial and revocation, verify
   reseeding doesn't restore access, test restart invalidation, and log out.
   Confirm hosted mode rejects both local endpoints and synthetic sessions.
5. Use a disposable database for tests that alter grants. A user-facing smoke
   command must identify and clean up its own fixtures, never restore or reset
   unrelated grants. Don't log credentials in assertions.
6. After app changes, restart the existing hosted workflows once and confirm
   their logs and preview still work. Don't replace them with local workflows.

If Docker cannot run in the agent environment, say so. Static configuration
validation or native PostgreSQL tests are not a Docker runtime test. A temporary
native database can validate app behavior only with an explicit isolated target
and scrubbed subprocess environment, never the workspace's hosted database.
Clean up processes you started. Provide reproducible Docker smoke commands and
state which checks actually ran.

Known implementation pitfalls:

- Node `fetch` may not preserve a supplied Host override in the runtime used
  for a container health check. Use `node:http` when probing an internal port
  with a different browser-facing Host, and verify the actual response.
- Node's `--watch-path` is not portable to Linux. Use a supported watcher and
  test that the API rebuilds and restarts after source edits.
- Workspace commands execute relative to the selected package, not necessarily
  the repository root. Verify paths in test/helper commands.
- Login responses can clear an old cookie and then set a new one. HTTP smoke
  clients must use the final cookie value, as a browser would.

## 6. Document the Git collaboration workflow

- Keep the existing remote/provider. Don't push, merge, alter visibility, or
  publish simply because local development was requested.
- Document clone, clean-tree checks, fetch, fast-forward update, feature branch,
  focused commits, review, explicit push, and pull-request merge.
- Use separate local and agent branches and a clear ownership handoff. The
  agent doesn't automatically see unpushed changes or watch remote branches.
- Don't force-push, reset hard, or clean the checkout to resolve divergence.
- Explain that Git transfers tracked source, not secrets, databases, volumes,
  running processes, or schema state. Apply each environment's schema workflow
  separately. Never migrate local test sessions to hosted data.
- Document first start, restart, login renewal, logs, smoke tests, dependency
  updates, optional external credentials, and destructive reset warnings.
- Summarize delivered files, exact quick-start commands, authentication choice,
  hosted-behavior checks, and any unverified Docker/architecture constraints.

## Example invocation

> Use the local-linux-development skill to add local Linux Docker development
> to this app. Keep its hosted behavior and Git remote unchanged. Include an
> optional VS Code Dev Container. If supported localhost OIDC isn't available,
> use a synthetic test account with a short-lived link printed by the helper.

