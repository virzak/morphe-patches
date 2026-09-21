# Agent notes

RuTube patches, built on the Morphe patcher (a hard fork of ReVanced). Scaffolded from
`MorpheApp/morphe-patches-template`.

## Build

```bash
GITHUB_ACTOR=<github-user> GITHUB_TOKEN=$(gh auth token) ./gradlew build -x test
```

The Morphe patcher plugin lives on GitHub Packages
(`maven.pkg.github.com/MorpheApp/registry`), which requires authentication even for
public artifacts. The token needs the **`read:packages`** scope; `repo` alone returns
401. `settings.gradle.kts` reads `gpr.user`/`gpr.key` from Gradle properties, falling
back to `GITHUB_ACTOR`/`GITHUB_TOKEN` from the environment.

Output is `patches/build/libs/patches-<version>.mpp`. Note `.mpp`, not ReVanced's
`.rvp` - the two managers cannot load each other's bundles.

### Morphe Manager needs `buildAndroid`, not `build`

`build` produces a bundle containing JVM `.class` files only. That is enough for
morphe-desktop, but Manager loads patches from a `classes.dex` and finds nothing:

```bash
ANDROID_HOME=~/Android/Sdk ./gradlew :patches:buildAndroid
```

The task adds `classes.dex` to the same `.mpp`. Confirm with
`unzip -l patches/build/libs/patches-<version>.mpp | grep classes.dex`.

**The failure mode is silent.** Manager adds the source, shows a green check on the
file, reports the correct version, and lists **Patches: 0**. Nothing says the bundle
was the wrong flavour, and the patch selection screen simply omits the app. If patches
do not appear for an app you know is compatible, check for the dex before re-checking
the compatibility declaration.

Anything a patch reads with `getResourceAsStream` must be loaded through a class
defined in the bundle. Inside `execute` the receiver is the patcher's own context, so
`javaClass` there resolves against the patcher's jar and silently returns null; declare
a private object in the patch file and go through that instead.

## Differences from ReVanced patches

Worth knowing if cribbing from a ReVanced patch, since the two are close enough to be
confusing:

- namespace is `app.morphe`, not `app.revanced`;
- the patch body is `execute { }`, not `apply { }`;
- extensions are attached with `extendWith("extensions/extension.mpe")`;
- compatibility is a `Compatibility` object (name, packageName, apkFileType,
  appIconColor, targets) rather than a bare version list, and it feeds Morphe
  Manager's UI directly.

## Before writing any patch

`Constants.kt` currently holds **guesses**. Confirm each against a real APK:

- package name - `aapt2 dump badging <apk> | head -1`;
- target version - prefer one available on apkmirror.com or uptodown.com, because
  Morphe web-search sends users there;
- icon colour - the launcher icon background.

A wrong package name fails silently: the patches simply never appear for the app in
Manager rather than reporting an error.

## After installing: re-enable link handling, or login breaks

Login is a web flow: the app opens a Chrome Custom Tab to `auth.gid.ru` (Gazprom ID)
and the result comes back through an `https://rutube.ru/...` redirect. That redirect
only reaches the app if Android treats it as a verified app link.

A patched build is signed with a different key, so it can never match the
`assetlinks.json` published on rutube.ru. Verification fails and Android leaves the
domains **disabled**, sending the callback to the browser instead of the app. Symptom:
login simply never completes, with nothing in the log to explain it.

```bash
adb shell pm get-app-links --user 0 ru.rutube.app
#   rutube.ru: 1024        <- >=1024 means the verifier reported failure
#   Selection state: Disabled: rutube.ru, rutube.dev
```

Fix, per install (the adb form of Settings > Apps > RUTUBE > Open by default >
Open supported links):

```bash
adb shell pm set-app-links-user-selection --user 0 --package ru.rutube.app true rutube.ru
adb shell pm set-app-links-user-selection --user 0 --package ru.rutube.app true rutube.dev
```

Verify by firing a link and checking it lands in the app rather than a chooser:

```bash
adb shell am start -a android.intent.action.VIEW -d "https://rutube.ru/video/<id>/"
adb shell dumpsys window | grep -m1 mCurrentFocus   # expect ru.rutube.app
```

No patch can fix this properly - it needs the app's original signing key.

## Verification status

Against RuTube 31.14.2-rustore, patched through Morphe Manager and installed on a
device. Recorded because "it applied" proves nothing and re-deriving this is slow.

| Patch | Verified | How |
|---|---|---|
| Enable background playback | yes | Audio keeps running with the launcher in the foreground: `dumpsys audio` still shows a started `USAGE_MEDIA` track and `dumpsys media_session` reports `PLAYING` with an advancing position. Confirmed on a signed-in account with no subscription. |
| Disable ads | yes | Same video played in the app and on the web at rutube.ru. The web player served ads, the patched app served none. The web side is the part that matters: it establishes the video actually carries ads, which is what makes the app's silence meaningful. |
| Lift download restrictions | yes | Against a 10 hour video (`408fdee3bbd5276eff0dc8f7c6ff0c7f`, 36085s, well past the 21600s limit). Before: the Скачать button is greyed with a padlock and does nothing. After: it is enabled, tapping it starts an ongoing `player_downloading_notifications_channel` foreground service titled with the video, and app storage grew ~80 MB in ten seconds. |
| Custom branding icon | yes | All 15 mipmap entries in the installed APK are pixel-identical to the badged sources, and the badge renders unclipped under the launcher's mask. |
| Unlock subscription features | no | Never needed. Background playback is covered by its own patch, which is narrower. Left off by default. |

Checking ads by playing a video in the app alone is not a test. Most RuTube videos
serve no ad at all, so "no ad appeared" is unfalsifiable without a control that proves
the video has one.

## The one rule worth carrying over

A patch applying is not a patch working. The patcher reports success when a
fingerprint resolved and code was written; it cannot tell you the hooked method is
ever called. Verify on a device, and prefer a log line or visible behaviour over a
green patch count.

## Fingerprint gotcha: class matching has no colon

The comparison type is inferred from the *shape* of the string:

| Declaration | Meaning |
|---|---|
| `"Lru/foo/Bar;"` | equals |
| `"Lru/foo/"` | starts with |
| `"Bar;"` | ends with |
| anything else | contains |

So a package is matched as `"ru/rutube/player/plugin/.../presentation/"`. The upstream
docs show a leading colon (`":com/some/app/ads/"`); that colon is compared literally
and the fingerprint will never match. Cost an hour to find, because a failed
fingerprint only reports "Failed to match" with no hint as to which field was wrong.

Bisecting a failing fingerprint works: start from an exact class plus method name,
confirm it matches, then relax one field at a time.

## Applying patches

There is no Morphe CLI, but `morphe-desktop` ships an executable jar with one:

```bash
java -jar morphe-desktop.jar list-patches --patches=patches/build/libs/patches-<v>.mpp
java -Xmx8g -jar morphe-desktop.jar patch \
  -p <bundle.mpp> -e "<Patch name>" -o out.apk -t ./tmp --keystore rutube.keystore <input.apk>
```

It needs **Java 21** (class file 65); this machine's default is Java 17 for the
ReVanced work, so invoke it as `/usr/lib/jvm/java-21-openjdk-amd64/bin/java`.

Flags differ from revanced-cli: purge is the default (`--disable-purge` opts out) and
the keystore entry alias must be `Morphe`, so pass a fresh `--keystore` path rather
than reusing a ReVanced one.
