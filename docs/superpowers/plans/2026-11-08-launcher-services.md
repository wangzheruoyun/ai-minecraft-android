# Launcher Services and AI Workflow Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement the complete external-AI command workflow, instances, real installation, mods, accounts and Android UI through existing libraries.

**Architecture:** A versioned schema and typed operations share application use cases with manual UI actions. Room journals serial execution; WorkManager manages long operations; existing installer/auth/catalog libraries perform underlying work.

**Tech Stack:** Kotlin/Compose, kotlinx.serialization, JSON Schema validator, Room/WorkManager, FlowUpdater, OpenLauncherLib, MinecraftAuth, Modrinth4J.

**Spec:** `docs/superpowers/specs/2026-11-08-ai-minecraft-android-design.md`, with loader-free touch override.

## Global Constraints

- All global constraints and file/package conventions from `2026-11-08-launcher-delivery.md` apply.
- Protocol v1: 65,536 UTF-8 bytes maximum, 1–64 operations, active UUID requestId, reject unknown fields/operations.
- Full first delivery includes current Minecraft and mod management; production handlers cannot be stubs.
- AI receives no credentials and cannot supply shell commands, native downloads, arbitrary JVM flags or file paths.
- Native compatibility is selected from verified runtime profiles; metadata discovery alone does not enable support.

## Review Focus

- UTF-8 byte limits, malformed/unknown fields and stale request IDs (S1).
- Duplicate/concurrent submission and process death around side effects (S2–S3).
- Partial/corrupt downloads, insufficient storage and version migration failure (S4).
- Declared mod dependency conflict, local import mismatch and mutation during play (S5).
- Account switch/expired token and private data leaking into prompts or logs (S6).

## Task S1: Versioned protocol and specialized prompt

**Files:** create `protocol/src/main/kotlin/io/github/wangzheruoyun/aiminecraft/protocol/{AiResponse,Operation,ProtocolValidator,PromptComposer,Capabilities}.kt`, `protocol/src/main/resources/{commands-v1.schema.json,prompt-v1.txt}`, tests `ProtocolTest.kt`, `PromptTest.kt`.

**Interfaces:** `ProtocolValidator.validate(json: String, expectedRequestId: String): ValidationResult` (Valid(response) or Invalid(errors)); `PromptComposer.compose(request: String, requestId: String, capabilities: Capabilities): String`. AiResponse is Plan or Clarification; Plan contains typed operations from the approved spec.

- [ ] Write tests asserting rejection of 65,537 UTF-8 bytes, 0/65 operations, unknown fields/type/schema version, invalid UUID/wrong active ID, malformed JSON and unsupported capability values. Accept a 64-operation valid plan and a clarification without side effects. Include a multibyte string that crosses the byte bound before the character bound.
- [ ] Run `:protocol:test` on Actions and record expected failures.
- [ ] Select and integrate an existing JSON Schema validator after verifying Android-compatible transitive dependencies. Define closed schemas and typed Kotlin DTOs for every approved operation. Use serializers/library validation, not a handwritten JSON parser/schema evaluator.
- [ ] Compose the exact schema and capability snapshot into the prompt with separately encoded request text, examples and clarification rule. Include catalog IDs rather than credentials or arbitrary download instructions. Test embedded quotes/newlines and user text that resembles protocol instructions; it remains data and cannot alter shipped schema.
- [ ] Run protocol tests and schema/DTO fixture consistency tests. Commit the protocol and prompt together.

## Task S2: Instances and durable execution journal

**Files:** create `services/.../store/{LauncherDatabase,InstanceEntity,OperationEntity,RequestEntity,LauncherDao,InstanceRepository,ExecutionJournal}.kt`, tests `LauncherDatabaseTest.kt`, `JournalTest.kt`.

**Interfaces:** `InstanceRecord(id, displayName, gameVersion, loaderSelection, memoryMiB, settings, installState)`; `InstanceRepository.get(id): InstanceRecord?`, `create(record)`, `update(record)`, `select(id)`; suspend database methods. `ExecutionJournal.claim(requestId, contentHash): ClaimResult` (New, Resume, Completed, Conflict, Busy); journal stores ordered operation state and failure text without tokens.

- [ ] Write Room tests for duplicate instance IDs, path-like invalid IDs, transactional settings changes, duplicate completed plan, changed hash under same ID, two concurrent claimants and recovery of Running→Indeterminate after process death.
- [ ] Run `:services:connectedDebugAndroidTest` remotely to record failures. Implement Room entities/transactions and app-owned path mapping (IDs restricted to `[a-z0-9][a-z0-9_-]{0,63}`); user display name is independent Unicode text.
- [ ] Hash a deterministic typed-plan serialization with a standard digest API; equivalent whitespace must not create different identities. Persist active request/capability snapshot so rotation does not invalidate legitimate responses.
- [ ] Test migration from the first supported schema fixture and atomic update rollback. Commit database and migration evidence.

## Task S3: Serial command execution and recovery

**Files:** create `services/.../commands/{CommandPreflight,CommandExecutor,CommandHandler,ExecutionWorker}.kt`, tests `CommandExecutorTest.kt`, `ExecutionRecoveryTest.kt`.

**Interfaces:** `CommandHandler.execute(operation: Operation): OperationResult`; result Success(data), NeedsInput(message), Failure(code,message). `CommandPreflight.check(plan, state): List<CommandError>` evaluates references created earlier in the same plan. `CommandExecutor.execute(plan): Flow<ExecutionEvent>` (Started, OperationStarted, OperationCompleted, Stopped, Completed).

- [ ] Write tests proving whole-plan preflight before any handler call, sequential execution, stop on first failure, no execution of clarification, progress after each actual completion, identical resubmission suppression and no automatic replay of indeterminate game.launch.
- [ ] Run `:services:testDebugUnitTest`; implement journal-backed dispatch through WorkManager and real handler interfaces. Persist result before moving to the next operation; recover uncertain side effects by operation-specific reconciliation rather than assuming transactions cover native/network work.
- [ ] Implement memory bounds from actual device/runtime policy and settings allowlist. Instance modifications acquire a per-instance lease shared with manual UI. A running game blocks install/mod/version mutations.
- [ ] Test cancellation during downloads, two plans targeting one instance, and interrupted state after a successful external operation but before journal completion. Reconcile installation from verified completion marker; launch remains uncertain until session state is checked.
- [ ] Run unit/Room/WorkManager tests in Actions; commit executor and recovery contract.

## Task S4: Existing installer and launch-profile integration

**Files:** create `services/.../install/{GameInstaller,FlowUpdaterInstaller,InstallWorker,VersionCatalog,LaunchProfileBuilder,VersionMigration}.kt`, `services/.../game/GameLaunchService.kt`, tests `InstallerTest.kt`, `LaunchProfileTest.kt`, `VersionMigrationTest.kt`.

**Interfaces:** `GameInstaller.install(instance: InstanceRecord): Flow<InstallProgress>`; `LaunchProfileBuilder.build(instance, account: AccountSession): LaunchSpec`; `GameLaunchService.launch(instanceId): LaunchOutcome` is the service port, with D3 supplying the Android implementation. `LaunchOutcome` is Requested(sessionId) or Rejected(reason), not a premature claim of gameplay.

- [ ] Inspect FlowUpdater and OpenLauncherLib current APIs/OS detection/native selection/version rules and Java dependencies. Compile a real library fixture on Android CI. Establish manifest handling/native substitution hooks; if absent, evaluate another established standalone library before implementing replacement installer logic.
- [ ] Write tests using MockWebServer/local manifest fixtures and actual installer library: assets/libraries fetched with checksums; interrupted/corrupt downloads not marked installed; retry does not trust bad cache; storage failure preserves prior completed installation; malformed archives cannot escape owned root through chosen archive library validation.
- [ ] Implement adapters around library calls, cancellation/progress and installation completion markers. Runtime/native substitutions come from D2 locked catalog, not AI input. Native executable loading remains D2's APK packaging responsibility.
- [ ] Build launch profiles through existing metadata/profile utilities, resolve runtime and auth placeholders at launch time; no tokens in persisted profile/logs. Vanilla profile assertions: expected original main class, original game hash, no loader/TouchController mod/agent/mixin arguments.
- [ ] Stage version migration separately; validate loaders/mods before switching active instance version. Tests assert migration failure leaves original files/metadata selected and running instance mutation is rejected.
- [ ] Run installer/profile unit and Android integration tests in Actions; commit real installation and native-substitution evidence.

## Task S5: Loader and mod management

**Files:** create `services/.../mods/{ModCatalog,ModrinthCatalog,ModManager,LoaderManager,ModManifest,LocalModImporter}.kt`, tests `ModManagerTest.kt`, `LocalModImporterTest.kt`, `LoaderManagerTest.kt`.

**Interfaces:** `ModCatalog.search(query, gameVersion, loader): List<ModProject>`; `ModManager.apply(instanceId, change: ModChange): ModResult`; changes Install(projectId,versionId?), Update(projectIds), Enable(projectIds), Disable(projectIds), Remove(projectIds). `LoaderManager.install(instanceId, kind, version)` uses existing installer. `LocalModImporter.import(instanceId, uri: Uri): ModResult` uses Android document access plus existing metadata readers.

- [ ] Verify Modrinth4J/FlowUpdater dependency/version/conflict behavior from source and fixtures. Choose an existing resolver/metadata reader for any missing required function; do not implement a new generic solver. Record the selected API in `docs/integration/mods.md` before committing adapter code.
- [ ] Write tests: exact game/loader/client-side filtering, required dependency inclusion, incompatible versions rejected, ambiguous name yields results rather than guessed install, enabling with missing dependency fails, removing a depended-on mod preserves valid set, updates preserve pinned requested versions, and running-game mutations reject.
- [ ] Run failing tests in Actions. Implement library-backed catalog/loader/mod adapters, stable provider/project/version IDs and checksum manifests; explicit dependency-safe enable/disable/remove through file staging and Room state.
- [ ] Test crash between staged download and manifest switch; recover to complete previous or new set, never claim success for mixed state. Local import tests cover invalid JAR, wrong loader/version, duplicate hash and absent identifiable metadata. Require explicit user resolution of unknown metadata rather than invent compatibility.
- [ ] Connect all mod/loader JSON commands to the same adapters used by manual UI; mod.search results feed the next prompt's stable IDs. No TouchController mod is installed as a dependency of vanilla touch.
- [ ] Run service tests and Android catalog/import integration in Actions; commit behavior and dependency records.

## Task S6: Public authentication and complete Android UI

**Files:** create `services/.../auth/{AccountSession,AccountRepository,MinecraftAuthAdapter,TokenVault}.kt`; `app/.../ui/{RequestScreen,JsonScreen,InstanceScreen,ModsScreen,AccountScreen,LauncherViewModel}.kt`; `app/src/androidTest/.../{LauncherFlowTest,AccountIsolationTest}.kt`.

**Interfaces:** `AccountSession(accountId, profileId, username, accessToken, expiresAt)` is memory-only; `AccountRepository.signIn(): Flow<AuthEvent>`, `activeSession(): AccountSession`, `signOut(accountId)`, `select(accountId)`. UI consumes S1/S3 flows and S2/S4/S5 services; composition root binds concrete production adapters.

- [ ] Write tests for token expiration/refresh/cancellation, sign-out deletion, switching account before launch, token isolation and no credentials in generated prompt/error/log output. Use library HTTP fixtures for automation; reserve real entitled account verification for D4.
- [ ] Integrate MinecraftAuth device-code and entitlement/profile flow using publisher public-client configuration; store tokens using Android Keystore-backed established crypto APIs. No client secret or borrowed launcher identity in APK. Missing publisher configuration is a visible setup error, not a fake login.
- [ ] Implement Compose request/copy/share, JSON paste/validate/execute, progress/recovery, account, instance/settings, versions/loaders and mod management screens. Keyboard and touch accessibility use framework controls. Manual actions call the same use cases as JSON execution.
- [ ] Write Compose tests: request produces schema-bearing clipboard text; pasted valid plan executes and shows real operation state; errors identify operation/field; clarification displays without execution; rotation preserves active request; mod search selection resolves stable ID; cancelled downloads show recoverable state.
- [ ] Connect real services, hide no unimplemented production path behind success messages. Run `:app:connectedDebugAndroidTest :app:lintDebug :app:assembleDebug` on Actions.
- [ ] Commit complete UI/auth integration and hand off to D3/D4 for native and physical end-to-end acceptance.
