# code-server WebView Android App Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a minimal Android app that shows `http://192.168.0.2:3000` (a code-server / VS Code Web instance) full-screen in a `WebView`, with no browser chrome or navigation.

**Architecture:** Single-`Activity`, single-`WebView` Kotlin app. No other screens, no ViewModel/data layer — this is a thin configuration shell around the platform `WebView`.

**Tech Stack:** Kotlin, Gradle (Kotlin DSL) with the version catalog-free `plugins {}` block, AGP 9.3.1 (Kotlin support is built into `com.android.application` as of AGP 9 — no separate Kotlin Gradle plugin), Gradle 9.5.0, `androidx.core:core-ktx:1.13.1`, `androidx.activity:activity-ktx:1.9.3`. `compileSdk`/`targetSdk` 36, `minSdk` 26 (needed for the adaptive-icon-only launcher icon, no legacy PNG fallback required).

**Spec:** [docs/superpowers/specs/2026-09-17-webview-code-server-app-design.md](../specs/2026-09-17-webview-code-server-app-design.md)

## Global Constraints

- Target URL is hardcoded: `http://192.168.0.2:3000` — no settings screen, no configurable URL.
- `android:usesCleartextTraffic="true"` at the application level (local network only, no TLS on the target host).
- No automated test suite (per spec: this is a UI-only shell with no business logic). Verification is `./gradlew assembleDebug` (compiles) plus a manual on-device checklist (Task 5).
- Package/namespace/applicationId: `com.litzrsh.codeserver`. App display name: `CodeServer`.
- Screen locked to landscape, immersive full-screen (status/nav bars hidden).
- Hardware back button: fully suppressed, does nothing.
- Notify-and-exit behavior applies **only** to the initial page load failing; drops after a successful initial load are left to the WebView's default behavior.

---

## Environment Notes (read before Task 1)

This machine already has everything needed to build fully offline (verified with a real `assembleDebug` dry run of this exact plan's files before Task 1 was dispatched — see the ledger's pre-flight scan):
- JDK 17 at `$JAVA_HOME` (`C:\Program Files\Eclipse Adoptium\jdk-17.0.20.101-hotspot`).
- Android SDK at `C:\dev\android-sdk` with platform `android-36` and build-tools `36.0.0` already installed, and SDK licenses already accepted.
- Gradle 9.5.0 (`bin` distribution) already unpacked under `~/.gradle/wrapper/dists/gradle-9.5.0-bin/`, and AGP 9.3.1 / the AndroidX artifacts this plan uses are already present in `~/.gradle/caches/modules-2`.

**Toolchain note:** AGP 9.3.1 requires Gradle ≥9.5.0 (Gradle 9.3.1 is too old for it) and, as of AGP 9, no longer accepts a separate `org.jetbrains.kotlin.android` plugin — Kotlin support is built directly into `com.android.application`. Do not add a Kotlin Gradle plugin anywhere in this project; the Kotlin blocks below reflect this already.

Because of this, Task 1 generates the Gradle wrapper using the already-unpacked local Gradle 9.5.0 binary (no download), and every `./gradlew` command in later tasks should resolve entirely from local caches (one transitive dependency, `androidx.collection:collection:1.0.0`, was not pre-cached and was fetched once during the pre-flight dry run — it is cached now, so later `./gradlew` runs in this plan do not need network access either).

---

### Task 1: Gradle project scaffolding and wrapper

**Files:**
- Create or modify: `.gitignore` (a root `.gitignore` containing only `.worktrees/` already exists in this worktree from earlier repo setup — append to it, do not overwrite it)
- Create: `settings.gradle.kts`
- Create: `build.gradle.kts`
- Create: `gradle.properties`
- Create: `local.properties`
- Create (generated): `gradlew`, `gradlew.bat`, `gradle/wrapper/gradle-wrapper.jar`, `gradle/wrapper/gradle-wrapper.properties`

**Interfaces:**
- Produces: a working Gradle wrapper (`./gradlew`) and `settings.gradle.kts`/`build.gradle.kts` for Task 2 to extend (Task 2 adds `include(":app")` to `settings.gradle.kts` once the `app/` directory exists — do NOT add that line in this task, see Step 2).

- [ ] **Step 1: Append to the existing `.gitignore`**

This worktree already has a root `.gitignore` containing exactly `.worktrees/` (unrelated repo-level setup, done before this plan). Append these lines to it — do not replace the file:

```gitignore
*.iml
.gradle/
/local.properties
/.idea/
.DS_Store
/build
/captures
.externalNativeBuild/
.cxx/
/app/build/
```

- [ ] **Step 2: Create `settings.gradle.kts`**

Deliberately no `include(":app")` yet — the `app/` directory doesn't exist until Task 2, and Gradle refuses to configure an `include()`d project whose directory is missing (verified: `BUILD FAILED` with "Configuring project ':app' without an existing directory is not allowed"). Task 2 appends the `include(":app")` line once the directory exists.

```kotlin
pluginManagement {
    repositories {
        google()
        mavenCentral()
        gradlePluginPortal()
    }
}

dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        google()
        mavenCentral()
    }
}

rootProject.name = "CodeServer"
```

- [ ] **Step 3: Create root `build.gradle.kts`**

Only the Android Gradle Plugin — no separate Kotlin plugin (AGP 9's built-in Kotlin support handles `.kt` sources):

```kotlin
plugins {
    id("com.android.application") version "9.3.1" apply false
}
```

- [ ] **Step 4: Create `gradle.properties`**

```properties
org.gradle.jvmargs=-Xmx2048m -Dfile.encoding=UTF-8
android.useAndroidX=true
kotlin.code.style=official
```

- [ ] **Step 5: Create `local.properties`**

This file is machine-specific and is excluded by `.gitignore` from Step 1 — it must still be created locally for Gradle to find the Android SDK.

```properties
sdk.dir=C:/dev/android-sdk
```

- [ ] **Step 6: Generate the Gradle wrapper from the locally cached Gradle 9.5.0 distribution**

Locate the cached Gradle 9.5.0 binary and use it to generate the wrapper (this avoids any network download — the same version is embedded in the wrapper it generates). Do not cap `find`'s depth — the distribution's `bin/gradle.bat` is 4 levels below the dists directory shown here, and a shallower `-maxdepth` misses it:

```bash
GRADLE_BIN=$(find "$HOME/.gradle/wrapper/dists/gradle-9.5.0-bin" -type f -name gradle.bat | head -1)
"$GRADLE_BIN" wrapper --gradle-version 9.5.0 --distribution-type bin
```

Expected: creates `gradlew`, `gradlew.bat`, `gradle/wrapper/gradle-wrapper.jar`, `gradle/wrapper/gradle-wrapper.properties` in the project root.

- [ ] **Step 7: Verify the wrapper works**

Run: `./gradlew tasks`
Expected: `BUILD SUCCESSFUL` (only root-project tasks are listed since `:app` doesn't exist yet — that's fine, this step only proves the wrapper itself runs).

- [ ] **Step 8: Commit**

```bash
git add .gitignore settings.gradle.kts build.gradle.kts gradle.properties gradlew gradlew.bat gradle/wrapper/gradle-wrapper.jar gradle/wrapper/gradle-wrapper.properties
git commit -m "Scaffold Gradle project and wrapper"
```

(`local.properties` is intentionally not committed — it's gitignored.)

---

### Task 2: App module — manifest, theme, launcher icon, blank Activity

**Files:**
- Modify: `settings.gradle.kts` (add the `include(":app")` line — see Step 1)
- Create: `app/build.gradle.kts`
- Create: `app/src/main/AndroidManifest.xml`
- Create: `app/src/main/java/com/litzrsh/codeserver/MainActivity.kt`
- Create: `app/src/main/res/layout/activity_main.xml`
- Create: `app/src/main/res/values/strings.xml`
- Create: `app/src/main/res/values/themes.xml`
- Create: `app/src/main/res/values/colors.xml`
- Create: `app/src/main/res/drawable/ic_launcher_foreground.xml`
- Create: `app/src/main/res/mipmap-anydpi-v26/ic_launcher.xml`
- Create: `app/src/main/res/mipmap-anydpi-v26/ic_launcher_round.xml`

**Interfaces:**
- Consumes: `:app` module slot from `settings.gradle.kts` (Task 1).
- Produces: `MainActivity` (launcher activity, currently just inflates a blank layout) and `R.id.webView` in `activity_main.xml` — Task 3 wires the `WebView` up.

- [ ] **Step 1: Add `include(":app")` to `settings.gradle.kts`**

Append this line to the end of the `settings.gradle.kts` created in Task 1:

```kotlin
include(":app")
```

- [ ] **Step 2: Create `app/build.gradle.kts`**

Only the Android Gradle Plugin in `plugins {}` — no separate Kotlin plugin, and no `kotlinOptions {}` block (both are rejected by AGP 9's built-in Kotlin support; `compileOptions` alone is enough for the Kotlin compiler to pick up Java 17 target):

```kotlin
plugins {
    id("com.android.application")
}

android {
    namespace = "com.litzrsh.codeserver"
    compileSdk = 36

    defaultConfig {
        applicationId = "com.litzrsh.codeserver"
        minSdk = 26
        targetSdk = 36
        versionCode = 1
        versionName = "1.0"
    }

    buildTypes {
        release {
            isMinifyEnabled = false
        }
    }

    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }
}

dependencies {
    implementation("androidx.core:core-ktx:1.13.1")
    implementation("androidx.activity:activity-ktx:1.9.3")
}
```

- [ ] **Step 3: Create `app/src/main/res/values/colors.xml`**

```xml
<resources>
    <color name="ic_launcher_background">#1E1E1E</color>
</resources>
```

- [ ] **Step 4: Create `app/src/main/res/drawable/ic_launcher_foreground.xml`**

```xml
<vector xmlns:android="http://schemas.android.com/apk/res/android"
    android:width="108dp"
    android:height="108dp"
    android:viewportWidth="108"
    android:viewportHeight="108">
    <path
        android:fillColor="#FFFFFF"
        android:pathData="M24,30 L84,30 L84,78 L24,78 Z" />
    <path
        android:fillColor="#1E1E1E"
        android:pathData="M24,30 L84,30 L84,38 L24,38 Z" />
    <path
        android:fillColor="#4FC3F7"
        android:pathData="M32,48 L44,54 L32,60 Z" />
    <path
        android:fillColor="#4FC3F7"
        android:pathData="M50,58 L64,58 L64,62 L50,62 Z" />
</vector>
```

- [ ] **Step 5: Create `app/src/main/res/mipmap-anydpi-v26/ic_launcher.xml`**

```xml
<?xml version="1.0" encoding="utf-8"?>
<adaptive-icon xmlns:android="http://schemas.android.com/apk/res/android">
    <background android:drawable="@color/ic_launcher_background" />
    <foreground android:drawable="@drawable/ic_launcher_foreground" />
</adaptive-icon>
```

- [ ] **Step 6: Create `app/src/main/res/mipmap-anydpi-v26/ic_launcher_round.xml`**

Identical content to Step 5 (the system applies the round mask automatically):

```xml
<?xml version="1.0" encoding="utf-8"?>
<adaptive-icon xmlns:android="http://schemas.android.com/apk/res/android">
    <background android:drawable="@color/ic_launcher_background" />
    <foreground android:drawable="@drawable/ic_launcher_foreground" />
</adaptive-icon>
```

- [ ] **Step 7: Create `app/src/main/res/values/strings.xml`**

```xml
<resources>
    <string name="app_name">CodeServer</string>
    <string name="load_failure_message">code-server에 연결할 수 없습니다. 네트워크와 서버 상태를 확인하세요.</string>
</resources>
```

- [ ] **Step 8: Create `app/src/main/res/values/themes.xml`**

```xml
<resources>
    <style name="Theme.CodeServer" parent="android:Theme.Material.NoActionBar">
        <item name="android:windowFullscreen">true</item>
        <item name="android:windowNoTitle">true</item>
        <item name="android:statusBarColor">@android:color/black</item>
        <item name="android:navigationBarColor">@android:color/black</item>
    </style>
</resources>
```

- [ ] **Step 9: Create `app/src/main/res/layout/activity_main.xml`**

```xml
<?xml version="1.0" encoding="utf-8"?>
<WebView xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/webView"
    android:layout_width="match_parent"
    android:layout_height="match_parent" />
```

- [ ] **Step 10: Create `app/src/main/java/com/litzrsh/codeserver/MainActivity.kt`**

```kotlin
package com.litzrsh.codeserver

import android.os.Bundle
import androidx.activity.ComponentActivity

class MainActivity : ComponentActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
    }
}
```

- [ ] **Step 11: Create `app/src/main/AndroidManifest.xml`**

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android">

    <uses-permission android:name="android.permission.INTERNET" />

    <application
        android:allowBackup="false"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:theme="@style/Theme.CodeServer"
        android:usesCleartextTraffic="true">
        <activity
            android:name=".MainActivity"
            android:configChanges="orientation|screenSize|keyboardHidden"
            android:exported="true"
            android:screenOrientation="landscape">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>

</manifest>
```

- [ ] **Step 12: Build the debug APK**

Run: `./gradlew assembleDebug`
Expected: `BUILD SUCCESSFUL`, and `app/build/outputs/apk/debug/app-debug.apk` exists. (This exact file set was already built successfully once during the plan's pre-flight check, so this should be a cache-warm, no-surprises build.)

- [ ] **Step 13: Commit**

```bash
git add settings.gradle.kts app/build.gradle.kts app/src/main/AndroidManifest.xml app/src/main/java app/src/main/res
git commit -m "Add app module skeleton with blank landscape Activity"
```

---

### Task 3: WebView loads code-server, immersive full-screen

**Files:**
- Modify: `app/src/main/java/com/litzrsh/codeserver/MainActivity.kt` (full rewrite)

**Interfaces:**
- Consumes: `R.layout.activity_main` / `R.id.webView` (Task 2), `R.string.load_failure_message` (Task 2, used in Task 4).
- Produces: a private top-level `TARGET_URL` constant and an `enableImmersiveMode()` private method in `MainActivity.kt` that Task 4 carries forward unchanged.

- [ ] **Step 1: Replace `MainActivity.kt` with the WebView + immersive-mode version**

```kotlin
package com.litzrsh.codeserver

import android.annotation.SuppressLint
import android.os.Bundle
import android.webkit.WebView
import androidx.activity.ComponentActivity
import androidx.core.view.WindowCompat
import androidx.core.view.WindowInsetsCompat
import androidx.core.view.WindowInsetsControllerCompat

private const val TARGET_URL = "http://192.168.0.2:3000"

class MainActivity : ComponentActivity() {

    @SuppressLint("SetJavaScriptEnabled")
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        enableImmersiveMode()

        val webView = findViewById<WebView>(R.id.webView)
        webView.settings.javaScriptEnabled = true
        webView.settings.domStorageEnabled = true
        webView.settings.useWideViewPort = true
        webView.settings.loadWithOverviewMode = true
        webView.loadUrl(TARGET_URL)
    }

    override fun onWindowFocusChanged(hasFocus: Boolean) {
        super.onWindowFocusChanged(hasFocus)
        if (hasFocus) {
            enableImmersiveMode()
        }
    }

    private fun enableImmersiveMode() {
        WindowCompat.setDecorFitsSystemWindows(window, false)
        val controller = WindowInsetsControllerCompat(window, window.decorView)
        controller.hide(WindowInsetsCompat.Type.systemBars())
        controller.systemBarsBehavior =
            WindowInsetsControllerCompat.BEHAVIOR_SHOW_TRANSIENT_BARS_BY_SWIPE
    }
}
```

- [ ] **Step 2: Build the debug APK**

Run: `./gradlew assembleDebug`
Expected: `BUILD SUCCESSFUL`.

- [ ] **Step 3: Commit**

```bash
git add app/src/main/java/com/litzrsh/codeserver/MainActivity.kt
git commit -m "Load code-server URL in a full-screen immersive WebView"
```

---

### Task 4: Initial-load failure handling and back-button suppression

**Files:**
- Modify: `app/src/main/java/com/litzrsh/codeserver/MainActivity.kt` (full rewrite)

**Interfaces:**
- Consumes: `TARGET_URL`, `enableImmersiveMode()` (Task 3), `R.string.load_failure_message` (Task 2).
- Produces: final `MainActivity` for this app — no further tasks modify it.

- [ ] **Step 1: Replace `MainActivity.kt` with the failure-handling + back-suppression version**

```kotlin
package com.litzrsh.codeserver

import android.annotation.SuppressLint
import android.os.Bundle
import android.webkit.WebResourceError
import android.webkit.WebResourceRequest
import android.webkit.WebView
import android.webkit.WebViewClient
import android.widget.Toast
import androidx.activity.ComponentActivity
import androidx.activity.addCallback
import androidx.core.view.WindowCompat
import androidx.core.view.WindowInsetsCompat
import androidx.core.view.WindowInsetsControllerCompat

private const val TARGET_URL = "http://192.168.0.2:3000"

class MainActivity : ComponentActivity() {

    private var initialLoadFinished = false

    @SuppressLint("SetJavaScriptEnabled")
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        enableImmersiveMode()
        suppressBackButton()

        val webView = findViewById<WebView>(R.id.webView)
        webView.settings.javaScriptEnabled = true
        webView.settings.domStorageEnabled = true
        webView.settings.useWideViewPort = true
        webView.settings.loadWithOverviewMode = true
        webView.webViewClient = object : WebViewClient() {
            override fun onReceivedError(
                view: WebView,
                request: WebResourceRequest,
                error: WebResourceError
            ) {
                super.onReceivedError(view, request, error)
                if (!initialLoadFinished && request.isForMainFrame) {
                    showLoadFailureAndExit()
                }
            }

            override fun onPageFinished(view: WebView, url: String?) {
                super.onPageFinished(view, url)
                initialLoadFinished = true
            }
        }
        webView.loadUrl(TARGET_URL)
    }

    override fun onWindowFocusChanged(hasFocus: Boolean) {
        super.onWindowFocusChanged(hasFocus)
        if (hasFocus) {
            enableImmersiveMode()
        }
    }

    private fun enableImmersiveMode() {
        WindowCompat.setDecorFitsSystemWindows(window, false)
        val controller = WindowInsetsControllerCompat(window, window.decorView)
        controller.hide(WindowInsetsCompat.Type.systemBars())
        controller.systemBarsBehavior =
            WindowInsetsControllerCompat.BEHAVIOR_SHOW_TRANSIENT_BARS_BY_SWIPE
    }

    private fun suppressBackButton() {
        onBackPressedDispatcher.addCallback(this) {
            // Intentionally does nothing: back navigation is disabled for this kiosk-style display.
        }
    }

    private fun showLoadFailureAndExit() {
        if (isFinishing) return
        Toast.makeText(this, getString(R.string.load_failure_message), Toast.LENGTH_LONG).show()
        finish()
    }
}
```

- [ ] **Step 2: Build the debug APK**

Run: `./gradlew assembleDebug`
Expected: `BUILD SUCCESSFUL`.

- [ ] **Step 3: Commit**

```bash
git add app/src/main/java/com/litzrsh/codeserver/MainActivity.kt
git commit -m "Handle initial load failure and suppress the back button"
```

---

### Task 5: Manual on-device verification

**Files:** none (verification only).

**Interfaces:**
- Consumes: the finished app from Task 4.

This app has no automated test suite (per spec). This task is a manual checklist to run on a physical device or emulator **on the same LAN as the code-server host (`192.168.0.2:3000`)**. It cannot be executed by an agent without such a device attached — hand this checklist to whoever has that device.

- [ ] **Step 1: Install and launch with the server reachable**

```bash
./gradlew installDebug
```

Launch the app from the device. Expected: code-server's VS Code Web UI loads, the screen is landscape, and the status bar / navigation bar are hidden (full-screen).

- [ ] **Step 2: Verify initial-load failure path**

Turn off the code-server host (or take the device off the LAN), then launch the app fresh. Expected: a short notification appears stating the connection failed, and the app closes shortly after.

- [ ] **Step 3: Verify a post-load drop does not crash or exit the app**

With the app already showing the loaded page, disable the code-server host or the device's network. Expected: the app does not crash and does not exit — the WebView may show its own in-page error, but the app process stays alive.

- [ ] **Step 4: Verify the back button does nothing**

With the app open (loaded or not), press the hardware/gesture back button repeatedly. Expected: no visible effect — the app does not exit, and nothing navigates.

- [ ] **Step 5: Verify the on-screen keyboard doesn't hide what you're typing**

Tap into the editor and type a line of code using the on-screen keyboard. Expected: the caret/cursor line stays visible above the keyboard, not hidden behind it. (Added after the final review flagged this as plausible but unconfirmed: the app doesn't currently resize its content or inset around the keyboard, and this is the app's primary use case, so it needs a real answer from a device.)

- [ ] **Step 6: Verify an outbound link can be recovered from**

Open something in code-server that links off-host — the Help menu, an extension's marketplace page, a release-notes link, anything that isn't `192.168.0.2`. Expected: it opens in the device's normal browser app, not inside this WebView, and you're still able to get back to the code-server app afterward (e.g. via the device's app switcher, since the hardware back button inside this app does nothing). (Added after the final review: since back is fully suppressed, an outbound link that stayed in-WebView would have had no way out.)

- [ ] **Step 7: Record the result**

If all six checks pass, this task is complete — no code changes are expected from this task. If any check fails, file it as a bug against the relevant task above (Task 3 for load/immersive issues, Task 4 for failure-handling, back-button, or navigation issues) rather than patching ad hoc.

---

## After This Plan

Pushing to `https://github.com/litzrsh/code-app.git` (already configured as the `origin` remote) is a separate, explicit step — do not push automatically as part of task execution. Ask the user before running `git push`.
