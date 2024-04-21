# Polyfrost Build Logic Shared Code

This repository contains shared build logic for Polyfrost projects. It is intended to be used as a git submodule in other projects. It is only intended for use within the Polyfrost organization, so support for other projects is not guaranteed.
If you are having problems with this repository, please contact the Polyfrost team via [Discord](https://polyfrost.cc/discord).

Some files may need to be symlinked or copied from this repository if they require being at the root of the file stucture to work.

## Spotless + Checkstyle

Do not use spotless if you are already using Kotlinter. Add the following to your `build.gradle.kts` file:

```kotlin
plugins {
    alias(libs.plugins.spotless)
    alias(libs.plugins.checkstyle)
}

allprojects {
    apply(plugin = rootProject.libs.plugins.spotless.get().pluginId)
    apply(plugin = rootProject.libs.plugins.checkstyle.get().pluginId)

    repositories {
        mavenCentral()
        maven("https://repo.polyfrost.org/releases")
    }

    checkstyle {
        configFile = rootProject.file("format/checkstyle.xml") // replace this with your buildformat submodule path
        toolVersion = rootProject.libs.plugins.checkstyle.get().pluginVersion
    }

    spotless {
        java {
            licenseHeaderFile(rootProject.file("HEADER")) // replace this with your header
            removeUnusedImports()
            importOrder('java', 'javax', '', 'net.minecraft', 'org.polyfrost')
            indentWithTabs()
            trimTrailingWhitespace()
        }
    }
}

spotless {
    groovyGradle {
        target 'src/**/*.gradle', '*.gradle', 'gradle/*.gradle'
        greclipse()
    }
}
```

Feel free to add git hooks or manage the Spotless and Checkstyle hooks if wanted.

## Kotlinter + Licenser

Do not use Kotlinter if you are already using Spotless. Add the following to your `build.gradle.kts` file:

```kotlin
plugins {
    alias(libs.plugins.kotlinter)
    alias(libs.plugins.licenser)
}

allprojects {
    apply(plugin = rootProject.libs.plugins.kotlinter.get().pluginId)
    apply(plugin = rootProject.libs.plugins.licenser.get().pluginId)

    repositories {
        mavenCentral()
        maven("https://repo.polyfrost.org/releases")
    }

    kotlinter {
        ignoreFailures = false
        reporters = arrayOf("checkstyle", "plain")
    }

    license {
        rule("${rootProject.projectDir}/HEADER") // replace this with your hearder
        include("**/*.kt")
    }
}
```

## Dokka

Add the following to your `build.gradle.kts` file to enable Dokka documentation generation:

```kotlin
import org.jetbrains.dokka.base.DokkaBase
import org.jetbrains.dokka.base.DokkaBaseConfiguration
import org.jetbrains.dokka.gradle.AbstractDokkaTask

plugins {
    alias(libs.plugins.dokka)
}

buildscript {
    dependencies {
        classpath(libs.dokka.base)
    }
}

allprojects {
    apply(plugin = rootProject.libs.plugins.dokka.get().pluginId)

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
