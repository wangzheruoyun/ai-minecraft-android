# Minecraft Android Launcher Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Deliver a publicly distributable Android Minecraft Java launcher with full loader-free TouchController gameplay, current game versions, mod management and external-AI JSON execution.

**Architecture:** Native Kotlin frontend and Java service libraries orchestrate existing Android JVM/native components. TouchController runs as an adapted Android overlay, delivering normal input to unmodified vanilla. Heavy compilation and automated testing run on GitHub Actions.

**Tech Stack:** Kotlin, Compose, Room, WorkManager, kotlinx.serialization, established JSON Schema validator, FlowUpdater, MinecraftAuth, Modrinth4J, TouchController/Combine, existing OpenJDK/LWJGL/GLFW/SDL/MobileGlues/OpenAL Android components.

**Spec:** `docs/superpowers/specs/2026-11-08-ai-minecraft-android-design.md` and overriding `docs/superpowers/specs/2026-11-08-loader-free-touchcontroller-design.md`.

## Global Constraints

- Permanently exclude Termux and Termux-based environments.
- Do not reuse an existing Minecraft launcher application. Separate component repositories are permitted.
- Vanilla game JARs/classes must not be patched, transformed, or injected with a Java agent/mixin.
- No loader or control mod is inserted into vanilla profiles.
- Reuse TouchController code; its authorized Android adaptation is not permission to reinvent other components.
- Full touch, newer Minecraft and mod management belong to the first public deliverable.
- ARM64 Android 10+; Android 14 physical acceptance; x86_64 is a CI test ABI.
- Current stable game and a representative Java 21/GLFW game must pass; research baseline is 26.3/Java 25/SDL and 1.21.1/Java 21/GLFW.
- Protocol schemaVersion=1, maximum UTF-8 input 65,536 bytes, 1–64 operations, UUID requestId.
- User requests go to an external AI chosen by the user; no integrated AI credentials or service.
- Existing libraries own networking, schema parsing/validation, installers and authentication.
- No completion claim from a UI-only APK, successful compilation or emulator-only gameplay.

## Review Focus

- App death between a successful side effect and journal completion: never replay an uncertain launch (service plan Task S3).
- Multiple touch pointers plus focus loss: release all held actions without dropping unrelated pointer ownership (touch plan Task T3).
- Remapped bindings and Unicode IME composition: correct actions/text once, not default-key assumptions (touch plan Task T4).
- Loader/version/mod changes while a game is running or dependencies conflict: preserve original runnable state (service plan Tasks S4–S5).
- New game/native artifacts or an upgrade with changed signing identity: no false compatibility or broken public upgrade (Tasks D2/D4).

## Plan decomposition and order

Three linked plans describe one complete deliverable. They are independently
testable subsystems, not reduced product milestones:

1. This plan: repository/CI, native runtime integration and final release.
2. `2026-11-08-touchcontroller-host.md`: Android TouchController library and input adapters.
3. `2026-11-08-launcher-services.md`: protocol, execution, installers, mods, auth and UI.

Execution order: D1 → T1 → D2 → T2–T4 → S1–S6 → D3 → D4.
T1 is first after CI because the authorized port's dependency closure is the
largest unverified assumption. A blocked native build does not justify claiming a
reduced deliverable; independent tasks may proceed while the exact blocker is resolved.

## File map

- `app/`: `io.github.wangzheruoyun.aiminecraft`, Compose screens and application composition.
- `protocol/`: pure Kotlin prompt, schema and typed operations.
- `services/`: Android instance/journal/auth/install/catalog adapters.
- `runtime/`: Android game process, native component integration and `InputSink` API.
- `touch-host/`: adapted upstream TouchController/Combine and Android providers.
- `upstream/sources.lock.json`, `upstream/patches/`: immutable source coordinates, provenance and reviewed patches.
- `gradle/libs.versions.toml`, `gradle/verification-metadata.xml`: dependency versions/checksums.
- `.github/workflows/{ci,native,emulator,release}.yml`: remote builds/checks/artifacts.
- `docs/integration/`: reproducible dependency/API evidence and exact outstanding gaps.
- `docs/acceptance/`: device results, vanilla-integrity evidence and release checklist.

## Task D1: Establish reproducible remote build and dependency inventory

**Files:** create `settings.gradle.kts`, `build.gradle.kts`, `gradle/libs.versions.toml`, Gradle wrapper, `app/build.gradle.kts`, `app/src/main/AndroidManifest.xml`, `.github/workflows/ci.yml`, `upstream/sources.lock.json`, `docs/integration/components.md`, `LICENSE`, `THIRD_PARTY_NOTICES.md`.

**Interfaces:** produces module boundaries listed above and pinned toolchain/dependency coordinates. Lock records contain `id`, `repository`, `commit`, `license`, `sourcePaths`, `patches`, `artifactSha256` when artifacts are consumed.

- [ ] Inspect applicable AGENTS instructions and use an isolated execution branch/worktree; initialize Git if necessary. Preserve existing docs. Use `gh` to create/check `wangzheruoyun/ai-minecraft-android`; do not overwrite a repository that appeared since planning.
- [ ] Record approved specs and these plans in Git. Use the authenticated account's verified Git identity rather than inventing attribution.
- [ ] Verify current compatible Gradle/AGP/Kotlin/Compose/SDK/NDK versions from upstream artifacts/docs through Firecrawl. Select minSdk 29; compile/target SDK follow the selected stable Android toolchain. Record versions instead of unpinned `latest` dependencies.
- [ ] Inspect licenses and pin component source revisions. TouchController research SHA is `f8128cc2b70aa98b8e2a7da422a5eeb110c6001c`; resolve and record its full toolchain/dependency closure. Preserve notices and source obligations; missing license evidence is a dependency selection issue.
- [ ] Configure CI to run `./gradlew check :app:assembleDebug --stacktrace`, cache by toolchain/lock, and upload APK/test reports. Use maintained upstream Actions pinned to commits; use `contents: read` and no local gh token in runners. Skip jobs for docs-only edits.
- [ ] Push and inspect the actual workflow with `gh run watch --exit-status` and `gh run view --log-failed`. Expected: real Gradle build succeeds and an APK artifact exists. Document any failure instead of installing a heavy local toolchain as a shortcut.
- [ ] Commit the verified foundation and source inventory.

## Task D2: Assemble existing JVM/native components and expose runtime contracts

**Files:** create `runtime/build.gradle.kts`, `runtime/src/main/cpp/{CMakeLists.txt,jvm_entry.cpp}`, `runtime/src/main/java/io/github/wangzheruoyun/aiminecraft/runtime/{InputSink,NativeRuntime,LaunchSpec,RuntimeCatalog,RuntimeEvent}.kt`, `runtime/src/main/aidl/io/github/wangzheruoyun/aiminecraft/runtime/IRuntimeCallback.aidl`, `.github/workflows/native.yml`, `docs/integration/native-contracts.md`.

**Interfaces:**
- `InputSink`: `key(code: Int, down: Boolean, modifiers: Int)`, `mouseButton(button: Int, down: Boolean)`, `relativeMotion(dx: Double, dy: Double)`, `absoluteMotion(x: Double, y: Double)`, `scroll(dx: Double, dy: Double)`, `text(value: String)`, `releaseAll()`.
- `LaunchSpec`: immutable instanceId, mainClass, classpath list, JVM/game argument lists, gameDirectory, runtimeId, nativeProfileId. Persist no plaintext credentials with it.
- `RuntimeCatalog.resolve(gameVersion: String, abi: String): RuntimeSelection`; selection contains exact runtime/native/library revisions and compatibility status.
- `NativeRuntime.start(spec: LaunchSpec, surface: Surface, callback: (RuntimeEvent) -> Unit)`; events `Starting`, `Running`, `Exited(code)`, `Failed(reason)`, `CaptureChanged(captured)`.

- [ ] Write `runtime/src/androidTest/.../NativeRuntimeTest.kt`: JNI smoke class reports the selected VM version; missing/wrong-ABI libraries yield `Failed`, never `Running`; host input recording consumer receives ordered press/release and Unicode codepoints. Run emulator job to record expected failures before integration.
- [ ] Read existing OpenJDK build scripts, unilwjgl3-builder workflow, GLFW/MojoSDL JNI and MojoExec native contracts. Record library initialization order, ELF dependencies, exported symbols, VM ownership, thread/surface requirements, native replacements and packaged JRE layout.
- [ ] Reuse their build scripts on Actions for ARM64 and x86_64, using exact source/toolchain hashes and matching Minecraft LWJGL versions. Build required OpenAL/shader/renderer dependencies through existing scripts. Select MobileGlues-compatible EGL/OpenGL path from actual library contracts, not assumptions based on names.
- [ ] Package executable `.so` files through APK native-library facilities and runtime class/module data through verified data archives. Handle duplicate JRE native names with isolated runtime packaging/search paths established by existing component contracts. Do not execute downloaded binaries from writable app storage or overwrite process-global runtime libraries.
- [ ] Implement only JNI invocation/lifecycle integration in `jvm_entry.cpp`; retain ART/HotSpot VM boundaries. Implement GLFW and SDL `InputSink` adapters in `runtime/.../input/{GlfwInputSink,SdlInputSink}.kt`, using their actual APIs and required thread dispatch.
- [ ] Implement library-backed runtime selection; reject unknown game/native combinations, mismatched Java versions and corrupted artifacts. Write `RuntimeCatalogTest` for these cases and current/representative profiles.
- [ ] Run native builds and `:runtime:connectedDebugAndroidTest` on Actions. Expected: JNI smoke, surface/input lifecycle and selected-runtime tests pass with exported logs. ARM64 runtime startup still requires physical-device evidence in D4.
- [ ] Commit pinned integration and evidence. If component source requires unapproved new porting, identify the exact missing symbol/API and resolve before marking complete.

## Task D3: Integrate real game process, overlay and end-to-end flow

**Files:** create `app/src/main/java/io/github/wangzheruoyun/aiminecraft/game/{GameActivity,GameSessionClient}.kt`, `runtime/src/main/java/io/github/wangzheruoyun/aiminecraft/runtime/GameSessionRegistry.kt`, `app/src/androidTest/.../{CommandToLaunchTest,GameLifecycleTest}.kt`, `.github/workflows/emulator.yml`.

**Interfaces:** consumes `LaunchSpec`, `NativeRuntime`, `TouchHostView` from T2 and `GameLaunchService` from S4. `GameSessionClient.launch(instanceId: String): LaunchOutcome`; session registry tracks actual process/session identity and last event without storing tokens.

- [ ] Write failing integration cases: command plan creates/configures/installs then requests launch; launching an uninstalled instance fails; app recreation reconnects to a live session; uncertain session status never launches twice; switching account clears stale credentials before launch.
- [ ] Compose GameActivity under `android:process=":game"`; attach surface, host overlay and backend before JVM startup. Use Android IPC callbacks; no cross-process singleton assumptions. Prevent a second simultaneous game session.
- [ ] Wire real service implementations from S4/S5 and native runtime from D2. Test doubles remain test-only; production cannot report successful install/launch when using them.
- [ ] Connect capture/mode toggle, viewport/insets, IME, lifecycle release and layout editor. On game exit, release native/session resources and keep the frontend usable.
- [ ] Run `:app:connectedDebugAndroidTest` and `:app:lintDebug` on Actions; collect test reports and debug APK. Exercise real game only with authorized device/session setup—never make offline identity a replacement for authentication acceptance.
- [ ] Commit integration and record actual coverage versus pending device tests.

## Task D4: Public release acceptance

**Files:** create `.github/workflows/release.yml`, `docs/acceptance/{android14,vanilla-integrity,mod-matrix,release}.md`, `docs/setup/publisher.md`.

**Interfaces:** consumes signed app artifact, component lock, account configuration and human/device acceptance records. Produces versioned public APK, checksums, notices/source references and compatibility matrix.

- [ ] Obtain Android 14 model/GPU/RAM and test access; arrange user-installed APK/log handoff if no remote device is available. Obtain publisher public-client registration and owner-controlled signing credentials through appropriate configuration, not chat logs.
- [ ] Run the complete touch-only acceptance list from the touch spec on 1.21.1 and current stable game; verify game JAR hashes and no control-mod/loader/agent classes or flags in vanilla profiles. Redact launch tokens from evidence.
- [ ] Test Fabric/NeoForge/current and compatible Forge profiles: install dependencies, update, enable/disable/remove, import local mod, failed download, declared conflicts and recovery. Confirm vanilla profiles never gain loaders from touch setup.
- [ ] Test Android 14 install/upgrade, preserved instances/layouts/account isolation and reauthentication. Use `apksigner verify --verbose --print-certs` and compare intended signing identity; verify native ABI and 16-KiB alignment compatibility where the selected platform/distribution requires it.
- [ ] Run final `./gradlew check :app:lintRelease :app:assembleRelease` on Actions with protected signing configuration. Missing signing/auth/device evidence is a release blocker, not a reason to publish a debug APK as production.
- [ ] Perform whole-branch review and address failures. Create GitHub Release only when all required acceptance evidence passes; attach APK/checksums/notices and accurate tested compatibility.
- [ ] Commit acceptance results and report the release URL plus remaining documented limitations. No universal future-version/mod/GPU claim.

## Self-review

- Specs mapped: touch T1–T4; protocol S1; execution S2–S3; install/runtime S4/D2;
  mods S5; auth/UI S6; integration D3; public/device acceptance D4.
- Interfaces are defined at their producing tasks; Kotlin packages use
  `io.github.wangzheruoyun.aiminecraft` consistently.
- Review-focus cases are assigned to tests, not deferred to general inspection.
- No full launcher or Termux dependency is proposed; TouchController adaptation
  is explicit. Dependency feasibility is measured before claiming integration.
- Plans await execution review. No product builds or tests have run yet.
