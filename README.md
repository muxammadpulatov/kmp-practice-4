# kmp-practice-4
# Practice 4 — KMP Project Structure (continuation)

**Course:** Android Development  
**Topics:** Lecture 10 — KMP Advanced Project Structure · Lecture 11 — Choosing a KMP Project Configuration  
**Student:** _Full Name_ · **Group:** _Group_

## Contents

| # | Question | Type |
|---|----------|------|
| [14](#question-14) | `dependsOn` vs `dependencies {}` | Theoretical |
| [15](#question-15) | Misuse of `dependsOn` | Theoretical |
| [16](#question-16) | Custom `jvmAndJsMain` source set | Hands-on |
| [17](#question-17) | Default hierarchy template caveat | Hands-on |
| [18](#question-18) | Dependency propagation | Theoretical |
| [19](#question-19) | Why `asExecutor()` fails in `commonMain` | Theoretical |
| [20](#question-20) | Version alignment | Theoretical |
| [21](#question-21) | Compilations | Hands-on |
| [22](#question-22) | Single vs. several shared modules | Theoretical |
| [23](#question-23) | Umbrella module | Theoretical |
| [24](#question-24) | Repository layouts | Theoretical |
| [25](#question-25) | Multi-app company design | Theoretical |

Project code for the hands-on questions (16, 17, 21) is in [`shared/`](shared/).

---

# Section 3 — KMP Advanced Project Structure (Lecture 10)

## Question 14

**Task:** Explain why "`dependsOn` is basically the same as the `dependencies {}` block" is wrong. List the four consequences of `A.dependsOn(B)` and say which kind of relation it is.

**Answer**

The claim is wrong because the two mechanisms are different kinds of relation:

- `dependencies {}` is a **Gradle (packaging/build) dependency**. It points at a library or another Gradle project, using coordinates or a project path.
- `dependsOn` is a **Kotlin-specific relation between two source sets**. It is not a Gradle dependency, and it is declared *outside* the `dependencies {}` block on purpose.

When `A.dependsOn(B)` is declared, all four of these hold **simultaneously**:

1. **A observes B's API**, including `internal` declarations.
2. **A can provide `actual` implementations** for B's `expect` declarations.
3. **B compiles to every target A compiles to**, plus its own targets.
4. **A inherits all of B's regular dependencies.**

**Kind of relation:** `dependsOn` is a **visibility / actualization relation** (what code can see, and who may implement `expect` declarations, plus target inheritance). It is **not** a packaging/build relation.

It reads naturally as "A depends on B for its common declarations", for example `iosArm64Main.dependsOn(iosMain)`.

---

## Question 15

**Task:** Explain why `commonTest.dependsOn(commonMain)` and calling `dependsOn` on a library's own `commonMain` are both mistakes.

**Answer**

**What `dependsOn` is reserved for.** It describes the *Kotlin source-set tree inside your own project*. The Kotlin Gradle plugin creates the default source sets and wires their `dependsOn` relations automatically. Manual calls are meant only for source sets Kotlin did *not* create for you, such as a custom `jvmAndJsMain`.

**Mistake 1: `commonTest.dependsOn(commonMain)`.**
- `dependsOn` is not a "share this code with that source set" switch. It makes `commonTest` a *child* in the hierarchy. It would inherit `commonMain`'s targets and take part in `expect`/`actual` actualization, and that is not what the student wants.
- Test source sets belong to a **separate tree** (Lecture 10, slide 10: "Test source sets align on their own separate tree"). Test code needs production code through the **test compilation's association with the main compilation**, which the plugin sets up itself. No manual `dependsOn` is needed.
- Beyond the lecture: current Kotlin versions do not support linking main and test source sets with `dependsOn` and report a diagnostic if you try.

**Mistake 2: calling `dependsOn` on a library's `commonMain`.**
- The lecture states that `dependsOn` "can't be used for a library or another Gradle project's source set."
- A library is something you *consume*, not part of your source-set tree.

**What `dependencies {}` is for.** It declares regular dependencies on published libraries (`"group:artifact:version"`) or sibling projects (`project(":x")`), with scopes such as `implementation` and `api`. Every source set has its own `dependencies {}` block. Kotlin then resolves that library dependency down to real *source set → source set* edges.

---

## Question 16

**Task:** Create a KMP module with `jvm()`, `js(IR) { browser() }`, `linuxX64()`. Declare `jvmAndJsMain`, wire it with three `dependsOn` calls, move one shared function into it, and confirm sync and build.

**Solution** — [`shared/build.gradle.kts`](shared/build.gradle.kts)

```kotlin
kotlin {
    jvm()
    js(IR) { browser() }
    linuxX64()

    sourceSets {
        val jvmAndJsMain by creating { }          // isolated at first, no dependsOn yet

        jvmAndJsMain.dependsOn(commonMain.get())  // 1. attach it to the tree
        jvmMain.get().dependsOn(jvmAndJsMain)     // 2. jvmMain sits below it
        jsMain.get().dependsOn(jvmAndJsMain)      // 3. jsMain sits below it
    }
}
```

Shared function — [`shared/src/jvmAndJsMain/kotlin/SharedGreeting.kt`](shared/src/jvmAndJsMain/kotlin/SharedGreeting.kt):

```kotlin
fun sharedGreeting(name: String): String = "Hello, $name! (shared between JVM and JS)"
```

**Why it works.** Before the wiring, `jvmAndJsMain` is isolated and has no target labels. Making the existing `jvmMain` and `jsMain` depend on it is what puts the **JVM and JS** labels on it (rule 3 of `dependsOn`: B compiles to every target A compiles to). The function can therefore be used from `jvmMain` and `jsMain` but **not** from `linuxX64Main`.

**Result:** Gradle sync and build succeeded.

![Q16 – sync and build](screenshots/q16-build-success.png)

---

## Question 17

**Task:** Add `iosArm64()` and `iosSimulatorArm64()`. Check the Project view for an auto-created `appleMain`/`iosMain` and explain what you observe.

**Code added** (see [`shared/build.gradle.kts`](shared/build.gradle.kts)):

```kotlin
iosArm64()
iosSimulatorArm64()
```

**Observation.** With the manual `dependsOn` calls from Question 16 in place, **no `appleMain` or `iosMain` intermediate source set is created**. Only `iosArm64Main` and `iosSimulatorArm64Main` appear, each attached directly under `commonMain`.

![Q17 – Project view](screenshots/q17-project-view.png)

**Explanation (default hierarchy template caveat).** Kotlin normally applies a *default hierarchy template* that creates useful intermediate source sets automatically (`appleMain`, `iosMain`, `nativeMain`, …). The lecture's warning is that **writing any manual `dependsOn` call opts the project out of that template**. After that, Kotlin no longer infers the tree for you ("New source sets need manual `dependsOn`; Kotlin won't infer the tree for you").

The consequence is that iOS code cannot be shared between the two iOS targets in a shared `iosMain`. That is exactly why the lecture's recap tells you to check "which default source sets disappeared".

**Ways to fix it:**
- Create `iosMain` by hand and add more `dependsOn` calls.
- Use the hierarchy template DSL: `applyDefaultHierarchyTemplate { common { group("jvmAndJs") { withJvm(); withJs() } } }`. This keeps the defaults *and* adds the custom group.

> Note: iOS targets can only be compiled on macOS. On Windows/Linux, Gradle sync still works but skips the Apple targets.

---

## Question 18

**Task:** Explain "propagation" for `kotlinx-coroutines-core:1.7.3` declared once in `commonMain`. Is it needed again in `androidMain` and `iosMain`? How can it be scoped to iOS only?

**Answer**

**Propagation** means a dependency declared once on `commonMain` is **automatically propagated to every source set that depends on `commonMain`, directly or transitively**. In this project that covers `androidMain`, `iosMain`, `iosArm64Main` and `iosSimulatorArm64Main`.

- **No**, you do **not** add it again to `androidMain` or `iosMain`. One declaration reaches all three targets, and Kotlin resolves it consistently everywhere.
- **iOS only:** declare it on **`iosMain`** instead of `commonMain`:

```kotlin
iosMain.dependencies {
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.7.3")
}
```

The lecture's point is that *the source set you pick is itself a scoping decision*.

---

## Question 19

**Task:** Explain why `Dispatchers.Main` and `runBlocking` resolve in `commonMain` (android + iosArm64 + iosSimulatorArm64) but `dispatcher.asExecutor()` does not.

**Answer**

**Compatibility rule:** a dependency's source set is compatible with yours only if it **covers at least all of your targets**.

A published KMP library ships its own source-set structure alongside the binaries. `kotlinx-coroutines-core:1.7.3` has:
- its own **`commonMain`**, covering every target;
- **`concurrentMain`**, covering every target except JS;
- **`jsMain`**, plus JVM-specific sources.

A dependency on "coroutines" is really a dependency on several of these source sets. Kotlin resolves it down to source-set → source-set edges and keeps only the compatible ones.

| Call | Resolved from (coroutines' own source set) | Covers android + iosArm64 + iosSimulatorArm64? |
|------|--------------------------------------------|---|
| `Dispatchers.Main` | `commonMain` (all targets) | Yes → resolves |
| `runBlocking` | `concurrentMain` (all targets except JS) | Yes → resolves |
| `dispatcher.asExecutor()` | the **JVM-only** source set | **No** → unresolved (red) |

`asExecutor()` comes from `java.util.concurrent.Executor`, which exists only on the JVM. The JVM-only source set doesn't cover the iOS targets, so it isn't visible from a `commonMain` that also compiles to iOS, and the compiler reports an unresolved reference.

> The slide only names `commonMain` and `concurrentMain` as the two source sets that cover the targets. Putting `Dispatchers.Main` in `commonMain` and `runBlocking` in `concurrentMain` follows the coroutines library's own layout.

---

## Question 20

**Task:** `commonMain` declares coroutines 1.7.3, while `androidMain` adds `navigation-compose:2.7.7`, which needs coroutines ≥ 1.8.0. What does Gradle do? Why doesn't an upgrade in `iosTest` affect `*Main` source sets?

**Answer**

`commonMain` is compiled **several times** (once per target), and **every compilation must agree on one version** of each dependency. Gradle therefore aligns the versions automatically:

1. Gradle sees that `navigation-compose:2.7.7` needs coroutines **1.8.0+**.
2. It **raises `commonMain`'s coroutines from 1.7.3 to 1.8.0**.
3. That version is **propagated everywhere**. `iosMain`, `iosArm64Main` and the rest therefore also use 1.8.0, even though they never saw the navigation library.

**Why `iosTest` doesn't affect production code.** Main and test source sets form **two separate alignment trees**. Gradle aligns versions among the `*Main` source sets and, independently, among the `*Test` source sets. Production code never depends on test code, so upgrading a dependency in `iosTest` changes only the test classpath. It never changes the versions used in any `*Main` source set.

---

## Question 21

**Task:** In `build.gradle.kts`, use `compilations.getByName("main")` on the `jvm()` target, print its name, confirm a `test` compilation exists, and explain how a compilation differs from a source set in one sentence.

**Code** (see [`shared/build.gradle.kts`](shared/build.gradle.kts)):

```kotlin
kotlin {
    jvm {
        val main = compilations.getByName("main")
        println("JVM main compilation name = ${main.name}")
        compilations.all { println("JVM compilation registered: $name") }
    }
}

tasks.register("printCompilations") {
    val info = kotlin.targets.associate { it.name to it.compilations.names.toList() }
    doLast { info.forEach { (target, comps) -> println("$target -> $comps") } }
}
```

Run `./gradlew :shared:printCompilations`. Expected output:

```
JVM main compilation name = main
JVM compilation registered: main
JVM compilation registered: test
jvm -> [main, test]
```

![Q21 – Gradle output](screenshots/q21-compilations.png)

Every target automatically gets two compilations: **`main`** (production sources) and **`test`** (test sources).

**Difference in one sentence:** a *source set* is a group of source files and dependencies (the code), whereas a *compilation* is one actual compiler run for a target that takes one or more source sets as input and produces an artifact.

---

# Section 4 — Choosing a KMP Project Configuration (Lecture 11)

## Question 22

**Task:** State the pros and cons of a single vs. several shared modules, and when a team moves from one to the other.

**Answer**

| | Pros | Cons |
|---|------|------|
| **Single shared module** | Simple design, lower cognitive load (no decisions about where functionality goes). Great starting point. | Compilation time grows as the module grows. No separate features, and no way to depend on only part of the shared code. |
| **Several shared modules** | Separation of concerns. Better scalability as the app grows. | More complicated setup (including the umbrella framework). More involved dependency management between modules. |

**When to switch.** A new team should **start with one shared module**, since the simplicity is worth more than scalability they don't need yet. As the app grows, compile times increase and the team wants features that are isolated and can be depended on separately. At that point they **split the shared module into feature modules** and accept the extra setup cost. Because iOS can only consume one framework, the split brings in an umbrella module (see Question 23).

---

## Question 23

**Task:** Why does generating one iOS framework per feature module break down? What is an umbrella module, and how does it fix this?

**Answer**

**The problem: dependency duplication.** Each KMP module compiled to its own iOS framework **bundles all of its dependencies inside it**. If two feature modules share a dependency and are exposed as two separate frameworks, that dependency is compiled into the app **twice, as two incompatible copies**. Kotlin can't share a dependency across separate builds, because each binary is trimmed to exactly what that one build needs.

**Concrete failure mode.** State passed through "the same" dependency from two different modules is **not actually connected**. Each module holds its own copy, so the shared state is not shared. This leads to bugs that **only show up on iOS**.

**The umbrella module.** It is an extra module that depends on all feature modules and is the only one exported as an iOS framework. The **umbrella framework bundles every shared module into a single compiled artifact**.

**Why it fixes the problem.** Every dependency then exists **exactly once**, with one consistent code structure, so state shared through a dependency really is shared. Android can still depend on any or all feature modules directly, while the iOS app consumes the umbrella framework only.

---

## Question 24

**Task:** For monorepo, two-repo and three-repo layouts, give one advantage and one disadvantage each, and explain the versioning difference between two and three repositories.

**Answer**

| Layout | Advantage | Disadvantage |
|---|---|---|
| **Monorepo** | Easy to set up with project wizards. iOS developers can work with shared code without hunting across repositories. | iOS developers must set up and learn unfamiliar tools (Gradle, Android Studio/IntelliJ). Often doesn't fit apps whose code already lives in separate repositories. |
| **Two repos** (Android + shared \| iOS) | The Xcode/iOS project stays separate, so Android and iOS can be versioned separately. | The shared module's version moves **in lockstep with the Android app**, because they share one repository. The iOS team consumes shared code owned by the Android repository. |
| **Three repos** (Android \| iOS \| shared) | Each platform and the shared code are versioned **independently**. | More overhead: shared modules must be published. Both teams consume the same versioned artifact, so version parity is required and the shared code stops feeling "owned" by the Android team. |

**Versioning consequence.**
- **Two repos:** the Android app and the shared module live in the same repository and are built together from source. Only the iOS app needs a published/built framework.
- **Three repos:** the shared modules must **additionally be published for Android/JVM, not just for iOS**. This can be done either for each feature module separately or for the umbrella module only. Android and iOS then both consume the **same versioned artifact**.

---

## Question 25

**Task:** Design a configuration for a 200-engineer company with five apps that reuse logging, networking and analytics, where no app should pull in libraries it doesn't use.

**Answer**

This is the **"many repositories"** configuration (Lecture 11, slide 9).

**1. Repositories for the shared libraries: 3 library repos.**
- `logging`, `networking` and `analytics` each get **their own repository and their own version**.
- Each is published for **Android/JVM and iOS**.
- *Why:* the libraries evolve at different speeds, and independent versions let an app upgrade one without taking the others.

**2. How each app consumes only what it needs: one umbrella repository per app.**
- The five apps use different subsets of the libraries, so each app gets its **own umbrella repository** that pulls in **only the libraries it needs**.
- The Android app depends on the libraries directly. The iOS app consumes **only its umbrella framework**.
- *Why:* the umbrella is required on iOS anyway, to avoid duplicated dependencies. For example, `networking` and `analytics` may both use `logging`, and without a single framework iOS would get two incompatible copies of it. Per-app umbrellas also mean no app is forced to ship libraries it doesn't use.

Total: **3 library repos + 5 umbrella repos** (the app repos themselves already exist).

**3. Distribution: remote, with the hybrid workflow on top.**
- **Remote (artifact) distribution as the default.** With 200 engineers, local/source distribution doesn't scale. It would require every iOS developer to have a full Gradle/Android Studio environment, and the lecture notes it "gets harder to manage as more code is shared and the team grows". The libraries are also already independently versioned and published, so remote is the natural fit. Most iOS developers never need to touch Kotlin, Gradle or Android Studio.
- **Hybrid for the people changing the libraries.** The cost of remote is a slow loop (publish → version → rebuild) and hard debugging. Engineers actively changing a library temporarily switch their app to a **local dependency** to get an immediate edit–build–debug loop. When the change is ready, they **publish the library first**, then update the app to the new remote version.
- *Tooling:* Swift Package Manager for the remote workflow with direct framework integration for local, or CocoaPods with an environment variable to switch between local and remote.

**Trade-off accepted:** publishing overhead and version management across many repositories, in exchange for platform independence, reuse across five apps, and a painless onboarding experience for non-Kotlin developers.

---

## Repository structure

```
kmp-practice-4/
├── README.md                  ← answers (this file)
├── settings.gradle.kts
├── .gitignore
├── screenshots/               ← proof for Q16, Q17, Q21
└── shared/
    ├── build.gradle.kts       ← Q16, Q17, Q21 build configuration
    └── src/jvmAndJsMain/kotlin/SharedGreeting.kt
```

## References

- Lecture 10 — KMP Advanced Project Structure
- Lecture 11 — Choosing a KMP Project Configuration
