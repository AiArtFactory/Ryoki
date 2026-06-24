# Ryoki — Mihon Extension Repository

Mihon/Tachiyomi extension repository. Extensions are built as Android APKs, signed with a keystore, then published here on the `repo` branch for Mihon to consume.

## Repo layout (must be preserved)

```
Ryoki/                          ← this directory
├── repo.json                   ← repo metadata — REQUIRED field: meta.signingKeyFingerprint
├── index.json                  ← full extension list (human-readable, not fetched by Mihon directly)
├── index.min.json              ← minified index — same content as index.json but compact
├── README.md
├── apk/                        ← APK files only
│   └── tachiyomi-{lang}.{sourceName}-v{version}.apk
└── icon/                       ← PNG icons only
    └── eu.kanade.tachiyomi.extension.{lang}.{sourceName}.png
```

**Never** put APKs or icons outside their respective directories. Mihon resolves URLs from `index.json` using these exact paths.

## How Mihon discovers extensions

1. User enters a URL pointing to `repo/repo.json` (NOT `index.json`).
2. Mihon reads `repo.json`'s `meta.signingKeyFingerprint` and stores it as the trusted key for this repo.
3. Mihon fetches `index.min.json` (or falls back to `index.json`) to list extensions.
4. When installing an extension, Mihon downloads the APK from `apkUrl`, extracts its signing certificate, and compares the SHA-256 fingerprint against the one from `repo.json`. **If they match → auto-trusted.** If not → user must manually trust it.

The `signingKey` in `index.json` is informational only; Mihon never reads it at runtime. Only the `signingKeyFingerprint` in `repo.json` matters for auto-trust.

## Prerequisites

- **Android Studio** (provides Gradle + JDK 21)
- **Android SDK Build-Tools** (provides `apksigner`, typically under `$ANDROID_SDK/build-tools/`)
- **Java 21** — on macOS with Android Studio: `/Applications/Android Studio.app/Contents/jbr/Contents/Home`
- **Homebrew** (for `pinentry-mac` if using GPG commit signing)
- **GPG keypair** for signed commits (optional but recommended; shows "Verified" badge on GitHub)
- **SSH key pair** for GitHub authentication

## Adding a new extension — step-by-step

### 1. Build the APK

Extensions are standalone Android Studio projects (not part of the mihon repo). Create a new Android application project with these settings:

```groovy
// build.gradle.kts
plugins {
    id("com.android.application")
    kotlin("android")
}

android {
    namespace = "eu.kanade.tachiyomi.extension.{lang}.{sourceName}"
    defaultConfig {
        applicationId = namespace
        minSdk = 26
        targetSdk = 36
        versionCode = 1
        versionName = "1.0.1"
    }
    buildTypes {
        release {
            isMinifyEnabled = false
            signingConfig = signingConfigs.named("debug").get()
        }
    }
}

dependencies {
    compileOnly(files("../mihon/source-api/src/commonMain"))
    compileOnly("org.jsoup:jsoup:1.22.2")
    compileOnly("com.squareup.okhttp3:okhttp:5.4.0")
    compileOnly("io.reactivex:rxjava:1.3.8")
}
```

You will need a local checkout of [mihonapp/mihon](https://github.com/mihonapp/mihon) to provide the `source-api` dependency. Clone it alongside this repo and reference `../mihon/source-api/src/commonMain` as shown above.

Build the APK with Android Studio (Build → Build Bundle(s) / APK(s) → Build APK) or Gradle from the command line. The APK will be at `build/outputs/apk/debug/{module}-debug.apk` (or release if configured).

### 2. Sign with your keystore

You need a one-time `.keystore` file generated via `keytool`:

```bash
# Generate keystore (one time only — keep this safe!)
"$JAVA_HOME/bin/keytool" -genkeypair \
  -keystore mihon-extensions.keystore \
  -alias mihon-ext \
  -keyalg RSA -keysize 2048 -validity 3650 \
  -dname "CN=YourName, OU=Dev, O=YourOrg, C=US" \
  -keypass YOUR_PASS \
  -storepass YOUR_PASS

# Sign the APK (apksigner path varies by SDK installation)
apksigner sign \
  --ks mihon-extensions.keystore \
  --ks-key-alias mihon-ext \
  --ks-pass pass:YOUR_PASS \
  --key-pass pass:YOUR_PASS \
  yourModule-release.apk

# Verify signature and copy the SHA-256 fingerprint
apksigner verify --print-certs yourModule-release.apk
```

**Critical:** Use the **same keystore for every extension in this repo**. The SHA-256 fingerprint goes into `repo.json` and `index.json`. Mixing keystores breaks auto-trust.

### 3. Place files in the repo

Copy the signed APK and icon into the correct directories:

```bash
# APK naming convention: tachiyomi-{lang}.{sourceName}-v{version}.apk
cp signed.apk apk/tachiyomi-en.manhwaus-v1.0.1.apk

# Icon naming convention: eu.kanade.tachiyomi.extension.{lang}.{sourceName}.png (512x512)
cp icon.png icon/eu.kanade.tachiyomi.extension.en.manhwaus.png
```

Icon must be a **square PNG** (preferably 512×512). Crop logos to square before placing them.

### 4. Update index.json

Add the new extension as an entry in `extensionList.extensions`. Each entry requires:

| Field | Required | Notes |
|-------|----------|-------|
| `name` | Yes | Display name in Mihon Browse |
| `packageName` | Yes | Must match the APK's Android package ID (e.g., `eu.kanade.tachiyomi.extension.en.manhwaus`) |
| `resources.apkUrl` | Yes | Full raw GitHub URL to the APK |
| `resources.iconUrl` | Yes | Full raw GitHub URL to the PNG icon |
| `extensionLib` | Yes | Extension SDK version — always `"1.4"` for modern extensions |
| `versionCode` | Yes | Integer, increment per release (0-based or 1-based) |
| `versionName` | Yes | Human-readable, format `X.Y.Z` |
| `contentWarning` | Yes | `"NSFW"` if the source contains adult content; omit otherwise |
| `sources[].id` | Yes | Unique 64-bit integer. Generate with: `python3 -c "import secrets; print(secrets.randbelow(2**63))"` |
| `sources[].name` | Yes | Display name |
| `sources[].language` | Yes | ISO code: `"en"`, `"all"`, `"ja"`, etc. |
| `sources[].homeUrl` | Yes | The website URL the extension scrapes |

**URL format rules for `apkUrl` and `iconUrl`:**
- Must start with `https://raw.githubusercontent.com/`
- Path: `/{owner}/{repo}/refs/heads/repo/{dir}/{filename}`
- Do NOT use `github.com/raw/` — that path does not exist
- Do NOT put the APK URL in `iconUrl` or vice versa

**Example of correct URLs:**
```json
"apkUrl": "https://raw.githubusercontent.com/AiArtFactory/Ryoki/refs/heads/repo/apk/tachiyomi-en.manhwaus-v1.0.1.apk",
"iconUrl": "https://raw.githubusercontent.com/AiArtFactory/Ryoki/refs/heads/repo/icon/eu.kanade.tachiyomi.extension.en.manhwaus.png"
```

### 5. Commit and push

```bash
cd path/to/Ryoki
git add apk/ icon/ index.json index.min.json
git commit -m "Add {SourceName} v{version}"
git push
```

## Common pitfalls (learned from experience)

### Pitfall 1: repo.json references non-existent index.pb
**Symptom:** Mihon shows HTTP 404 or "Field 'meta' is required" error.
**Cause:** `repo.json` has an `index_v2` key pointing to `index.pb`, but the file doesn't exist on GitHub.
**Fix:** Do NOT include `index_v2` in `repo.json`. Mihon falls back to `index.min.json` automatically.

### Pitfall 2: Wrong URL for adding repo in Mihon
**Wrong:** `https://raw.githubusercontent.com/{owner}/{repo}/refs/heads/repo/index.json`
**Correct:** `https://raw.githubusercontent.com/{owner}/{repo}/repo/repo.json`
Mihon reads `repo.json` for the `meta` field. `index.json` has no `meta`.

### Pitfall 3: iconUrl points to APK instead of PNG
Both `apkUrl` and `iconUrl` must point to their respective file types. A common copy-paste error puts the APK URL in both fields.

### Pitfall 4: GitHub URLs use wrong format
`https://github.com/{owner}/{repo}/raw/refs/heads/repo/...` returns 404.
Use `https://raw.githubusercontent.com/{owner}/{repo}/refs/heads/repo/...` instead.

### Pitfall 5: APK icon resource not compiled into APK
The mipmap resources (`res/mipmap-*/ic_launcher.png`) must exist in the source module before building. If they're missing, the APK has no launcher icon and Mihon shows a blank placeholder.

### Pitfall 6: GPG commit signing fails in certain terminals (e.g., Ghostty)
**Symptom:** `gpg: signing failed: No pinentry` or `Inappropriate ioctl for device`.
**Fix:** Install `pinentry-mac` via Homebrew, set the correct path in gpg-agent.conf:
```bash
echo "pinentry-program /opt/homebrew/bin/pinentry-mac" > ~/.gnupg/gpg-agent.conf
gpgconf --kill gpg-agent
```

### Pitfall 7: SSH key not used for push
**Symptom:** `Permission denied (publickey)` even though the key exists.
**Fix:** Create or update `~/.ssh/config`:
```
Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/your-github-key
```

### Pitfall 8: Git config leaking PII
Always set `user.name` and `user.email` in the repo's **local** git config, not global. This prevents personal info from appearing in commits across different projects.

## index.json field reference

```json
{
  "name": "Display name of the extension repository",
  "badgeLabel": "3-letter abbreviation shown next to repo name (e.g., KEI, MUS, RYK)",
  "signingKey": "SHA-256 fingerprint of the APK signing certificate",
  "contact": {
    "website": "URL to repo website or GitHub page"
  },
  "extensionList": {
    "extensions": [
      {
        "name": "Source display name",
        "packageName": "eu.kanade.tachiyomi.extension.{lang}.{sourceName}",
        "resources": {
          "apkUrl": "https://raw.githubusercontent.com/{owner}/{repo}/refs/heads/repo/apk/{filename}.apk",
          "iconUrl": "https://raw.githubusercontent.com/{owner}/{repo}/refs/heads/repo/icon/{filename}.png"
        },
        "extensionLib": "1.4",
        "versionCode": 0,
        "versionName": "1.0.0",
        "contentWarning": "NSFW",
        "sources": [
          {
            "id": 1234567890123456789,
            "name": "Source display name",
            "language": "en",
            "homeUrl": "https://example.com"
          }
        ]
      }
    ]
  }
}
```

## How to create a new extension source (Kotlin)

Create a standalone Android Studio project (not a mihon module). The `source-api` from [mihonapp/mihon](https://github.com/mihonapp/mihon) provides the interface contract.

1. Set up the Android project with `compileOnly` dependencies on `source-api`, `jsoup`, `okhttp`, `injekt`, and `rxjava`.
2. Create an `AndroidManifest.xml` declaring `<uses-feature android:name="tachiyomi.extension" />` and metadata for the extension class + NSFW flag.
3. Implement a source class extending `HttpSource` (preferred) or `ParsedHttpSource` (deprecated but simpler). The source-api lives at `source-api/src/commonMain/kotlin/eu/kanade/tachiyomi/source/`.
4. Override required properties: `name`, `baseUrl`, `lang`, `supportsLatest`.
5. Implement the data-fetching methods — either the modern suspend API (`getPopularManga`, `getSearchManga`, `getMangaUpdate`, `getPageList`) or the deprecated helper-based API (`popularMangaRequest`, `popularMangaParse`, etc.).
6. Generate launcher icons: crop logo to square, scale to 48/72/96/144/192px, save as `ic_launcher.png` in `res/mipmap-{mdpi,hdpi,xhdpi,xxhdpi,xxxhdpi}/`.
7. Build and sign as described above.

**Key source-api classes to know:**
- `eu.kanade.tachiyomi.source.CatalogueSource` — base interface for browse-capable sources
- `eu.kanade.tachiyomi.source.online.HttpSource` — abstract class with HTTP/Jsoup helpers (preferred)
- `eu.kanade.tachiyomi.source.online.ParsedHttpSource` — deprecated convenience class with Jsoup selectors
- `eu.kanade.tachiyomi.source.model.{SManga,SChapter,Page,MangasPage}` — data models
- `eu.kanade.tachiyomi.source.model.FilterList` — filter support for search

## GitHub Pages requirements

- Deploy from the `repo` branch (not `main`).
- Source: **Branch** → `repo`, folder: `/ (root)`.
- All files in the repo root and subdirectories (`apk/`, `icon/`) are served at their raw paths.
- The APK is served as `application/octet-stream` by GitHub, which Mihon accepts.

## Updating an existing extension

1. Rebuild the APK with a bumped `versionCode` and `versionName`.
2. Re-sign with the **same keystore** (same fingerprint = auto-trust preserved).
3. Replace the old APK in `apk/` with the new one.
4. Update `index.json`: bump `versionCode`, `versionName`, and optionally update `apkUrl` if the filename changed.
5. Commit and push.
