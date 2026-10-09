---
name: Integrate From Git
description: Run this when you've pulled changes in to your replit from git that the agent is unaware of
---

---
name: integrate-git-changes
description: Reconcile an existing Replit development app after changes arrive through Git: inspect the incoming diff, install changed dependencies, update generated code and the development database safely, restart affected services, and verify the changed features. Use when the user has pulled changes, asks to integrate a branch, or asks to restart after a Git update. Preserve uncommitted work, hosted authentication, and production data.
---

# Integrate Git Changes into a Running Replit App

## Core lesson

A successful server restart does not mean incoming Git changes are integrated.
The code may compile while new pages fail because tables or columns are missing.
Check dependencies, generated code, database schema, and service configuration
before declaring the app ready.

Git transfers source files, not installed packages, database state, secrets, or
running processes. Never assume a manual Git pull ran the post-merge setup.

## Scope and safety

- Default to the current Replit **development** workspace, not production.
- If the user already pulled changes, do not pull, switch branches, or merge again.
- If asked to integrate a remote branch, inspect the working tree and confirm
  the intended branch/base from context before changing Git state.
- Preserve uncommitted work and existing remotes. Never reset hard, clean files,
  force-push, publish, or change repository visibility as routine integration.
- An integration request authorizes necessary non-destructive development
  setup. Stop for data deletion, ambiguous renames, incompatible migrations,
  access-policy changes, or unclear database targets.
- If the request explicitly limits work to a restart, respect that limit:
  report required schema/dependency work rather than silently expanding scope.
- Do not enable local test authentication in Replit, run Docker-local helpers
  there, reseed administrator grants, or import local sessions/data.
- Use applicable database, package-management, secrets, workflows, and
  post-merge-setup guidance. Never print connection strings or credentials.

## 1. Establish what changed

1. Read repository instructions and relevant memory.
2. Inspect `git status --short`, branch, recent history, and available reflog.
3. Identify the pre-update revision from an actual pull/merge/reflog event or
   user-provided base. Do not assume `HEAD~1` or `ORIG_HEAD` is the right base.
4. Inspect the incoming file list and relevant diffs. If the baseline cannot be
   established, say so and reconcile the current checkout against its declared
   dependencies/schema rather than guessing a complete change list.
5. Check these categories:
   - Manifests, lockfiles, runtime versions, native packages.
   - API contracts and generated clients/validators.
   - ORM schemas, migrations, schema exports, data backfills.
   - Environment/configuration changes and service commands.
   - New routes/pages whose behavior depends on these changes.
6. Inspect any existing reconciliation script before running it. Verify package
   selectors resolve to real packages; a command that selected no packages is
   not a successful database migration.

Do not turn integration into a broad refactor or review every unrelated file.

## 2. Reconcile dependencies and generated code

- If dependency inputs changed, install using the repository's pinned package
  manager and committed lockfile. For pnpm, normally use
  `pnpm install --frozen-lockfile`.
- Do not regenerate the lockfile merely to hide an installation failure.
- If API contracts or code-generation inputs changed, run the existing
  generator when outputs are missing or inconsistent. Avoid unnecessary
  regeneration if the incoming generated files already match.
- Review generated diffs. Never hand-edit generated output to suppress errors.
- Run relevant type checks/builds; fix integration failures before restarting.
- Check new environment requirements without exposing secret values. Request
  missing credentials through the supported secrets flow, not chat.
- Do not install or reconfigure unrelated services.

## 3. Reconcile the development database explicitly

1. Identify the actual database engine, ORM, configuration, and documented
   migration strategy. Confirm that the target is development without printing
   credentials. Do not substitute a different database provider.
2. Review schema/migration changes, including exports that determine which
   models the migration tool discovers.
3. Compare with current database metadata or use the migration tool's supported
   preview/status mode. Review proposed DDL before applying risky changes.
   Use strict confirmation/verbose preview options when available; verify the
   installed tool's syntax rather than assuming a flag.
4. Apply the established development migration command. Use versioned migrations
   when the project uses them, not schema push as a shortcut.
5. Never use force/reset/truncate flags to dismiss warnings. Stop if a rename
   is ambiguous, a table/column would be dropped, existing values cannot satisfy
   a constraint, or the operation may lose data. Explain the specific impact
   and obtain approval for an appropriate migration.
6. Do not insert invented business data or automatically seed/restore permissions.
   Review explicit backfills separately for scope, idempotency, and impact.
7. Verify expected tables/columns/indexes with read-only metadata queries after
   the update. Do not expose application rows merely to prove a table exists.

**Production is separate.** For Replit-managed PostgreSQL, follow the current
database skill's Publish schema flow; do not add startup DDL or scripts that
directly migrate managed production. For an external production database, use
its documented process only with explicit authorization and target confirmation.
Do not publish as part of a development integration request.

## 4. Restart once, then verify the right things

- Finish the coherent dependency/code-generation/schema changes first.
- Restart only affected existing managed workflows, using their exact current
  names. Don't invent replacement workflows or duplicate servers.
- Inspect fresh startup logs and browser errors, distinguishing old errors from
  errors produced after the restart.
- Confirm the app responds through its real preview routing. An API returning
  404 at `/` may be normal; use a defined health or application endpoint.
- Check the newly changed behavior, not only the landing page. A login page
  screenshot or health endpoint does not establish that a database-backed page
  works. Use appropriate metadata checks, authenticated requests, focused tests,
  and/or a preview check.
- Prefer cheap, targeted verification. Reserve browser automation for critical
  changed flows that cannot be established with simpler checks.
- Do not trigger real payments, external billing operations, or destructive
  business transitions solely as smoke tests.
- If something remains blocked, report the exact missing prerequisite and what
  was verified. Don't equate successful compilation with end-to-end success.

## Current Billing Export project reference

These are known starting points, not universal defaults. Recheck them in the
target checkout before use, especially when copying this skill to another app.

| Concern | Existing location or command |
| --- | --- |
| Full type check | `pnpm run typecheck` |
| Full build (includes type check) | `pnpm run build` |
| API generation | `pnpm --filter @workspace/api-spec run codegen` |
| Development schema update | `pnpm --filter @workspace/db run push` |
| Schema discovery | `lib/db/drizzle.config.ts`, `lib/db/src/schema/index.ts` |
| Reconciliation script to inspect | `scripts/post-merge.sh` |
| API workflow | `artifacts/api-server: API Server` |
| Web workflow | `artifacts/billing-export: web` |

Do not use `push-force`. Confirm the package selector matches
`lib/db/package.json`; prefer the exact scoped name over assuming a shorthand
worked. Leave the mockup preview workflow alone unless its inputs changed.

For example, a monthly-review feature can add closure/history tables while the
API still starts normally without them. Verify those tables exist and test the
feature's read path, rather than closing or reopening a real billing month just
to exercise the new code.

## Completion report

Briefly state:

- Which development setup actions were applied, or why they weren't needed.
- Whether the schema changed and how the relevant objects were verified.
- Which services restarted and which functional checks passed.
- Any warnings, blockers, or behavior not verified.
- Whether production was left unchanged.

Do not claim data was preserved based solely on the absence of a warning.
Do not run commands twice just to pad the report.

## Example invocation

> I've pulled new changes from Git. Use the integrate-git-changes skill to
> reconcile dependencies, generated code, and the development database, then
> restart the affected services and verify the app. Leave production unchanged.

