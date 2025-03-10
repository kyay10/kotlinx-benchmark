[![Kotlin Alpha](https://kotl.in/badges/alpha.svg)](https://kotlinlang.org/docs/components-stability.html)
[![JetBrains incubator project](https://jb.gg/badges/incubator.svg)](https://confluence.jetbrains.com/display/ALL/JetBrains+on+GitHub)
[![GitHub license](https://img.shields.io/badge/license-Apache%20License%202.0-blue.svg?style=flat)](https://www.apache.org/licenses/LICENSE-2.0)
[![Build status](https://teamcity.jetbrains.com/guestAuth/app/rest/builds/buildType:(id:KotlinTools_KotlinxCollectionsImmutable_Build_All)/statusIcon.svg)](https://teamcity.jetbrains.com/viewType.html?buildTypeId=KotlinTools_KotlinxBenchmark_Build_All)
[![Maven Central](https://img.shields.io/maven-central/v/org.jetbrains.kotlinx/kotlinx-benchmark-runtime.svg?label=Maven%20Central)](https://search.maven.org/search?q=g:%22org.jetbrains.kotlinx%22%20AND%20a:%22kotlinx-benchmark-runtime%22)
[![Gradle Plugin Portal](https://img.shields.io/maven-metadata/v?label=Gradle%20Plugin&metadataUrl=https://plugins.gradle.org/m2/org/jetbrains/kotlinx/kotlinx-benchmark-plugin/maven-metadata.xml)](https://plugins.gradle.org/plugin/org.jetbrains.kotlinx.benchmark)
[![IR](https://img.shields.io/badge/Kotlin%2FJS-IR%20supported-yellow)](https://kotl.in/jsirsupported)


> **_NOTE:_** &nbsp; Starting from version 0.3.0 of the library:
> * The library runtime is published to Maven Central and no longer published to Bintray.
> * The Gradle plugin is published to Gradle Plugin Portal
> * The Gradle plugin id has changed to `org.jetbrains.kotlinx.benchmark`
> * The library runtime artifact id has changed to `kotlinx-benchmark-runtime`


**kotlinx.benchmark** is a toolkit for running benchmarks for multiplatform code written in Kotlin 
and running on the following supported targets: JVM, JavaScript and Native.

Both Legacy and IR backends are supported for JS, however `kotlin.js.compiler=both` or `js(BOTH)` target declaration won't work.
You should declare each targeted backend separately. See build script of the [kotlin-multiplatform example project](https://github.com/Kotlin/kotlinx-benchmark/tree/master/examples/kotlin-multiplatform).

On JVM [JMH](https://openjdk.java.net/projects/code-tools/jmh/) is used under the hoods to run benchmarks.
This library has a very similar way of defining benchmark methods. Thus, using this library you can run your JMH-based 
Kotlin/JVM benchmarks on other platforms with minimum modifications, if any at all. 

# Requirements

Gradle 7.0 or newer

Kotlin 1.7.20 or newer

# Gradle plugin

Use plugin in `build.gradle`:

```groovy
plugins {
    id 'org.jetbrains.kotlinx.benchmark' version '0.4.4'
}
```

For Kotlin/JS specify building `nodejs` flavour:

```groovy
kotlin {
    js {
        nodejs()
        …
    }   
}
```

For Kotlin/JVM code, add `allopen` plugin to make JMH happy. Alternatively, make all benchmark classes and methods `open`.

For example, if you annotated each of your benchmark classes with `@State(Scope.Benchmark)`:
```kotlin
@State(Scope.Benchmark)
class Benchmark {
    …
}
```
and added the following code to your `build.gradle`:
```groovy
plugins {
    id 'org.jetbrains.kotlin.plugin.allopen'
}

allOpen {
    annotation("org.openjdk.jmh.annotations.State")
}
```
then you don't have to make benchmark classes and methods `open`. 

# Runtime Library

You need a runtime library with annotations and code that will run benchmarks.

Enable Maven Central for dependencies lookup:
```groovy
repositories {
    mavenCentral()
}
```

Add the runtime to dependencies of the platform source set, e.g.:
```  
kotlin {
    sourceSets {
        commonMain {
             dependencies {
                 implementation("org.jetbrains.kotlinx:kotlinx-benchmark-runtime:0.4.4")
             }
        }
    }
}
```

# Configuration

In a `build.gradle` file create `benchmark` section, and inside it add a `targets` section.
In this section register all compilations you want to run benchmarks for. 
`register` should either be called on the name of a target (e.g. `"jvm"`) which will register its `main` compilation
(meaning that `register("jvm")` and `register("jvmMain")` register the same compilation)
Or on the name of a source set (e.g. `"jvmTest"`, `"jsBenchmark"`) which will register the apt compilation
(e.g. `register("jsFoo")` uses the `foo` compilation defined for the `js` target)
Example for multiplatform project:

```groovy
benchmark {
    targets {
        register("jvm") 
        register("js")
        register("native")
        register("wasm") // Experimental
    }
}
```

This package can also be used for Java and Kotlin/JVM projects. Register a Java sourceSet as a target:

```groovy
benchmark {
    targets {
        register("main") 
    }
}
```

To configure benchmarks and create multiple profiles, create a `configurations` section in the `benchmark` block,
and place options inside. Toolkit creates `main` configuration by default, and you can create as many additional
configurations, as you need.   


```groovy
benchmark {
    configurations {
        main { 
            // configure default configuration
        }
        smoke { 
            // create and configure "smoke" configuration, e.g. with several fast benchmarks to quickly check
            // if code changes result in something very wrong, or very right. 
        }       
    }
}
```

Available configuration options:

* `iterations` – number of measuring iterations
* `warmups` – number of warm up iterations
* `iterationTime` – time to run each iteration (measuring and warmup)
* `iterationTimeUnit` – time unit for `iterationTime` (default is seconds)
* `outputTimeUnit` – time unit for results output
* `mode`
  - "thrpt" (default) – measures number of benchmark function invocations per time
  - "avgt" – measures time per benchmark function invocation
* `include("…")` – regular expression to include benchmarks with fully qualified names matching it, as a substring
* `exclude("…")` – regular expression to exclude benchmarks with fully qualified names matching it, as a substring
* `param("name", "value1", "value2")` – specify a parameter for a public mutable property `name` annotated with `@Param`
* `reportFormat` – format of report, can be `json`(default), `csv`, `scsv` or `text`
* There are also some advanced platform-specific settings that can be configured using `advanced("…", …)` function, 
  where the first argument is the name of the configuration parameter, and the second is its value. Valid options:
  * (Kotlin/Native) `nativeFork`
    - "perBenchmark" (default) – executes all iterations of a benchmark in the same process (one binary execution)
    - "perIteration" – executes each iteration of a benchmark in a separate process, measures in cold Kotlin/Native runtime environment
  * (Kotlin/Native) `nativeGCAfterIteration` – when set to `true`, additionally collects garbage after each measuring iteration (default is `false`).
  * (Kotlin/JVM) `jvmForks` – number of times harness should fork (default is `1`)
    - a non-negative integer value – the amount to use for all benchmarks included in this configuration, zero means "no fork"
    - "definedByJmh" – let the underlying JMH determine, which uses the amount specified in [`@Fork` annotation](https://javadoc.io/static/org.openjdk.jmh/jmh-core/1.21/org/openjdk/jmh/annotations/Fork.html) defined for the benchmark function or its enclosing class,
      or [Defaults.MEASUREMENT_FORKS (`5`)](https://javadoc.io/static/org.openjdk.jmh/jmh-core/1.21/org/openjdk/jmh/runner/Defaults.html#MEASUREMENT_FORKS) if it is not specified by `@Fork`.
  * (Kotlin/Js and Wasm) `jsUseBridge` – when `false` disables to generate special benchmark bridges to prevent inlining optimisations (only for `BuiltIn` benchmark executors).

Time units can be NANOSECONDS, MICROSECONDS, MILLISECONDS, SECONDS, MINUTES, or their short variants such as "ms" or "ns".  
  
Example: 

```groovy
benchmark {
    // Create configurations
    configurations {
        main { // main configuration is created automatically, but you can change its defaults
            warmups = 20 // number of warmup iterations
            iterations = 10 // number of iterations
            iterationTime = 3 // time in seconds per iteration
        }
        smoke {
            warmups = 5 // number of warmup iterations
            iterations = 3 // number of iterations
            iterationTime = 500 // time in seconds per iteration
            iterationTimeUnit = "ms" // time unit for iterationTime, default is seconds
        }   
    }
    
    // Setup targets
    targets {
        // This one matches compilation base name, e.g. 'jvm', 'jvmTest', etc
        register("jvm") {
            jmhVersion = "1.21" // available only for JVM compilations & Java source sets
        }
        register("js") {
            // Note, that benchmarks.js uses a different approach of minTime & maxTime and run benchmarks
            // until results are stable. We estimate minTime as iterationTime and maxTime as iterationTime*iterations
            //
            // You can configure benchmark executor - benchmarkJs or buildIn (works only for JsIr backend) with the next line:
            // jsBenchmarksExecutor = JsBenchmarksExecutor.BuiltIn
        }
        register("native")
        register("wasm") // Experimental
    }
}
```  
  
# Separate source sets for benchmarks

Often you want to have benchmarks in the same project, but separated from main code, much like tests. Here is how:
For a Kotlin/JVM project:

Define source set:
```groovy
sourceSets {
    benchmarks
}
```

Propagate dependencies and output from `main` sourceSet. 

```groovy
dependencies {
    benchmarksCompile sourceSets.main.output + sourceSets.main.runtimeClasspath 
}
```

You can also add output and compileClasspath from `sourceSets.test` in the same way if you want 
to reuse some of the test infrastructure.


Register `benchmarks` source set:

```groovy
benchmark {
    targets {
        register("benchmarks")    
    }
}
```

For a Kotlin Multiplatform project:

Define a new compilation in whichever target you'd like (e.g. `jvm`, `js`, etc):
```groovy
kotlin {
    jvm {
        compilations.create('benchmark') { associateWith(compilations.main) }
    }
}
```

Register it by its source set name (`jvmBenchmark` is the name for the `benchmark` compilation for `jvm` target):

```groovy
benchmark {
    targets {
        register("jvmBenchmark")    
    }
}
```

# Examples

The project contains [examples](https://github.com/Kotlin/kotlinx-benchmark/tree/master/examples) subproject that demonstrates using the library.
 
