# Modular Java Builds with Gradle: From Setup to Strategy

<img src="assets/Modular_Building.png" title="" alt="Modular Building.png" data-align="center">

In this hands-on guide we'll see how we can configure composite builds in a simple way using the Gradle command line utils as much as possible — plus a bit of how I used them to break up a monolith on the way to microservices.

## Why Use Composite Builds?

Composite builds are a powerful Gradle feature that address the challenges of modular development, particularly in projects that need to evolve independently but still integrate reliably. They offer several practical advantages:

- **Live dependency substitution**: You can replace the typical change–publish–install workflow with a direct project dependency. This means that any changes you make to a library are immediately visible to the consuming application—no need to publish intermediate versions.

- **Modular development within monorepos**: Composite builds let you split a monorepo into multiple independent builds. You can work with each module in isolation for faster feedback, and use the full composite to verify everything still works together.

- **Temporary or ad hoc integration**: You can integrate separately developed projects without merging their structures. This is especially useful for testing changes across repositories or coordinating temporary collaborations before publishing a new version.

## Scaffolding the Projects

Let's create the root directory that will contain the projects:

```bash
$ mkdir composite
```

In this example, we'll create and application and a library projects.

```bash
$ cd composite
$ mkdir application
$ mkdir library
```

A `tree` command run should return:

```textile
.
├── application
└── library

2 directories, 0 files
```

Use the `gradle init` command to create the structure for the projects.

```bash
$ cd application
$ gradle init \
  --type java-application \
  --dsl kotlin \
  --test-framework junit-jupiter \
  --package com.example.app \
  --project-name application \
  --no-split-project \
  --java-version 21 \
  --incubating
```

Next the library project:

```bash
$ cd ../library
$ gradle init \
  --type java-library \
  --dsl kotlin \
  --test-framework junit-jupiter \
  --package com.example.lib \
  --project-name library \
  --no-split-project \
  --java-version 21 \
  --incubating
```

At this point, a `tree -d -L 4` command run from the root directory should return:

```
.
├── application
│   ├── app
│   │   └── src
│   │       ├── main
│   │       └── test
│   └── gradle
│       └── wrapper
└── library
    ├── gradle
    │   └── wrapper
    └── lib
        └── src
            ├── main
            └── test
```

Now, we've the basic structure of two unconnected projects. Let's make a composite build with them.

## Composite Build Configuration

From the root directory, we'll create an empty Gradle settings file so we can generate the Gradle wrapper.

```bash
$ touch settings.gradle.kts
$ gradle wrapper
```

After that, in `settings.gradle.kts` at the root directory, we must inform to Gradle which builds are included in the composite:

```kotlin
// ../composite/settings.gradle.kts
rootProject.name = "composite"

includeBuild("application")
includeBuild("library")
```

To test our configuration, run the following from the root project directory:

```bash
$ ./gradlew projects
```

That should return the following:

```
> Task :projects

Projects:

------------------------------------------------------------
Root project 'composite'
------------------------------------------------------------

Root project 'composite'
No sub-projects

Included builds:

+--- Included build ':application'
\--- Included build ':library'

To see a list of the tasks of a project, run gradlew <project-path>:tasks
For example, try running gradlew :tasks

BUILD SUCCESSFUL in 789ms
1 actionable task: 1 executed
```

Now the projects are included in the same composite, but still unrelated without any real dependency between them.

## Connecting the Builds

Let's use the library in the application project. In order to do that we need to edit the `build.gradle.kts` files in the `lib` and in the `app` modules. 

```kotlin
// ../composite/library/lib/build.gradle.kts
group = "com.example"
version = "0.0.1
```

Now, in the application module, we can add the library as a dependency:

```kotlin
// ../composite/application/app/build.gradle.kts

dependencies {
    //...
    implementation("com.example:lib:0.0.1")
}
```

One interesting thing to notice here is that we declared the library as if it were a binary dependency, using its group, name, and version. From the application’s perspective, it looks just like any third-party dependency — but with one huge advantage: thanks to composite builds, Gradle automatically substitutes it with the local project source. Any change made in the library is immediately seen by the application, without needing to publish or install anything. That’s the magic of composite builds.

Let's make some use of the library from the app code:

```java
package com.example.app;

import com.example.lib.Library; // Import the library

public class App {
    public String getGreeting() {
        return Library.getGreeting(); // Get the greeating from library
    }

    public static void main(String[] args) {
        System.out.println(new App().getGreeting());
    }
}
```

In `Library`, just add the necessary method: 

```java
package com.example.lib;

public class Library {
    public static String getGreeting() {
        return "Hello from Libray!";
    }
    //...
}
```

Let's run `app`:

```bash
$ ./gradlew :application:app:run                                                                                            
```

Here's the output:

```
> Task :application:app:run
Hello from Libray!

BUILD SUCCESSFUL in 1s
4 actionable tasks: 1 executed, 3 up-to-date
```

If we then change the library greeting message and run the app again, the app will notice the change immediately.

```java
public class Library {
    public static String getGreeting() {
        return "Hello there!";
    }
    //...
}
```

Voila!

```
> Task :application:app:run
Hello there!

BUILD SUCCESSFUL in 10s
4 actionable tasks: 3 executed, 1 up-to-date
```

## Other Considerations

#### Internal Wrappers

We used the `gradle init` command to generate the project structure, which also created a Gradle wrapper (`./gradlew`). You should keep an internal Gradle wrapper in a project **only if that project is meant to be developed or used independently** — for example, if it’s published separately, run in its own CI pipeline, or needs to pin a different Gradle version. In typical composite builds or multi-project setups where everything is built together from the root, you don’t need internal wrappers — they just add clutter. Keep a single wrapper at the root unless there's a strong reason not to.

#### Internal Versions Catalogs

The same consideration applies to version catalogs. You should define a `libs.versions.toml` file in an included build or individual project **only if that project is intended to be consumed in isolation**. If you're working in a composite build where modules are developed together, it's best to define a single shared catalog in the root build. This avoids duplication, reduces maintenance overhead, and keeps dependency versions consistent across builds. As with wrappers, use separate version catalogs only when you genuinely need independence.

By default, the `libs.versions.toml` file is placed in the `gradle/` directory alongside the wrapper JAR in every generated module. If you want to use a single shared version catalog, you'll need to add an extra piece of configuration to the `settings.gradle.kts` file of each included build: the `dependencyResolutionManagement` block.

```kotlin
rootProject.name = "library"

include("lib")

dependencyResolutionManagement {
    versionCatalogs {
        create("libs") {
            from(files("../gradle/libs.versions.toml"))
        }
    }
}
```

Now the `libs` version catalog refers to the `libs.vesions.toml` file in the root directory and you can safety remove the ones inside the modules. 

#### Root Build Script

One last thing you should consider is adding a build script in the composite's root directory to define some global tasks. Here's an example:

```kotlin
defaultTasks("run")

tasks.register("run") {
    dependsOn(gradle.includedBuild("application").task(":app:run"))
}

tasks.register("testAll") {
    dependsOn(gradle.includedBuild("application").task(":app:test"))
    dependsOn(gradle.includedBuild("library").task(":lib:test"))
}
```

This setup allows you to run `./gradlew testAll` from the root directory to execute all tests across the composite build.

If you decide to use a single wrapper, a single version catalog and a root build script you'll end up with the following directory structure.

```
.
├── application
│   ├── app
│   │   ├── build.gradle.kts
│   │   └── src
│   │       ├── main
│   │       │   ├── java
│   │       │   │   └── org
│   │       │   │       └── example
│   │       │   │           └── App.java
│   │       │   └── resources
│   │       └── test
│   │           ├── java
│   │           │   └── org
│   │           │       └── example
│   │           │           └── AppTest.java
│   │           └── resources
│   └── settings.gradle.kts
├── build.gradle.kts
├── gradle
│   ├── libs.versions.toml
│   └── wrapper
│       ├── gradle-wrapper.jar
│       └── gradle-wrapper.properties
├── gradlew
├── gradlew.bat
├── library
│   ├── lib
│   │   ├── build.gradle.kts
│   │   └── src
│   │       ├── main
│   │       │   ├── java
│   │       │   │   └── org
│   │       │   │       └── example
│   │       │   │           └── Library.java
│   │       │   └── resources
│   │       └── test
│   │           ├── java
│   │           │   └── org
│   │           │       └── example
│   │           │           └── LibraryTest.java
│   │           └── resources
│   └── settings.gradle.kts
└── settings.gradle.kts
```

## Composites in a Monolith-to-Microservices Scenario

A few years ago, I was working on a system that had started to show serious scalability issues — both in terms of performance and design flexibility. It followed a classic three-layer architecture:

- A REST API layer that handled requests from the UI and external systems

- A middle layer containing the business logic

- An integration layer responsible for communicating with various third-party providers

The integration layer, in particular, had become a bottleneck. It was monolithic in structure and hard to evolve. Performance was another challenge — each provider had very different response characteristics, which made it difficult to scale or even size that layer appropriately.

We knew we needed to evolve toward a more modular architecture, but a full rewrite wasn’t realistic. Instead, we chose a path of **incremental decomposition**. The goal was to isolate provider-specific logic that could be developed and tested independently, without breaking the existing deployment model.

To enable this, we adopted Gradle composite builds. Rather than splitting the integration layer into separate microservices immediately — which would have required significant coordination, deployment, and CI complexity — we began extracting provider connectors into standalone Gradle builds. These builds could be version-controlled separately, tested in isolation, and evolved at their own pace.

Using `includeBuild(...)`, we wired them back into the main system as source dependencies. This gave us the benefits of modularity without forcing early service boundaries or dependency publication. Because Gradle treated them as if they were published artifacts, nothing had to change in the rest of the system.

To keep the business logic layer insulated from these changes, we introduced a small internal SDK — a module that exposed a stable interface to the provider connectors. It encapsulated provider selection logic and served as a boundary between the modularized integration code and the core application. In some cases, the SDK imported the connector directly using includeBuild(...); in others, it routed requests over HTTP to connectors that had been promoted to microservices. This dual-mode setup allowed us to evolve the architecture incrementally behind a unified interface, without impacting the rest of the system. It also helped maintain a consistent developer experience throughout the transition.

This approach gave us several advantages:

- We could work on or refactor connectors without having to touch unrelated parts of the system, because each connector lived in its own isolated build and interacted with the rest of the application only through a stable SDK interface.

- Teams could build and test connectors independently, which sped up local development and reduced unnecessary work in CI, because the connectors were split into separate builds with their own lifecycle and caching behavior.

- Over time, we could graduate some of these connectors into fully independent microservices — often without changing their internal APIs

This setup wasn’t without tradeoffs. It introduced more build logic and required engineers to become comfortable working across multiple Gradle builds. But the gains in modularity, velocity, and long-term flexibility far outweighed the initial complexity.

In practice, composite builds gave us an architectural bridge between monolith and microservices. They enabled controlled, observable evolution — allowing us to restructure the system without stopping the world.

## Wrapping Up

Gradle composite builds offer a clean and flexible way to work with multiple independent projects as if they were part of a single unified system. In this tutorial, we used `gradle init` to scaffold the project structure, configured the composite with `includeBuild`, and wired everything together using a shared wrapper, version catalog, and root build script. This setup allows changes in one project to be immediately reflected in others, without the need to publish or install artifacts — enabling faster feedback loops and smoother cross-project collaboration.

Beyond simple modularization, composite builds can also serve as a practical bridge during architectural transitions. As shown in the monolith-to-microservices example, they make it possible to incrementally extract parts of a system and defer deployment decisions without sacrificing testability or development speed. Whether you're modernizing a legacy system or building modular applications from scratch, composite builds give you the structure to scale with confidence — both technically and organizationally.

*Licensed under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/).*

---

*References*:

[Introducing Composite Builds](https://blog.gradle.org/introducing-composite-builds?utm_source=chatgpt.com)

[Composite Builds](https://docs.gradle.org/current/userguide/composite_builds.html)
