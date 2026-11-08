# External-AI Minecraft Java Android launcher

Status: approved for implementation planning by the user's “Continue” after the
loader-free adaptation design was presented. Full touch, newer versions, mod
management and public distribution are required. No product implementation exists.

## Controlling revision: loader-free TouchController for unmodified vanilla

After the assistant requested authorization for launcher-side adaptation, the
user said “Continue.” The resulting proposed architecture is specified in
`2026-11-08-loader-free-touchcontroller-design.md`, which supersedes the touch
sections below. The user subsequently approved proceeding with this design.

The user subsequently required full touchscreen gameplay for completely unmodified
vanilla and reuse of TouchController without a mod loader. This supersedes the
mod-dependent touch proposal below. Do not install Fabric/NeoForge/TouchController
into a vanilla profile to satisfy touch support. User-selected modded profiles
remain separately supported by the mod-management requirements.

Source inspection identified a specific integration gap:
- `proxy-client-android` provides touch forwarding and callback/transport APIs,
  not a standalone control overlay or vanilla keyboard/mouse translation engine.
- `touchcontroller/versions/fabric/26.3/TouchController.kt` registers HUD/tick/
  lifecycle callbacks through Fabric and renders through Combine's Minecraft
  CanvasImpl.
- `touchcontroller/versions/26.3/BUILD.bazel` includes versioned game abstraction
  implementations and mixins. `GameActionImpl.kt` directly invokes Minecraft
  client APIs. The Android proxy cannot replace these pieces by itself.
- The inspected `master` tree contains Combine backend build files under
  `combine/backend/minecraft`; no separate Android rendering backend was found
  in that tree inspection. This does not establish that no external fork exists.

Approved adaptation architecture, with runtime feasibility still to be verified:
reuse suitable TouchController layout/configuration/input components in the
launcher, render the controls outside the game, and map user actions through
the existing GLFW/SDL keyboard/mouse input interfaces. Keep vanilla game JARs
and game classes unmodified; no agent/mixin injection as a substitute for a loader.
Game-state-dependent features need evaluation because vanilla does not expose
TouchController's player/inventory/world APIs to an external overlay.

This requires rendering/action/state adapters and potentially substantial porting,
not verified minimal glue. The user's subsequent “Continue” is the working
authorization for this specific adaptation. Other functionality still follows the
reuse-first constraint; do not extend the exception to unrelated reimplementation.

The design uses the loader-free adaptation for vanilla and modded profiles.
Upstream mod/proxy findings remain in the research document as historical evidence.

## Intent

Deliver a self-contained Android launcher for Minecraft Java Edition whose key
interaction is: write a request, copy a specialized prompt into any AI, paste its
JSON response into the launcher, and execute the requested supported operations.
The user's purpose is to achieve this by integrating established open-source
technologies with minimal custom code and GitHub Actions doing heavy build work.

## Requirements and decisions

User requirements:
- Permanently exclude Termux and Termux-based environments.
- Do not reuse an existing Minecraft launcher application.
- Prefer maintained, high-star libraries and existing build pipelines.
- Use Firecrawl for research and the authenticated GitHub CLI account for remote
  development workflows.
- Keep the external-AI copy/paste interaction; AI provider choice belongs to users.
- Full touchscreen gameplay is mandatory in the first deliverable.
- The first milestone must include full functionality, newer Minecraft versions
  and mod management; a vanilla-only or keyboard/mouse-only milestone is rejected.
- Android 14 is the supplied device-test OS. Model, GPU and RAM are not yet known.
- The intended audience is the public, not only the repository owner.

Working interpretations stated to the user:
- Minimal application-specific integration is permissible. Frameworks, a JVM,
  rendering backends, JSON validators, authentication, and installers are reused.
- Separately published library/runtime repositories maintained by launcher teams
  may be evaluated; the user replied “Continue” after this distinction was asked.
  This does not authorize copying a launcher application's source tree.

Engineering calls covered by the approved design:
- Kotlin, Jetpack Compose, AndroidX lifecycle/ViewModel, coroutines, Room and
  WorkManager for the Android application.
- ARM64 Android 10+ for the first device release. x86_64 emulator builds are for
  integration checks; they do not establish ARM64/GPU compatibility.
- First public acceptance includes current stable Minecraft (26.3 at research
  time, Java 25/SDL) and the representative Java 21/GLFW 1.21.1 path, plus modded
  instances and full touch. Recheck the current stable release before delivery.
  Minecraft versions use a tested compatibility catalog; discovery of a new
  release does not automatically assert its native or mod compatibility.
- Full-function scope proposed below includes instances/version selection,
  installation, runtime selection, settings, authentication, loader management,
  mod discovery/install/update/enable/disable/remove, touchscreen gameplay and
  the complete external-AI command loop. This is not a promise that every historical
  version, arbitrary third-party mod or future release will run on every GPU.
- Reuse and adapt TouchController's components to a launcher-side control overlay
  as specified by `2026-11-08-loader-free-touchcontroller-design.md`. Vanilla
  profiles contain no loader or control mod; no runtime class transformation is used.
- Proposed loader coverage: Fabric and NeoForge for their supported current
  versions, Forge for compatible older profiles. The launcher overlay supplies
  touch independently of these loaders. Verify each claimed profile on the
  appropriate game/runtime/native combination.

## Approaches

1. **Selected proposal: native Android frontend + Java service libraries + native
   Android component repositories.** Reuses the platform's established app model,
   Java ecosystem and existing runtime build scripts. Main cost is connecting
   the components and validating their actual Android compatibility.
2. **Considered: native frontend + Python installation library via an embedded
   Python runtime.** Reuses minecraft-launcher-lib but adds another runtime and
   inter-language interface. Not preferred while Java libraries supply the needed
   installation operations. Reconsider only with concrete incompatibility evidence.

Termux was explicitly rejected by the user. Forking an entire launcher violates
the user's requirements. Neither is an implementation alternative.

## Component selection

These are preferred candidates, not an already verified dependency lock. The
implementation plan must pin the selected revisions/artifacts, verify licenses,
check Android dependencies and confirm the native graph before promising gameplay.
No candidate may be silently replaced by copied application code or a new port.

| Responsibility | Preferred existing component | Integration facts / remaining checks |
| --- | --- | --- |
| App UI/state/persistence/jobs | AndroidX / Compose / Room / WorkManager | Standard Android architecture; background installers use progress and cancellation |
| Android JVM | FCL-Team/Android-OpenJDK-Build, Java 21/25 branches | Existing branch-specific scripts/Actions; verify archive layout, licensing, ABI, embedding and relocatability; Mojo's packaged runtimes are an alternate within the same architecture |
| LWJGL native bindings | MojoLauncher/unilwjgl3-builder | Existing multi-architecture build machinery; verify matching game LWJGL version and dependencies; pin source/toolchain rather than mutable branch heads |
| Window/surface/input | MojoLauncher/glfw and MojoLauncher/MojoSDL | Separate Android JNI bindings; select GLFW or SDL by the game version's requirements |
| Full touch controls | TouchController/TouchController layout/widget/configuration modules, adapted to Android | Launcher-side Canvas/resources/input providers; reuse actual controls; no game-side mod or loader required |
| Renderer configuration | MojoLauncher/mojoexec | MIT component; Java API configures EGL/display/native-library locations; it is not itself a complete JVM launcher |
| OpenGL translation | MobileGL-Dev/MobileGlues | Existing native renderer; existing Gradle build and separate AAR build wrapper; verify exact renderer/backend API compatibility |
| Audio | OpenAL Soft / MojoLauncher/alsoft-android-aar | Reuse native audio implementation; inspect artifact and lifecycle contract |
| File installation | FlowArg/FlowUpdater | Published Java library; manifest/library/assets downloads and progress; Java 8 bytecode; verify Android OS detection and native replacement hooks |
| Mod catalog | masecla22/Modrinth4J plus FlowUpdater mod integration | Existing Java API wrapper and installer; verify Android compatibility and dependency semantics; Modrinth is the proposed initial catalog provider |
| Launch argument construction | FlowArg/OpenLauncherLib | Reuse supported Minecraft metadata/profile routines if compatible; inspect current APIs rather than copying its old README example; do not use desktop ProcessBuilder path as Android JVM bootstrap |
| Microsoft/Minecraft authentication | RaphiMC/MinecraftAuth | Device-code flow, refresh and token serialization; verify HTTP dependencies on Android and provision a permitted application/client ID |
| JSON decode and schema validation | kotlinx.serialization plus an established Android-compatible JSON Schema validator | OptimumCode validator is a candidate; check alternatives/compatibility before locking; no handwritten generic validator |
| Networking/archive extraction | Existing HTTP/archive libraries selected through dependencies | Checksums and app-owned directories; no custom HTTP/archive implementation |

High-star upstream foundations (AndroidX, LWJGL, SDL) and specialized downstream
components have different popularity levels. Several specialized repositories
have fewer than 20 stars; they must not be advertised as high-star projects.
Their value is existing platform integration, explicit APIs, history and tests.

For the required 26.3 target, MojoLauncher/MojoSDL already documents offscreen-window
support and MojoExec integration for that game version. Prefer this maintained
component over constructing a new SDL/ART/HotSpot bridge.

## Application architecture

```text
Compose request / JSON / instances screens
             |
        ViewModels
             |
  Prompt composer + command use cases
             |
  schema validator -> typed command dispatcher
             |
  instance store (Room) | install jobs (WorkManager) | runtime adapter
             |                    |                         |
      app-private data      FlowUpdater              Game Activity/process
                                                        |
                               existing JVM + LWJGL + GLFW + renderer + audio
```

Custom code is limited to composing the prompt, defining the application's typed
command DTOs/schema, wiring Android screens/state, mapping operations to libraries,
and adapting documented runtime/lifecycle/input interfaces. Calling JNI invocation
APIs is integration; implementing a JVM port or replacement renderer is not.

### Touchscreen integration

Use the dedicated loader-free TouchController adaptation specification. The
overlay runs under Android ART in the game activity, reusing TouchController's
layout/widget/configuration logic with host rendering and input adapters. Input
is delivered through existing GLFW/SDL keyboard/mouse/text interfaces. The Android
proxy AAR by itself is not the standalone controls engine and is not presented as
the solution. Vanilla JARs and classes remain unmodified.

Touch acceptance includes simultaneous movement/look/actions, jump/sprint/sneak,
attack/use/break/place, hotbar selection, inventory/crafting/container use, chat
and text entry with Android IME, menus/pause, swimming/flying and the upstream
layout customization facilities. Test pointer cancellation, rotation/surface
changes and background/foreground transitions without stuck inputs.

### Mod management

Provide mod browsing/search and instance-scoped installation, updates, enable/
disable and removal using the same use cases as JSON commands. The proposed
catalog is Modrinth through an existing library; provider coverage is explicit,
not implied support for every hosting service. Local mod imports use Android's
document picker and existing archive/metadata libraries.

Resolve against exact game version, loader and client-side applicability. Use
published dependency metadata and installer capabilities for required dependencies,
optional dependencies and declared incompatibilities. Persist project/version
IDs, hashes, source, enabled state and resolved dependency relationships. Do not
claim that metadata alone detects every runtime mod conflict. If existing
libraries lack a necessary resolver feature, identify a suitable existing resolver
before implementation; do not silently build a general-purpose solver.

Mutations happen while the instance is stopped, with downloads staged and verified
before replacing the active mod set. Keep a recoverable previous manifest and
files when applying updates. Failures retain a usable prior set where possible;
report residual changes if an operation cannot be rolled back. Dependency removal
must not silently leave enabled dependants broken. Loader/version changes validate
the installed mod set and preserve the original instance until migration succeeds.

Use a separate Android game process (`:game`) so the native/JVM session can end
without terminating the frontend. All cross-process app data uses explicit
Android IPC or persisted instance identifiers, not shared Kotlin singletons.
The process shares the app UID and app-private filesystem; it is not a security
sandbox. UI reports start, running, exit and failure through explicit callbacks.

Native executable libraries must be packaged through Android-supported APK/native
library mechanisms. Do not assume downloaded executables can run from writable
app storage. Runtime image/class files may be managed as data with verified
layout; exact split between native libraries and runtime data is a plan task.

The game process must establish the host library callbacks and surface before
starting the JVM, pass only library-generated/classified arguments, and release
keys/surface resources on pause/destruction. JNI objects from ART and HotSpot
cannot be interchanged. Existing JNI bindings must own their expected VM state.

## User interaction and command contract

The app contains a request field, Copy Prompt action, a JSON paste field,
Execute action, per-operation progress, and an instance list. Android's clipboard
and share facilities provide the copy/paste transport. No AI API key is required.

The generated prompt contains:
1. Instructions to return only one JSON object conforming to the supplied schema.
2. Protocol version, supported operations, constraints and examples.
3. The current supported version catalog, instance identifiers and device limits.
4. The user's request encoded as a separate data value.
5. Instructions to use a clarification response when the request cannot map to
   supported operations, rather than inventing operations or downloads.

Credentials/tokens are never included in the prompt. The AI plans operations;
the app owns authentication, version metadata and download resolution.

Protocol v1 has two response kinds:
- `plan`: `schemaVersion`, `requestId`, `operations` (1–64 typed operations).
- `clarification`: `schemaVersion`, `requestId`, `message`; displayed, not executed.

The app supplies a UUID requestId in the prompt and requires the response to match
the active request. JSON input is bounded to 64 KiB. Unknown properties, versions,
operation kinds and invalid bounds produce a precise error before execution.

Operations:
- `instance.create`: stable instanceId, displayName, supported gameVersion.
- `instance.select`: existing instanceId.
- `game.install`: instanceId; installer resolves its stored gameVersion.
- `instance.memory.set`: instanceId, memoryMiB (bounded by app/device policy).
- `game.launch`: instanceId; requires successful installation and signed-in account.
- `instance.version.set`: instanceId, catalog gameVersion; resolves runtime and
  validates loader/mod compatibility before applying the change.
- `loader.install`: instanceId, loader kind and available loader version.
- `mod.search`: instanceId and search query; display catalog results with stable IDs.
- `mod.install`: instanceId, catalog project ID/slug and optional version ID;
  resolve a compatible version and required dependencies.
- `mod.update`: instanceId and installed project IDs; update only to a compatible set.
- `mod.enable`, `mod.disable`, `mod.remove`: instanceId and installed project IDs,
  with dependency checks.
- `instance.settings.set`: instanceId and documented bounded display/game settings;
  map through existing configuration APIs, not arbitrary file edits.

The prompt carries available loader/game/catalog capabilities. When a search
has multiple plausible results, show them and compose a follow-up prompt with
their IDs. An ambiguous request must not install a guessed project. AI commands
and manual UI controls share operation handlers and validation.

Example (instance and version are illustrative):

```json
{
  "schemaVersion": 1,
  "kind": "plan",
  "requestId": "00000000-0000-4000-8000-000000000001",
  "operations": [
    {"type": "instance.create", "instanceId": "vanilla", "displayName": "Vanilla", "gameVersion": "1.21.1"},
    {"type": "instance.memory.set", "instanceId": "vanilla", "memoryMiB": 2048},
    {"type": "game.install", "instanceId": "vanilla"},
    {"type": "game.launch", "instanceId": "vanilla"}
  ]
}
```

Schema and typed handlers are versioned together. The prompt embeds the shipped
schema and the same supported-capability catalog used by the executor. JSON is a
bounded application protocol, not a shell/programming-language interface.

## Execution semantics

- Validate the entire plan structurally and semantically before starting it,
  including references to instances created earlier in that plan.
- Execute operations serially and stop on the first failure. Report the failing
  operation and successful prior operations; do not claim transactional rollback
  across downloads or game launch.
- Persist request hash and per-operation status in Room. Identical completed
  plans are not executed again. Reuse of an ID with different content is rejected.
- An interrupted installation may resume through existing installer semantics
  after checksum verification. An indeterminate game launch is not automatically
  replayed after process death; inspect process/session state and report it.
- Instance configuration updates are atomic database transactions. Installs use
  library checksum support and completion markers; partial content is not marked
  playable. Before launch, revalidate installation and compatibility.
- All game files are under an app-owned instance root. Instance IDs are logical
  keys, never arbitrary paths. AI output cannot provide native libraries, shell
  commands, JVM flags or arbitrary download URLs.

## Authentication

Use the library's Microsoft device-code flow and Minecraft profile/entitlement
checks with a properly provisioned client identity. Android Keystore-backed
storage protects tokens; logging redacts tokens and launch credentials. If an app
registration requires user action, expose that exact setup task. Do not borrow
another launcher's credentials or claim end-to-end auth was tested without a real
authorized account. Login is an app session prerequisite, not an AI operation.

This is a public-distribution application: users sign in with their own Microsoft
accounts and entitlements. The publisher supplies an appropriate public-client
application registration and any required Minecraft API access; end users do not
create developer registrations. A public client ID may ship in the app; no client
secret is embedded. Test sign-in/refresh/sign-out/account switching and token
isolation with permitted accounts before public release.

## GitHub-first workflow

After design approval, create a repository under `wangzheruoyun` with a working
name `ai-minecraft-android` (or a collision-free descriptive variant). Proposed
distribution is a public source repository and signed APKs on GitHub Releases,
with a GPL-3.0-compatible project license appropriate to selected dependencies.
The user has requested public distribution, but has not specifically requested
Google Play publication; store-specific publishing is not assumed.

Use the authenticated gh account to push changes, trigger workflows, read logs
and retrieve artifacts. Never place the local authentication token in workflow
files or pass it to upstream scripts. Jobs use least-required GITHUB_TOKEN scope.

Workflows:
1. Dependency integration: fetch pinned upstream revisions/artifacts, verify
   hashes, execute existing upstream build commands, publish internal artifacts.
2. App CI: Gradle compile, lint, command/schema/installer tests, APK assembly.
3. Emulator: app UI/copy-paste flow, persistence/replay, library loading and runtime
   tests where the ABI and emulator support them.
4. Device acceptance: explicit matrix/results for actual game launch, rendering,
   audio/input and lifecycle; device access remains external to hosted CI.

Cache Gradle/native outputs by revision/toolchain/ABI. Keep heavy builds remote.
Use pinned action revisions and dependency verification. Preserve third-party
notices/source obligations. Debug APK artifacts support development; release
signing requires a separately configured owner-controlled keystore. The public
release requires stable application ID, retained signing identity, install/upgrade
tests on Android 14 and published compatibility/dependency notices. No shared
debug signing key is used for production releases.

## Delivery and acceptance

Implementation order:
1. Lock reusable components and their integration contracts; run their builds in
   Actions. Resolve the installer/native substitutions/JVM bootstrap interfaces.
2. Implement the prompt/JSON loop, instance/version/settings management, actual
   installation, loader and mod operations using existing libraries.
3. Connect both required runtime/window paths and existing input/audio/renderer
   components, including full TouchController integration.
4. Verify the full loop with authenticated accounts and physical Android 14
   ARM64 hardware, current stable Minecraft and representative modded profiles.
5. Produce signed public-release artifacts and verify installation/upgrades.

These are internal engineering steps, not reduced public milestones. First
delivery must include the full user-requested functionality.

Success requires a user request to produce a usable prompt, schema-valid AI JSON
to produce the requested instance/settings/install, and the installed game to
reach its menu and a playable world with rendering/audio/full-touch input verified.
Acceptance includes loader installation, mod discovery and dependency installation,
updates, enable/disable/remove, incompatible-mod handling and retained instance
state after interrupted operations. Current stable and representative Java 21
profiles must both be covered. Tests
must include unsupported operations, malformed JSON, wrong request IDs, replay,
interruption, download failure, missing runtime and lifecycle transitions.

An app that only validates JSON or installs files is an intermediate artifact,
not completion of the launcher. CI green does not prove device gameplay.

## Non-goals and change boundaries

No new JVM port, GLFW implementation, graphics translator, authentication stack,
package installer or schema-validation framework. No bundled existing launcher.
No Termux. No integrated AI subscription/backend. No unrelated social/store/server
features. No universal Minecraft/mod/GPU compatibility claim.

If a preferred library lacks a needed interface, first evaluate another maintained
component or existing upstream adaptation. A substantial missing implementation
is reported as a specific blocker, not filled with a placeholder or a silent
expansion of the agreed minimal-integration scope.

## Builder-owned details

Exact compatible dependency versions, revision hashes, module/package names,
theme/layout styling, archive library choice, retry/backoff tuning and cache layout
are implementation decisions. They must preserve the above behavior and be
recorded with reproducible build evidence. Current research is in
`docs/research/2026-11-08-launcher-feasibility.md`.

## Builder review and remaining setup

User answers resolve the previous scope questions: full touch is mandatory;
first milestone includes newer Minecraft and mod management; device OS is Android
14; audience is public. The earlier keyboard/mouse-only and vanilla-only proposals
are superseded.

The user explicitly requires unmodified vanilla with full touch and has approved
the loader-free TouchController adaptation design. A fresh builder check found
no remaining product/architecture decisions; component compatibility remains an
engineering verification responsibility.

Device model/GPU/RAM and physical test access remain delivery prerequisites.
Android 14 by itself does not identify the hardware. Publisher authentication
registration and signing identity are setup tasks before public release, not
features delegated to end users. Modpack import/export and additional catalogs
are not assumed by this draft's definition of mod management; identify them
explicitly if the user intends them as part of “full functionality.”
