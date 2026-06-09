# metro-setup-skill

A reusable Claude Code skill for adding the [Metro](https://github.com/ZacSweers/metro) DI framework as a dual-build option in Android projects.

## Supported project types

| Current DI | Path | Status |
|---|---|---|
| Anvil + Dagger | Path A | Full support |
| Plain Dagger | Path B | Full support |
| Hilt | — | Not supported (too complex for automation) |

## What it does

- Detects your current DI framework (Anvil/Dagger or plain Dagger)
- Chooses the correct Metro version based on your Kotlin version
- Adds a `ddg.di` qualifier to `gradle.properties`
- Sets up conditional source sets for dual-build coexistence
- Creates Metro-specific DI graph files (`AppComponent`, `AppComponentFactory`, `ActivityComponent`)
- Configures Gradle plugin application (Anvil hook or direct, depending on path)
- Updates KSP processors (Anvil path only)
- Adds `@HasMemberInjections` lint enforcement
- Configures CI matrices and nightly Metro workflows
- Handles Develocity scan tagging for cache separation

## Usage

Copy `metro-setup.md` into your project's `.claude/skills/` directory, then invoke:

```
/metro-setup
```

Or ask Claude: "add Metro DI to this project as a dual-build option"

The skill will auto-detect your DI framework and follow the appropriate path.

## Verification

```bash
# Default DI framework
./gradlew assembleInternalDebug

# Metro
./gradlew assembleInternalDebug -Pddg.di=Metro
```

## Requirements

- Kotlin 2.0+
- JDK 21+
- Android project using Anvil+Dagger or plain Dagger
- **Not** Hilt (architectural migration required — see skill file for details)

## Known Limitations

See the "Known Limitations" section in the skill file for details on import ordering, annotation placement, build config classes, and Kotlin version overrides.
