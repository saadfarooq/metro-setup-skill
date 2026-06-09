# Metro DI Setup Skill

Adds the [Metro](https://github.com/ZacSweers/metro) DI framework as a dual-build option alongside Anvil/Dagger in Android projects. Controlled by `ddg.di` property in `gradle.properties`.

## Prerequisites

- Kotlin 2.0+ (Metro requires K2 compiler)
- JDK 21+
- Android project using Anvil + Dagger
- `anvil-ksp` module with `ContributesSubComponentProcessor`

## Step 1: Choose Metro Version

Read `versions.properties` to find the project's Kotlin version (`version.kotlin=X.Y.Z`):

| Kotlin version | Metro version |
|---|---|
| 2.0.x | 1.0.0 |
| 2.1.x | 1.0.1 |
| 2.2.x | 1.0.1 |
| 2.3.x | 1.0.2 (or latest) |

Add to `versions.properties`:
```
version.dev.zacsweers.metro..gradle-plugin=<version>
version.dev.zacsweers.metro..runtime=<version>
```

## Step 2: Add Qualifier to gradle.properties

Append after `com.squareup.anvil.trackSourceFiles=true`:
```properties
# DI framework: AnvilDagger (default) or Metro
ddg.di=AnvilDagger
```

Default is `AnvilDagger`. Set `ddg.di=Metro` to switch.

## Step 3: Root build.gradle — Metro Classpath

In the `buildscript { dependencies { } }` block, after the Android plugin classpath, add:
```groovy
// Metro DI plugin (requires JVM 21+ to load)
classpath "dev.zacsweers.metro:gradle-plugin:_"
```

Also update the existing JDK 21 comment to mention Metro:
```groovy
// Use JDK 21 for Kotlin compilation (required for Metro compiler plugin)
```

## Step 4: Root build.gradle — Metro Dual-Build Section

Add the KSP `ddg.di` arg inside `plugins.withId('com.google.devtools.ksp')`:
```groovy
plugins.withId('com.google.devtools.ksp') {
    ksp {
        arg("ddg.di", findProperty('ddg.di') ?: 'AnvilDagger')
    }
    // ... keep existing room.generateKotlin arg inside afterEvaluate
}
```

After the `subprojects { }` closing `}`, add the Metro dual-build switch:
```groovy
// Metro dual-build: conditionally swap Anvil for Metro based on ddg.di property
def diFramework = findProperty('ddg.di') ?: 'AnvilDagger'
logger.lifecycle("DI FRAMEWORK: ${diFramework}")
if (diFramework == 'Metro') {
    subprojects {
        def applyMetro = {
            pluginManager.withPlugin('com.squareup.anvil') {
                anvil {
                    disableComponentMerging.set(true)
                }
                apply plugin: 'dev.zacsweers.metro'
                metro {
                    enableKClassToClassMapKeyInterop.set(true)
                    interop {
                        includeDaggerAnnotations.set(true)
                        includeJavaxAnnotations.set(true)
                        includeAnvilAnnotations.set(true)
                        enableDaggerRuntimeInterop.set(true)
                        enableDaggerAnvilInterop.set(true)
                    }
                }
                afterEvaluate {
                    tasks.withType(org.jetbrains.kotlin.gradle.tasks.KotlinCompile).configureEach {
                        compilerOptions {
                            languageVersion.set(org.jetbrains.kotlin.gradle.dsl.KotlinVersion.KOTLIN_2_2)
                            apiVersion.set(org.jetbrains.kotlin.gradle.dsl.KotlinVersion.KOTLIN_2_2)
                        }
                        pluginClasspath.setFrom(pluginClasspath.filter {
                            !it.name.contains("anvil")
                        })
                    }
                    tasks.matching { it.name.contains("kapt") || it.name.contains("Kapt") }.configureEach {
                        enabled = false
                    }
                }
            }
        }
        pluginManager.withPlugin('org.jetbrains.kotlin.android', applyMetro)
        pluginManager.withPlugin('org.jetbrains.kotlin.jvm', applyMetro)
    }
}
```

Update the test configuration to expose the DI framework:
```groovy
systemProperty 'ddg.di', findProperty('ddg.di') ?: 'AnvilDagger'
```

## Step 5: di/build.gradle — Conditional Source Sets

Read the existing AnvilDagger sources to understand the contracts:
- `di/src/anvilDagger/java/dagger/SingleInstanceIn.kt`
- `di/src/anvilDagger/java/dagger/android/InjectorFactoryMap.kt`

Add the `diFramework` variable and conditional source set:
```groovy
def diFramework = findProperty('ddg.di') ?: 'AnvilDagger'

android {
    sourceSets {
        main {
            if (diFramework == 'Metro') {
                java.srcDirs += 'src/metro/java'
            } else {
                java.srcDirs += 'src/anvilDagger/java'
            }
        }
    }
}
```

Add Metro runtime dependency:
```groovy
dependencies {
    api "dev.zacsweers.metro:runtime:_"
}
```

Create `di/src/metro/java/dagger/SingleInstanceIn.kt`:
```kotlin
package dagger

/**
 * In Metro mode, [SingleInstanceIn] is a typealias for Metro's [dev.zacsweers.metro.SingleIn].
 * This ensures that @SingleInstanceIn(AppScope::class) on bindings is the same annotation
 * as the synthetic @SingleIn(AppScope::class) that Metro creates from @DependencyGraph(scope).
 */
typealias SingleInstanceIn = dev.zacsweers.metro.SingleIn
```

Create `di/src/metro/java/dagger/android/InjectorFactoryMap.kt`:
```kotlin
package dagger.android

import kotlin.reflect.KClass

/** Uses KClass<*> keys, which is what Metro's @ClassKey interop produces. */
typealias InjectorFactoryMap = Map<@JvmSuppressWildcards KClass<*>, @JvmSuppressWildcards AndroidInjector.Factory<*, *>>

fun InjectorFactoryMap.getFactory(key: Class<*>): AndroidInjector.Factory<*, *>? = this[key.kotlin]
```

## Step 6: app/build.gradle — Conditional Source Sets

Read the existing AnvilDagger sources:
- `app/src/anvilDagger/java/.../AppComponent.kt` — Dagger `@MergeComponent`
- `app/src/anvilDagger/java/.../AppComponentFactory.kt` — creates `DaggerAppComponent`
- `app/src/anvilDagger/java/.../ActivityComponent.kt` — Dagger `@MergeSubcomponent`

Add DI_FRAMEWORK buildConfigField inside `defaultConfig`:
```groovy
buildConfigField "String", "DI_FRAMEWORK", "\"${findProperty('ddg.di') ?: 'AnvilDagger'}\""
```

Add conditional source set inside the `android { }` block:
```groovy
// Conditional source set: swap DI graph creation based on ddg.di property
def diFramework = findProperty('ddg.di') ?: 'AnvilDagger'
sourceSets {
    main {
        if (diFramework == 'Metro') {
            java.srcDirs += 'src/metro/java'
        } else {
            java.srcDirs += 'src/anvilDagger/java'
        }
    }
}
```

Create `app/src/metro/java/.../AppComponent.kt`:
- Copy the AnvilDagger version, replacing Dagger with Metro equivalents:
  - `@MergeComponent(scope = AppScope::class, modules = [...])` → `@DependencyGraph(scope = AppScope::class)`
  - `@Component.Factory` → `@DependencyGraph.Factory`
  - `@BindsInstance` → `@Provides`
  - `AndroidInjector<App>` → no supertype (declare `inject` methods directly)
  - Remove module declarations (Metro auto-discovers via `@ContributesTo`)
  - `dagger.SingleInstanceIn` → `dev.zacsweers.metro.SingleIn`
  - All `inject()` methods and accessor methods stay the same

Create `app/src/metro/java/.../AppComponentFactory.kt`:
```kotlin
object AppComponentFactory {
    fun create(application: Application, applicationCoroutineScope: CoroutineScope): AppComponent {
        return createGraphFactory<AppComponent.Factory>()
            .create(application, applicationCoroutineScope)
    }
}
```

Create `app/src/metro/java/.../ActivityComponent.kt`:
- Copy the AnvilDagger version but:
  - `@MergeSubcomponent(ActivityScope::class)` → `@ContributesSubcomponent(scope = ActivityScope::class, parentScope = AppScope::class)`
  - Remove `@SingleInstanceIn(ActivityScope::class)` — Metro derives scope from `@ContributesSubcomponent`
  - `@Subcomponent.Factory` → `@ContributesSubcomponent.Factory`
  - Keep `@ContributesTo`, `@Module`, `@Binds`, `@IntoMap`, `@ClassKey` (Metro interop handles these)

## Step 7: RealAppBuildConfig.kt — DI Framework in Version Name

If the project has a build config class, add the DI framework suffix to version names for debug/internal builds. This uses `BuildConfig.DI_FRAMEWORK` from Step 6:
```kotlin
override val versionName: String
    get() = if (isInternalBuild() || isDebug) {
        "${BuildConfig.VERSION_NAME}-$diFramework"
    } else {
        BuildConfig.VERSION_NAME
    }

private val diFramework: String = if (BuildConfig.DEBUG || BuildConfig.FLAVOR == "internal") {
    BuildConfig.DI_FRAMEWORK
} else {
    ""
}
```

## Step 8: settings.gradle — Develocity Tagging

Update the Develocity scan tagging to include DI framework:
```groovy
// Tag scans with the active DI framework so cache hits are queryable per-variant
def diFrameworkTag = providers.gradleProperty('ddg.di').getOrElse('AnvilDagger')
tag(diFrameworkTag)
value('ddg.di', diFrameworkTag)
```

## Step 9: anvil-ksp — KSP Processor

In `ContributesSubComponentProcessor.kt`:
- Add `private val isMetro: Boolean` constructor parameter
- In `generateSubComponent`, conditionally omit `@SingleInstanceIn` when `isMetro`:
  - Metro derives scope from `@ContributesSubcomponent(scope = ...)`
  - Dagger requires the explicit scope annotation on generated subcomponents

In `ContributesSubComponentProcessorProvider.kt`:
- Read `ddg.di` from KSP options
- Pass `isMetro = diFramework == "Metro"` to the processor

## Step 10: Lint Rules

### Add MissingHasMemberInjectionsDetector

Metro requires `@HasMemberInjections` on every non-final class with `@Inject` field/property injection. Create a lint detector that enforces this. The detector:
- Checks for `@Inject` on fields (not constructor params)
- Reports an error if `@HasMemberInjections` is missing
- Skips `final` classes

Register the issue in the project's `IssueRegistry`.

### Update Existing Detectors

Update `MissingContributesToOnModuleDetector` error messages to mention Metro compatibility:
```
"Dagger @Module must be annotated with @ContributesTo to be discovered by Metro DI."
```

Update `MissingExplicitReturnTypeOnProvidesBindsDetector` error messages similarly:
```
"@Provides/@Binds methods must declare an explicit return type for Metro compatibility."
```

## Step 11: Add @HasMemberInjections to Source Files

For every class with `@Inject` on a field/property (not constructor), add:
1. `import dev.zacsweers.metro.HasMemberInjections` (in alphabetical order among imports)
2. `@HasMemberInjections` annotation on the class (before other annotations)

Typical files that need this (search with `@Inject` + `lateinit` patterns):
- `DaggerActivity.kt`, `DaggerFragment.kt` (base classes in `di/`)
- `DuckDuckGoApplication.kt` (application class)
- `DuckDuckGoActivity.kt`, `DuckDuckGoFragment.kt` (base classes in design-system)
- `BrowserActivity.kt`, `SearchWidget.kt`, feedback/navigation fragments
- Any view/widget class with `@Inject lateinit` fields

**Important:** Place the import alphabetically — `dev.zacsweers` comes before `javax.inject`. Also, preserve the existing annotation order on the class (e.g., `@SuppressLint` before `@HasMemberInjections`, `@InjectWith` before `@HasMemberInjections`).

## Step 12: CI Workflows

### ci.yml — Add Metro to Unit Test Matrix
```yaml
unit_tests:
  strategy:
    matrix:
      di: [AnvilDagger, Metro]
  # name includes matrix.di for non-default
  # Pass -Pddg.di=${{ matrix.di }} to Gradle
  # Artifact names include matrix.di
```

### build-debug-apk.yaml — Add Metro Matrix
```yaml
strategy:
  matrix:
    di: [AnvilDagger, Metro]
# name, artifact names, and -Pddg.di flag include matrix.di
```

### nightly.yml, privacy.yml, e2e-nightly-full-suite.yml — Add ddg_di Input
Add to `workflow_call` and `workflow_dispatch` inputs:
```yaml
ddg_di:
  description: 'DI framework to build with (AnvilDagger or Metro)'
  required: false
  type: string
  default: 'AnvilDagger'
```

Add env vars for DI-variant suffixes (empty for AnvilDagger):
```yaml
DDG_DI: ${{ inputs.ddg_di || 'AnvilDagger' }}
DI_HYPHEN_SUFFIX: ${{ (inputs.ddg_di || 'AnvilDagger') != 'AnvilDagger' && format('-{0}', inputs.ddg_di) || '' }}
```

Add `# Only block releases for the default Anvil path. Metro is informational.` on release-blocker steps. Pass `-Pddg.di=${{ env.DDG_DI }}` to all Gradle commands.

### release_upload_internal.yml — Metro as Default
```yaml
ddg_di:
  default: 'Metro'

DDG_DI: ${{ inputs.ddg_di || 'Metro' }}
```

### Create nightly-metro.yml
A scheduled workflow (04:30 UTC) that runs the full nightly suite under Metro. Calls nightly.yml, e2e-nightly-full-suite.yml, and privacy.yml with `ddg_di: Metro`, plus assembles all build variants with `-Pddg.di=Metro`. Creates Asana tasks on failure.

## Step 13: Test Guards

In test files that assert Dagger-specific behavior (e.g., `_Factory` class generation), add:
```kotlin
assumeFalse(
    "Dagger `_Factory` codegen does not apply under Metro",
    System.getProperty("ddg.di") == "Metro"
)
```

## Verification

After setup, verify with:
```bash
# AnvilDagger (default)
./gradlew assembleInternalDebug

# Metro
./gradlew assembleInternalDebug -Pddg.di=Metro
```

## Known Limitations

- Import ordering and annotation placement need manual verification — automated insertion may not match project conventions exactly.
- `RealAppBuildConfig.kt` (or equivalent build-config class) requires manual inspection to add DI framework to version names.
- Some files may need `@HasMemberInjections` but not directly import `javax.inject.Inject` — these need individual attention.
- The Kotlin language version override (`KOTLIN_2_2`) in the Metro dual-build section may need adjustment for your Kotlin version.
