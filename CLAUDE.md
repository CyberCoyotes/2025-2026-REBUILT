# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

FRC (FIRST Robotics Competition) robot code for Team 3603, built on WPILib + GradleRIO for the 2026 game season. Java 17, deployed to a roboRIO. This is `2025-2026-Rebuilt`.

## Commands

All commands run through the Gradle wrapper (`gradlew`/`gradlew.bat`) — there is no separate lint step.

- Build: `./gradlew build`
- Run all tests: `./gradlew test`
- Run a single test class: `./gradlew test --tests "frc.robot.SomeTest"`
- Simulate (desktop, with sim GUI): `./gradlew simulateJava`
- Deploy to the roboRIO: `./gradlew deploy` (requires the robot on the network; team number comes from `.wpilib/wpilib_preferences.json`, currently 3603)
- Replay a log through AdvantageKit: `./gradlew replayWatch` (runs `org.littletonrobotics.junction.ReplayWatch`)

There are currently no test sources under `src/test` — `./gradlew test` runs the (empty) JUnit 5 harness configured in `build.gradle`.

## Architecture

### Entry point and control flow

`Main.java` → `Robot.java` (extends AdvantageKit's `LoggedRobot`, not WPILib's plain `TimedRobot`) → constructs `RobotContainer` once. `RobotContainer` builds every subsystem, wires driver/operator controller bindings in `configureBindings()`, and registers autonomous routines with a Choreo `AutoChooser`. Look here first when tracing "what does button X do" or "what runs in auto."

`Robot.robotPeriodic()` is more than boilerplate — it owns the **vision pose-fusion math**: it reads MegaTag2 (falls back to MegaTag1) from `LimelightHelpers`, gates fusion on rotation rate (`OMEGA_FILTER_MAX_RPS`), scales the standard deviation by distance², and fuses into the drivetrain's pose estimator with `theta stddev = 9999999` (the gyro always owns heading, vision never corrects yaw). `VisionSubsystem` is deliberately just a *data provider* — it does not call `addVisionMeasurement` itself, to avoid double-counting the same measurement. If you touch vision fusion, read `Robot.java` and `VisionSubsystem.java` together.

### IO / Hardware abstraction pattern (every subsystem)

Every subsystem follows the same three-layer split — see `docs/io-hardware-subsystem-pattern.md`:

```
XxxIO (interface, default no-op methods) → XxxIOHardware (real CTRE/hardware impl) → XxxSubsystem (wraps IO calls as named Java methods) → command factories (call subsystem methods only)
```

Rule to internalize: **command factories never call other command factories** — they call subsystem methods. `XxxIOInputs` classes split fields into "fast" (read every 20ms cycle, control-critical) and "slow" (read at 10Hz via `updateSlowInputs`, diagnostics-only) to avoid re-reading CAN signals faster than they're actually published. There is no `XxxIOSim` implementation for any subsystem currently — only `Hardware`.

Subsystems: `intake`, `indexer`, `shooter`, `vision` (all under `src/main/java/frc/robot/subsystems/`), plus `CommandSwerveDrivetrain` (CTRE swerve-generator output, extended with a custom `resetPoseFromVisionCommand()`). `generated/TunerConstants.java` is generator output — regenerate via the CTRE Tuner X swerve project generator rather than hand-editing when drivetrain hardware changes.

The shooter (`ShooterSubsystem`) is the one subsystem with a real finite-state machine (`ShooterState`: IDLE/STANDBY/SPINNING_UP/READY/PASS/EJECT/POPPER) and is the reference implementation for FSM patterns in this codebase — the indexer's state enum is closer to a status label than a true FSM.

### Commands

Command factories live in `src/main/java/frc/robot/commands/` (`FuelCommands`, `AlignAndShootCommand`, `AlignOnlyCommand`) as static methods taking subsystems as parameters. `AlignOnlyCommand` exists specifically so rotation/vision alignment can be tuned in isolation without engaging the flywheel/hood — prefer it over `AlignAndShootCommand` for PID tuning work.

### Constants

`Constants.java` is a single file with one `static final class` per subsystem (`Intake`, `Indexer`, `Flywheel`, `Hood`, `Shooter`, `Vision`, `Led`, `Auto`), each with a private constructor. CAN IDs are documented in a quick-reference comment block at the top of the file — cross-check `TunerConstants.java` for the CANivore (swerve) bus IDs, since those are generator-owned. Many constants carry inline tuning-history comments (`// Tuned 4-24-2026`, `// was 2.25`) — preserve that trail when changing a value; don't just overwrite silently.

### Autonomous

Choreo trajectories + `AutoFactory`/`AutoRoutine` (`AutoRoutines.java`, ~1100 lines) define named routines; only a subset are actually registered with `autoChooser` in `RobotContainer` — the chooser registration list is the source of truth for what's live, not the full set of methods in `AutoRoutines.java` (many are commented out or superseded). Trajectory files come from Choreo (`choreo_export.csv`, generated via `export_choreo_csv.py`).

### Telemetry / dashboards

Two logging paths run in parallel and are both intentional:
- **AdvantageKit** (`Logger.recordOutput`, `Logger.processInputs`) — for replay and post-match log analysis.
- **Raw NetworkTables publishers** per subsystem (e.g. `shooterTable.getDoubleTopic(...).publish()`) — for the **Elastic** dashboard (`elastic-dashboard.json` / `elastic-layout.json` at repo root), which doesn't read AKit logs directly.

`Telemetry.java` / `GameDataTelemetry.java` / `ScoringTelemetry.java` / `HubActiveState.java` (under `utilities/`) provide additional dashboard/game-data publishing. `PhoenixUtil.applyConfig()` wraps CTRE config `apply()` calls with 5x retry — RIO CAN can return OK prematurely while a device is still booting; use it instead of calling `.apply()` directly on Phoenix 6 configs.

`LimelightHelpers.java` is a vendored third-party helper (from Limelight, ~1950 lines) — don't restyle it to match this repo's comment conventions; treat it as external code, upgraded in place when Limelight ships a new version.

### Vendor dependencies

Declared in `vendordeps/*.json`: AdvantageKit, ChoreoLib (2026), CTRE Phoenix 6 (swerve, replay), CTRE Phoenix 5 (replay-only, legacy), libgrapplefrc, WPILib New Commands.

## Conventions

- **Comment style** is documented in `docs/style-guide_comments.md` — read it before making non-trivial edits to any subsystem/command/`RobotContainer`/`Constants` file. Key points: 3-line `====` banners only for major sections (not tiny groupings), Javadoc on public APIs, no restating-the-obvious inline comments, `TODO`/`FIXME` must name the specific thing that needs to happen (not "TODO tune").
- Controller bindings live only in `RobotContainer.configureBindings()` — `docs/controller-reference.md` exists but has drifted from the current bindings; treat the code as the source of truth for what's actually bound, not that doc.
- `.gitattributes` normalizes all text files to LF (`*.bat` stays CRLF) — don't fight this by hand-setting line endings.

## docs/ folder

Beyond the style guide, `docs/` contains a large post-season retrospective binder (`RETROSPECTIVE_*.md`, one per subsystem/area, indexed by `docs/RETROSPECTIVE_README.md`) plus `docs/OFFSEASON_BACKLOG.md`, an ordered backlog compiled from every retro's "lessons for next season." These are historical/planning documents, not living specs — treat them as context on *why* the code looks the way it does and what's already a known trade-off, not as instructions to refactor unprompted. Tuning guides (`docs/tuning-guide_*.md`) and hardware notes (`docs/hardware-reference.md`, `docs/hardware-LED-specs.md`) are more operational and worth checking before changing motor/PID configuration.
