# OpenDoodler cross-platform migration plan

Status: architecture baseline. Branch: `feature/cross-platform-desktop`. Original WPF app remains untouched.

## Objective

Ship the same OpenDoodler document format and animation behavior on Windows, Linux and macOS. Linux is the first runtime target; macOS is an explicit acceptance target from day one. No Linux-specific UI fork or rewrite of the C# domain.

## Verified baseline (main, 2026-10-26)

- `src/OpenBoardAnim` targets `net10.0-windows` with WPF. It contains Views, ViewModels, models, services, renderer, bundled Windows FFmpeg executables, and Windows-specific packages (HandyControl, MaterialDesignThemes, SharpVectors, NAudio).
- `src/OpenBoardAnim.Library` already targets `net10.0`: EF Core + SQLite persistence. `src/OpenBoardAnim.Utilities` targets `net10.0`.
- Existing `tests/OpenBoardAnim.Tests` depend on WPF and target Windows.
- Current CI executes only on Windows.
- Existing user experience includes canvas manipulation, SVG/text, drawing animation, scenes/timeline, audio, preview, MP4 export, undo/redo, recovery and settings.

## Architectural decision

**Preferred UI:** Avalonia (.NET 10 compatible version, verify before pinning) for one desktop shell shared by Windows/Linux/macOS. Avalonia is not a drop-in WPF binary replacement: XAML, controls, binding behavior, event handling and render code must be adapted.

**Dependency direction:**

```text
OpenDoodler.Desktop (Avalonia)
    -> OpenDoodler.Application (use cases, UI-agnostic state, commands)
        -> OpenDoodler.Core (project/scene/animation model, timeline math)
        -> OpenDoodler.Infrastructure (EF Core/SQLite, serialization, FFmpeg)
```

Application interfaces must not expose WPF/Avalonia types, `System.Windows`, `MessageBoxResult`, `Dispatcher`, `BitmapSource` or platform window handles. Place view models in Application only after untangling UI dependencies. Retain existing `OpenBoardAnim` WPF as a functional reference, not a permanent independent product implementation.

**Rendering and video:** first inventory current WPF geometry, SVG, stroke-order rendering, camera effects, and export/preview dependencies. Define renderer interfaces and an OS-neutral representation of shapes/animation frames. Evaluate SkiaSharp/SVG support for deterministic cross-platform frame generation; do not claim parity until golden-frame tests pass. FFmpeg must be resolved per OS and architecture (system binary or correctly licensed, verified bundle), invoked with argument lists instead of shell interpolation. Preview and MP4 must use the same timing and frame model.

**System adapters:** filesystem/user-data directory, file dialogs, clipboard, fonts/text shaping, screen scaling, audio output, timers, process invocation, logging and theme. Data path: use per-platform application-data directories with backward-compatible migration on Windows. Never hardcode `ffmpeg.exe` or `%LocalAppData%` in domain logic.

## Migration slices

1. **Inventory and characterization (first gate).** Create a dependency map with exact files and packages tied to WPF/Windows; document on-disk schema and project compatibility; capture representative baseline projects and expected export output; make an explicit parity checklist.
2. **Shared core.** Extract serializable project, scene, timeline and animation state plus pure calculations into platform-neutral `net10.0` projects. Keep the existing WPF build and tests passing; add OS-neutral tests on Ubuntu.
3. **Infrastructure and rendering contracts.** Extract database/files/FFmpeg interfaces; create common render/timing contracts with test fixtures; validate Linux output without GUI.
4. **Avalonia desktop MVP on Linux.** Open/save existing projects, show editor, display SVG and text, select/move/resize, undo/redo. Reproduce existing workflows before visual redesign. Keyboard shortcuts, HiDPI, keyboard focus and accessible labels are baseline quality requirements.
5. **Feature parity.** Multi-scene timeline, templates, audio preview, stroke animation, pan/zoom, transitions, video export and recovery; tests from the same reference projects.
6. **macOS verification.** CI/build on macOS and runtime smoke tests on actual macOS hardware where possible; validate Apple Silicon, application packaging, dialogs, font metrics and FFmpeg. Windows regression runs on each major slice.

## Explicit non-goals of the first slice

- Do not rewrite the repository into a different language or framework.
- Do not delete WPF or change existing `.db` records/project files.
- Do not promise cross-platform execution merely because `dotnet build` succeeds.
- Do not merge into main until the appropriate per-slice tests and review are complete.
- Do not modify editor UX before functional parity; make improvements through shared UX specifications rather than platform forks.

## Acceptance evidence

- Build and unit tests for new cross-platform projects on Ubuntu + Windows; add macOS runner as early as possible.
- WPF regression workflow continues to pass on Windows during migration.
- Same fixture project opens, edits, saves and reopens without data loss across OSes.
- Rendered reference frames checked at defined timestamps, including SVG, text, transforms and transitions; differences recorded and approved.
- Linux video has expected frame count, resolution, duration, audio alignment, and playable MP4 output.
- Document executable commands, CI run URLs, test output and any environmental limitations in each migration PR.

## First implementation PR boundaries

Only dependency inventory, architecture decision, compatibility baseline and initial OS-neutral test/build scaffolding. No bulk XAML replacement. Keep each PR focused, reviewable and reversible.
