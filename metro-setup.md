# Metro DI Setup Skill

Adds the [Metro](https://github.com/ZacSweers/metro) DI framework as a dual-build option alongside your existing DI framework in Android projects. Controlled by `ddg.di` property in `gradle.properties`.

## Step 0: Detect the DI Framework

First, determine what the project currently uses:

```bash
# Check for Anvil
grep -r 'com.squareup.anvil' *.gradle build.gradle settings.gradle 2>/dev/null
# Check for Hilt
grep -r 'dagger.hilt\|com.google.dagger.hilt' *.gradle build.gradle 2>/dev/null
# Check for Dagger (without Anvil/Hilt)
grep 'com.google.dagger\|id.*dagger' build.gradle */build.gradle 2>/dev/null | grep -v hilt
```

| Framework | Path | Complexity |
|---|---|---|
| Anvil + Dagger | [Path A](#path-a-anvil--dagger) | Medium |
| Plain Dagger (no Anvil/Hilt) | [Path B](#path-b-plain-dagger) | Low |
| Hilt | Not supported by this skill | High — manual migration |

> **Hilt projects:** Metro doesn't provide equivalents for `@HiltAndroidApp`, `@AndroidEntryPoint`, `@HiltViewModel`, or Hilt's compile-time validation. Migrating a Hilt project to Metro is a substantial effort requiring architectural changes. This skill does not cover Hilt. Consider keeping Hilt or migrating to plain Dagger first.

---

## Common Steps (All Paths)

These steps apply regardless of your current DI framework.

### Step 1: Choose Metro Version

Read `versions.properties` to find the project's Kotlin version (`version.kotlin=X.Y.Z`):

| Kotlin version | Metro version |
|---|---|
| 2.0.x | 1.0.0 |
| 2.1.x | 1.0.1 |
| 2.2.x | 1.0.1 |
| 2.3.x | 1.0.2 (or latest) |

Add to `versions.properties`:
```properties
version.dev.zacsweers.metro..gradle-plugin=<version>
version.dev.zacsweers.metro..runtime=<version>
```

### Step 2: Add Qualifier to gradle.properties

Append:
```properties
# DI framework: AnvilDagger (default) or Metro
ddg.di=AnvilDagger
```

> **Plain Dagger projects:** Use `ddg.di=Dagger` as default and `ddg.di=Metro` to switch. Adjust all references below accordingly.

### Step 3: Root build.gradle — Metro Classpath

In the `buildscript { dependencies { } }` block, add:
```groovy
// Metro DI plugin (requires JVM 21+ to load)
classpath "dev.zacsweers.metro:gradle-plugin:_"
```

Also update the existing JDK 21 comment:
```groovy
// Use JDK 21 for Kotlin compilation (required for Metro compiler plugin)
```

### Step 4: settings.gradle — Develocity Tagging

Update Develocity scan tagging:
```groovy
def diFrameworkTag = providers.gradleProperty('ddg.di').getOrElse('AnvilDagger')
tag(diFrameworkTag)
value('ddg.di', diFrameworkTag)
```

---

## Path A: Anvil + Dagger

For projects using `com.squareup.anvil` with Dagger.

### A1. Root build.gradle — Metro Dual-Build Section

Add the KSP `ddg.di` arg inside `plugins.withId('com.google.devtools.ksp')`:
```groovy
plugins.withId('com.google.devtools.ksp') {
    ksp {
        arg("ddg.di", findProperty('ddg.di') ?: 'AnvilDagger')
    }
    // ... keep existing room.generateKotlin arg inside afterEvaluate
}
```

After `subprojects { }`, add:
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

Update test configuration:
```groovy
systemProperty 'ddg.di', findProperty('ddg.di') ?: 'AnvilDagger'
```

### A2. di/build.gradle — Conditional Source Sets

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

dependencies {
    api "dev.zacsweers.metro:runtime:_"
}
```

### A3. Create di/src/metro/... Files

Create `di/src/metro/java/dagger/SingleInstanceIn.kt`:
```kotlin
package dagger

/**
 * In Metro mode, [SingleInstanceIn] is a typealias for Metro's [dev.zacsweers.metro.SingleIn].
 * Ensures @SingleInstanceIn(AppScope::class) on bindings matches Metro's synthetic
 * @SingleIn(AppScope::class) from @DependencyGraph(scope), preventing scope mismatch errors.
 */
typealias SingleInstanceIn = dev.zacsweers.metro.SingleIn
```

Create `di/src/metro/java/dagger/android/InjectorFactoryMap.kt`:
```kotlin
package dagger.android

import kotlin.reflect.KClass

/** Uses KClass<*> keys — what Metro's @ClassKey interop produces. */
typealias InjectorFactoryMap = Map<@JvmSuppressWildcards KClass<*>, @JvmSuppressWildcards AndroidInjector.Factory<*, *>>

fun InjectorFactoryMap.getFactory(key: Class<*>): AndroidInjector.Factory<*, *>? = this[key.kotlin]
```

### A4. app/build.gradle — Conditional Source Sets

```groovy
buildConfigField "String", "DI_FRAMEWORK", "\"${findProperty('ddg.di') ?: 'AnvilDagger'}\""

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

### A5. Create app/src/metro/... Files

Read the existing AnvilDagger sources (`app/src/anvilDagger/`) to understand the contracts, then create Metro equivalents:

**AppComponent.kt** — Convert from Dagger `@MergeComponent` to Metro `@DependencyGraph`:
- `@MergeComponent(scope = AppScope::class, modules = [...])` → `@DependencyGraph(scope = AppScope::class)`
- `@Component.Factory` → `@DependencyGraph.Factory`
- `@BindsInstance` → `@Provides`
- `AndroidInjector<App>` → no supertype (declare `inject` methods directly)
- Remove module declarations (Metro discovers via `@ContributesTo`)
- `dagger.SingleInstanceIn` → `dev.zacsweers.metro.SingleIn`
- Keep all `inject()` and accessor method signatures

**AppComponentFactory.kt:**
```kotlin
object AppComponentFactory {
    fun create(application: Application, applicationCoroutineScope: CoroutineScope): AppComponent {
        return createGraphFactory<AppComponent.Factory>()
            .create(application, applicationCoroutineScope)
    }
}
```

**ActivityComponent.kt** — Convert from Dagger `@MergeSubcomponent`:
- `@MergeSubcomponent(ActivityScope::class)` → `@ContributesSubcomponent(scope = ActivityScope::class, parentScope = AppScope::class)`
- Remove `@SingleInstanceIn(ActivityScope::class)` — Metro derives scope from `@ContributesSubcomponent`
- `@Subcomponent.Factory` → `@ContributesSubcomponent.Factory`
- Keep `@ContributesTo`, `@Module`, `@Binds`, `@IntoMap`, `@ClassKey`

### A6. anvil-ksp — KSP Processor

In `ContributesSubComponentProcessor.kt`:
- Add `private val isMetro: Boolean` constructor parameter
- In `generateSubComponent`, conditionally omit `@SingleInstanceIn` when `isMetro`

In `ContributesSubComponentProcessorProvider.kt`:
- Read `ddg.di` from KSP options
- Pass `isMetro = diFramework == "Metro"`

---

## Path B: Plain Dagger

For projects using Dagger without Anvil or Hilt.

### B1. Root build.gradle — Metro Dual-Build Section

Unlike the Anvil path, Metro is applied directly without hooking into Anvil's plugin lifecycle:

```groovy
// Metro dual-build: conditionally swap Dagger codegen for Metro
def diFramework = findProperty('ddg.di') ?: 'Dagger'
logger.lifecycle("DI FRAMEWORK: ${diFramework}")
if (diFramework == 'Metro') {
    subprojects {
        pluginManager.withPlugin('org.jetbrains.kotlin.android') {
            pluginManager.withPlugin('kotlin-kapt') {
                // Metro must load after Kotlin and before kapt runs
                apply plugin: 'dev.zacsweers.metro'
                metro {
                    enableKClassToClassMapKeyInterop.set(true)
                    interop {
                        includeDaggerAnnotations.set(true)
                        includeJavaxAnnotations.set(true)
                        enableDaggerRuntimeInterop.set(true)
                    }
                }
                // Disable KAPT — Metro replaces Dagger's annotation processor
                afterEvaluate {
                    tasks.matching { it.name.contains("kapt") || it.name.contains("Kapt") }.configureEach {
                        enabled = false
                    }
                }
            }
        }
    }
}
```

> **Key difference from Path A:** No `enableDaggerAnvilInterop`, no `includeAnvilAnnotations`, no `disableComponentMerging`. Metro reads Dagger annotations (`@Module`, `@Provides`, `@Binds`, `@Inject`, `@Component`) directly.

### B2. di/build.gradle — Conditional Source Sets

```groovy
def diFramework = findProperty('ddg.di') ?: 'Dagger'

android {
    sourceSets {
        main {
            if (diFramework == 'Metro') {
                java.srcDirs += 'src/metro/java'
            } else {
                java.srcDirs += 'src/dagger/java'
            }
        }
    }
}

dependencies {
    api "dev.zacsweers.metro:runtime:_"
}
```

> Adjust `src/dagger/java` to match your existing Dagger-specific source directory. If your Dagger graph is in the main source set, you may not need conditional compilation at all — you can place the Metro variant in `src/metro/java` and keep Dagger in `src/main/java`.

### B3. Create Metro Source Files

Create `di/src/metro/java/dagger/SingleInstanceIn.kt`:
```kotlin
package dagger

/** In Metro mode, maps @SingleInstanceIn to Metro's @SingleIn for scope compatibility. */
typealias SingleInstanceIn = dev.zacsweers.metro.SingleIn
```

Create `di/src/metro/java/dagger/android/InjectorFactoryMap.kt`:
```kotlin
package dagger.android

import kotlin.reflect.KClass

typealias InjectorFactoryMap = Map<@JvmSuppressWildcards KClass<*>, @JvmSuppressWildcards AndroidInjector.Factory<*, *>>

fun InjectorFactoryMap.getFactory(key: Class<*>): AndroidInjector.Factory<*, *>? = this[key.kotlin]
```

### B4. Create app/src/metro/... Files

**AppComponent.kt** — Convert from Dagger `@Component` to Metro `@DependencyGraph`:

Your existing Dagger component likely looks like:
```kotlin
@Singleton
@Component(modules = [NetworkModule::class, DatabaseModule::class, ...])
interface AppComponent {
    fun inject(app: MyApplication)

    @Component.Factory
    interface Factory {
        fun create(@BindsInstance app: Application): AppComponent
    }
}
```

Convert to Metro:
```kotlin
@SingleIn(AppScope::class)
@DependencyGraph(scope = AppScope::class)
interface AppComponent {
    fun inject(app: MyApplication)

    @DependencyGraph.Factory
    interface Factory {
        fun create(@Provides app: Application): AppComponent
    }
}
```

> **Module discovery:** Without Anvil, you have two options:
> 1. **Explicit modules** — keep `modules = [...]` in `@DependencyGraph`. Metro's interop reads Dagger `@Module`.
> 2. **Auto-discovery** — add `dev.zacsweers.metro.ContributesTo` annotations to your modules. Metro reads these even without Anvil's KSP. This enables gradual adoption.

**AppComponentFactory.kt:**
```kotlin
object AppComponentFactory {
    fun create(application: Application): AppComponent {
        return createGraphFactory<AppComponent.Factory>()
            .create(application)
    }
}
```

**ActivityComponent.kt** — Convert from Dagger `@Subcomponent`:

Before (Dagger):
```kotlin
@Subcomponent(modules = [FooModule::class])
interface ActivityComponent : AndroidInjector<MyActivity> {
    @Subcomponent.Factory
    interface Factory : AndroidInjector.Factory<MyActivity>
}
```

After (Metro):
```kotlin
@ContributesSubcomponent(scope = ActivityScope::class, parentScope = AppScope::class)
interface ActivityComponent : AndroidInjector<MyActivity> {
    @ContributesSubcomponent.Factory
    interface Factory : AndroidInjector.Factory<MyActivity, ActivityComponent> {
        override fun create(@BindsInstance instance: MyActivity): ActivityComponent
    }

    @ContributesTo(AppScope::class)
    interface ParentComponent {
        fun activityComponentFactory(): Factory
    }
}
```

### B5. app/build.gradle — Source Sets

```groovy
buildConfigField "String", "DI_FRAMEWORK", "\"${findProperty('ddg.di') ?: 'Dagger'}\""

def diFramework = findProperty('ddg.di') ?: 'Dagger'
sourceSets {
    main {
        if (diFramework == 'Metro') {
            java.srcDirs += 'src/metro/java'
        } else {
            java.srcDirs += 'src/dagger/java'
        }
    }
}
```

### B6. No KSP Changes Needed

Plain Dagger projects typically don't have an `anvil-ksp` module or `ContributesSubComponentProcessor`. Skip the KSP processor changes from Path A.

---

## Shared Steps (All Paths)

These apply to both Path A and Path B.

### Lint Rules

**Add MissingHasMemberInjectionsDetector:**

Metro requires `@HasMemberInjections` on every non-final class with `@Inject` field/property injection. Create a lint detector that:
- Checks for `@Inject` on fields (not constructor params)
- Reports an error if `@HasMemberInjections` is missing
- Skips `final` classes

Register the issue in the project's `IssueRegistry`.

**Update existing detectors** to mention Metro:
- `MissingContributesToOnModuleDetector`: "must be annotated with @ContributesTo to be discovered by Metro DI"
- `MissingExplicitReturnTypeOnProvidesBindsDetector`: "must declare an explicit return type for Metro compatibility"

### Add @HasMemberInjections to Source Files

For every class with `@Inject` on a field/property (not constructor), add:
1. `import dev.zacsweers.metro.HasMemberInjections` (alphabetical order among imports — `dev.zacsweers` before `javax.inject`)
2. `@HasMemberInjections` annotation on the class (preserve existing annotation order)

Find affected files with:
```bash
grep -rl '@Inject' --include='*.kt' | xargs grep -l 'lateinit\|@Inject.*var'
```

**Important:** Place the import alphabetically. Preserve existing annotation order on the class (e.g., `@SuppressLint` before `@HasMemberInjections`, `@InjectWith` before `@HasMemberInjections`).

### Build Config — DI Framework in Version Name

If the project has a build config class that constructs display version names, add the DI framework suffix for debug/internal builds:
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

### CI Workflows

**Add Metro to CI matrices** in `ci.yml` and `build-debug-apk.yaml`:
```yaml
strategy:
  matrix:
    di: [AnvilDagger, Metro]  # or [Dagger, Metro] for Path B
```

**Add `ddg_di` input** to nightly/privacy/e2e workflow files:
```yaml
ddg_di:
  description: 'DI framework to build with (AnvilDagger or Metro)'
  required: false
  type: string
  default: 'AnvilDagger'
```

Add DI-variant env vars (empty suffix for default path):
```yaml
DDG_DI: ${{ inputs.ddg_di || 'AnvilDagger' }}
DI_HYPHEN_SUFFIX: ${{ (inputs.ddg_di || 'AnvilDagger') != 'AnvilDagger' && format('-{0}', inputs.ddg_di) || '' }}
```

Pass `-Pddg.di=${{ env.DDG_DI }}` to all Gradle commands.

**Create `nightly-metro.yml`** — scheduled workflow (e.g., 04:30 UTC) that calls nightly/e2e/privacy workflows with `ddg_di: Metro`, plus assembles all build variants with `-Pddg.di=Metro`.

### Test Guards

In tests that assert Dagger-specific behavior (e.g., `_Factory` class generation), add:
```kotlin
assumeFalse(
    "Dagger `_Factory` codegen does not apply under Metro",
    System.getProperty("ddg.di") == "Metro"
)
```

---

## Verification

```bash
# Default DI framework
./gradlew assembleInternalDebug

# Metro
./gradlew assembleInternalDebug -Pddg.di=Metro
```

## Known Limitations

- **Import ordering:** Automated insertion may not match project conventions. Verify `dev.zacsweers.metro.HasMemberInjections` appears alphabetically.
- **Annotation ordering:** Multiple annotations on a class need manual verification to preserve existing order.
- **Build config class:** Files like `RealAppBuildConfig.kt` that use version name logic need individual attention.
- **Kotlin version override:** The `KOTLIN_2_2` language version in the Metro section may need adjustment for newer Kotlin versions.
- **Hilt:** Not supported. Hilt's component hierarchy (@HiltAndroidApp, @AndroidEntryPoint, @HiltViewModel) has no direct Metro equivalent.
