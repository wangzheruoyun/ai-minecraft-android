# Loader-free TouchController Host Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Reuse TouchController for full touch gameplay in unmodified vanilla through an Android-hosted overlay.

**Architecture:** Adapt a pinned upstream source dependency, keeping its layout/widget/state logic and replacing game-side rendering/resource/action boundaries. Deliver normal key/mouse/text events through existing GLFW/SDL libraries.

**Tech Stack:** TouchController, Combine, Kotlin, Android Canvas/Paint/View/IME, existing GLFW/SDL JNI bindings.

**Spec:** `docs/superpowers/specs/2026-11-08-loader-free-touchcontroller-design.md`.

## Global Constraints

- Apply every constraint and execution convention in `2026-11-08-launcher-delivery.md`.
- No Minecraft/Fabric/NeoForge runtime dependency in `touch-host`.
- No injected control mod, agent, mixin or class modification in vanilla.
- Reuse real upstream widget behavior, preserve attribution and record adaptations.
- Full gameplay access, not fabricated game-state awareness; digital keyboard movement is explicit.
- Build and run automated tests on GitHub Actions; physical Android 14 tests remain mandatory.

## Review Focus

- Cancel/focus loss releases all keys and pointer locks (T3).
- Capture-mode changes during a drag do not leave a mouse button held (T3).
- 30/60/120-Hz timing preserves short taps and hold thresholds (T3).
- IME composition, non-BMP text and delete do not duplicate characters (T4).
- Presets with unavailable game-state conditions do not produce invisible/unusable controls (T4).

## File map and shared types

`touch-host/src/main/java/io/github/wangzheruoyun/aiminecraft/touch/` holds host adapters only. Upstream source paths remain intact under `upstream/touchcontroller` (pinned checkout/submodule), with patches in `upstream/patches/touchcontroller/`. Build source sets include an audited module list, not the whole mod.

Types in `TouchTypes.kt`:
- `Viewport(left: Float, top: Float, width: Float, height: Float, density: Float)`.
- `TouchMode`: `Gameplay`, `Pointer`, `Edit`.
- `BindingMap`: immutable logical action → key/mouse binding map.
- `HostPointer(id: Int, x: Float, y: Float, phase: PointerPhase, timeNanos: Long)`; phases Down/Move/Up/Cancel.
- `LayoutId(value: String)` and `TouchProfile(instanceId: String, layoutId: LayoutId, bindings: BindingMap)`.

## Task T1: Close the actual upstream dependency graph

**Files:** create `touch-host/build.gradle.kts`, `upstream/touchcontroller` pinned source, `upstream/patches/touchcontroller/series`, `docs/integration/touchcontroller-host.md`, `touch-host/src/test/.../UpstreamJoystickTest.kt`.

**Interfaces:** produces buildable upstream `Context`, `Joystick`, `DrawQueue`, key-state/configuration modules and required generated factories. Exact upstream symbols stay unchanged unless a documented patch is necessary.

- [ ] Add an integration test instantiating the actual upstream Joystick, a host Context and pointer: center produces zero movement; right-edge drag yields negative `result.left`; pointer release yields no held movement in the next frame. This must execute upstream widget code, not a replica. Run `:touch-host:testDebugUnitTest` remotely and retain failure evidence.
- [ ] Enumerate Bazel dependencies for Context/Joystick/D-pad/buttons/presets/editor, generated factories, textures and text resources from the pinned revision. Record modules/providers and classify each as reusable, host-adapted or excluded Minecraft integration.
- [ ] Use upstream generation/build rules to produce resources/factory bindings where possible. Adapt packaging/visibility into an Android library; pin Kotlin compatibility, including upstream `kotlin.uuid` usage. Do not add empty player/window factories simply to get compilation green.
- [ ] Supply real host providers for only the required interfaces; convert game-state-dependent entrypoints into explicit host actions through reviewed patches. If the editor closure requires larger changes, document them before adding new code rather than silently dropping customization.
- [ ] Run upstream joystick test and Android library assembly. Inspect runtime classpath and bytecode with existing tooling (`jdeps`/ASM-based test) to assert no forbidden game/loader classes, including reflective bootstrap references. Preserve useful upstream tests.
- [ ] Commit reused modules, generated source provenance, patches and build evidence. If closure cannot compile, stop this task with exact dependency evidence; no placeholder controls.

## Task T2: Render upstream controls in Android

**Files:** create `touch-host/.../{TouchTypes,AndroidCombineCanvas,AndroidTextProvider,AndroidTextureProvider,TouchHostView,HostController}.kt`, `touch-host/src/androidTest/.../TouchRenderingTest.kt`.

**Interfaces:**
- `HostController.frame(timeNanos: Long, viewport: Viewport, pointers: List<HostPointer>): DrawQueue` uses upstream layout/configuration/state.
- `TouchHostView.attach(profile: TouchProfile, sink: InputSink)` and `setViewport(viewport: Viewport)`.
- `AndroidCombineCanvas` implements upstream Canvas; texture/text factories use Android APIs and licensed upstream resources.

- [ ] Write instrumentation tests rendering actual joystick/D-pad/buttons at two densities with clipped/scaled viewport. Assert expected control bounds, nonempty texture output, clipping and hit/draw coordinate agreement. Record failures before implementation.
- [ ] Implement Canvas operations through Android Canvas/Paint, including balanced state/clips, transforms, text measurement and resource/text adapters; implement upstream texture drawing contract through bitmaps. Reuse standard Android resource/image APIs.
- [ ] Host upstream layout and DrawQueue in TouchHostView, using Choreographer and an app-local frame clock. View draws only over the game viewport; Edit mode blocks delivery to the game.
- [ ] Run `:touch-host:connectedDebugAndroidTest` on Actions. Expected: all render/geometry tests pass and no Minecraft runtime is loaded.
- [ ] Commit renderer/resources and instrumentation evidence.

## Task T3: Translate actual control output and release input reliably

**Files:** create `touch-host/.../{HostKeyBindings,HostActions,PointerRouter,HostInputClock}.kt`, tests `HostInputTest.kt`, `PointerRouterTest.kt`, `HostTimingTest.kt`.

**Interfaces:** consumes D2 `InputSink` and T2 `HostController`; `HostActions.apply(result: ContextResult, timeNanos: Long)`, `cancelAll()`, `setMode(mode: TouchMode)`; `PointerRouter.onMotion(event: MotionEvent, viewport: Viewport): List<HostPointer>`.

- [ ] Write tests using a recording InputSink: simultaneous move/look/attack; releasing attack leaves movement held; CANCEL then reconnect emits no stuck/replayed input; opposite movement directions release before new press; press/release within one display frame survives in order; capture-mode switch while dragging releases mouse state.
- [ ] Run tests remotely and confirm expected failure. Implement host binding/action adapters over upstream key state and movement results; maintain ordered transitions rather than frame-only snapshots. Use an explicit dead zone configurable in profile, initial default 0.15 of normalized joystick radius.
- [ ] Reuse/patch upstream view gesture calculation to output relative motion without PlayerHandle/crosshair queries. Explicit attack/use controls replace target-dependent dispatch. Pointer mode routes absolute coordinates, buttons, wheel and modifiers to backend APIs.
- [ ] Adapt hold/tap timing to monotonic host time; write parameterized tests at 30/60/120 Hz asserting identical action outcomes and no short-tap loss. Persist no pending held state across activity/process restart.
- [ ] Wire cancellation on focus loss, ACTION_CANCEL, surface destruction, rotation and profile/mode changes. Respect the native input thread contract from D2.
- [ ] Run JVM tests and backend recording-consumer instrumentation for both GLFW/SDL. Commit only when event order and cancellation tests pass.

## Task T4: Complete text entry, bindings and reusable layout editing

**Files:** create `touch-host/.../{HostInputConnection,TouchLayoutStore,TouchLayoutEditor,VanillaPresetPolicy}.kt`, tests `HostInputConnectionTest.kt`, `TouchLayoutStoreTest.kt`, `TouchEditorTest.kt`.

**Interfaces:** `TouchLayoutStore.load(instanceId: String): TouchProfile`, `save(instanceId: String, profile: TouchProfile)`; `VanillaPresetPolicy.validate(layout: upstream layout model): List<UnsupportedCondition>`; editor consumes upstream configuration/preset/editor modules from T1.

- [ ] Write tests: remapping jump changes the emitted key; editor drag/resize/opacity/binding edits round-trip; malformed/imported unsupported conditions yield actionable errors; an interrupted save preserves prior valid layout; restoring a profile resets all held input.
- [ ] Implement storage with upstream serialization and atomic Android file/database facilities; adapt the existing editor to the Canvas/input/resource host, preserving upstream model/behavior. Always retain a visible mode/keyboard/settings recovery control.
- [ ] Provide Android IME connection using framework BaseInputConnection/composition handling and existing backend text APIs. Test composing/replacing/committing text, emoji/non-BMP characters, delete and enter; the game receives one committed string, not both composition previews and commits. Do not claim access to game's internal text selection.
- [ ] Implement vanilla presets covering movement/look/actions/hotbar/inventory/menu/chat/custom keys and pointer-mode inventory modifiers. Support manual mode selection when cursor capture is ambiguous. Map unsupported imported game-state conditions explicitly; never invent world/player state.
- [ ] Run all touch-host tests and editor/IME instrumentation in Actions; then hand the library to D3. Commit and retain the complete physical touch acceptance list for D4.
