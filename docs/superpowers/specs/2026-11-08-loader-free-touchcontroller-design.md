# Loader-free TouchController adaptation

Status: approved for implementation planning by the user's subsequent “Continue.”
This document replaces
the touch sections of `2026-11-08-ai-minecraft-android-design.md`; other launcher
requirements, including current Minecraft, mod management and public distribution,
remain in force.

## Requirement and authorized scope

The user requires full touchscreen gameplay for completely unmodified vanilla
Minecraft, reusing TouchController without a mod loader. After the assistant
asked permission for the necessary launcher-side adaptation, the user said
“Continue.” The working authorization is to adapt TouchController, not to rewrite
its controls or relax the unmodified-vanilla requirement.

Vanilla game JARs/classes must not be patched, transformed, or injected with a
Java agent/mixin. Runtime/native libraries provide Android compatibility through
existing components. No loader or control mod is inserted into vanilla profiles.
Optional modded profiles retain explicit, user-selected loader/mod management.

## Approaches

1. **Selected proposal: host reusable TouchController components in the Android
   game activity.** Adapt their rendering/resource/action boundaries to Android
   and existing GLFW/SDL inputs. This meets the no-loader requirement while
   reusing the control implementation and keeping changes reviewable.
2. **Existing Android proxy + TouchController game mod:** well-documented upstream
   integration but rejected for vanilla because it needs a game-side component.
3. **Load the mod through another bootstrap or Java agent:** rejected because
   bypassing the conventional loader does not leave the running game unmodified.

The first approach is a proposed port, not an existing ready-to-use artifact or
a claim of proven compatibility. The adaptation may be substantial; do not
relabel a newly written control engine as reuse.

## Evidence and source baseline

Research baseline: TouchController/TouchController
`f8128cc2b70aa98b8e2a7da422a5eeb110c6001c`.

Inspected source:
- `combine/core/paint/Canvas.kt`: transform, shape, text and clipping interface.
- `touchcontroller/common/layout/Context.kt`: layout context with pointers,
  dimensions, key-binding handler, status, configuration and draw queue.
- `touchcontroller/common/layout/queue/DrawQueue.kt`: queues Canvas draw calls.
- `touchcontroller/common/control/widget/joystick/Joystick.kt`: existing pointer
  ownership, joystick geometry, textures and movement output in ContextResult.
- `touchcontroller/common/gal/key/KeyBindingHandler.kt`: reusable action/key state
  interfaces with click/hold lifecycles.
- `touchcontroller/common/layout/data/ContextInput.kt`: includes dependencies on
  PlayerHandle, game conditions, perspective and crosshair target.
- `touchcontroller/common/layout/widget/game/View.kt`: camera gestures also query
  player/item/target state to choose attack/use behavior.
- `touchcontroller/common/layout/widget/game/Inventory.kt`: reads player-selected
  hotbar state; not a standalone vanilla inventory adapter.
- `touchcontroller/versions/fabric/26.3/TouchController.kt`: mod callback lifecycle
  and Minecraft render integration, excluded from the vanilla host.

The research checkout is at `/tmp/opencode/touchcontroller-research` and is not
product source. Current branch contents must be pinned before implementation.
These files demonstrate reusable boundaries but not a closed dependency graph:
generated factories, texture/text providers, UI and game abstractions still need
to be resolved during the first integration task.

## Runtime architecture

```text
Android :game process
  Game Activity
    game SurfaceView -> existing GLFW or SDL -> vanilla game JVM
    TouchController host overlay (Android ART)
      reused configuration/layout/widget logic
      adapted Combine Canvas/resource/text providers
      reused pointer and key-state logic
      host key/mouse action adapters -> GLFW or SDL input API
      Android IME -> existing backend text input API
```

The overlay is an ordinary child view above the app's own game surface; no system
overlay or accessibility service is needed. It runs on Android ART and never
loads Minecraft classes. The game runs in its embedded JVM with existing native
bridges. All overlay input is sent through explicit backend interfaces rather
than passing ART objects into HotSpot.

## Reuse boundaries and implementation responsibilities

| Reuse from TouchController/Combine | Required host adaptation |
| --- | --- |
| Layout/context/alignment and widget configuration | Supply app-local viewport, density, insets and settings |
| Joystick, D-pad and generic button behavior where dependency closure permits | Translate movement/action output into existing keyboard/mouse events |
| Draw queue and Canvas abstraction | Android Canvas/Paint text, shape, clip and transform adapter; adapt texture drawing through Android bitmap APIs |
| Text/resource/texture abstractions and assets with appropriate licenses | Android resource/text measurement/provider implementations |
| Pointer state and button press/hold lifecycle | Feed Android pointer events and advance host input timing |
| Preset/configuration serialization and editor logic that is host-independent | Persist in app-private instance data; host the existing editor through adapted rendering/input interfaces |

Use a pinned fork/patch set or pinned source dependency with preserved source paths
and attribution. Maintain an explicit list of reused modules and modified files.
No complete launcher application source is imported. Do not substitute a new
Compose joystick for the existing joystick while calling it a TouchController port.

Generated `ExpectFactory`/implementation bindings must resolve to host providers
using existing upstream tooling where feasible. Upstream visibility/build metadata
may need adaptation to package these modules as an Android library. Exclude the
Minecraft backend, loader entrypoints, mixins, and game-specific rendering code
from the Android runtime dependency closure.

A separate dependency-closure task must prove the host can compile without
`net.minecraft`, Fabric or NeoForge runtime dependencies before native gameplay
integration. If a needed component is inseparable without a significant redesign,
report the exact component and proposed adaptation; do not hide it behind a stub.

## Input behavior for unmodified vanilla

Full gameplay means all normal player actions are accessible from touch. It does
not mean reproducing automatic game-state awareness supplied by the mod.

- Movement: reused joystick/D-pad outputs become W/A/S/D press/release states,
  respecting configured bindings. Vanilla keyboard input is digital: do not
  promise analog movement without a game modification. Apply bounded dead zones
  and release opposite directions deterministically.
- Look: relative drag motions feed the backend's mouse motion mechanism while
  captured. Preserve upstream reusable gesture calculations; adapt game-state
  dependencies explicitly rather than inventing player/target observations.
- Combat/building: separate attack and use buttons map to normal mouse buttons;
  hold works for breaking/using, tap for clicks. Never guess whether the aimed
  object is an entity or block.
- Jump/sprint/sneak/fly/swim: normal vanilla bindings and press/hold/double-press
  combinations. Keep required actions accessible without knowing player state.
- Hotbar: explicit 1–9 selection and wheel events. Display input intent rather
  than claiming authoritative knowledge of the selected slot.
- Inventory/crafting/containers: use the game's normal GUI with absolute pointer,
  left/right click, drag, shift-click, scrolling and number-key actions. Do not
  reimplement inventory or inspect its internal slot/item model.
- Chat/text: provide a visible keyboard button and Android IME bridge. Text goes
  through the backend's character/text API, editing/navigation through key events.
  Verify composition/Unicode and avoid duplicate character delivery. The host
  does not pretend to know vanilla text selection/composition internals.
- Menus: escape/pause and pointer control remain available at all times.
- Perspective/drop/offhand/player list and user-configurable keys: ordinary key
  bindings exposed as existing generic buttons.

Use backend cursor-grab notifications as a hint for gameplay versus pointer mode;
provide an explicit user toggle because capture state does not identify every
screen. The generic controls are always recoverable. No screen OCR or memory
inspection is introduced to infer hidden state.

Support standard vanilla key bindings and a configurable binding map. External
changes to game bindings must be reflected through an existing configuration
reader if suitable or through explicit user settings; do not assume default keys
after the user remaps them. Persist mappings per instance.

## Host timing and lifecycle

The overlay has a host frame/input clock, not a simulated Minecraft client tick.
Adapt upstream timer/key-state call sites to host time while preserving configured
gesture durations; verify behavior at 30/60/120 Hz. Input event ordering must
preserve short press/release pairs across frames.

Each pointer can own a control; overlapping controls must not steal or duplicate
events. Both attack/use buttons and movement/look may be active concurrently.
Clear held buttons, movement and pointer ownership on ACTION_CANCEL, focus loss,
surface teardown, profile change and process exit. Reconnect must not replay old
held input. Calls into the native backend follow its documented thread rules.

Use the actual rendered viewport (including scaling/letterboxing/insets) for
coordinate transforms. Layout editing pauses input delivery to the game. The
game surface and overlay must share rotation/resize geometry.

## Control layouts

Reuse TouchController's layout/preset models and suitable existing editor logic.
Support moving/resizing controls, opacity, bindings and saving/restoring per-
instance layouts. Host adapters must supply text, texture, clipboard, storage
and input dependencies needed by the editor. This is part of the port's dependency
audit, not assumed to work merely because the layout model exists.

Vanilla-safe presets use explicit controls and manual mode switching. Conditions
that require player/world state are not silently populated with guessed values.
If importing an upstream preset that relies on such conditions, explain which
conditions cannot be evaluated and require a user-visible compatible mapping.
Do not import a preset and leave its controls invisibly unusable.

## GitHub Actions and deliverables

After design approval, implementation planning will create the project repository
under the authenticated GitHub account and run heavy compilation/testing remotely.
Use the existing TouchController Bazel/build tooling where it fits, existing native
component build workflows and conventional Android Gradle packaging.

Internal integration checks (not reduced public milestones):
1. Build pinned TouchController host modules as an Android library, with no loader
   or Minecraft dependency in their runtime classpath.
2. Render reused widgets and editor paths in a host test screen and test pointer
   ownership, state transitions, coordinates, timing and cancellation.
3. Run the same overlay against GLFW and SDL input adapters; verify received
   ordered key/mouse/text events using an instrumented test consumer.
4. Launch unmodified vanilla on the Java 21/GLFW and current stable Java 25/SDL
   paths. Verify vanilla JAR hash, selected main class, game classpath and launch
   flags contain no injected control mod, loader, mixin or instrumentation agent.
5. Complete touch-only gameplay acceptance on physical Android 14 hardware,
   followed by the complete launcher/AI/mod-management/public-release checks.

Device acceptance: navigate menus, create/load world, move/look simultaneously,
combat, mine/place, jump/sprint/sneak/swim/fly, hotbar/offhand/drop, craft and move
items between containers, type chat/world names, pause/resume, edit/save/reload
layout, and survive Android backgrounding/rotation without stuck controls.
Latest stable game and representative older game must both pass. A screenshot
of an overlay or a successful build is not proof of playable vanilla support.

## Remaining setup and acceptance facts

- Android 14 is known; phone model/GPU/RAM and device-test access remain unknown.
- Publisher Microsoft registration and release signing identity remain setup tasks.
- Android port compilation, game execution and gameplay have not been performed.
- Public delivery includes full touch, newer Minecraft, mod management and the
  external-AI JSON workflow; this adaptation is one part of that deliverable.

## Builder-owned choices

Module names, packaging layout, compatible dependency versions, exact pinned
revisions of other libraries, native input thread dispatch and texture caching
are engineering choices, to be established with source/API and test evidence.
No permission to weaken the unmodified-vanilla or TouchController-reuse requirement
is implied by an implementation detail.
