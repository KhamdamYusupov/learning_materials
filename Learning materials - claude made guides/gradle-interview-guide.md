# Gradle Interview Guide for Middle-Level Developers

---

## 1. Executive Summary

Gradle is a **programmable build automation platform** built on a Directed Acyclic Graph (DAG) task execution engine. Unlike Maven's rigid lifecycle, Gradle gives you a general-purpose JVM-based scripting environment (Groovy or Kotlin) that happens to specialize in building software. This flexibility is Gradle's greatest strength and its greatest source of complexity.

**Why modern projects use Gradle over alternatives:**

- **Incremental builds**: Gradle tracks inputs and outputs of every task. If nothing changed, the task is skipped. This alone can cut local build times by 60-80%.
- **Build cache**: Task outputs are content-addressed and cacheable across machines. A CI build can reuse outputs from a previous run or even from a teammate's machine.
- **Programmability**: Build logic is real code, not XML configuration. Conditional logic, loops, API calls—all available natively.
- **Performance at scale**: Parallel task execution, configuration avoidance, and the Gradle Daemon eliminate repeated startup costs.

**What interviewers expect from mid-level developers:**

| Topic | Expected Depth |
|---|---|
| Task graph and DAG execution | Understand configuration vs execution phases, task dependencies |
| Dependency management | Know scopes, transitive resolution, conflict resolution strategy |
| Multi-project builds | Set up and reason about inter-module dependencies |
| Build performance | Explain incremental builds, build cache, and Gradle Daemon |
| Custom tasks & plugins | Write basic custom tasks; understand how plugins extend the build |
| Debugging builds | Use `--scan`, `--info`, `dependencyInsight`; read build output critically |
| CI integration | Know how Gradle behaves in CI, caching strategies, common failures |

You are not expected to know Gradle internals at the source-code level. You ARE expected to reason about why things happen the way they do, what trade-offs exist, and how to debug problems systematically.

---

## 2. Core Mental Models

### 2.1 Gradle as a Task-Based Build System

Maven operates on a fixed lifecycle: you plug goals into predetermined phases. Gradle operates on a **task graph**: you define tasks, declare dependencies between them, and Gradle figures out the execution order.

There is no fixed "compile → test → package" chain. The `java` plugin creates tasks (`compileJava`, `test`, `jar`) and wires their dependencies, but this wiring is just configuration—you can modify it, extend it, or replace it entirely.

A task is a unit of work with:
- **Inputs**: Source files, configuration properties, dependency artifacts
- **Outputs**: Compiled classes, JAR files, reports
- **Actions**: The actual code that transforms inputs into outputs
- **Dependencies**: Other tasks that must complete first

### 2.2 The Task Graph (DAG)

When you run `gradle build`, Gradle constructs a Directed Acyclic Graph of all tasks that need to execute, then walks the graph in topological order.

```
┌──────────────────────────────────────────────────────────┐
│                    TASK DAG FOR `gradle build`            │
│                                                          │
│                    ┌─────────┐                           │
│                    │  build  │                           │
│                    └────┬────┘                           │
│               ┌─────────┼──────────┐                    │
│               ▼         ▼          ▼                    │
│          ┌────────┐ ┌───────┐ ┌─────────┐              │
│          │ check  │ │ assemble│ │ (other)│              │
│          └───┬────┘ └───┬────┘ └─────────┘              │
│              │          │                                │
│              ▼          ▼                                │
│          ┌──────┐  ┌─────────┐                          │
│          │ test │  │   jar   │                          │
│          └──┬───┘  └────┬────┘                          │
│             │           │                                │
│        ┌────┴────┐      │                                │
│        ▼         ▼      ▼                                │
│  ┌───────────┐ ┌────────────┐                           │
│  │testClasses│ │  classes   │                           │
│  └─────┬─────┘ └──┬─────┬──┘                           │
│        │          │     │                                │
│        ▼          ▼     ▼                                │
│  ┌───────────┐ ┌──────────┐ ┌────────────────┐         │
│  │compileTest│ │compileJava│ │processResources│         │
│  │   Java    │ │          │ │                │         │
│  └───────────┘ └──────────┘ └────────────────┘         │
│                                                          │
│  Execution order (one valid topological sort):           │
│  compileJava → processResources → classes →              │
│  compileTestJava → testClasses → test → check            │
│  classes → jar → assemble → build                        │
└──────────────────────────────────────────────────────────┘
```

**Key insight**: Gradle doesn't execute tasks in a predetermined order. It computes the minimal set of tasks needed to satisfy your request, then executes them respecting dependency edges. If you run `gradle test`, Gradle will NOT execute `jar` or `assemble` because `test` doesn't depend on them.

### 2.3 Configuration Phase vs Execution Phase

This is the single most important concept for understanding Gradle behavior. Every Gradle build has two distinct phases:

```
┌─────────────────────────────────────────────────────┐
│                GRADLE BUILD PHASES                   │
│                                                     │
│  ┌─────────────────────────────────────────────┐    │
│  │  INITIALIZATION PHASE                        │    │
│  │  • Read settings.gradle                      │    │
│  │  • Determine which projects are in the build │    │
│  │  • Create Project instances                  │    │
│  └──────────────────┬──────────────────────────┘    │
│                     ▼                               │
│  ┌─────────────────────────────────────────────┐    │
│  │  CONFIGURATION PHASE                         │    │
│  │  • Execute ALL build.gradle scripts          │    │
│  │  • Configure ALL tasks (even ones you won't  │    │
│  │    run)                                      │    │
│  │  • Resolve task graph dependencies           │    │
│  │  • Build the DAG                             │    │
│  └──────────────────┬──────────────────────────┘    │
│                     ▼                               │
│  ┌─────────────────────────────────────────────┐    │
│  │  EXECUTION PHASE                             │    │
│  │  • Execute only requested tasks + their      │    │
│  │    dependencies                              │    │
│  │  • Run task actions in topological order     │    │
│  │  • Check up-to-date status before executing  │    │
│  └─────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────┘
```

**Why this distinction matters**: Code in `build.gradle` that is NOT inside a task action runs during **configuration**, for EVERY build, regardless of which tasks you requested. This is the source of a very common performance mistake:

```groovy
// BAD: This runs during CONFIGURATION phase (every build)
task generateReport {
    def data = file('data.json').text       // File read happens at configuration time!
    def parsed = new JsonSlurper().parse(data)
    doLast {
        println parsed.summary
    }
}

// GOOD: File read happens during EXECUTION phase (only when task runs)
task generateReport {
    doLast {
        def data = file('data.json').text
        def parsed = new JsonSlurper().parse(data)
        println parsed.summary
    }
}
```

In the bad example, even running `gradle help` will read and parse `data.json` because configuration runs for every task.

### 2.4 Incremental Builds and Up-to-Date Checks

When a task declares its inputs and outputs, Gradle stores a snapshot of them after execution. On the next build, Gradle compares current inputs/outputs against the snapshot. If nothing changed, the task is marked `UP-TO-DATE` and skipped.

```groovy
// Kotlin DSL
tasks.register("processConfig") {
    // Declare inputs
    inputs.file("src/config/application.yml")
    inputs.property("env", project.findProperty("env") ?: "dev")

    // Declare outputs
    outputs.file("$buildDir/config/application-processed.yml")

    doLast {
        // ... transformation logic
    }
}
```

**What Gradle tracks for up-to-date checks:**
- File content hashes (not timestamps—unlike Maven)
- File paths
- Property values
- Task class implementation (if you change the task code, it re-executes)

**Common pitfall**: A task without declared outputs is NEVER up-to-date. It runs every time. This is why poorly written custom tasks can silently destroy incremental build performance.

---

## 3. Dependency Management

### 3.1 Dependency Configurations (Scopes)

Gradle uses **configurations** to group dependencies by their purpose. The Java plugin defines these key configurations:

| Configuration | Compile Classpath | Runtime Classpath | Exposed to Consumers | Typical Use |
|---|---|---|---|---|
| `api` | Yes | Yes | Yes (compile + runtime) | Library APIs visible to consumers |
| `implementation` | Yes | Yes | No (runtime only) | Internal implementation details |
| `compileOnly` | Yes | No | No | Compile-time annotations, provided deps |
| `runtimeOnly` | No | Yes | No | JDBC drivers, SLF4J bindings |
| `testImplementation` | Test only | Test only | No | JUnit, Mockito |
| `testRuntimeOnly` | No | Test only | No | Test engines, test-only drivers |

**The `api` vs `implementation` distinction** is the most commonly asked about in interviews:

```groovy
// library-module/build.gradle
dependencies {
    // Guava is part of this library's PUBLIC API
    // (appears in method signatures, return types, etc.)
    // Consumers need it on their compile classpath
    api 'com.google.guava:guava:33.0.0-jre'

    // OkHttp is used INTERNALLY only
    // Consumers don't need to know about it at compile time
    // They'll get it on their runtime classpath automatically
    implementation 'com.squareup.okhttp3:okhttp:4.12.0'
}
```

**Why this matters for build performance**: When you change a dependency declared as `implementation`, only the current module recompiles. When you change an `api` dependency, every module that depends on this one must also recompile. In a 20-module project, using `api` everywhere instead of `implementation` can double or triple incremental compile times.

```
┌───────────────────────────────────────────────────────────┐
│         IMPACT OF api vs implementation                    │
│                                                           │
│  Module A (library)                                       │
│    ├── api: guava           ──► Change guava version      │
│    └── implementation: okhttp   │                         │
│                                 │                         │
│  Module B (depends on A)        ▼                         │
│    └── implementation(A)     B MUST recompile             │
│                              (guava is on B's             │
│                               compile classpath)          │
│                                                           │
│  If guava were 'implementation' instead of 'api':         │
│    Change guava version → ONLY A recompiles               │
│    B does NOT recompile (guava not on B's                 │
│    compile classpath)                                     │
└───────────────────────────────────────────────────────────┘
```

### 3.2 Transitive Dependencies

When you depend on `library-a`, and `library-a` depends on `library-b`, Gradle automatically brings `library-b` onto your classpath. This is transitive resolution.

Gradle builds a full dependency graph and resolves it. You can inspect it:

```bash
# Full dependency tree
gradle dependencies --configuration runtimeClasspath

# Find why a specific dependency is included
gradle dependencyInsight --dependency jackson-databind --configuration runtimeClasspath
```

### 3.3 Version Conflicts and Resolution Strategy

**Gradle's default strategy: newest version wins.** This is different from Maven's nearest-wins.

```
your-app
├── lib-a:1.0 → jackson-databind:2.14.0
└── lib-b:2.0 → jackson-databind:2.16.0

Gradle resolves: jackson-databind:2.16.0 (newest wins)
Maven would resolve: jackson-databind:2.14.0 (nearest/first-encountered wins)
```

This is generally safer than Maven's approach because newer versions are more likely to be backward-compatible. But "newest" does NOT mean "correct"—a major version bump can break binary compatibility.

**Forcing a specific version:**

```groovy
// build.gradle
dependencies {
    implementation 'com.example:lib-a:1.0'
    implementation 'com.example:lib-b:2.0'

    // Force a specific version globally
    implementation('com.fasterxml.jackson.core:jackson-databind') {
        version {
            strictly '2.15.3'  // Reject any other version, fail the build if incompatible
        }
    }
}
```

**Failing on conflicts instead of silently resolving:**

```groovy
configurations.all {
    resolutionStrategy {
        failOnVersionConflict()  // Build fails if any version conflict exists
    }
}
```

**Using a platform (BOM) for consistent versions:**

```groovy
dependencies {
    // Import Spring Boot's BOM to align all Spring-related versions
    implementation platform('org.springframework.boot:spring-boot-dependencies:3.2.1')

    // Now declare without version — the platform provides it
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'com.fasterxml.jackson.core:jackson-databind'
}
```

### 3.4 Dependency Constraints

Constraints let you influence version selection without adding a dependency:

```groovy
dependencies {
    constraints {
        // "If anyone in the graph pulls in log4j-core, use at least 2.17.1"
        implementation('org.apache.logging.log4j:log4j-core') {
            version {
                require '2.17.1'   // Minimum version (can be upgraded by conflict resolution)
            }
            because 'CVE-2021-44228 (Log4Shell) - versions before 2.17.1 are vulnerable'
        }
    }
}
```

**Constraints vs direct dependencies:** A constraint only takes effect if the dependency is already in the graph (pulled transitively). A direct dependency always adds the artifact.

### 3.5 Dependency Locking

For reproducible builds, you can lock resolved versions:

```groovy
// build.gradle
dependencyLocking {
    lockAllConfigurations()
}
```

```bash
# Generate lock files
gradle dependencies --write-locks
# Creates gradle/dependency-locks/*.lockfile
```

Lock files record the exact resolved version of every dependency. On subsequent builds, Gradle verifies that resolution produces the same versions. If not, the build fails.

**When to use locking**: Production applications that need reproducible builds. NOT for libraries (you want consumers to resolve versions flexibly).

### 3.6 Common Dependency Mistakes

**Mistake 1: Using `api` everywhere**
```groovy
// BAD: Everything leaks to consumers, slow incremental builds
dependencies {
    api 'com.google.guava:guava:33.0.0-jre'
    api 'io.netty:netty-all:4.1.100.Final'
    api 'com.fasterxml.jackson.core:jackson-databind:2.16.0'
}

// GOOD: Only expose what consumers actually need
dependencies {
    api 'com.google.guava:guava:33.0.0-jre'              // Used in public API
    implementation 'io.netty:netty-all:4.1.100.Final'     // Internal networking
    implementation 'com.fasterxml.jackson.core:jackson-databind:2.16.0' // Internal serialization
}
```

**Mistake 2: Not excluding unwanted transitives**
```groovy
dependencies {
    implementation('org.springframework.boot:spring-boot-starter-web') {
        exclude group: 'org.springframework.boot', module: 'spring-boot-starter-tomcat'
    }
    implementation 'org.springframework.boot:spring-boot-starter-undertow'
}
```

**Mistake 3: Mixing version declaration strategies**
```groovy
// BAD: Some versions from BOM, some hardcoded, some from ext — impossible to maintain
dependencies {
    implementation platform('org.springframework.boot:spring-boot-dependencies:3.2.1')
    implementation 'org.springframework.boot:spring-boot-starter-web'        // version from BOM
    implementation 'com.google.guava:guava:33.0.0-jre'                      // hardcoded
    implementation "io.netty:netty-all:${nettyVersion}"                      // from ext property
}

// BETTER: Centralize ALL versions in one place
// gradle/libs.versions.toml (Gradle version catalog)
```

### 3.7 Version Catalogs (Modern Approach)

```toml
# gradle/libs.versions.toml
[versions]
spring-boot = "3.2.1"
jackson = "2.16.0"
guava = "33.0.0-jre"

[libraries]
spring-boot-starter-web = { module = "org.springframework.boot:spring-boot-starter-web", version.ref = "spring-boot" }
jackson-databind = { module = "com.fasterxml.jackson.core:jackson-databind", version.ref = "jackson" }
guava = { module = "com.google.guava:guava", version.ref = "guava" }

[bundles]
spring-web = ["spring-boot-starter-web", "jackson-databind"]

[plugins]
spring-boot = { id = "org.springframework.boot", version.ref = "spring-boot" }
```

```groovy
// build.gradle — type-safe accessors generated from the catalog
dependencies {
    implementation libs.spring.boot.starter.web
    implementation libs.jackson.databind
    implementation libs.guava

    // Or use a bundle
    implementation libs.bundles.spring.web
}
```

**Real-world issue: Transitive version conflict causing runtime crash**

A team used `spring-boot-starter-web` (which pulls Jackson 2.15.x) alongside `aws-sdk-java-v2` (which pulls Jackson 2.14.x). Gradle resolved to 2.15.x (newest wins). AWS SDK methods that relied on 2.14.x-specific internal Jackson APIs threw `NoSuchMethodError` at runtime when serializing DynamoDB responses. The fix: add a dependency constraint forcing a version compatible with both, or use the AWS BOM which aligns Jackson versions.

---

## 4. Build Lifecycle & Task Execution

### 4.1 Configuration Phase Deep Dive

During configuration, Gradle executes your entire `build.gradle` script top to bottom. Every `task {}` block, every `dependencies {}` block, every plugin application—all of this runs during configuration.

```groovy
// Everything here runs during CONFIGURATION
println "Configuring project: ${project.name}"  // Runs even for `gradle help`

plugins {
    id 'java'  // Configures Java tasks during configuration
}

// This block CONFIGURES the compileJava task (doesn't execute it)
tasks.named('compileJava') {
    options.encoding = 'UTF-8'
    options.compilerArgs.add('-Xlint:unchecked')
}

// This REGISTERS a new task (lazy — configured only when needed)
tasks.register('myTask') {
    doLast {
        println "This runs during EXECUTION"
    }
}
```

**`tasks.register` vs `tasks.create`**: `register` is lazy—the task's configuration block only runs if the task is actually needed for the build. `create` (or the older `task` keyword) eagerly configures the task during the configuration phase regardless of whether it will execute. Use `register` for better configuration-phase performance.

```groovy
// BAD: Task is configured even if never executed
task processData {                            // Eager creation
    def data = file('large-data.csv').text    // Runs during configuration!
    doLast { println data }
}

// GOOD: Task is configured only when needed
tasks.register('processData') {               // Lazy registration
    doLast {
        def data = file('large-data.csv').text  // Runs during execution
        println data
    }
}
```

### 4.2 Execution Phase and Up-to-Date Checks

```
┌─────────────────────────────────────────────────────────────┐
│          TASK EXECUTION DECISION FLOW                        │
│                                                             │
│  For each task in topological order:                        │
│                                                             │
│  ┌──────────────────┐                                      │
│  │  Check inputs &  │                                      │
│  │  outputs snapshot │                                      │
│  └────────┬─────────┘                                      │
│           │                                                 │
│      Changed?                                               │
│       │      │                                              │
│      YES     NO                                             │
│       │      │                                              │
│       ▼      ▼                                              │
│  ┌────────┐ ┌──────────────┐                               │
│  │EXECUTE │ │ UP-TO-DATE   │                               │
│  │  task  │ │  (skip)      │                               │
│  └───┬────┘ └──────────────┘                               │
│      │                                                      │
│      ▼                                                      │
│  ┌───────────────────┐                                     │
│  │ Check build cache │                                     │
│  │ (if cache enabled)│                                     │
│  └────────┬──────────┘                                     │
│      Hit?                                                   │
│       │      │                                              │
│      YES     NO                                             │
│       │      │                                              │
│       ▼      ▼                                              │
│  ┌────────┐ ┌──────────────┐                               │
│  │FROM    │ │  RUN task    │                               │
│  │CACHE   │ │  actions     │                               │
│  └────────┘ └──────────────┘                               │
│                                                             │
│  Possible task outcomes:                                    │
│  • UP-TO-DATE — inputs/outputs unchanged                    │
│  • FROM-CACHE — outputs restored from build cache           │
│  • executed   — task actions ran                            │
│  • SKIPPED    — task was disabled (enabled = false)         │
│  • NO-SOURCE  — task has no input files to process          │
└─────────────────────────────────────────────────────────────┘
```

### 4.3 Task Dependencies and Ordering

```groovy
tasks.register('generateSchema') {
    outputs.file("$buildDir/schema.sql")
    doLast {
        file("$buildDir/schema.sql").text = "CREATE TABLE users(...);"
    }
}

tasks.register('migrateDb') {
    // Explicit dependency: migrateDb runs AFTER generateSchema
    dependsOn 'generateSchema'
    doLast {
        println "Running migration with ${file("$buildDir/schema.sql").text}"
    }
}

tasks.register('seedData') {
    // mustRunAfter: ordering constraint WITHOUT dependency
    // seedData won't CAUSE migrateDb to run, but IF both run, this order is enforced
    mustRunAfter 'migrateDb'
    doLast {
        println "Inserting seed data..."
    }
}

tasks.register('validateSchema') {
    // finalizedBy: cleanup tasks that ALWAYS run after this task, even on failure
    finalizedBy 'reportResults'
    doLast {
        println "Validating schema..."
    }
}
```

**`dependsOn` vs `mustRunAfter` vs `shouldRunAfter`:**

| Mechanism | Creates dependency edge? | Ordering guaranteed? | Forces task inclusion? |
|---|---|---|---|
| `dependsOn` | Yes | Yes | Yes — dependent task is added to the graph |
| `mustRunAfter` | No | Yes (hard) | No — only orders IF both are in graph |
| `shouldRunAfter` | No | Soft (can be broken to avoid cycles) | No |
| `finalizedBy` | Yes (reverse) | Yes | Yes — finalizer always added |

### 4.4 Custom Tasks

```groovy
// Inline task (quick and simple)
tasks.register('printVersion') {
    group = 'info'
    description = 'Prints the project version'
    doLast {
        println "Version: ${project.version}"
    }
}

// Typed task class (reusable, supports incremental builds)
abstract class GenerateConfigTask extends DefaultTask {

    @Input
    abstract Property<String> getEnvironment()

    @InputFile
    abstract RegularFileProperty getTemplateFile()

    @OutputFile
    abstract RegularFileProperty getOutputFile()

    @TaskAction
    void generate() {
        def template = templateFile.get().asFile.text
        def result = template.replace('{{ENV}}', environment.get())
        outputFile.get().asFile.text = result
    }
}

// Register typed task
tasks.register('generateConfig', GenerateConfigTask) {
    environment.set(project.findProperty('env') ?: 'dev')
    templateFile.set(file('src/config/template.yml'))
    outputFile.set(file("$buildDir/config/application.yml"))
}
```

**Why typed tasks matter**: The `@Input`, `@InputFile`, `@OutputFile` annotations tell Gradle exactly what to track for up-to-date checks and caching. Without them, the task runs every time.

---

## 5. Multi-Project Builds

### 5.1 Project Structure

```
┌──────────────────────────────────────────────────────────┐
│              MULTI-PROJECT STRUCTURE                      │
│                                                          │
│  my-application/                                         │
│  ├── settings.gradle          ← Defines project tree    │
│  ├── build.gradle             ← Root build config        │
│  ├── gradle.properties        ← Shared properties        │
│  ├── gradle/                                             │
│  │   ├── libs.versions.toml   ← Version catalog          │
│  │   └── wrapper/                                        │
│  ├── common/                                             │
│  │   ├── build.gradle                                    │
│  │   └── src/main/java/...                               │
│  ├── api/                                                │
│  │   ├── build.gradle                                    │
│  │   └── src/main/java/...                               │
│  ├── service/                                            │
│  │   ├── build.gradle                                    │
│  │   └── src/main/java/...                               │
│  └── webapp/                                             │
│      ├── build.gradle                                    │
│      └── src/main/java/...                               │
│                                                          │
│  Dependency graph:                                       │
│                                                          │
│         common                                           │
│        /      \                                          │
│      api    (standalone)                                 │
│       |                                                  │
│    service                                               │
│       |                                                  │
│    webapp                                                │
└──────────────────────────────────────────────────────────┘
```

**settings.gradle:**

```groovy
rootProject.name = 'my-application'
include 'common', 'api', 'service', 'webapp'
```

**Root build.gradle (shared configuration):**

```groovy
// Apply to ALL subprojects
subprojects {
    apply plugin: 'java-library'

    java {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }

    repositories {
        mavenCentral()
    }

    test {
        useJUnitPlatform()
    }
}
```

**Module build.gradle (service/build.gradle):**

```groovy
dependencies {
    // Inter-project dependency
    implementation project(':api')
    implementation project(':common')

    implementation 'org.springframework.boot:spring-boot-starter-web'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
```

### 5.2 Modern Approach: Convention Plugins

Instead of `subprojects {}` / `allprojects {}` blocks (which eagerly configure everything), use convention plugins in `buildSrc`:

```
my-application/
├── buildSrc/
│   ├── build.gradle
│   └── src/main/groovy/
│       └── myapp.java-conventions.gradle
├── common/
│   └── build.gradle
├── service/
│   └── build.gradle
└── settings.gradle
```

```groovy
// buildSrc/src/main/groovy/myapp.java-conventions.gradle
plugins {
    id 'java-library'
}

java {
    sourceCompatibility = JavaVersion.VERSION_17
}

repositories {
    mavenCentral()
}

test {
    useJUnitPlatform()
}
```

```groovy
// service/build.gradle — much cleaner
plugins {
    id 'myapp.java-conventions'
    id 'org.springframework.boot'
}

dependencies {
    implementation project(':api')
}
```

**Why convention plugins are better than `subprojects {}`:**
- Lazy configuration: Each project only applies the plugins it needs
- Type safety: Plugins can use Kotlin DSL with full IDE support
- Reusability: Convention plugins can be published and shared across repositories
- Clarity: Each build file explicitly declares what conventions it follows

### 5.3 Inter-Project Dependencies and the Classpath

When you write `implementation project(':api')`, Gradle creates a dependency edge between projects. During compilation, the `:service:compileJava` task depends on `:api:compileJava` (and its outputs).

```
┌──────────────────────────────────────────────────────┐
│    INTER-PROJECT DEPENDENCY RESOLUTION               │
│                                                      │
│  :service:compileJava                                │
│     │                                                │
│     ├── depends on → :api:jar (or :api:classes)      │
│     │                  │                             │
│     │                  └── depends on → :api:        │
│     │                       compileJava              │
│     │                                                │
│     └── depends on → :common:jar (or :common:classes)│
│                        │                             │
│                        └── depends on → :common:     │
│                             compileJava              │
│                                                      │
│  Gradle configures these dependencies automatically  │
│  when you use project(':api')                        │
└──────────────────────────────────────────────────────┘
```

### 5.4 Composite Builds

Composite builds let you develop against a local version of an external dependency without publishing it:

```groovy
// settings.gradle
includeBuild '../shared-library'  // Local checkout of a library
```

When Gradle encounters a dependency that matches an included build's published coordinates, it substitutes the local project automatically. This is useful for developing a library and its consumer simultaneously.

### 5.5 Common Multi-Module Mistakes

**Mistake 1: Circular dependencies**
```
:module-a depends on :module-b
:module-b depends on :module-a
→ Gradle fails at configuration time with a clear error
```
Fix: Extract shared code into a `:common` module that both depend on.

**Mistake 2: Using `allprojects {}` for everything**
```groovy
// BAD: Forces Java plugin on the root project (which is just an aggregator)
allprojects {
    apply plugin: 'java'
}

// BETTER: Only apply to subprojects that need it
subprojects {
    if (it.name != 'docs') {
        apply plugin: 'java-library'
    }
}

// BEST: Use convention plugins (each module opts in)
```

**Mistake 3: Not using `api` vs `implementation` correctly between modules**
```groovy
// In :api module
dependencies {
    // If :service depends on :api, and :service needs Guava at compile time:
    api 'com.google.guava:guava:33.0.0-jre'
    // If :service doesn't need Jackson at compile time:
    implementation 'com.fasterxml.jackson.core:jackson-databind:2.16.0'
}
```

---

## 6. Plugins & Customization

### 6.1 Applying Plugins

```groovy
// Modern approach: plugins {} block (recommended)
plugins {
    id 'java-library'                              // Core plugin (no version needed)
    id 'org.springframework.boot' version '3.2.1'  // Community plugin (version required)
    id 'io.freefair.lombok' version '8.4'
}

// Legacy approach: apply plugin (still works, sometimes needed)
apply plugin: 'java'
apply plugin: 'jacoco'
```

**`plugins {}` vs `apply plugin`**: The `plugins {}` block is resolved against the Gradle Plugin Portal at configuration time and supports version management. `apply plugin` requires the plugin to already be on the build classpath. For new code, always prefer `plugins {}`.

### 6.2 Plugin Version Management

Centralize plugin versions in `settings.gradle`:

```groovy
// settings.gradle
pluginManagement {
    plugins {
        id 'org.springframework.boot' version '3.2.1'
        id 'io.freefair.lombok' version '8.4'
    }
    repositories {
        gradlePluginPortal()
        mavenCentral()
    }
}
```

```groovy
// build.gradle — no version needed, comes from pluginManagement
plugins {
    id 'org.springframework.boot'
    id 'io.freefair.lombok'
}
```

Or use a version catalog:

```toml
# gradle/libs.versions.toml
[plugins]
spring-boot = { id = "org.springframework.boot", version = "3.2.1" }
lombok = { id = "io.freefair.lombok", version = "8.4" }
```

```groovy
plugins {
    alias(libs.plugins.spring.boot)
    alias(libs.plugins.lombok)
}
```

### 6.3 Writing Simple Custom Task Logic

Beyond registering ad-hoc tasks, you can create buildSrc-based tasks for reuse:

```groovy
// buildSrc/src/main/groovy/com/example/VersionBumpTask.groovy
package com.example

import org.gradle.api.DefaultTask
import org.gradle.api.tasks.*

abstract class VersionBumpTask extends DefaultTask {

    @Input
    abstract Property<String> getBumpType()  // "major", "minor", "patch"

    @InputFile
    abstract RegularFileProperty getVersionFile()

    @TaskAction
    void bump() {
        def file = versionFile.get().asFile
        def current = file.text.trim()
        def parts = current.split('\\.').collect { it.toInteger() }

        switch (bumpType.get()) {
            case 'major': parts[0]++; parts[1] = 0; parts[2] = 0; break
            case 'minor': parts[1]++; parts[2] = 0; break
            case 'patch': parts[2]++; break
        }

        def newVersion = parts.join('.')
        file.text = newVersion
        logger.lifecycle("Version bumped: $current → $newVersion")
    }
}
```

```groovy
// build.gradle
tasks.register('bumpVersion', com.example.VersionBumpTask) {
    bumpType.set(project.findProperty('bump') ?: 'patch')
    versionFile.set(file('version.txt'))
}
```

```bash
gradle bumpVersion -Pbump=minor
```

### 6.4 Real-World Plugin Misconfiguration

**Scenario**: Team applies `spring-boot` plugin to a library module.

```groovy
// common/build.gradle
plugins {
    id 'java-library'
    id 'org.springframework.boot'  // WRONG: This is a library, not a bootable app
}
```

**What happens**: The `spring-boot` plugin disables the `jar` task and enables `bootJar` instead. Now `:common` produces a fat JAR with embedded Spring Boot launcher classes. When `:service` depends on `:common`, it gets the fat JAR on its classpath, which contains repackaged classes under `BOOT-INF/classes/`. The compiler can't find `:common`'s classes. Build fails with confusing "class not found" errors.

**Fix**: Only apply `spring-boot` plugin to the module that produces the deployable application. Libraries use `java-library`.

```groovy
// common/build.gradle — CORRECT
plugins {
    id 'java-library'
    // No spring-boot plugin here
}

// webapp/build.gradle — application module
plugins {
    id 'java'
    id 'org.springframework.boot'
}
```

---

## 7. Testing & Quality Integration

### 7.1 Test Task Configuration

```groovy
// build.gradle
test {
    useJUnitPlatform()

    // JVM settings for test execution
    jvmArgs '-Xmx1024m', '-XX:+UseG1GC'

    // System properties available in tests
    systemProperty 'spring.profiles.active', 'test'

    // Environment variables
    environment 'DB_URL', 'jdbc:h2:mem:testdb'

    // Logging
    testLogging {
        events 'passed', 'skipped', 'failed'
        showExceptions true
        showCauses true
        showStackTraces true
        exceptionFormat 'full'
    }

    // Fail fast: stop on first failure
    failFast = false

    // Retry flaky tests (requires plugin)
    // retry { maxRetries = 2; maxFailures = 5 }
}
```

### 7.2 Parallel Test Execution

```groovy
test {
    // Fork multiple JVMs for test execution
    maxParallelForks = Runtime.runtime.availableProcessors().intdiv(2) ?: 1

    // OR: Use JUnit 5 parallel execution (within a single JVM)
    systemProperty 'junit.jupiter.execution.parallel.enabled', 'true'
    systemProperty 'junit.jupiter.execution.parallel.mode.default', 'concurrent'
    systemProperty 'junit.jupiter.execution.parallel.config.fixed.parallelism', '4'
}
```

| Strategy | Isolation Level | Memory Cost | Speedup | Use When |
|---|---|---|---|---|
| `maxParallelForks = N` | Separate JVMs | N × JVM heap | Good | Tests have JVM-level side effects |
| JUnit 5 parallel | Same JVM, threads | Low | Better | Tests are thread-safe |
| Both combined | Multiple JVMs, each with threads | High | Best | Large test suites with mixed isolation needs |

**Gotcha**: `maxParallelForks` means separate JVMs. Each fork loads the full classpath and Spring context (if using Spring). For a Spring Boot project, each fork can use 500MB-1GB. Four forks on an 8GB CI machine can cause OOM.

### 7.3 Integration Tests with Separate Source Set

```groovy
// Define a separate source set for integration tests
sourceSets {
    integrationTest {
        java.srcDir 'src/integrationTest/java'
        resources.srcDir 'src/integrationTest/resources'
        compileClasspath += sourceSets.main.output + sourceSets.test.output
        runtimeClasspath += sourceSets.main.output + sourceSets.test.output
    }
}

configurations {
    integrationTestImplementation.extendsFrom testImplementation
    integrationTestRuntimeOnly.extendsFrom testRuntimeOnly
}

tasks.register('integrationTest', Test) {
    description = 'Runs integration tests'
    group = 'verification'
    testClassesDirs = sourceSets.integrationTest.output.classesDirs
    classpath = sourceSets.integrationTest.runtimeClasspath
    useJUnitPlatform()
    mustRunAfter test

    // Integration tests might need more memory and time
    jvmArgs '-Xmx2g'
    systemProperty 'spring.profiles.active', 'integration'
}

// Include integration tests in the `check` lifecycle
tasks.named('check') {
    dependsOn 'integrationTest'
}
```

### 7.4 JaCoCo Code Coverage

```groovy
plugins {
    id 'java'
    id 'jacoco'
}

jacoco {
    toolVersion = '0.8.11'
}

test {
    finalizedBy jacocoTestReport  // Generate report after tests
}

jacocoTestReport {
    dependsOn test
    reports {
        xml.required = true    // For CI tools (SonarQube, Codecov)
        html.required = true   // For human reading
        csv.required = false
    }
}

jacocoTestCoverageVerification {
    violationRules {
        rule {
            limit {
                counter = 'LINE'
                value = 'COVEREDRATIO'
                minimum = 0.80
            }
        }
        rule {
            limit {
                counter = 'BRANCH'
                value = 'COVEREDRATIO'
                minimum = 0.70
            }
        }
    }
}

// Fail the build if coverage thresholds are not met
tasks.named('check') {
    dependsOn jacocoTestCoverageVerification
}
```

**Performance note**: JaCoCo uses a Java agent that instruments bytecode at class load time. This adds 5-15% overhead to test execution. For very large test suites (10,000+ tests), this overhead can add minutes. Consider running coverage checks only on CI, not during local development:

```groovy
jacocoTestReport {
    enabled = System.getenv('CI') != null
}
```

---

## 8. Build Performance Basics

### 8.1 Incremental Builds

Gradle's incremental build feature skips tasks whose inputs and outputs haven't changed. This is the single biggest performance feature for local development.

```bash
# First run: everything executes
$ gradle build
BUILD SUCCESSFUL in 45s
8 actionable tasks: 8 executed

# Second run (no changes): everything skipped
$ gradle build
BUILD SUCCESSFUL in 2s
8 actionable tasks: 8 up-to-date

# After editing one Java file: only affected tasks re-execute
$ gradle build
BUILD SUCCESSFUL in 8s
8 actionable tasks: 3 executed, 5 up-to-date
```

**What breaks incremental builds:**
- Tasks without declared inputs/outputs (always re-execute)
- Using `new Date()` or `System.currentTimeMillis()` as an input (changes every time)
- Reading from a non-declared file (Gradle doesn't know to track it)
- Writing outputs outside of `$buildDir` (Gradle can't clean them)

### 8.2 Build Cache

The build cache stores task outputs keyed by a hash of the task's inputs. If inputs match, outputs are restored from cache instead of re-executing.

```properties
# gradle.properties — enable local build cache
org.gradle.caching=true
```

```groovy
// settings.gradle — configure remote cache (for sharing across CI machines)
buildCache {
    local {
        enabled = true
        directory = new File(rootDir, '.gradle/build-cache')
    }
    remote(HttpBuildCache) {
        url = 'https://cache.company.com/cache/'
        push = System.getenv('CI') != null  // Only CI pushes to remote cache
        credentials {
            username = System.getenv('CACHE_USER') ?: ''
            password = System.getenv('CACHE_PASS') ?: ''
        }
    }
}
```

**Local cache vs remote cache:**

| Feature | Local Cache | Remote Cache |
|---|---|---|
| Stored | `~/.gradle/caches/build-cache-1/` | HTTP server (Gradle Enterprise, custom) |
| Shared | No (single machine) | Yes (across CI agents, developers) |
| Speed | Very fast (disk read) | Network-dependent (10-100ms per entry) |
| Value | Speeds up repeated local builds | Speeds up CI cold starts, branch switches |

**Cache hit example**: Developer A builds `main` branch. Outputs cached. Developer B checks out `main` and builds. All tasks are `FROM-CACHE` — instant build.

### 8.3 The Gradle Daemon

Gradle starts a long-lived background JVM (the Daemon) that persists between builds. This eliminates JVM startup time (~2-5 seconds) and allows the JIT compiler to optimize Gradle's own code over repeated builds.

```properties
# gradle.properties
org.gradle.daemon=true          # Enabled by default since Gradle 3.0
org.gradle.jvmargs=-Xmx2048m   # Daemon heap size
```

```bash
# Check daemon status
gradle --status

# Stop all daemons
gradle --stop
```

**Daemon lifecycle**: Idle daemons are killed after 3 hours. If you change `org.gradle.jvmargs`, a new daemon spawns (old one becomes orphaned). On CI, daemons are often disabled because each build runs in a fresh container:

```properties
# CI gradle.properties
org.gradle.daemon=false
```

### 8.4 Configuration Avoidance

Configuration avoidance means not configuring tasks that won't execute. This reduces configuration phase time.

```groovy
// BAD: Eagerly creates and configures the task
task myExpensiveTask {
    // Configuration logic runs for EVERY build
    def files = fileTree('src').matching { include '**/*.xml' }
    inputs.files(files)
    doLast { /* ... */ }
}

// GOOD: Lazily registers the task
tasks.register('myExpensiveTask') {
    // Configuration logic runs ONLY if this task is in the execution graph
    def files = fileTree('src').matching { include '**/*.xml' }
    inputs.files(files)
    doLast { /* ... */ }
}
```

### 8.5 Parallel Project Execution

```properties
# gradle.properties
org.gradle.parallel=true   # Build independent projects in parallel
```

```
┌──────────────────────────────────────────────────────────┐
│        PARALLEL PROJECT EXECUTION                         │
│                                                          │
│  Without parallel:                                       │
│  :common ──────► :api ──────► :service ──────► :webapp   │
│  Total: 40s       30s          35s              25s      │
│  = 130s sequential                                       │
│                                                          │
│  With parallel (4 workers):                              │
│  Worker 1: :common ──────► :service ──────► :webapp      │
│  Worker 2:         :api ──────►                          │
│  Total: ≈ 100s (common→service→webapp = critical path)   │
│                                                          │
│  Speedup limited by critical path in dependency graph     │
└──────────────────────────────────────────────────────────┘
```

### 8.6 Common Performance Bottlenecks

| Bottleneck | Symptom | Diagnostic | Fix |
|---|---|---|---|
| Slow configuration | `gradle help` takes > 5s | `--profile` report, look at configuration time | Use `tasks.register`, avoid eager file I/O |
| No incremental build | Tasks always execute | Look for "executed" instead of "up-to-date" | Declare inputs/outputs on all tasks |
| Large dependency resolution | First build very slow | `--info` shows resolution time per configuration | Cache `~/.gradle/caches/`, use repository mirrors |
| Test-heavy builds | Tests take 70%+ of build time | `--profile` report, check test execution time | `maxParallelForks`, skip tests locally with `-x test` |
| buildSrc changes | Full rebuild on any buildSrc change | Any change in buildSrc invalidates all caches | Move logic to included build instead |

### 8.7 Debugging Slow Builds

```bash
# Generate a build profile report (HTML)
gradle build --profile
# Opens build/reports/profile/profile-<timestamp>.html

# Detailed logging
gradle build --info      # Logs why tasks execute/skip
gradle build --debug     # Very verbose (usually too much)

# Build scan (detailed web-based report)
gradle build --scan
# Requires accepting Gradle terms; generates a URL with detailed build analysis

# Specific diagnostics
gradle build --dry-run          # Show what WOULD execute without doing it
gradle compileJava --info       # Why did this specific task execute?
```

---

## 9. CI/CD Integration Basics

### 9.1 Running Gradle in CI

```bash
# Standard CI build command
./gradlew clean build

# Key flags for CI
./gradlew build \
    --no-daemon \             # Don't use daemon (ephemeral CI containers)
    --build-cache \           # Use build cache
    --parallel \              # Parallel project execution
    --stacktrace \            # Full stack traces on failure
    --warning-mode all        # Surface deprecation warnings
```

**Always use the Gradle Wrapper (`./gradlew`)**: The wrapper ensures every CI agent uses the exact same Gradle version. Never install Gradle globally on CI.

### 9.2 Caching in CI

```yaml
# GitHub Actions example
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Setup Gradle
        uses: gradle/actions/setup-gradle@v3
        with:
          cache-read-only: ${{ github.ref != 'refs/heads/main' }}
          # PRs read from cache but don't pollute it

      - name: Build
        run: ./gradlew build --parallel --build-cache
```

**What to cache:**
- `~/.gradle/caches/` — Downloaded dependencies, transforms
- `~/.gradle/wrapper/` — Gradle distribution
- `.gradle/` (project-level) — Local build cache, configuration cache

**What NOT to cache:**
- `build/` directories — These are outputs, not inputs; caching them can cause stale outputs
- `~/.gradle/daemon/` — Daemon state is machine-specific

### 9.3 Artifact Publishing

```groovy
plugins {
    id 'java-library'
    id 'maven-publish'
}

publishing {
    publications {
        mavenJava(MavenPublication) {
            from components.java  // Publishes jar + dependencies metadata

            pom {
                name = 'My Library'
                description = 'A useful library'
            }
        }
    }
    repositories {
        maven {
            name = 'internal'
            url = project.version.endsWith('-SNAPSHOT')
                ? 'https://nexus.company.com/repository/maven-snapshots/'
                : 'https://nexus.company.com/repository/maven-releases/'
            credentials {
                username = System.getenv('NEXUS_USER')
                password = System.getenv('NEXUS_PASS')
            }
        }
    }
}
```

### 9.4 Common CI Failures

**Failure 1: "Could not resolve all files for configuration"**
- **Cause**: Network issue, dependency removed from repository, or authentication failure
- **Fix**: Check repository URLs, verify credentials, check if the dependency still exists

**Failure 2: Flaky tests pass locally, fail in CI**
- **Cause**: Test depends on execution order, timezone, locale, or file system behavior
- **Fix**: Run tests with `--no-parallel` to isolate; check for shared mutable state

**Failure 3: Out of memory during build**
- **Cause**: Too many parallel forks, large project, insufficient CI agent resources
- **Fix**: Tune `org.gradle.jvmargs`, reduce `maxParallelForks`, use `--no-parallel`

**Example CI issue and resolution:**

A CI build intermittently fails with:
```
> Could not resolve com.example:shared-lib:1.5.0-SNAPSHOT
  > Could not get resource
    'https://nexus.company.com/repository/maven-snapshots/com/example/shared-lib/1.5.0-SNAPSHOT/maven-metadata.xml'
    > Read timed out
```

**Root cause**: Nexus was under heavy load during peak CI hours. SNAPSHOT metadata resolution requires a network round-trip every time (by default, every 24 hours for changing modules).

**Resolution**:
```groovy
// build.gradle — increase timeout and reduce SNAPSHOT check frequency
repositories {
    maven {
        url 'https://nexus.company.com/repository/maven-snapshots/'
        content {
            includeGroup 'com.example'  // Only use this repo for internal artifacts
        }
        metadataSources {
            mavenPom()
            artifact()
        }
    }
}

configurations.all {
    resolutionStrategy.cacheChangingModulesFor 4, 'hours'  // Don't re-check SNAPSHOTs every build
}
```

---

## 10. Common Failure Scenarios

### Scenario 1: The Mysterious "Task Not Found" After Plugin Upgrade

**What happened**: After upgrading from Spring Boot 2.x to 3.x, the build fails with `Task 'bootRun' not found in root project`. The same `build.gradle` worked yesterday.

**Root cause**: The Spring Boot Gradle plugin 3.x requires Gradle 7.5+ and Java 17+. The CI agent was running Gradle 7.3 (from the wrapper). The plugin loaded but failed silently during configuration, not registering its tasks.

**How to debug**:
```bash
# Check Gradle version
./gradlew --version

# Run with --info to see plugin application details
./gradlew bootRun --info

# Check if the task exists
./gradlew tasks --all | grep boot
```

**How to prevent**:
- Always read plugin upgrade guides before updating
- Update `gradle/wrapper/gradle-wrapper.properties` when upgrading plugins that require newer Gradle
- Pin Gradle version in wrapper and test upgrades in a branch

```properties
# gradle/wrapper/gradle-wrapper.properties
distributionUrl=https\://services.gradle.org/distributions/gradle-8.5-bin.zip
```

---

### Scenario 2: Build Cache Poisoning — "FROM-CACHE" But Wrong Output

**What happened**: A developer's build produced a JAR with incorrect contents. Tests passed locally. In production, a class had stale code from a previous build. Running `gradle clean build` fixed it.

**Root cause**: A custom task did not declare all its inputs. It read a configuration file that wasn't tracked:

```groovy
tasks.register('generateCode') {
    inputs.dir('src/templates')         // Declared
    outputs.dir("$buildDir/generated")  // Declared
    // MISSING: inputs.file('config/codegen.yml')  ← Not tracked!
    doLast {
        def config = file('config/codegen.yml').text  // Read but not tracked
        // ... generate code based on config ...
    }
}
```

When `codegen.yml` changed, Gradle didn't detect it. The task was `UP-TO-DATE` (or `FROM-CACHE`) with stale output.

**How to debug**:
```bash
# Check why a task was UP-TO-DATE
gradle generateCode --info
# Look for "Skipping task ':generateCode' as it is up-to-date"
# Check listed inputs — is your file there?
```

**How to prevent**:
- Always declare ALL inputs and outputs for custom tasks
- Use typed tasks with `@Input`, `@InputFile`, `@OutputDirectory` annotations
- Periodically run `gradle clean build` to verify cache correctness
- Use `--rerun-tasks` to force execution and compare outputs

---

### Scenario 3: Dependency Resolution Deadlock in Multi-Module Build

**What happened**: Build hangs indefinitely during configuration phase of a 15-module project. No error, no progress, CPU at 0%.

**Root cause**: Two modules had circular configuration-time dependencies through custom configurations:

```groovy
// module-a/build.gradle
configurations {
    schemaExport
}
dependencies {
    schemaExport project(path: ':module-b', configuration: 'schemaExport')
}

// module-b/build.gradle
configurations {
    schemaExport
}
dependencies {
    schemaExport project(path: ':module-a', configuration: 'schemaExport')
}
```

This created a cycle that Gradle's configuration-time dependency resolver couldn't detect (it's not a compile dependency cycle, it's a configuration resolution cycle).

**How to debug**:
```bash
# Thread dump to see where Gradle is stuck
# Kill -3 <gradle-daemon-pid>  (on Linux/Mac)
# Or use jstack

# Simplify: build modules individually to isolate the hang
./gradlew :module-a:build
./gradlew :module-b:build
```

**How to prevent**:
- Avoid custom cross-project configurations that can create cycles
- Extract shared schemas/contracts into a dedicated module
- Use `dependencyInsight` to visualize cross-project edges
- Keep inter-module dependencies unidirectional

---

## 11. Interview Question Bank

### 11.1 Core Concepts

**Q1: "What is the difference between the configuration phase and the execution phase in Gradle?"**

**Weak answer**: "Configuration is when Gradle reads the build file, execution is when it runs tasks."

**Strong answer**: "During the configuration phase, Gradle evaluates all `build.gradle` scripts and configures all tasks—even tasks that won't execute in this build. This phase builds the task DAG. During the execution phase, Gradle walks the DAG in topological order and runs only the tasks that are needed. The key practical implication is that code written directly in a task block (outside `doLast`/`doFirst`) runs at configuration time, which can cause surprising behavior. For example, file I/O in the configuration block runs on every build invocation, even `gradle help`. The fix is to put work inside `doLast {}` or use lazy configuration APIs like `tasks.register` instead of `task` keyword."

**Follow-up**: "What is configuration avoidance and why does it matter?"

**Expected answer**: "Configuration avoidance means not configuring tasks that won't execute. `tasks.register` creates a task lazily—its configuration block only runs if the task is in the execution graph. `tasks.create` (or `task` keyword) configures eagerly. In a large project with hundreds of tasks, eager configuration adds seconds to every build invocation. Gradle's modern APIs are designed around laziness: `Provider`, `Property`, `TaskProvider` all defer evaluation until needed."

---

**Q2: "How does Gradle handle dependency version conflicts?"**

**Weak answer**: "It picks the latest version."

**Strong answer**: "Gradle's default conflict resolution strategy selects the highest version when multiple versions of the same module appear in the dependency graph. This differs from Maven's nearest-wins strategy. You can change this behavior: `resolutionStrategy.failOnVersionConflict()` makes the build fail on any conflict, forcing explicit resolution. `resolutionStrategy.force('group:artifact:version')` pins a specific version. For richer control, dependency constraints let you specify minimum versions with explanations. And `strictly` version constraints reject any other version entirely. For production applications, I'd recommend using a platform (BOM) for consistent versions across the project and enabling `failOnVersionConflict` in CI to catch unexpected conflicts early."

---

**Q3: "Explain `implementation` vs `api` dependency configurations."**

**Weak answer**: "API is for public dependencies, implementation is for private."

**Strong answer**: "`implementation` means the dependency is on the module's compile and runtime classpath, but does NOT leak to consumers' compile classpath—consumers only get it at runtime. `api` means the dependency IS on consumers' compile classpath too. The practical impact is on incremental compilation: changing an `implementation` dependency only recompiles the current module. Changing an `api` dependency recompiles the current module AND all modules that depend on it. In a 20-module project, marking everything as `api` can double incremental build time because a change anywhere cascades through the graph. The rule of thumb: use `implementation` by default; only use `api` when the dependency's types appear in your module's public method signatures, return types, or superclass hierarchy."

---

### 11.2 Build Performance

**Q4: "How does Gradle's build cache work?"**

**Expected answer**: "Gradle computes a cache key for each task based on all declared inputs: file content hashes, property values, and the task implementation class. After execution, it stores the task outputs keyed by this hash. On a subsequent build, if the inputs hash matches a cache entry, Gradle restores outputs from cache instead of re-executing. There are two levels: a local cache on disk (`~/.gradle/caches/build-cache-1/`) and an optional remote cache (HTTP server, Gradle Enterprise). The local cache speeds up repeated local builds. The remote cache enables sharing across CI agents and developers. Cache correctness depends entirely on complete input/output declarations—if a task reads an undeclared file, the cache key won't reflect that file's changes, leading to stale outputs."

**Follow-up**: "When would a build cache entry NOT be reused?"

**Expected answer**: "Cache misses happen when any input changes: different source code, different dependency version, different JDK version, different task configuration property, different Gradle plugin version (which changes the task implementation class). Also, tasks that use absolute paths in their inputs get different cache keys on different machines. Gradle provides path sensitivity annotations (`@PathSensitive(PathSensitivity.RELATIVE)`) to normalize paths for cache-friendly behavior."

---

### 11.3 Multi-Project Builds

**Q5: "How would you optimize a slow multi-project Gradle build?"**

**Expected answer**: "I'd take a systematic approach:
1. **Profile first**: Run `gradle build --profile` or `--scan` to identify what's slow.
2. **Enable parallel builds**: `org.gradle.parallel=true` executes independent projects concurrently.
3. **Enable build cache**: `org.gradle.caching=true` skips tasks with unchanged inputs.
4. **Fix configuration time**: Replace eager `task` with `tasks.register`; avoid heavy I/O during configuration.
5. **Use `implementation` instead of `api`**: Reduces incremental recompilation scope.
6. **Parallelize tests**: `maxParallelForks` in each module's test task.
7. **Targeted builds**: During development, use `-x test` or build specific modules with `gradle :module:build`.
8. **CI-specific**: Cache `~/.gradle/caches`, use remote build cache, consider configuration cache for repeated builds."

---

### 11.4 Troubleshooting

**Q6: "A task that should be skipped (UP-TO-DATE) is running every time. How do you debug this?"**

**Expected answer**: "I'd run the task with `--info` flag. Gradle logs exactly why a task is not up-to-date: which input changed, which output is missing, or whether the task has no declared outputs (which makes it always execute). Common causes: (1) the task doesn't declare outputs, so it's never up-to-date; (2) an input uses a timestamp or random value that changes every build; (3) an output file is outside `buildDir` and gets deleted by other processes; (4) the task class itself was recompiled (buildSrc change); (5) a `doLast` or `doFirst` closure was added as a lambda, and Gradle can't determine implementation stability."

---

**Q7: "What is the Gradle Daemon and when should you disable it?"**

**Expected answer**: "The Gradle Daemon is a long-lived background JVM that survives between builds. It eliminates JVM cold start overhead and benefits from JIT compilation over time—subsequent builds are faster because hot paths are already compiled. You should disable it in CI environments where each build runs in a fresh container—the daemon provides no benefit since there's no 'next build' to speed up, and it consumes memory. Also disable it if you're running into memory issues locally, since each daemon holds onto heap even when idle. The daemon is killed automatically after 3 hours of inactivity."

---

## 12. Practice Exercises

### Exercise 1: Dependency Conflict Debugging

**Setup**: Create a project with these dependencies:

```groovy
dependencies {
    implementation 'org.apache.httpcomponents.client5:httpclient5:5.3'
    implementation 'org.elasticsearch.client:elasticsearch-rest-client:8.11.0'
}
```

**Tasks**:
1. Run `gradle dependencies --configuration runtimeClasspath` and identify any version conflicts
2. Run `gradle dependencyInsight --dependency httpcore5 --configuration runtimeClasspath` to trace why multiple versions appear
3. Resolve the conflict using three different strategies:
   a. Force a version with `resolutionStrategy`
   b. Use a `constraints` block
   c. Exclude the transitive and add a direct dependency

**Evaluation**:
- [2 pts] Correctly identifies the conflict
- [2 pts] Uses `dependencyInsight` to explain the resolution path
- [3 pts] Implements all three resolution strategies correctly
- [3 pts] Explains which strategy is best for production and why (constraints > force > exclude for maintainability)

---

### Exercise 2: Multi-Module Setup Challenge

**Task**: Create a 4-module Gradle project from scratch:

```
my-app/
├── settings.gradle
├── build.gradle
├── model/          (plain Java library — data classes)
├── persistence/    (depends on model — database access)
├── service/        (depends on persistence and model)
└── web/            (Spring Boot app — depends on service)
```

**Requirements**:
1. Use a version catalog (`libs.versions.toml`) for all dependency versions
2. Use convention plugins in `buildSrc` (no `subprojects {}` block)
3. Only `:web` module applies the Spring Boot plugin
4. All modules use `implementation` except where `api` is justified (document why)
5. `:model` module has no framework dependencies

**Evaluation**:
- [2 pts] Correct `settings.gradle` and project structure
- [2 pts] Working version catalog
- [2 pts] Convention plugin applied cleanly
- [2 pts] Correct `api` vs `implementation` usage with justification
- [2 pts] Only `:web` produces a bootable JAR

---

### Exercise 3: Performance Tuning

**Setup**: Given a multi-module project where `gradle build` takes 4 minutes:

```
Build profile (from --profile):
  Configuration:     12s
  :common:compileJava:    8s
  :common:test:          45s
  :api:compileJava:       5s
  :api:test:             30s
  :service:compileJava:  10s
  :service:test:         90s   ← Bottleneck
  :webapp:compileJava:    3s
  :webapp:test:          20s
  Other tasks:           17s
  Total:               240s
```

**Tasks**:
1. Identify the top 3 bottlenecks
2. Propose specific fixes for each (with configuration examples)
3. Estimate the new build time after optimizations
4. Explain what you would NOT change and why

**Evaluation**:
- [3 pts] Correctly identifies: (1) `:service:test` at 90s, (2) configuration at 12s, (3) `:common:test` at 45s
- [3 pts] Proposes: (1) `maxParallelForks` for tests, (2) `tasks.register` and configuration avoidance, (3) `--parallel` for concurrent module execution
- [2 pts] Reasonable time estimate (e.g., ~100-120s with parallel + faster tests)
- [2 pts] Explains what not to change (e.g., don't skip tests, don't disable incremental compilation)

---

### Exercise 4: Custom Task Creation

**Task**: Write a Gradle task type `LicenseCheckTask` that:

1. Scans all `*.java` files in `src/main/java`
2. Checks that each file starts with a license header comment
3. Reports files missing the header
4. Fails the build if any file is missing the header
5. Supports a `skip` property (`-PlicenseCheck.skip=true`)
6. Is fully incremental (up-to-date when no source files change)

**Requirements**:
- Use `@InputFiles`, `@Input`, `@OutputFile` annotations
- Write it as a typed task class
- Register it bound to the `check` lifecycle
- Write a test that verifies the task works

**Evaluation**:
- [2 pts] Correct annotations for incremental build support
- [3 pts] Working implementation that finds missing headers
- [2 pts] Properly fails the build with clear error message
- [1 pt] Skip property works correctly
- [2 pts] Task is up-to-date on second run without changes

---

## 13. 30-Day Gradle Improvement Plan

### Week 1: Foundations & Dependency Mastery

**Day 1-2: Build Phases**
- Read Gradle docs on [Build Lifecycle](https://docs.gradle.org/current/userguide/build_lifecycle.html)
- Create a sample project with `println` statements at configuration vs execution time
- Lab: Add a `println` inside and outside of `doLast {}`. Run `gradle help`. Observe which prints.
- Lab: Replace a `task` declaration with `tasks.register`. Run `gradle help` again. Observe the difference.

**Day 3-4: Dependency Configurations**
- Run `gradle dependencies --configuration compileClasspath` and `--configuration runtimeClasspath` on your real project
- Identify 5 dependencies that should be `implementation` instead of `api`
- Lab: Change them and measure incremental build time before/after (change one file, run `gradle build`)
- Lab: Run `gradle dependencyInsight --dependency <some-transitive> --configuration runtimeClasspath` and trace the resolution path

**Day 5: Version Conflicts**
- Add a dependency that creates a version conflict (e.g., two libraries pulling different Jackson versions)
- Resolve it three ways: `force`, `constraints`, `exclude`
- Enable `failOnVersionConflict()` and fix all violations
- Lab: Set up a version catalog and migrate all version declarations to `libs.versions.toml`

---

### Week 2: Task Graph & Build Performance

**Day 6-7: Task Graph Deep Dive**
- Run `gradle build --dry-run` on your project and study the task execution order
- Create three custom tasks with `dependsOn`, `mustRunAfter`, and `finalizedBy`
- Lab: Draw the task DAG for your project's `build` task (manually, from `--dry-run` output)

**Day 8-9: Incremental Builds**
- Write a custom task with `@InputFile` and `@OutputFile`
- Run it twice—verify `UP-TO-DATE` on second run
- Deliberately break incrementality (read a file without declaring it as input). Observe the consequence.
- Lab: Run `gradle build --info` and search for "out of date" messages. Identify any tasks that always execute.

**Day 10: Build Profiling**
- Run `gradle build --profile` on your project
- Identify the three slowest tasks
- Enable parallel builds (`--parallel`) and compare
- Enable build cache and compare
- Lab: Create a spreadsheet with columns: configuration, module, task, time. Track across 5 configurations.

---

### Week 3: Multi-Module & Plugins

**Day 11-12: Multi-Module Architecture**
- Complete Exercise 2 (multi-module setup from scratch)
- Set up a convention plugin in `buildSrc`
- Lab: Add a fifth module that depends on two others. Verify the build order with `--dry-run`.

**Day 13-14: Plugin Development**
- Complete Exercise 4 (LicenseCheckTask)
- Write a test for your task using `TestKit`
- Lab: Publish the task as a standalone plugin to your local Maven repository. Apply it from another project.

**Day 15: Composite Builds**
- Check out a library and its consumer as separate Gradle projects
- Use `includeBuild` in `settings.gradle` to develop both simultaneously
- Lab: Change a class in the library. Verify the consumer picks up the change without publishing.

---

### Week 4: CI Integration & Interview Prep

**Day 16-17: CI Pipeline**
- Set up a Gradle build in GitHub Actions (or your CI of choice)
- Configure dependency caching
- Configure build cache
- Lab: Measure cold vs warm CI build time. Document the improvement.

**Day 18-19: Artifact Publishing**
- Configure `maven-publish` plugin to publish a library module
- Set up SNAPSHOT vs release repository routing
- Lab: Publish a SNAPSHOT, depend on it from another project, update the SNAPSHOT, verify the consumer picks up the new version.

**Day 20-21: Mock Interview Practice**

Practice answering every question from Section 11 out loud. Time yourself—aim for 60-90 seconds per answer.

**Model answer for self-assessment (Q1 — Configuration vs Execution):**

"Gradle build has three phases: initialization reads `settings.gradle` to determine which projects are in the build. Configuration evaluates all `build.gradle` scripts and constructs the task DAG—this happens for every build invocation, even `gradle help`. Execution walks the DAG and runs only the tasks needed for the requested goal.

The practical implication is that anything outside `doLast`/`doFirst` in a task block runs during configuration. Heavy work there—like reading files, calling APIs, or computing values—slows down every build. The fix is to use `doLast {}` for work and lazy APIs like `Property<T>` and `Provider<T>` for configuration. Also, `tasks.register` defers even the configuration of tasks that won't execute, which is important in large builds with hundreds of tasks."

**Model answer for self-assessment (Q3 — `implementation` vs `api`):**

"`implementation` declares a dependency that's internal to the module. Consumers get it on their runtime classpath but NOT their compile classpath. `api` declares a dependency that's part of the module's public surface—consumers get it at both compile and runtime. The build performance implication is significant: when an `api` dependency changes, all downstream modules must recompile because the change might affect types they use. With `implementation`, only the declaring module recompiles. In a real project, I'd default to `implementation` for everything and only promote to `api` when a dependency's types actually appear in public method signatures, return types, or parent classes. In a 20-module project I worked on, switching overused `api` to `implementation` reduced incremental build time from 90 seconds to 35 seconds."

**Day 22-25: Consolidation**
- Review all exercises and fill knowledge gaps
- Re-read this guide and identify any concept you can't explain confidently
- Practice explaining the task DAG, dependency resolution, and build cache to a colleague
- Run through a full debugging scenario: introduce a dependency conflict, trace it with `dependencyInsight`, resolve it, verify with `--scan`

**Day 26-30: Final Preparation**
- Do a full mock interview (have someone ask you questions from Section 11 in random order)
- Complete any remaining exercises
- Review your build profiling spreadsheet and prepare talking points about real improvements you made
- Write a one-page summary of "what I learned about Gradle" — this becomes your narrative for interviews

---

### Success Criteria After 30 Days

You should be able to:

1. **Explain build phases**: Clearly distinguish configuration vs execution phase and explain their impact on build performance
2. **Resolve dependency conflicts**: Trace any version conflict using `dependencyInsight`, explain why Gradle chose a specific version, and fix it
3. **Optimize builds**: Profile a build, identify the top bottleneck, and apply the right fix (parallel, cache, configuration avoidance, or test tuning)
4. **Set up multi-module builds**: Create a clean multi-module project with convention plugins, version catalogs, and correct `api`/`implementation` usage
5. **Write custom tasks**: Create typed tasks with proper input/output annotations that support incremental builds
6. **Debug confidently**: Use `--info`, `--profile`, `--scan`, `dependencyInsight`, and `--dry-run` to investigate any build issue
7. **Answer interview questions**: Respond to any question from Section 11 with practical depth and real examples within 90 seconds

---

*This guide reflects the depth expected of a mid-level developer ready to demonstrate strong build engineering fundamentals. Gradle mastery is not about memorizing DSL syntax—it is about understanding the task graph model, the performance implications of your configuration choices, and having a systematic approach to debugging when things go wrong.*
