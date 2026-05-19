# Gradle init script modernization (M8)

Authoring date: 2026-05-19
Owner: TEESimulator multi-user build infrastructure
Trigger: Gradle 10 release, OR project wrapper bump past Gradle 9.x

## Context

`/etc/gradle-init.d/per-user-builds.gradle.kts` redirects each user's `buildDir` and `projectCacheDir` to `/mnt/companion/$USER/builds/<projectSlug>/` so concurrent users sharing `~/Git-repo-success/` do not collide on Gradle output. The script is installed system-wide and applied to every Gradle invocation across all users.

It uses two APIs that Gradle has been deprecating:

- `Settings.settingsEvaluated { }` (init script DSL) — deprecated since Gradle 7.6, slated for removal in Gradle 10.
- `Gradle.beforeProject { }` (via implicit init script binding) — collides with Gradle's Isolated Projects feature and is also on the removal track.

Today the wrapper is pinned at Gradle 9.2.0 (`gradle/wrapper/gradle-wrapper.properties`), so the deprecation warnings are tolerable. Once the project bumps past 9.x — or once another consumer of the shared tree forces a Gradle 10 upgrade — the script will start failing.

## Current implementation (verbatim)

```kotlin
val USER: String = System.getProperty("user.name") ?: System.getenv("USER") ?: "unknown"
val SHARED_TREE_PREFIX = "/home/president/Git-repo-success/"
val COMPANION_MOUNT = "/mnt/companion"
val COMPANION_BUILDS = "$COMPANION_MOUNT/$USER/builds"

fun companionMounted(): Boolean =
    try {
        java.io.File("/proc/mounts").readLines().any { line ->
            val fields = line.split(" ")
            fields.size >= 2 && fields[1] == COMPANION_MOUNT
        }
    } catch (_: Exception) {
        false
    }

settingsEvaluated {
    val rd = rootDir
    if (!rd.absolutePath.startsWith(SHARED_TREE_PREFIX)) return@settingsEvaluated
    if (!companionMounted()) {
        System.err.println(
            "[per-user-builds] $COMPANION_MOUNT not mounted; skipping redirect"
        )
        return@settingsEvaluated
    }
    val projectRoot = file("$COMPANION_BUILDS/${rd.name}")
    projectRoot.mkdirs()
    startParameter.projectCacheDir = projectRoot.resolve(".gradle")
}

beforeProject {
    val rd = rootDir
    if (!rd.absolutePath.startsWith(SHARED_TREE_PREFIX)) return@beforeProject
    if (!companionMounted()) return@beforeProject
    val projectRoot = file("$COMPANION_BUILDS/${rd.name}")
    val sub = if (this == rootProject) "build" else "${name}-build"
    layout.buildDirectory.set(projectRoot.resolve(sub))
}
```

## Migration paths

### Option A — init-script-only refactor

Replace `settingsEvaluated` with the non-deprecated `Settings.gradle.allprojects { }` form, and replace the `beforeProject` block with `Settings.gradle.beforeProject` (the Settings receiver, not Gradle's). Verify both against the targeted Gradle version's API surface before committing.

- Pro: stays in `/etc/gradle-init.d/`. No project changes. No new files to track in the repo.
- Con: the modern non-deprecated equivalents of both hooks are themselves moving targets across Gradle versions. Requires re-verification on each Gradle bump.

### Option B — convention plugin in shared buildSrc

Move the per-user redirect into a settings convention plugin published to a shared location (e.g., a small Gradle plugin published to a local `~/Git-repo-success/.gradle-plugins/` repo). Each project's `settings.gradle.kts` applies the plugin.

- Pro: lives inside the project tree, version-controlled, testable. Survives Gradle major bumps because it uses the public plugin SPI.
- Con: every project in the shared tree must opt in via `settings.gradle.kts` apply. Adds a published-plugin maintenance burden.

### Option C — make Gradle Build Service

Implement the redirect logic as a Gradle Build Service registered in a settings plugin. The service intercepts project configuration and applies the redirect. Compatible with Isolated Projects.

- Pro: future-proof (Isolated Projects is the long-term direction). Modern API surface.
- Con: heaviest implementation. Requires Gradle 8+ Build Service knowledge. Probably overkill for a per-user buildDir redirect.

## Recommendation

Start with Option A on the next Gradle bump (no infrastructure change), only escalate to B if A breaks again within a year. Hold C in reserve until either Isolated Projects becomes mandatory or B's plugin maintenance gets noisy.

## Trigger conditions

Act when any of:

- `gradle-wrapper.properties` bumps to a version where `settingsEvaluated` or `beforeProject` becomes an error rather than a warning.
- A user reports `[per-user-builds]` no longer firing on a Gradle bump.
- Isolated Projects becomes mandatory and the current init script throws at runtime.

## Verification plan when migrating

1. Pin the target Gradle version locally; rebuild from clean.
2. Inspect `--warning-mode all` output to confirm no remaining deprecation warnings from the init script.
3. Verify per-user `/mnt/companion/$USER/builds/<projectSlug>/` is populated as today.
4. Run multi-user concurrent build smoke test: two users build the same shared project simultaneously; confirm no file lock contention or output cross-pollution.
5. Verify mountpoint guard still fires when `/mnt/companion` is unmounted (test by unmounting the bind temporarily).

## References

- Gradle 7.6 release notes — `settingsEvaluated` deprecation (consult `https://docs.gradle.org/7.6/` for the canonical entry).
- Gradle 9.x user guide — Settings plugin authoring.
- Gradle 8.x / 9.x roadmap — Isolated Projects (consult the docs site at the version pinned in the project wrapper).
