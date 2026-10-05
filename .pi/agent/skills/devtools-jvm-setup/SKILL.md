---
name: devtools-jvm-setup
description: Install and manage JVM devtools (JDK, Maven, Gradle, etc.) using SDKMAN!. Use when compiling, running, or testing JVM code that requires missing or mismatched dev tools.
---

# JVM Dev Tool Setup with SDKMAN!

Use this skill whenever JVM code must be compiled, run, or tested and the
required toolchain (java, maven, gradle, kotlin, ...) is missing or at the
wrong version. SDKMAN! is the standard way to provision these tools.

## 1. Check current state first

```shell
java -version 2>&1; mvn -version 2>&1 | head -3   # or the tool you need
```

If the tool is present at an acceptable version, do nothing. Do not
reinstall or "upgrade" working toolchains without being asked.

## 2. Install SDKMAN! (if missing)

Guard: only install if SDKMAN itself is not already present! You can easily validate SDKMAN installation by running:

```shell
command -v sdk >/dev/null 2>&1 && echo "SDKMAN is already installed and available in this shell" || { [ -s "$HOME/.sdkman/bin/sdkman-init.sh" ] && echo "SDKMAN installed but needs to be activated in this shell" || echo "SDKMAN is not installed"; }
```

If the guard above reports SDKMAN is installed, skip to step 3. Otherwise:

Install CI optimized version cause CI mode automatically:

- Answers all prompts (sets sdkman_auto_answer=true)
- Disables colored output for cleaner logs (sets sdkman_colour_enable=false)
- Turns off the self-update feature to prevent unexpected updates (sets sdkman_selfupdate_feature=false)

```shell
curl -s "https://get.sdkman.io?ci=true&rcupdate=false" | bash
```

## 3. Install the required tools

DO NOT SOURCE `"$HOME/.sdkman/bin/sdkman-init.sh"` if you received `SDKMAN is already installed and available in this shell` in step 2!!!
IF YOU RECEIVED `SDKMAN installed but needs to be activated in this shell` then do `source "$HOME/.sdkman/bin/sdkman-init.sh"` before running any `sdk` commands.

```shell
sdk install java 21.0.4-tem        # specific version (preferred, reproducible)
sdk install java                   # latest stable
sdk install maven                  # build tools as needed
```

Version identifiers carry a vendor suffix (e.g. `-tem` = Eclipse Temurin,
`-zulu`, `-graal`, `-amzn`). List available versions with
`sdk list java` (output is paged; append `| cat` or use `q` to exit).

Make the choice permanent (default for future shells):

```shell
sdk default java 21.0.4-tem
```
IMPORTANT!
Never manually delete `~/.sdkman/tmp` or other state directories; use
`sdk flush` if state is corrupted.

## 4. Project-pinned environments (.sdkmanrc)

If the project root contains an `.sdkmanrc` (key=value pairs, e.g.
`java=21.0.4-tem`), activate it:

```shell
sdk env              # switch to pinned versions
sdk env install      # install any missing pinned SDKs, then switch
```

Do not edit `.sdkmanrc` versions unless the user asks or the build fails
with a version-specific error.

## 5. Verification

After setup, verify before running the build:

```shell
java -version && mvn -version   # match what the project expects
```

If the project uses the Maven wrapper (`./mvnw`), it still requires a JDK;
the wrapper supplies Maven itself, so usually only `java` needs installing.

