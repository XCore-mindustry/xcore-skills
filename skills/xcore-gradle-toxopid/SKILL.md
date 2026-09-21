---
name: xcore-gradle-toxopid
description: Gradle and Toxopid build guidelines for Java 25 Mindustry plugins and mods. Use when editing build.gradle.kts, adding dependencies, configuring shadowJar, or troubleshooting builds.
---

# Gradle & Toxopid Guidelines for Mindustry Plugins and Mods

## 1. The Golden Rule of Dependencies

**All dependencies on Mindustry, Arc, or its submodules MUST be declared as `compileOnly`.**

```kotlin
// CORRECT:
compileOnly(toxopid.dependencies.mindustryCore)
compileOnly(toxopid.dependencies.arcCore)
compileOnly(toxopid.dependencies.mindustryHeadless)

// FATAL ERROR - NEVER DO THIS:
implementation(toxopid.dependencies.mindustryCore) // Corrupts runtime ClassLoader!
```

---

## 2. Server Plugins vs. Client Mods

### A. Dedicated Server Plugins (Java 25)

Server plugins run on the dedicated server host under modern Java 25:

```kotlin
java {
    toolchain {
        languageVersion.set(JavaLanguageVersion.of(25))
    }
}
```

### B. Client Mods (Android DEX & Desktop Compatibility)

Client mods (`AdminTools`) must support both Desktop and Android (D8 dexer). Because D8 cannot dex Java 25 bytecode, client mods MUST target Java 17 compatibility:

```kotlin
toxopid {
    compileVersion.set("v${metadata.minGameVersion}")
    platforms.set(setOf(ModPlatform.ANDROID, ModPlatform.DESKTOP))
}

java {
    sourceCompatibility = JavaVersion.VERSION_17
    targetCompatibility = JavaVersion.VERSION_17
}

tasks.withType<JavaCompile>().configureEach {
    options.encoding = "UTF-8"
    options.compilerArgs.addAll(listOf("--release", "17"))
}
```

---

## 3. ShadowJar & Avaje Inject

Because Avaje Inject generates compile-time dependency injection metadata in `META-INF/services`, you MUST enable `mergeServiceFiles()` in `ShadowJar`:

```kotlin
tasks.named<ShadowJar>("shadowJar") {
    archiveBaseName.set(project.name)
    mergeServiceFiles() // Required for Avaje Inject service provider registration
    duplicatesStrategy = DuplicatesStrategy.EXCLUDE
}
```

---

## 4. Client Mod Packaging & Android Dexing (`mergeJar`)

Toxopid produces a dexed jar during build. Configure `mergeJar` to copy the Android-ready JAR into `build/libs/`:

```kotlin
tasks {
    mergeJar {
        doLast {
            val dexedJar = outputs.files.singleFile
            val finalJar = project.file("build/libs/${project.rootDir.name}.jar")

            if (dexedJar.exists()) {
                dexedJar.copyTo(finalJar, overwrite = true)
                println("✅ Android-ready JAR created: ${finalJar.name}")
            } else {
                println("⚠️ Dexed jar not found!")
            }
        }
    }

    build {
        dependsOn(mergeJar)
    }
}
```

---

## 5. Release Stripping & Property Forwarding

When running or building client mods:

1. **Property Forwarding for Testing:**
   Forward `-P` and `-D` test properties to `runMindustryDesktop`:

   ```kotlin
   named<JavaExec>("runMindustryDesktop") {
       listOf("screen", "mobile", "screenshot", "width", "height", "delay", "exit", "dev").forEach { key ->
           val prop = "admintools.$key"
           if (System.getProperty(prop) != null) systemProperty(prop, System.getProperty(prop))
           else if (project.hasProperty(key)) systemProperty(prop, project.property(key).toString())
       }
   }
   ```

2. **Release Stripping:**
   Exclude test and developer packages when `-Prelease` is passed:

   ```kotlin
   val isRelease = project.hasProperty("release")
   tasks.withType<Jar> {
       if (isRelease) {
           exclude("**/dev/**")
       }
   }
   ```

---

## 6. Build Commands Reference

```bash
# Server Plugin Build
./gradlew shadowJar

# Client Mod Dev Build
./gradlew build

# Client Mod Production Release (Stripped)
./gradlew build -Prelease
```
