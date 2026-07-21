# Code Review — codraw/entity-migrator

Reviewed at package root `/Users/tlacroix/Sites/localhost/codraw/codraw-entity-migrator` (namespace `Draw\Component\EntityMigrator`). All PHP source, DI integration, and `composer.json` were read in full; tests were skimmed for coverage assessment.

## Fixes applied (2026-07-20)

- **composer.json:** PHP version constraint changed from unbounded `>=8.5` to `^8.5` (version-compatibility debt: prevents a future PHP 9 from installing against this package; no effect on any currently existing PHP version).
- **H1** — `composer.json`: added `"php": ">=8.5"`, `doctrine/orm ^3.6`, `doctrine/persistence ^2.2 || ^3.0`, `codraw/log ^0.39`, `psr/container ^1.1 || ^2.0`, `psr/log ^3` and `symfony/dependency-injection ^6.4.0` to `require`; added `codraw/dependency-injection ^0.39` to `require-dev` (the `DependencyInjection/` integration only activates through `codraw/framework-extra-bundle`, matching the `codraw/messenger`/`codraw/cron-job` precedent); added a `suggest` section for `codraw/framework-extra-bundle` and `symfony/security-core` (the only usage is an `instanceof UserInterface` check in `BaseEntityMigration::setState()`, which safely degrades to `false` when the class is absent, so it is soft-optional rather than a hard requirement). `composer validate --no-check-publish` passes.
- **H2** — `EventListener/MigrationWorkflowListener.php::process()`: the count/dispatch query builder now also filters on `entity_migration.migration = :migration`, so only the migration being processed is counted and dispatched.
- **M1** — `Command/MigrateCommand.php::execute()`: `null` from `findOneBy(['name' => ...])` now produces `$io->error('Migration "..." not found.')` and `Command::FAILURE` instead of a `TypeError` in `Registry::get()`. The secondary edge (empty `ChoiceQuestion` in `interact()` when no migration rows exist) is left open.
- **M4** — `Migrator.php::migrate()`: the catch block now applies `TRANSITION_FAIL` only when the workflow can actually take it, and rethrows the original error otherwise — previously a failure during `pause`/`skip` produced a `NotEnabledTransitionException` that masked the original exception (that path always threw; it now throws the real error).
- **L5** (partial) — `Entity/Migration.php`: `$state` now defaults to `MigrationWorkflow::PLACE_NEW`, matching the workflow's `initial_marking` and the column's non-nullable mapping, so `getState()`/marking-store reads on a fresh instance no longer throw. `$id` was left uninitialized because fixing it requires changing the `getId(): int` signature to `?int` (public API change).

### Validation pass (2026-07-20)

The fixes above were validated as CI would run them; no test fallout was found and no additional code changes were needed:

- `composer install --optimize-autoloader --no-interaction --prefer-dist --no-scripts` resolves and installs cleanly with the corrected `composer.json` (no constraint adjustments were necessary).
- `vendor/bin/phpunit` (with `DATABASE_URL` pointing at a dedicated `codraw_entity_migrator` MySQL database): 2 tests, 5 assertions, OK. The 2 PHPUnit notices (mocks without expectations in `Tests/Message/MigrateEntityCommandTest.php`) are pre-existing and do not fail the run (exit code 0).
- PHPStan (`analyse -c phpstan.dist.neon`): 4 errors, all verified pre-existing via `git stash` (identical error list with the fixes stashed): `EntityMigratorIntegration.php:60` (`NodeParentInterface::end()` not found), `Entity/BaseEntityMigration.php:87-88` (`Symfony\Component\Security\Core\User\UserInterface` not installed — consistent with it being `suggest`-only), `Message/MigrateEntityCommand.php:12` (`property.unusedType`). No baseline changes were needed.
- `markdownlint-cli2 "**/*.md"`: 0 errors, no fixes required.
- No further CODEREVIEW findings were fixed in this pass: the only existing tests cover `MigrateEntityCommand`, so none of the remaining findings (M2, M3, M5, L1-L4) are exercised by existing unit tests.

## Overall assessment

This is a small, well-architected component: entity migrations are modeled as two Symfony state machines (one per migration, one per entity/migration pair), driven through guard/entered/completed listeners, with per-entity locking and messenger-based fan-out. The design is genuinely good. However, the package has notable delivery-quality problems: `composer.json` omits several packages the code uses directly (including `codraw/log`, whose class is called exactly in the failure-logging path, and `doctrine/orm`, on which the whole package is built), the bulk-dispatch query forgets to filter by migration, a CLI command crashes with a `TypeError` on an unknown migration name, and package-local test coverage is close to zero. These are fixable, but today the package leans heavily on being installed alongside the rest of the codraw monorepo to work at all.

## Findings

### High

#### **[FIXED]** H1. Undeclared runtime dependencies; failure-logging path fatals if `codraw/log` is absent

`composer.json:17-24` declares only `codraw/messenger`, `symfony/console`, `symfony/event-dispatcher`, `symfony/lock`, `symfony/messenger`, `symfony/workflow`. The code directly uses:

- `Draw\Component\Log\Monolog\ErrorToArray` (`Entity/BaseEntityMigration.php:8,99`) — package `codraw/log`, **not** a dependency of this package nor of `codraw/messenger` (verified in `codraw-messenger/composer.json`). `ErrorToArray::convert()` is invoked precisely when a workflow transition carries an `error` context (i.e., when a migration fails, `Migrator.php:113`). In an app without `codraw/log`, the first failed migration throws `Error: Class "Draw\Component\Log\Monolog\ErrorToArray" not found`, masking the original failure.
- `doctrine/orm` (`Entity/*` attributes, `MigrationInterface.php:5` `QueryBuilder`, `EventListener/MigrationWorkflowListener.php:5-6` `EntityManagerInterface`/`Query\Parser`) — only a dev dependency of `codraw/messenger`, so nothing guarantees it.
- `doctrine/persistence`, `psr/log`, `psr/container`, `symfony/dependency-injection` + `symfony/config` (DI integration, `#[Autowire]`, `#[AutoconfigureTag]`), `codraw/dependency-injection` (`DependencyInjection/EntityMigratorIntegration.php:5-8`), `symfony/security-core` (`Entity/BaseEntityMigration.php:9`) — all used directly, none declared (some arrive transitively today).
- There is also no `"php"` version constraint at all, while sibling `codraw/messenger` requires `>=8.5`.

#### **[FIXED]** H2. Bulk dispatch is not filtered by migration

`EventListener/MigrationWorkflowListener.php:176-205`: after the `INSERT IGNORE ... SELECT` seeds `EntityMigration` rows for the migration being processed, the follow-up query selects rows only by `entity_migration.state = 'new'` — there is **no** `entity_migration.migration = :migration` predicate. Multiple migrations targeting the same entity class share one entity-migration class/table, so processing migration A will also count and dispatch every `new` row belonging to migration B (e.g., rows created just-in-time by `EntityMigrationRepository::load()` or seeded by a not-yet-started migration). Consequences: migration B's entities are processed (or paused/skipped) prematurely, and the progress bar max count (`:183-189`) is wrong.

### Medium

#### **[FIXED]** M1. `MigrateCommand` crashes with `TypeError` on unknown migration name

`Command/MigrateCommand.php:69-74`: `findOneBy(['name' => $input->getArgument('migration-name')])` can return `null` (typo, or migration not yet set up via `draw:entity-migrator:setup`), and `$migrationEntity` is passed straight to `Registry::get()`, producing an uncaught `TypeError` instead of a user-facing error message. `interact()` only fills the argument when it was omitted; a wrongly typed argument goes straight to `execute()`. Also, with an empty `draw_entity_migrator__migration` table, `interact()` builds a `ChoiceQuestion` over an empty array (`:50-56`).

#### M2. Raw SQL in `MigrationWorkflowListener::process()` is MySQL-only and partially hardcoded

`EventListener/MigrationWorkflowListener.php:161-174`: the seeding statement uses `INSERT IGNORE` (MySQL/MariaDB-specific — fails on PostgreSQL/SQLite, undermining the DBAL abstraction used elsewhere) and hardcodes the `entity_id` column name while `transitionLogs`/`createdAt` column names are correctly resolved from class metadata (`:167-168`). A subclass mapping the `entity` join column under another name breaks silently. `date('Y-m-d H:i:s')` (`:137`) also injects the PHP-process timezone into a DB literal. The `INSERT IGNORE` deduplication additionally *assumes* a unique constraint on `(entity_id, migration_id)` that neither `BaseEntityMigration` defines nor the README documents.

#### M3. Find-or-create race in `EntityMigrationRepository::load()`

`Repository/EntityMigrationRepository.php:17-33`: `findOneBy` + `persist`/`flush` with no locking or upsert. Two concurrent requests migrating the same entity just-in-time (the documented use case of `Migrator::migrateEntity()`) either create duplicate rows (no unique constraint in the base mapping — each duplicate is then migrated independently) or, if the app added a unique constraint, the second request gets an uncaught `UniqueConstraintViolationException` and a closed EntityManager instead of a retry of the find.

#### **[FIXED]** M4. `Migrator::migrate()` failure handling assumes the failing transition was `process`

`Migrator.php:104-117`: the try/catch wraps the application of whichever of `pause`/`skip`/`process` is enabled, but the catch always applies `TRANSITION_FAIL`, which is only defined `from: processing` (`DependencyInjection/EntityMigratorIntegration.php:200-205`). If a listener throws during the `pause` or `skip` transition (marking already moved to `paused`/`skipped` before `entered` listeners run), `apply(..., 'fail')` itself throws a `NotEnabledTransitionException`, replacing and hiding the original error, and the entity is left in an inconsistent, unflushed state.

#### M5. `assert()`-only validation in `Migrator::migrateEntity()`

`Migrator.php:42-46`: the check that a class-string argument actually implements `MigrationInterface` is an `assert()`, compiled out in production (`zend.assertions=-1`). An arbitrary class name then hits `$migrationName::getName()` — at best an `Error` for an undefined method, at worst calling an unrelated static `getName()` and silently resolving the wrong migration. The input is developer-supplied, so this is a robustness rather than a security issue.

### Low

#### L1. `sleep(1)` to "wait for database replication"

`Migrator.php:96-101`: after a blocking lock acquisition, a hardcoded one-second sleep stands in for replication consistency. It blocks a messenger worker, and one second is neither guaranteed sufficient nor configurable.

#### L2. Lock relies on destructor release and default TTL

`Migrator.php:66-70`: the lock is never explicitly released (relies on `Lock::__destruct` auto-release) and is created with the default TTL (300 s for TTL-based stores). A migration of a single entity running longer than the TTL loses mutual exclusion; an explicit `$lock->release()` in a `finally` plus a documented TTL would be safer.

#### L3. Typo in the generated index name

`EventListener/DoctrineSchemaListener.php:30`: `draw_migration_sate` (missing "t") is baked into every consumer's schema; renaming later means a schema migration in every application.

#### L4. Nullable `json` column vs non-nullable typed property

`Entity/BaseEntityMigration.php:33-36`: `transitionLogs` is mapped `nullable: true` but the PHP property is a non-nullable `array`. A `NULL` in the DB (any writer other than this package's own `INSERT ... '{}'`) produces a `TypeError` on hydration. Either map `nullable: false` with a default or make the property nullable.

#### **[FIXED]** (partially) L5. Uninitialized typed properties in `Migration`

`Entity/Migration.php:20,30`: `$id` and `$state` have no default; `getId()`/`getState()` on a freshly constructed, not-yet-persisted instance throws "must not be accessed before initialization" (e.g., a `Migration` created without the `SetupCommand` flow, or the workflow marking store reading `state` before `setState()` was called).

#### L6. Compiler pass assumes resolvable class on tagged definitions

`DependencyInjection/Compiler/EntityMigratorCompilerPass.php:24-28`: `$container->getDefinition($id)->getClass()::getName()` breaks (call on `null` or on an unresolved `%parameter%`) for definitions whose class is not set literally; `$container->getParameterBag()->resolveValue(...)` or a guard would be more robust. It also silently assumes the renamed id `draw.entity_migrator.migrator` (`:34`) stays in sync with `EntityMigratorIntegration::renameDefinitions()`.

#### L7. Documentation does not cover the integration contract

`README.md` is nine lines. Consumers must know to: implement `MigrationTargetEntityInterface` + a `BaseEntityMigration` subclass, map the `entity` association themselves (the base class leaves `$entity` unmapped, `Entity/BaseEntityMigration.php:20`), add a unique constraint on `(entity, migration)` (required by M2/M3), define the `ENTITY_MIGRATOR_LOCK_DSN` env var (unconditionally required by `DependencyInjection/EntityMigratorIntegration.php:79-88`), and have an `async_low_priority` messenger transport (default routing, `:59`). None of this is documented.

## Strengths

- **Sound state-machine design.** Both workflows are declared centrally with places/transitions derived from constants (`Workflow/*.php`), guards enforce business rules (pause only when the parent migration is paused, skip only when `needMigration()` is false — `EventListener/EntityWorkflowListener.php:32-49`), and the parent migration auto-completes/errors from an `AsCompletedListener` at priority −255 after flush.
- **Well-ordered side effects.** Flush happens in `completed` listeners, so an exception thrown inside an `entered` listener (the actual `migrate()` call) prevents the flush of the optimistic marking, and the `fail` transition then records the error — including a structured error trace in `transitionLogs` (`Entity/BaseEntityMigration.php:75-104`) with user attribution.
- **Concurrency awareness.** Per-entity, per-migration named locks (`Migrator.php:66-70`) with a distinct just-in-time "wait" mode plus entity refresh after contention; queueing is idempotent (`queue()` guarded by `workflow->can()`).
- **Performance-conscious bulk path.** Seeding uses a single `INSERT ... SELECT` derived from the migration's own id query builder (with correct DQL parameter re-mapping via `Query\Parser`), and dispatching streams ids with `toIterable()` + `getReference()` instead of hydrating entities (`EventListener/MigrationWorkflowListener.php:129-205`). A composite `(migration_id, state)` index is added automatically for all implementing entities (`EventListener/DoctrineSchemaListener.php`).
- **Clean DI integration.** `MigrationInterface` implementations are autoconfigured into a tagged service locator with lazy resolution by name (`DependencyInjection/Compiler/EntityMigratorCompilerPass.php`), and workflow/messenger/doctrine/monolog/lock configuration is prepended so consumers get a working setup for free.
- **Deferred entity references in messages.** `MigrateEntityCommand` implements `DoctrineReferenceAwareInterface` so the entity is serialized as a reference stamp and re-resolved at handling time, with a proper `ObjectNotFoundException` when it is gone.
- **Empty phpstan baseline** (`phpstan-baseline.neon`) — the code is clean under the configured static-analysis level.

## Test coverage

Package-local coverage is minimal: a single test file, `Tests/Message/MigrateEntityCommandTest.php`, covering only the `MigrateEntityCommand` message (entity accessor, not-found exception, reference-property list). Untested in-package:

- `Migrator` (lock acquisition/wait semantics, transition-selection loop, failure path) — the most intricate logic in the package;
- `MigrationWorkflowListener::process()` (raw SQL seeding, parameter remapping, dispatch loop) and both guard methods;
- `EntityWorkflowListener` (guards, process/queued/flush/updateState listeners);
- `EntityMigrationRepository::load()`;
- both console commands;
- `EntityMigratorIntegration` (prepend config, workflow definitions) and `EntityMigratorCompilerPass`;
- `DoctrineSchemaListener`.

Integration coverage may exist elsewhere in the monorepo (e.g., `codraw-framework-extra-bundle`), but as a standalone package nearly all behavior — including the H2 and M1 bugs, which a basic functional test would have caught — is unverified.
