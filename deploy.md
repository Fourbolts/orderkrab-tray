# PrintKrab: build and release

All installers are built on the ARM Mac, including the Windows one. The build
downloads the right JDK per platform and architecture on first use and caches
it under `out/jlink/`.

## One-time setup

```bash
brew install openjdk@11 ant nsis makeself
```

```bash
npm install -g appdmg
```

Only needed if installers are ever committed (they are LFS-tracked via `.gitattributes`):

```bash
brew install git-lfs && git lfs install
```

## Before every release

1. Check out the release branch (currently `orderkrab-2.2.5`).
2. The version number lives in two places and must match:
   - `src/qz/common/Constants.java` (`VERSION`)
   - `js/qz-tray.js` (`VERSION` and the `@version` header)
   Output file names use this version. Replace `2.2.5` in the commands below accordingly.
3. Every `ant` run wipes `out/` (except the cached JDKs), so copy each installer
   into `releases/` before starting the next build. The commands below do that.
4. On this ARM Mac the build targets the host architecture unless
   `-Dtarget.arch=x86_64` is given. Forgetting it produces ARM64 installers.

## Build commands

macOS ARM package:

```bash
ant pkgbuild && cp out/qz-tray-2.2.5-arm64.pkg releases/printkrab-2.2.5-arm64_arm_mac.pkg
```

macOS ARM disk image (optional):

```bash
ant dmg && cp out/qz-tray-2.2.5-arm64.dmg releases/printkrab-2.2.5_arm_mac.dmg
```

macOS Intel package:

```bash
ant pkgbuild -Dtarget.arch=x86_64 && cp out/qz-tray-2.2.5-x86_64.pkg releases/printkrab-2.2.5-x86_64_intel_mac.pkg
```

macOS Intel disk image (optional):

```bash
ant dmg -Dtarget.arch=x86_64 && cp out/qz-tray-2.2.5-x86_64.dmg releases/printkrab-2.2.5-x86_64_intel_mac.dmg
```

Windows 64-bit installer (the arch flag is required):

```bash
ant nsis -Dtarget.arch=x86_64 && cp out/qz-tray-2.2.5-x86_64.exe releases/printkrab-2.2.5-x86_64_win.exe
```

Linux installer (optional):

```bash
ant makeself -Dtarget.arch=x86_64 && cp out/qz-tray-2.2.5-x86_64.run releases/printkrab-2.2.5-x86_64_linux.run
```

## Quick checks after building

Version and architecture of a Mac package (`arm64` or `x86_64`):

```bash
pkgutil --expand-full releases/printkrab-2.2.5-arm64_arm_mac.pkg /tmp/pk && /usr/libexec/PlistBuddy -c 'Print CFBundleShortVersionString' /tmp/pk/*/Scripts/payload/PrintKrab.app/Contents/Info.plist && file /tmp/pk/*/Scripts/payload/PrintKrab.app/Contents/PlugIns/Java.runtime/Contents/Home/bin/java && rm -rf /tmp/pk
```

Architecture of the runtime packed into the last Windows build (must say `x86-64`):

```bash
file out/dist/runtime/bin/java.exe
```

Signature state of a Mac package (currently "no signature" until certificates are set up):

```bash
pkgutil --check-signature releases/printkrab-2.2.5-arm64_arm_mac.pkg
```

## Push

```bash
git push -u origin orderkrab-2.2.5
```

Do not `git add releases/` unless Git LFS is installed and initialized. Without
it the installers are committed as plain files over 100 MB and GitHub rejects
the push.

## Code signing (not set up yet)

Until this is done, macOS packages are ad-hoc signed and Windows installers
carry the QZ test key, so both operating systems warn on install.

macOS:

1. In the OrderKrab Apple Developer account create a "Developer ID Application"
   and a "Developer ID Installer" certificate and install both in the Keychain
   of the build Mac. `security find-identity -v -p codesigning` must list them.
2. Put the OrderKrab Team ID in `apple.packager.signid` in
   `ant/apple/apple.properties` (it still holds QZ's `P5DMU6659X`).
3. `ant pkgbuild` then signs the app, runtime, and package automatically.
4. Notarize and staple each package (one-time: create the keychain profile with
   `xcrun notarytool store-credentials orderkrab-notary --apple-id <apple id> --team-id <team id>`):

```bash
xcrun notarytool submit releases/printkrab-2.2.5-arm64_arm_mac.pkg --keychain-profile orderkrab-notary --wait && xcrun stapler staple releases/printkrab-2.2.5-arm64_arm_mac.pkg
```

Windows:

1. Obtain a code-signing certificate (EV certificate or Azure Trusted Signing
   for immediate SmartScreen reputation; jsign 7.1 in `ant/lib/` supports both,
   plus hardware tokens and cloud HSMs).
2. Create `../private/private.properties` one directory above the repo (kept
   out of git) with `signing.keystore`, `signing.alias`, `signing.storepass`,
   `signing.keypass`, and `signing.tsaurl=http://timestamp.digicert.com`.
   Setting `signing.tsaurl` is what switches the build from the test key to
   real signing. For token or cloud signing add the matching `--storetype`
   in `ant/windows/installer.xml`.
3. `ant nsis -Dtarget.arch=x86_64` then signs all exe files and the installer.

## Pulling upstream QZ Tray updates

- Remote `upstream` points at https://github.com/qzind/tray.git.
- Our code is upstream v2.2.5 plus the PrintKrab commits plus selected 2.2.6
  fixes. PrintKrab-specific edits that must survive any update: branding
  (`ant/project.properties`, `Constants.java`, icons, pkg background), the
  removed update button in `AboutDialog.java`, the always-trusted stubs in
  `src/qz/auth/Certificate.java` and `src/qz/auth/RequestState.java`, and the
  `callCert` catch block in `js/qz-tray.js`.
- Preferred method: rebase the PrintKrab commits onto the new upstream tag on a
  fresh branch, then cherry-pick individual fixes. Skipped from 2.2.6 on
  purpose: raw image refactor, headless dialog rewrite, browser policy
  installer, `.qz.surf` hostname default, log window redesign, macOS icon.
- Upstream master after 2.2.6 moves to Java 25, Ivy dependencies, and the
  FlatLaf theme. Do not merge it wholesale.
