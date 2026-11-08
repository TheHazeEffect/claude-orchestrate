# Orchestration profile

Read by the `orchestrate` plugin's skills and agents. Keep it short: only what differs from project to project. Architecture and conventions belong in `CLAUDE.md`.

## Verify

Commands every executor and reviewer runs. List them in order; all must pass.

```bash
# e.g. Flutter:  fvm flutter analyze --no-fatal-infos && fvm flutter test
# e.g. Next.js:  pnpm build && npx tsc --noEmit
```

Formatter check (optional): e.g. `fvm dart format --set-exit-if-changed .`

Known failures that are already there (so agents don't chase them): none

## Never touch

Files and folders agents must not edit:

- e.g. `.env*`, `key.properties`, `lib/firebase_options.dart`, `wrangler.jsonc` secrets

## Review checklist

Project-specific things the reviewer must check whenever the diff touches them:

- e.g. Every Firestore query is scoped by `tenantId`.
- e.g. Hive type IDs and `@HiveField` indexes are never reused.

## Established patterns

Things the project does on purpose; the reviewer must not flag them:

- e.g. Fake repositories hold in-memory mutable state.
- e.g. Approximations recorded in `docs/design-notes.md` are intentional.

## Tester

- Kind: `mobile` | `web` | `mobile, web` | `none`
- Default job: `suite` (run the automated suite below) | `cases` (run manual cases from the test plans)
- Manual cases need owner approval: yes | no
- Suite (optional): command, e.g. `DEVICE=emulator-5556 scripts/integration_test.sh`; about how long a full run takes
- Suite device: e.g. `emulator-5556` (AVD `App_IT`) and how to start it. Never use: e.g. `emulator-5554`, which holds data the manual cases need. If the suite wipes app data, say so.
- Not covered by automated tests (needs the tester): e.g. WebView rendering, payments flow
- Test plans: e.g. `test/regression/*_PLAN.md`
- Results folder: e.g. `test/regression/results/`
- Design reference (optional): e.g. `docs/design/` mockups

Mobile:

- Build and install: e.g. `fvm flutter build apk --debug && fvm flutter install`
- App id: e.g. `com.example.app`

Web:

- Dev server: e.g. `pnpm dev` at `http://localhost:3000`
- Test accounts: where they are defined (seed file or example config), never the values themselves
