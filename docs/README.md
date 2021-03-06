# clojurephant

{% include nav.md %}

## Quick Start

- Install the [Clojure command line tool](https://clojure.org/guides/getting_started) (i.e. clj).
- Add an alias for [clj-new](https://github.com/seancorfield/clj-new/) to your `~/.clojure/deps.edn`

Create a new Clojure library:

```
clj -X:new :template clojurephant-clj-lib :name myname/mylib
```

Create a new Clojure application:

```
clj -X:new :template clojurephant-clj-app :name myname/myapp
```

Create a new ClojureScript appliation:

```
clj -X:new :template clojurephant-cljs-app :name myname/myapp
```

If the documentation doesn't answer your questions, please visit the [Clojurephant Discussions](https://github.com/clojurephant/clojurephant/discussions) [ClojureVerse gradle-clojure category](https://clojureverse.org/c/projects/gradle-clojure) or the [Clojurian's Slack #gradle channel](http://clojurians.net/).

## Plugins

clojurephant uses the common Gradle pattern of providing _capability_ plugins and _convention_ plugins. Capability plugins provide the basic machinery for using the language, but leaves it to you to configure. Convention plugins provide configuration on top of the capabilities to support common use cases.

| Convention                       | Capability                            |
| -------------------------------- | ------------------------------------- |
| `dev.clojurephant.clojure`       | `dev.clojurephant.clojure-base`       |
| `dev.clojurephant.clojurescript` | `dev.clojurephant.clojurescript-base` |

### dev.clojurephant.clojure-base

- Applies `java-base`, which lets you configure source sets. Each source set will get:
  - A Java compilation task
  - Configurations for compile (`implementation`, `compileOnly`) and runtime (`runtimeOnly`) dependencies
- Adds a `clojure` extension which allows you to configure builds of your Clojure code.
  - A build is added for each source set (with the same name as that source set)
  - Additional builds can be configured by the user
  - Each build gets a `check<Build>Clojure` task that can be used to ensure namespaces compile, and optionally warn or fail on reflection. (by default no namespaces are compiled)
  - Each build gets a `compile<Build>Clojure` task that can be used for AOT compilation. (by default no namespaces are AOTd)
  - If any namespaces are configured to be AOTed for the source sets build, the source sets output will be the AOTd classes. Otherwise, the Clojure source will be the output (i.e. what would get included in a JAR)

Tip: `gradlew tasks --all` shows all these created tasks. Beware: The `main` build is somewhat special and its name is not included in the task names so it has e.g. `compileClojure`.

#### Clojure Builds

You can define a custom build:

```groovy
clojure {
 builds {
   // Defaults noted here are for custom builds, the convention plugin configures the builds it adds differently
   mybuild {
     sourceSet = sourceSets.mystuff // no default
     // Configuration of the check<Build>Clojure task
     reflection = 'fail' // defaults to 'silent', can also be 'warn'
     checkNamespaces = ['my.core', 'my.base'] // defaults to no namespaces checked
     checkNamespaces.add('my-core') // just add a single namespace
     checkAll() // checks any namespaces found in the source set
     // Configuration of the compile<Build>Clojure task
     compiler {
       disableLocalsClearing = true // defaults to false
       elideMeta = ['doc', 'file'] // defaults to empty list
       directLinking = true // defaults to false
     }
     aotNamespaces = ['my.core', 'my.base'] // defaults to no namespaces aoted
     aotNamespaces.add('my-core') // just add a single namespace
     aotAll() // aots any namespaces found in the source set
   }
 }
}
```

You can also _modify_ the configuration of the auto-added builds, f.ex. the "main" one:

```
clojure {
    builds {
      main {
        reflection = 'warn'
      }
    }
}
```

### dev.clojurephant.clojure

- Applies the `dev.clojurephant.clojure-base` plugin (see above)
- Applies the `java` plugin:
  - Creates a main source set, whose output is packaged into a JAR via the `jar` task.
  - Creates a test source set, which extends the main source set.
  - Creates a `test` task that runs tests within the test source set.
- Applies the internal `ClojureCommonPlugin` which:
  - Creates a dev source set, to be used for REPL development, which extends the test source set.
  - Adds 'nrepl:nrepl' as a dependency of that source set.
  - Adds a `clojureRepl` task which will start an nREPL server.
  - Configures dependency rules to indicate that:
    - `org.clojure:tools.nrepl` is replaced by `nrepl:nrepl`
    - If you are using a Java 9+ JVM, any `org.clojure:java.classpath` dependency must be bumped to at least 1.0.0 to support the new classloader hierarchy.
- Configures the `main` Clojure build to `checkAll()` namespaces.
- Configures any build whose name includes `test` to:
  - `aotAll()` namespaces (required for the current JUnit4 integration)
- Configures the `dev` Clojure build to `checkNamespaces = ['user']` (if you have a user namespace). This ensures that your REPL will start successfully.

### dev.clojurephant.clojurescript-base

- Applies `java-base`, which lets you configure source sets. Each source set will get:
  - A Java compilation task
  - Configurations for compile (`implementation`, `compileOnly`) and runtime (`runtimeOnly`) dependencies
- Adds a `clojurescript` extension which allows you to configure builds of your ClojureScript code.
  - A build is added for each source set (with the same name as that source set)
  - Additional builds can be configured by the user
  - Each build gets a `compile<Build>ClojureScript` task that can be used for compilation. (by default no compiler options are set)
  - If `outputTo` is configured (either the top level one or for a module) for the source sets build, the source sets output will be the compiled JS. Otherwise, the ClojureScript source will be the output (i.e. what would get included in a JAR).

#### ClojureScript Builds

See [ClojureScript compiler options](https://clojurescript.org/reference/compiler-options) for details on what each option does and defaults to.

```groovy
clojurescript {
 builds {
   // Defaults noted here are for custom builds, the convention plugin configures the builds it adds differently
   mybuild {
     sourceSet = sourceSets.mystuff // no default
     // Configuration of the compile<Build>ClojureScript task (defaults match what is defaulted in the ClojureScript compile options)
     compiler {
       outputTo = 'public/some/file/path.js' // path is relative to the task's destinationDir
       outputDir = 'public/some/path' // path is relative to the task's destinationDir
       optimizations = 'advanced'
       main = 'foo.bar'
       assetPath = 'public/some/path'
       sourceMap = 'public/some/file/path.js.map' // path is relative to the task's destinationDir
       verbose = true
       prettyPrint = false
       target = 'nodejs'
       // foreignLibs
       externs = ['jquery-externs.js']
       // modules
       // stableNames
       preloads = ['foo.dev']
       npmDeps = ['lodash': '4.17.4']
       installDeps = true
       checkedArrays = 'warn'
     }
   }
 }
}
```

### dev.clojurephant.clojurescript

- Applies the `dev.clojurephant.clojurescript-base` plugin (see above)
- Applies the `java` plugin:
  - Creates a main source set, whose output is packaged into a JAR via the `jar` task.
  - Creates a test source set, which extends the main source set.
  - Creates a `test` task that runs tests within the test source set.
- Applies the internal `ClojureCommonPlugin` which:
  - Creates a dev source set, to be used for REPL development, which extends the test source set.
  - Adds 'nrepl:nrepl' as a dependency of that source set.
  - Adds a `clojureRepl` task which will start an nREPL server.
  - Configures dependency rules to indicate that:
    - `org.clojure:tools.nrepl` is replaced by `nrepl:nrepl`
    - If you are using a Java 9+ JVM, any `org.clojure:java.classpath` dependency must be bumped to at least 1.0.0 to support the new classloader hierarchy.
- Wires your ClojureScript build configuration into the nREPL for use by Figwheel.
- Configures the REPL for Piggieback:
  - Adds a dev dependency `cider:piggieback`
  - Adds the Piggieback middleware: `cider.piggieback/wrap-cljs-repl`

## Project Layout

```
<project>/
  src/
    main/
      clojure/
        sample_clojure/
          core.clj
      clojurescript/
        sample_clojure/
          main.cljs
    test/
      clojure/
        sample_clojure/
          core_test.clj
      clojurescript/
        sample_clojure/
          main_test.cljs // right now we don't support cljs.test
    dev/
      clojure/
        user.clj
      clojurescript/
        user.cljs
  gradle/
    wrapper/
      gradle-wrapper.jar
      gradle-wrapper.properties
  build.gradle
  gradlew
  gradlew.bat
```

## Task Configuration

### ClojureNRepl

```groovy
clojureRepl {
  port = 55555 // defaults to a random open port (which will be written to a .nrepl-port file)

  // handler and middleware are both optional, but don't provide both
  handler = 'cider.nrepl/cider-nrepl-handler' // fully-qualified name of function
  middleware = ['my.stuff/wrap-stuff'] // list of fully-qualified middleware function names (override any existing)
  middleware 'dev/my-middleware', 'dev/my-other-middleware' // one or more full-qualified middleware function names (append to any existing)

  // clojureRepl provides fork options to customize the Java process for compilation
  forkOptions {
    memoryMaximumSize = '2048m'
    jvmArgs = ['-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=5005', '-Djava.awt.headless=true']
  }
}
```

The `ClojureNRepl` task also supports command-line options for some of it's parameters. Multiple `middleware` must be specified as separate options.

```
./gradlew clojureRepl --port=1234 --handler=cider.nrepl/cider-nrepl-handler
./gradlew clojureRepl --port=4321 --middleware=dev/my-middleware --middleware=dev/my-other-middleware
```

### check or compile tasks

Always configure compiler options and reflection settings via the `clojure` or `clojurescript` extensions. These options may be immutable on the tasks at some point in the future.

The only settings you should configure directly on the tasks are the forkOptions, if you need to customize the JVM that is used.

```groovy
checkClojure {
  // to customize the Java process for compilation
  forkOptions {
    memoryMaximumSize = '2048m'
    jvmArgs = ['-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=5005', '-Djava.awt.headless=true']
  }
}
```
