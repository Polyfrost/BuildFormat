# Polyfrost Build Logic Shared Code

This repository contains shared build logic for Polyfrost projects. It is intended to be used as a git submodule in other projects. It is only intended for use within the Polyfrost organization, so support for other projects is not guaranteed.
If you are having problems with this repository, please contact the Polyfrost team via [Discord](https://polyfrost.cc/discord).

## Dokka

Add the following to your `build.gradle.kts` file to enable Dokka documentation generation:

```kotlin
import org.jetbrains.dokka.base.DokkaBase
import org.jetbrains.dokka.base.DokkaBaseConfiguration
import org.jetbrains.dokka.gradle.AbstractDokkaTask

plugins {
    id("org.jetbrains.dokka") version "1.7.20"
}

buildscript {
    dependencies {
        classpath("org.jetbrains.dokka:dokka-base:1.7.20")
    }
}

allprojects {
    apply(plugin = "org.jetbrains.dokka")

    repositories {
        mavenCentral()
        maven("https://repo.polyfrost.cc/releases")
    }

    tasks {
        withType<AbstractDokkaTask> {
            pluginConfiguration<DokkaBase, DokkaBaseConfiguration> {
                val rootPath = "${rootProject.projectDir.absolutePath}/{BUILDFORMAT_DIR}/dokka"
                customStyleSheets = file("$rootPath/styles").listFiles()!!.toList()
                customAssets = file("$rootPath/assets").listFiles()!!.toList()
                templatesDir = file("$rootPath/templates")

                footerMessage = "© ${Year.now().value} Polyfrost"
            }

            doLast {
                // Script overriding does not work, so we have to do it manually
                val scriptsOut = outputDirectory.get().resolve("scripts")
                val scriptsIn = file("${rootProject.projectDir}/{BUILDFORMAT_DIR}/dokka/scripts")
                if (project != rootProject) return@doLast
                scriptsIn.listFiles()!!.forEach {
                    it.copyTo(scriptsOut.resolve(it.name), overwrite = true)
                }
            }
        }

        val dokkaJavadocJar by creating(Jar::class.java) {
            group = "documentation"
            archiveClassifier.set("javadoc")
            from(dokkaJavadoc)
        }
    }

    // If you want to integrate with the `maven-publish` plugin, add this to your publishing config, but DO NOT replace
    // your existing config -- this will make it so your actual jar isn't published to maven.
    publishing {
        publications {
            create<MavenPublication>("Maven") {
                artifactId = project.name
                version = projectVersion

                artifact(tasks.getByName<Jar>("dokkaJavadocJar").archiveFile) {
                    builtBy(tasks.getByName("dokkaJavadocJar"))
                    this.classifier = "javadoc"
                }
            }
        }

        repositories {
            mavenLocal()
            // replace this with your own maven repo/maven publishing config :D
            if (System.getenv("MAVEN_URL") != null) {
                maven {
                    setUrl(System.getenv("MAVEN_URL"))
                    credentials {
                        username = System.getenv("MAVEN_USERNAME")
                        password = System.getenv("MAVEN_PASSWORD")
                    }
                    name = "Maven"
                }
            } else if (System.getenv("SNAPSHOTS_URL") != null) {
                maven {
                    setUrl(System.getenv("SNAPSHOTS_URL"))
                    credentials {
                        username = System.getenv("SNAPSHOTS_USERNAME")
                        password = System.getenv("SNAPSHOTS_PASSWORD")
                    }
                    name = "Maven"
                }
            }
        }
    }
}

subprojects {
    tasks {
        named<Jar>("dokkaJavadocJar") {
            archiveBaseName.set("${rootProject.name}-${project.name}")
        }
    }
}

tasks {
    create("dokkaHtmlJar", Jar::class.java) {
        group = "documentation"
        archiveBaseName.set(rootProject.name)
        archiveClassifier.set("dokka")
        from(dokkaHtmlMultiModule.get().outputDirectory)
        duplicatesStrategy = DuplicatesStrategy.FAIL
    }
}

// If you want to integrate with the `maven-publish` plugin
publishing {
    publications {
        getByName<MavenPublication>("Maven") {
            val dokka = rootProject.tasks.getByName<Jar>("dokkaHtmlJar")
            artifact(dokka.archiveFile) {
                builtBy(dokka)
                classifier = "dokka"
            }
        }
    }
}
```
