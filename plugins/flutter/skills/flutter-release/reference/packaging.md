# Packaging — per-platform commands and CI templates

Verify versions against current Flutter tooling before use; flags do change between releases.

---

## Android

```bash
# Play Store
flutter build appbundle \
  --release --flavor prod \
  --dart-define-from-file=config/prod.json \
  --obfuscate --split-debug-info=build/symbols/android

# Direct distribution / QA
flutter build apk --release --flavor prod --split-per-abi \
  --dart-define-from-file=config/prod.json \
  --obfuscate --split-debug-info=build/symbols/android
```

`android/key.properties` — **gitignored**, written by CI:

```properties
storePassword=...
keyPassword=...
keyAlias=upload
storeFile=/tmp/upload.jks
```

`android/app/build.gradle.kts`:

```kotlin
signingConfigs {
    create("release") {
        val props = Properties().apply {
            file(rootProject.file("key.properties")).takeIf { it.exists() }?.inputStream()?.use { load(it) }
        }
        storeFile = props["storeFile"]?.let { file(it) }
        storePassword = props["storePassword"] as String?
        keyAlias = props["keyAlias"] as String?
        keyPassword = props["keyPassword"] as String?
    }
}
buildTypes {
    release {
        signingConfig = signingConfigs.getByName("release")
        isMinifyEnabled = true
        isShrinkResources = true
        proguardFiles(getDefaultProguardFile("proguard-android-optimize.txt"), file("proguard-rules.pro"))
    }
}
flavorDimensions += "env"
productFlavors {
    create("dev")     { dimension = "env"; applicationIdSuffix = ".dev";     versionNameSuffix = "-dev" }
    create("staging") { dimension = "env"; applicationIdSuffix = ".staging"; versionNameSuffix = "-stg" }
    create("prod")    { dimension = "env" }
}
```

`android/app/proguard-rules.pro` — keep rules for reflection users:

```proguard
# Flutter deferred components / embedding
-keep class io.flutter.** { *; }
-keep class io.flutter.plugins.** { *; }
# Anything read reflectively by a JSON layer or native SDK
-keep class com.yourorg.models.** { *; }
-keepattributes Signature,InnerClasses,*Annotation*
```

**Decode a keystore in CI:**

```bash
echo "$ANDROID_KEYSTORE_B64" | base64 -d > /tmp/upload.jks
```

---

## Windows

```bash
flutter build windows --release --dart-define-from-file=config/prod.json
# artifact: build/windows/x64/runner/Release/   <- ship the whole directory
```

**MSIX** — `pubspec.yaml`:

```yaml
msix_config:
  display_name: Your App
  publisher_display_name: Your Org
  publisher: CN=Your Org, O=Your Org, C=GB   # must match the certificate subject exactly
  identity_name: com.yourorg.yourapp
  msix_version: 1.4.2.0                       # four parts, must increase
  logo_path: assets/icon/app_icon.png
  capabilities: internetClient
  file_extension: .yourext
```

```bash
dart run msix:create --release
```

**Signing:**

```bash
signtool sign /fd SHA256 \
  /f cert.pfx /p "$CERT_PASSWORD" \
  /tr http://timestamp.digicert.com /td SHA256 \
  build/windows/x64/runner/Release/your_app.exe

signtool verify /pa /v your_app.msix
```

Always timestamp (`/tr`). Without it the signature stops validating when the certificate expires.

**Inno Setup** alternative (`installer.iss`):

```ini
[Setup]
AppName=Your App
AppVersion=1.4.2
DefaultDirName={autopf}\YourApp
PrivilegesRequired=lowest          ; per-user install, no UAC prompt
OutputBaseFilename=YourApp-1.4.2-setup
WizardStyle=modern

[Files]
Source: "build\windows\x64\runner\Release\*"; DestDir: "{app}"; Flags: recursesubdirs

[Icons]
Name: "{autoprograms}\Your App"; Filename: "{app}\your_app.exe"
```

Verify the VC++ runtime on a clean VM; add the redistributable to `[Files]` if the app fails to start.

---

## Linux

```bash
flutter build linux --release --dart-define-from-file=config/prod.json
# artifact: build/linux/x64/release/bundle/

ldd build/linux/x64/release/bundle/your_app | grep "not found"   # must be empty
```

**`.deb`** layout:

```
usr/local/yourapp/            <- the whole bundle directory
usr/share/applications/com.yourorg.yourapp.desktop
usr/share/icons/hicolor/{64x64,128x128,256x256}/apps/yourapp.png
DEBIAN/control
```

`DEBIAN/control`:

```
Package: yourapp
Version: 1.4.2
Architecture: amd64
Maintainer: Your Org <dev@yourorg.example>
Depends: libgtk-3-0, libblkid1, liblzma5
Description: One-line summary.
 Indented continuation lines for the long description.
```

```bash
dpkg-deb --build --root-owner-group dist/deb yourapp_1.4.2_amd64.deb
```

`.desktop` file:

```ini
[Desktop Entry]
Type=Application
Name=Your App
Exec=/usr/local/yourapp/your_app
Icon=yourapp
Categories=Utility;
StartupWMClass=your_app
```

`StartupWMClass` must match the window class or the taskbar shows a second, iconless entry.

**AppImage:** build an `AppDir` with the same layout plus `AppRun`, then `appimagetool AppDir`.
**Flatpak:** a manifest with `org.gnome.Platform` as the runtime; plan for portal-mediated file
access and no arbitrary filesystem reads.

**glibc:** build inside an old container (e.g. `ubuntu:20.04`) to support older LTS releases. A
binary built on a current distro will not start on an older one.

---

## Release workflow (GitHub Actions)

```yaml
name: release
on:
  push:
    tags: ["v*"]

jobs:
  android:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { distribution: temurin, java-version: "17" }
      - uses: subosito/flutter-action@v2
        with: { channel: stable, cache: true }
      - run: flutter pub get
      - run: flutter analyze
      - run: flutter test
      - name: Write signing + config secrets
        run: |
          echo "${{ secrets.ANDROID_KEYSTORE_B64 }}" | base64 -d > /tmp/upload.jks
          cat > android/key.properties <<EOF
          storeFile=/tmp/upload.jks
          storePassword=${{ secrets.ANDROID_STORE_PASSWORD }}
          keyAlias=${{ secrets.ANDROID_KEY_ALIAS }}
          keyPassword=${{ secrets.ANDROID_KEY_PASSWORD }}
          EOF
          echo '${{ secrets.PROD_CONFIG_JSON }}' > config/prod.json
      - run: |
          flutter build appbundle --release --flavor prod \
            --dart-define-from-file=config/prod.json \
            --obfuscate --split-debug-info=build/symbols/android
      - uses: actions/upload-artifact@v4
        with:
          name: android-symbols-${{ github.ref_name }}     # keep these; crashes are unreadable without them
          path: build/symbols/android
          retention-days: 365
      - uses: actions/upload-artifact@v4
        with: { name: android-aab, path: build/app/outputs/bundle/prodRelease/*.aab }

  windows:
    runs-on: windows-latest
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
        with: { channel: stable, cache: true }
      - run: flutter pub get
      - run: flutter build windows --release --dart-define-from-file=config/prod.json
      - run: dart run msix:create --release
      - uses: actions/upload-artifact@v4
        with: { name: windows-msix, path: build/windows/x64/runner/Release/*.msix }

  linux:
    runs-on: ubuntu-latest
    container: ubuntu:20.04          # oldest supported glibc
    steps:
      - uses: actions/checkout@v4
      - run: |
          apt-get update && apt-get install -y \
            clang cmake ninja-build pkg-config libgtk-3-dev liblzma-dev curl git unzip xz-utils
      - uses: subosito/flutter-action@v2
        with: { channel: stable, cache: true }
      - run: flutter pub get
      - run: flutter build linux --release --dart-define-from-file=config/prod.json
      - run: ldd build/linux/x64/release/bundle/your_app | grep "not found" && exit 1 || true
      - uses: actions/upload-artifact@v4
        with: { name: linux-bundle, path: build/linux/x64/release/bundle }
```

Secrets referenced: `ANDROID_KEYSTORE_B64`, `ANDROID_STORE_PASSWORD`, `ANDROID_KEY_ALIAS`,
`ANDROID_KEY_PASSWORD`, `PROD_CONFIG_JSON`, and for signed Windows builds `WINDOWS_CERT_B64` /
`WINDOWS_CERT_PASSWORD`.

---

## Pre-tag checks

```bash
flutter analyze                       # zero issues
flutter test                          # green
grep -n "^version:" pubspec.yaml      # bumped, build number increased
git status --porcelain                # clean
rg -n "TODO|FIXME|print\(" lib/       # nothing embarrassing shipping
```

Then run `secure-ship/prod-readiness-gate` — that skill, not this one, decides whether to ship.
