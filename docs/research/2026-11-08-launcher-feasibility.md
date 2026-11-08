# Minecraft Java Android launcher: feasibility research

Status: research only; Termux permanently excluded by the user; self-contained
Android runtime feasibility remains unresolved; no gameplay verified.

## Requirements

- Android launcher for Minecraft Java Edition.
- User writes a natural-language request, copies an app-generated specialized
  prompt to an AI of their choice, and pastes the resulting JSON into the app.
- The app executes supported instructions described by that JSON.
- Prefer established, maintained, high-star projects; do not reuse existing
  Minecraft launchers.
- HARD CONSTRAINT (user decision): Termux is permanently ruled out. Do not use
  Termux, Termux:X11, Termux runtime packages, or a bundled/hidden Termux-based
  environment. Do not propose Termux again as a fallback. Independent upstream
  projects such as OpenJDK and LWJGL remain candidates. The user's prohibition
  covers existing Minecraft launchers. After the assistant asked whether separate
  component repositories maintained by launcher teams may be used, the user said
  “Continue.” The assistant explicitly proceeded with evaluating those components.
  This is the working interpretation for the proposed design; it does not allow
  copying complete launcher applications.
- Minimize custom code to integration. This working interpretation was stated
  after the user's instruction to continue; it is not permission to build a new
  JVM, graphics backend, or general-purpose framework.
- Use Firecrawl for research and authenticated GitHub CLI / Actions for as much
  build and verification work as possible.
- Subsequent user clarification: full touchscreen gameplay is required; the first
  milestone must include newer Minecraft versions and mod management with full
  functionality; known test OS is Android 14; target audience is public.

## Environment

- Workspace was empty at initial inspection.
- GitHub CLI authenticated as `wangzheruoyun`; workflow/repository access present.
- No repository created and no Actions jobs started during research.

## Evidence

The Termux findings below are retained solely as historical research about a
rejected approach. They are not approved dependencies or implementation guidance.

Firecrawl searches and upstream page retrieval were supplemented with `gh api`
to inspect exact build recipes and current repository metadata. Star counts
below were returned by GitHub on 2026-11-08; they are not compatibility evidence.

| Project | Stars | Potential role | Findings |
| --- | ---: | --- | --- |
| termux/termux-app | 64,411 | External runtime host | Documented third-party RUN_COMMAND intent and result callbacks |
| termux/termux-packages | 17,320 | Maintained JVM/native packages | OpenJDK 21 and versioned LWJGL 3 recipes exist |
| termux/termux-x11 | 5,103 | Display and input | Android 8+; app and companion package both required |
| androidx/androidx | 6,130 | Native Android UI, persistence, lifecycle/work | Candidate; dependency versions not selected |
| Kotlin/kotlinx.serialization | 5,950 | Typed JSON decoding | Candidate; command schema still needs design |
| LWJGL/lwjgl3 | 5,491 | Java native graphics/audio bindings | Termux builds from upstream LWJGL tags |
| ptitSeb/gl4es | 864 | Desktop GL translation | Describes OpenGL 2.x support; insufficient evidence for arbitrary modern game versions |
| OptimumCode/json-schema-validator | 66 | Kotlin JSON Schema validation | Small project; evaluate against more established compatible validators |

The request for `square/okhttp` metadata redirected to `lysine-dev/okhttp`
(47,106 stars). Verify ownership/release provenance before selecting artifacts;
the redirect alone does not establish suitability.

### Runtime details

- `packages/openjdk-21/build.sh` reports OpenJDK 21.0.13 and depends on Android/
  Termux-specific libraries. Its paths target the Termux installation prefix.
  These binaries cannot simply be assumed relocatable into another APK.
- `packages/lwjgl3/build.sh` builds versions 3.2.1, 3.2.2, 3.3.1, 3.3.2, 3.3.3,
  3.3.6, 3.4.1, 3.4.2, and 3.4.3 from upstream LWJGL source and per-version
  patches. JARs and native libraries are installed in version-specific paths.
- Issue #32975 reports problems with LWJGL >=3.3.3, including GLFW functions and
  jemalloc. A search excerpt is not proof of current resolution or universal failure.
- RUN_COMMAND requires the Android permission and the user's Termux setting
  `allow-external-apps=true`. Its documented API supports argument arrays and
  PendingIntent results. Arbitrary AI-produced shell strings are unnecessary:
  application-owned handlers can map typed operations to fixed executables.
- Termux:X11 documents mouse/touch input and third-party integration. Its source
  is GPL-3.0. Licensing and packaged dependencies need per-component review before
  redistribution, particularly for any embedded design.
- Upstream GLFW documents desktop platforms. This research has not identified a
  complete, verified drop-in native Android Minecraft runtime independent of
  launcher-derived compatibility layers.

## Architecture disposition after user decision

1. Native Android frontend plus Termux and Termux:X11: permanently rejected by
   the user. This also excludes embedding their runtime to conceal the dependency.
2. Self-contained Android APK integrating independently maintained runtime and
   display components: the remaining direction to investigate. Packaging, native
   library paths, process lifecycle and input integration require feasibility
   research. This is not yet established as achievable with minimal glue alone,
   and no full design has been approved.

The external-AI prompt/JSON workflow remains a core requirement.

## Unresolved implementation facts

- A viable independently sourced JVM, graphics, windowing and input combination
  that uses neither Termux nor existing Minecraft launcher implementations.
- Initial Minecraft version / GPU / ABI compatibility matrix.
- A reusable non-launcher installation/metadata library and authentication flow.
- First supported command set and failure semantics.
- Device-level launch, render, audio and input evidence.

## Independent follow-up research after Termux exclusion

Firecrawl identified the official OpenJDK Mobile project as a candidate:
https://openjdk-mobile.github.io/openjdk-mobile/ describes an official downstream
of OpenJDK with Android/iOS patches. This search result does not establish a
production-ready Android runtime for the required Minecraft version. The older
https://openjdk.org/projects/mobile/android.html documentation specifically
describes JDK 9, so it must not be treated as evidence for current JDK support.

A separate search returned https://docs.neoforged.net/primer/docs/26.3/ reporting
a change from GLFW to SDL. Verify this against actual game artifacts and official
version metadata before selecting a game version or windowing integration. SDL
is only a candidate; its availability does not prove LWJGL/JVM/Android compatibility.

### Verified follow-up: Minecraft 26.3 and SDL

Mojang's version manifest returned `26.3` as the latest release and
`26.4-snapshot-7` as the latest snapshot during this research. The immutable
version metadata URL returned for 26.3 was:
https://piston-meta.mojang.com/v1/packages/80608ff6753705b6a82e2cb86a5937d4d233abd4/26.3.json

Its `javaVersion.majorVersion` is 25. Its libraries include LWJGL 3.4.3 with
SDL, OpenGL, Vulkan, shaderc, SPIRV-Cross, VMA, OpenAL, FreeType, jemalloc and stb.
The retrieved native classifiers cover desktop operating systems; this is not
an Android native distribution. Actual game execution has not been inspected.

NeoForge's 26.3 migration documentation confirms the GLFW-to-SDL API transition.
This makes a 26.3-focused SDL path worth investigating, rather than assuming
that a custom GLFW Android backend is required for every Minecraft version.
Older GLFW-based versions would need their own verified integration path.

SDL's official Android documentation describes Android AAR/Prefab integration,
subclassing SDLActivity, native build support, lifecycle handling, and JNI thread
attachment. These are reusable facilities, not proof that a separate HotSpot JVM
and LWJGL will interoperate with Android's ART/JNI state without additional work.

`libsdl-org/SDL` had 16,948 GitHub stars and was not archived at inspection.
Source: https://wiki.libsdl.org/SDL3/README-android

### Corrected assessment: OpenJDK Mobile Android readiness

The general OpenJDK Mobile overview is insufficient evidence of usable Android
builds. The platform-specific documentation explicitly says:

> Work for Android is still TODO.

It lists defining the build process and establishing Android CI artifacts as
next steps. The roadmap likewise lists starting Android builds as medium-term
work. This establishes a documentation/readiness gap, not proof that no source
revision can build or that the project is technically impossible.

- https://openjdk-mobile.github.io/android/
- https://openjdk-mobile.github.io/roadmap/

`openjdk/mobile` had 217 GitHub stars and was not archived at inspection. The
listed workflow filenames did not show an Android-specific workflow; this alone
does not establish that Android logic is absent from all workflows.

### Superseded proposal: custom runtime feasibility spike

The assistant proposed a bounded, throwaway runtime feasibility spike on GitHub Actions before
full product implementation. Candidate scope: upstream OpenJDK Mobile, upstream
LWJGL 3.4.3, SDL Android AAR, one Android ABI and Minecraft 26.3 requirements.

The spike should establish in order:

1. Whether a pinned upstream JVM source can cross-compile for Android using an
   existing documented recipe without importing excluded patches.
2. Whether the resulting JVM can load inside an Android app and execute a Java
   test class (build success alone is not sufficient).
3. Whether required LWJGL JNI bindings can load and an SDL surface can render
   through a compatible graphics path while delivering input.
4. What remains to reach actual Minecraft launch, including dependent native
   modules and device/GPU requirements.

If a step requires a new runtime port, substantial native implementation or
launcher-derived compatibility code, report the exact gap and stop that path.
Do not represent a UI-only APK or an SDL test as a functioning Minecraft launcher.
Actions may build and run suitable emulator checks; ARM64 device/GPU results need
a suitable device or runner. No such runner has been established in this session.

This spike has not been authorized or started. No repository, workflow, runtime
binary, or APK has been created.

## Correction after user feedback: existing component repositories

The user pointed out that repositories already exist for the proposed work.
Further Firecrawl discovery and GitHub README/release/workflow inspection verified
that the earlier focus on upstream OpenJDK Mobile was too narrow. Its Android TODO
does not establish absence of usable downstream Android JDK projects.

| Repository | Verified existing work | Qualification |
| --- | --- | --- |
| FCL-Team/Android-OpenJDK-Build | README lists JDK 8/11/17/21/25 Android branches; Build_JRE_25 contains .github/workflows/build.yml | Separate runtime build repository maintained by a launcher team; 18 stars; license/provenance and permitted-use boundary need resolution |
| MojoLauncher/android-openjdk-build-17-25 | Published rolling release lists jre17-pojav.zip, jre21-pojav.zip, jre25-pojav.zip | Separate runtime repository; 11 stars; README contains older instructions; artifacts not executed or audited |
| Project-crafting/openjdk-android-hostcraft | Published multi-ABI runtime archives; existing build/verification scripts; README describes JNI_CreateJavaVM embedding and no Termux sibling libraries at runtime | Redirected from artruntime/openjdk-android-hostcraft; 2 stars; headless-only documentation; launcher-team patch ancestry explicitly disclosed; device claims are upstream claims |
| LWJGL/android-test | Existing Android AAR build instructions, gears and Vulkan sample apps | 36 stars; last pushed 2023-08-30; references lwjgl3 android branch; not evidence of current game binding compatibility |
| MobileGL-Dev/MobileGlues | Existing desktop GL-over-GLES implementation aimed at Minecraft; source linked by MobileGlues-release | Release README states LGPL-2.1 and GLES 3.0 minimum / 3.2 preferred; evaluate embedding and exact version compatibility |
| RaphiMC/MinecraftAuth | Device-code auth, token lifecycle, serialization, published Maven library | 146 stars; LGPL-3.0; Android dependency compatibility and application registration need verification |

An older PojavLauncherTeam/lwjgl3-glfw-java component also exists, but GitHub
marks it archived, last pushed 2020-12-31. It is not a selected dependency.
Likewise xCuri0/glfw-android-example exists but was last pushed 2017-11-04.
Repository existence alone does not establish suitability or maintenance.

Next step is component selection and integration-contract inspection, using
existing artifacts and CI recipes where suitable. The custom upstream-runtime
spike is no longer the recommended starting point. Narrow runtime repositories
often have modest stars; high-star foundational libraries remain preferred.

Follow-up: the user said “Continue” after this boundary question. The working
interpretation is to evaluate separately published components, including those
maintained by launcher teams, while excluding complete launcher applications.

## Integration-contract findings and design progress

- MojoLauncher/glfw has a separate `jni_bindings` Android module and exposes
  surface creation/destruction, key/mouse/scroll injection, clipboard/cursor,
  grab-state and initialization callbacks. The default branch was `glfw34`.
- MojoLauncher/MojoSDL explicitly documents desktop-app Android adaptations,
  Minecraft 26.3+ offscreen-window support, MojoExec integration and separate
  JNI bindings. These are upstream claims, not execution results from this session.
- MojoLauncher/mojoexec is MIT-licensed and has a small Java JNI API for EGL,
  display and native library configuration. It must not be mistaken for a
  complete JVM bootstrap merely because of its name.
- MojoLauncher/unilwjgl3-builder contains Android architecture mappings,
  toolchain/libffi/dyncall build integration and existing Actions workflows.
- FCL-Team/MobileGlues-Build documents existing native AAR packaging; MobileGlues
  itself has a Gradle build. Its wrapper follows a development branch, so its
  mutable release is not automatically the production dependency choice.
- FlowArg/FlowUpdater is a standalone installation library (GPL-3.0) with
  Maven publication, Java 8 output and version/assets/library installation APIs.
- FlowArg/OpenLauncherLib supplies launch-profile facilities, but its README's
  old example is insufficient evidence of current Minecraft/Android compatibility.

The proposed design is at
`docs/superpowers/specs/2026-11-08-ai-minecraft-android-design.md`.
A fresh builder review returned four product/environment questions: first-release
touch controls, initial game/action scope, physical-device testing, and permitted
Microsoft client identity plus entitled test-account availability. No product
code or GitHub repository has been created pending design approval.

## Sources

- https://github.com/termux/termux-app/wiki/RUN_COMMAND-Intent
- https://github.com/termux/termux-x11
- https://github.com/termux/termux-packages/blob/master/packages/openjdk-21/build.sh
- https://github.com/termux/termux-packages/blob/master/packages/lwjgl3/build.sh
- https://github.com/termux/termux-packages/issues/32975
- https://github.com/glfw/glfw
- https://github.com/LWJGL/lwjgl3
- https://github.com/ptitSeb/gl4es
- https://github.com/OptimumCode/json-schema-validator

## Design progress

### Loader-free adaptation authorized for design

The user said “Continue” after being asked to authorize the necessary
launcher-side TouchController backend. Working interpretation: this specific
port/adaptation is allowed; other components remain reuse-first.

A sparse source checkout of TouchController at research baseline
`f8128cc2b70aa98b8e2a7da422a5eeb110c6001c` was inspected at
`/tmp/opencode/touchcontroller-research`. Canvas, DrawQueue, Context, Joystick and
KeyBindingHandler provide concrete reuse boundaries. ContextInput, View and
Inventory retain game-state coupling, so the host needs explicit key/mouse actions
and a vanilla-safe context rather than pretending those APIs exist outside the game.

The adaptation design is
`docs/superpowers/specs/2026-11-08-loader-free-touchcontroller-design.md`.
A fresh builder check found no remaining product/architecture questions; device
model/access and publisher auth/signing setup remain delivery prerequisites.
The document awaits user approval before implementation planning. No product code,
repository creation, build, APK or gameplay test has been performed.

### Full-function scope follow-up

Later user requirement: completely unmodified vanilla must have full touch by
reusing TouchController without a mod loader. The prior mod-based vanilla route
is rejected. Investigation of the current upstream source, not just README,
shows the Android AAR forwards events to game-side controls; Fabric entrypoints
register HUD/tick callbacks, versioned GameActionImpl calls Minecraft directly,
and the version build depends on mixins and game-specific implementations.
No standalone launcher-side rendering backend was found in the inspected tree.
A Firecrawl search for standalone vanilla TouchController returned the upstream
mod and unrelated results; it did not establish a usable external port.

Evidence:
- https://github.com/TouchController/TouchController/blob/master/touchcontroller/versions/fabric/26.3/TouchController.kt
- https://github.com/TouchController/TouchController/blob/master/touchcontroller/versions/26.3/BUILD.bazel
- https://github.com/TouchController/TouchController/blob/master/touchcontroller/versions/gal/action/26.3/GameActionImpl.kt
- https://github.com/TouchController/TouchController/blob/master/touchcontroller/proxy/client/android/TouchControllerLayout.kt

Conclusion: loader-free reuse is a porting/integration question, not a property
of the current Android proxy artifact. Reusable configuration/layout abstractions
exist, but an Android renderer and external input/action backend have not been
established. No custom port or game modification has been implemented. The design
records this as a blocking gap against the minimal-custom-code restriction.

The keyboard/mouse-first, vanilla-only milestone has been rejected by the user.
The proposed design now includes current stable Minecraft/Java 25/SDL and a
representative Java 21/GLFW profile, full touch and mod management in first delivery.

TouchController's current README lists 1.21.1, 1.21.10, 1.21.11, 26.1, 26.1.1,
26.1.2, 26.2 and 26.3 with Fabric/NeoForge. It documents the independently
published `top.fifthlight.touchcontroller:proxy-client-android:0.0.5` library,
UnixSocketTransport, LauncherProxyClient, `TOUCH_CONTROLLER_PROXY_SOCKET`, and
TouchControllerLayout. Source inspection confirms pointer/cancel handling and
keyboard/text callback interfaces. These are reuse opportunities; not claims
that this session tested gameplay or that every listed loader build is released.

Source: https://github.com/TouchController/TouchController/blob/master/touchcontroller/README.md

Important distinction: the touch UI lives in a client-side mod. Using this route
for vanilla gameplay requires a loader/mod; unmodified vanilla support needs a
separate reusable touch-overlay solution if required.

Modrinth4J is an existing MIT Java API wrapper listed in Modrinth's awesome list
and published through Maven Central; GitHub reported 25 stars and a 2026-01-05
last push. Android compatibility and resolver coverage are unverified. FlowUpdater
already documents Fabric/Forge/NeoForge and Modrinth mod installation facilities.
Source: https://github.com/masecla22/Modrinth4J

- [x] Inspect workspace and GitHub authentication.
- [x] Research independent runtime components using Firecrawl.
- [x] Verify key upstream integration contracts and package recipes.
- [x] Record the user's permanent exclusion of Termux and all Termux-based options.
- [ ] Establish feasibility of a self-contained runtime under the remaining constraints.
- [ ] Set initial compatibility and supported-command scope.
- [x] Write proposed full design and obtain a fresh builder review.
- [ ] Obtain user approval of the design.
- [ ] Write implementation plan; execute builds/tests through GitHub Actions.
