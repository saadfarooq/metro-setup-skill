# metro-setup-skill

A reusable Claude Code skill for adding the [Metro](https://github.com/ZacSweers/metro) DI framework as a dual-build option alongside Anvil/Dagger in Android projects.

## What it does

- Chooses the correct Metro version based on your project's Kotlin version
- Adds a `ddg.di` qualifier to `gradle.properties` for switching between Anvil/Dagger and Metro
- Sets up conditional source sets so both DI frameworks coexist
- Creates Metro-specific DI graph files (`AppComponent`, `AppComponentFactory`, `ActivityComponent`)
- Updates KSP processors for Metro compatibility
- Adds `@HasMemberInjections` lint enforcement
- Configures CI matrices and nightly Metro workflows
- Handles Develocity scan tagging for cache separation

## Usage

Copy `metro-setup.md` into your project's `.claude/skills/` directory, then invoke:

```
/metro-setup
```

Or ask Claude: "add Metro DI to this project as a dual-build option"

## Verification

```bash
# AnvilDagger (default)
./gradlew assembleInternalDebug

# Metro
./gradlew assembleInternalDebug -Pddg.di=Metro
```

## Requirements

- Kotlin 2.0+
- JDK 21+
- Android project using Anvil + Dagger
- An existing `anvil-ksp` module with `ContributesSubComponentProcessor`

## Known Limitations

See the "Known Limitations" section in the skill file for details on import ordering, `RealAppBuildConfig`, and other edge cases that may need manual attention.
