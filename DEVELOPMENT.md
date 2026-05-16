# Development

How to build, deploy, and run `collection-app` during development. This document captures the working setup and the side-quests that wasted time on the way there. Read the "Friction points" section before you start fresh on another machine, or you will repeat them.

Concrete identifiers (Apple Team ID, cert fingerprints, device serials, the build-keychain password) are NOT in this document on purpose. Each one is either out-of-band (you will look it up locally) or stored only in the Mac-side scripts under `~/bin/` on the signing host. The placeholders in this guide are written as `<NAME>`.

## Architecture

Three machines, divided by responsibility:

- **Linux dev box**. Primary development. Edit code, run Android builds, run the Android emulator, drive everything else via SSH.
- **macOS system**. Signing host for iOS. Apple does not permit iOS compilation or signing from non-Apple hardware. Reachable from the Linux box over LAN via SSH host alias `mac` (configure in `~/.ssh/config`).
- **Devices**. Physical iPhone, iPad, and one Android phone. All deployed to over the network during normal use. The iPhone and iPad reach the Mac via Wi-Fi after one-time Xcode device pairing. The Android phone reaches the Linux box via wireless adb.

```
┌─────────────────┐                 ┌─────────────────┐
│  Linux dev box  │  SSH (key-auth) │  macOS          │
│  ─ edit code    ├────────────────►│  ─ codesign     │
│  ─ Android NDK  │                 │  ─ dx serve     │
│  ─ Android sim  │                 │     --ios       │
│  ─ Android adb  │                 └────────┬────────┘
└────┬────────────┘                          │
     │ wireless adb (Wi-Fi)             Wi-Fi│ (Xcode "Connect via network")
     ▼                                       ▼
┌──────────────┐                 ┌──────────────┐  ┌──────────────┐
│ Android      │                 │ iPhone       │  │ iPad         │
│ phone        │                 │              │  │              │
└──────────────┘                 └──────────────┘  └──────────────┘
```

## Identifiers you will need

Look these up at setup time. None of them belong in this document.

| Item | How to find or set |
| --- | --- |
| Bundle identifier | Set in `Dioxus.toml` `[bundle] identifier`. Reverse-DNS style. Must be Java-valid (no hyphens), see warning below. |
| Apple Team ID | On Apple Developer site under Membership Details, or in any provisioning profile (`TeamIdentifier`), or in the cert's OU field. |
| Dev cert SHA1 | `security find-identity -v -p codesigning` on the Mac. The 40-character hex prefix is the SHA1. |
| Provisioning profile | Xcode generates one under `~/Library/Developer/Xcode/UserData/Provisioning Profiles/<UUID>.mobileprovision` on the Mac. See "Free-tier provisioning profile bootstrap" below. |
| Build keychain password | Pick any string. Bake into the Mac-side scripts at `~/bin/dx-*.sh`. Rotate immediately if it leaves the Mac. |
| Android phone serial | `adb devices -l` on the Linux box. For wireless adb, also visible as `adb-<SERIAL>-<random>._adb-tls-connect._tcp`. |

## Daily workflow

```bash
# 1. Edit on Linux
$EDITOR src/main.rs

# 2. (When you want to test on iOS) sync to Mac
rsync -av --exclude target --exclude .git --exclude .claude \
  ./ mac:~/collection-app/

# 3. Deploy
#    Android emulator (Linux-local):
dx serve --android

#    Android phone (wireless adb, must be paired, see one-time setup):
dx run --android --device <DEVICE_SERIAL>

#    iPhone (wireless, via Mac over SSH):
ssh mac '~/bin/dx-run.sh'

#    iPad (wireless, via Mac over SSH):
ssh mac 'DX_IOS_DEVICE="<iPad device name>" ~/bin/dx-run.sh'

#    iOS Simulator on the Mac:
ssh mac 'cd ~/collection-app && . ~/.cargo/env && dx serve --ios'
```

## Mac scripts (`~/bin/` on the Mac)

| Script | Purpose |
| --- | --- |
| `setup-build-keychain.sh` | One-time bootstrap. Creates dedicated build keychain, imports dev cert, sets partition list. Must be run in the Mac's local terminal because `security export` from the login keychain triggers a GUI password prompt. |
| `dx-run.sh` | One-shot install and launch (`dx run`). Best for verification. Default target is the iPhone. Override with `DX_IOS_DEVICE` env var. |
| `dx-deploy.sh` | Same but with `dx serve` (stays attached, holds dev server open). |
| `dx-serve-bg.sh` | Detached `dx serve` via `nohup`, logs to `~/collection-app/dx-serve.log`. Use this when you want a persistent dev loop you can return to. |

The three deploy scripts hard-code the build keychain password and the dev cert SHA1. Edit the scripts to put YOUR values in. Do not commit those scripts to a public repo, do not paste them into a doc. They are sized to fit in one terminal screen on purpose.

## Hot-reload

`dx serve` watches files in the project dir and rebuilds on change. With `dx-serve-bg.sh` running on the Mac and you editing on Linux, the workflow is: edit, save, rsync, dx auto-rebuilds and pushes to the connected device.

> [!WARNING]
> `dx serve` over wireless adb on a physical Android phone crashed the app at runtime with `FORTIFY: pthread_mutex_lock called on a destroyed mutex`, then signal-9 kill. Suspected interaction between Dioxus's dev-server hot-reload path and the wireless-adb transport (`adb reverse` works, so it is something subtler). For the Android phone use `dx run` (one-shot, no dev server). For the iOS devices `dx serve` is fine.

## Weekly chore: re-sign iOS apps

Free-tier Apple signing certificates produce app bundles that stop launching after 7 days on the device. To get them back, re-run any of the iOS deploy commands. They re-sign and re-install. No re-provisioning needed. The cert and profile both stay valid for about 12 months.

> [!NOTE]
> Upgrading to the paid Apple Developer Program ($99/yr) lifts the 7-day expiry and enables TestFlight (over-the-air installs without re-running the Mac build).

---

# Friction points (read this before redoing the setup from scratch)

Each subsection is a thing that wasted real time on the way to working. The bold sentence is the takeaway. The warning callout is the gotcha verbatim.

## Android phone deeply discharged, looked bricked, wasn't

An Android phone that had been sitting unused refused to boot when first plugged in. No display, no logo, nothing. After a few minutes it started pulsing and vibrating every 6 seconds, but still no display. The device looked completely dead.

> [!WARNING]
> An Android phone whose battery has drained to absolute 0% can take an hour or more of charging before it will even attempt to boot. During that time it will appear bricked (no screen, no LED) or "vibrating brick" (about 6s pulse). This is normal behaviour after long storage and is documented in community knowledge bases. Do not conclude the phone is dead. Leave it on the charger for at least an hour before troubleshooting.

The phone autonomously decided to boot after about an hour on the charger. No magic combo of buttons needed.

## `dx new --yes` does not actually skip all prompts

The Dioxus CLI's project generator has six interactive questions (sub-template, fullstack, router, tailwind, LLM prompts, default platform). `--yes` is supposed to take defaults for all of them.

> [!WARNING]
> In practice `dx new --yes` still requires an interactive TTY and will silently hang in any non-TTY context (including SSH sessions, `script(1)` pty wrappers, agent harnesses). Always run `dx new` in a real terminal where you can answer prompts yourself. Workaround attempts to fake a TTY produced empty output and zombie processes.

The wizard also ignored some of the explicit "no" answers. It generated `AGENTS.md` and `tailwind.css` files despite declining both. Manually delete after.

## Bundle ID hyphens: works on iOS, breaks on Android

A bundle identifier like `com.example.collection-app` is valid for iOS. Android Gradle then fails with:

```
Namespace 'com.example.collection-app' is not a valid Java package name
as 'collection-app' is not a valid Java identifier.
```

> [!WARNING]
> Java package names disallow hyphens. iOS bundle IDs allow them. `dx` injects the same string into both Android's `namespace`/`applicationId` AND iOS's bundle ID, with no per-platform override available in `Dioxus.toml`. Pick a Java-valid identifier (no hyphens) from the start: `com.example.collectionapp`. Changing it mid-project forces redoing iOS provisioning. The existing `.mobileprovision` is bound to the old bundle ID.

The fix is doing the Xcode dummy-app dance again to get a new provisioning profile for the renamed bundle ID. About 15 minutes of avoidable work.

## JRE vs JDK on Ubuntu

Ubuntu 24.04 ships `default-jre` and `openjdk-21-jre` but not the corresponding `-jdk` packages by default. Gradle reports:

```
Toolchain installation '/usr/lib/jvm/java-21-openjdk-amd64' does not provide
the required capabilities: [JAVA_COMPILER]
```

> [!WARNING]
> `java -version` working doesn't mean you have a JDK. It only means you have a JVM. Gradle's Android builds need `javac`, which lives only in `openjdk-21-jdk` (or whichever `-jdk` package matches your version). Install `openjdk-21-jdk` explicitly, even if `java` already works.

## `JAVA_HOME` must be set explicitly

Even after installing `openjdk-21-jdk`, Gradle continued to report the same JAVA_COMPILER missing error. Setting `JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64` resolved it.

> [!WARNING]
> Gradle's auto-detection of JDKs from `/usr/lib/jvm` is fragile. It sometimes flags valid JDKs as missing the compiler. Set `JAVA_HOME` explicitly in `~/.bashrc` and forget about it.

## `--apple-team-id` is misnamed

`dx`'s `--apple-team-id` flag passes its value to `codesign --sign`. `codesign` matches against the cert's Common Name (CN) by substring, not against the Apple Team ID.

> [!WARNING]
> Despite the flag name, you must pass either the cert's SHA1 fingerprint or a substring of the cert's CN. The actual 10-character Team ID does NOT work as the flag value. It doesn't appear in the cert CN. The Team ID lives in the cert's OU field, which codesign doesn't search.
>
> Also: the value in parentheses in the cert's display name (e.g. `Apple Development: <email> (XXXXXXXXXX)`) is not the team ID either. It is a cert serial. The real team ID is in the provisioning profile (`TeamIdentifier`) or the cert's OU.

## `errSecInternalComponent` codesigning over SSH

The Mac's login keychain is unlocked when the user logs into the GUI session, but SSH sessions have a separate unlock state. Calls to `codesign` from SSH against the login keychain fail with `errSecInternalComponent`. Opaque, undocumented, no useful diagnostic.

> [!WARNING]
> Running `security set-key-partition-list -S apple-tool:,apple:` on the login keychain's key does NOT fix this. The partition list controls which app categories can use the key without prompting, but if the SSH session's keychain context is "locked", codesign can't reach the key at all.
>
> The fix used here: create a dedicated build keychain on the Mac with a known fixed password, import the dev cert and key into it, set its partition list, and have SSH-driven build scripts call `security unlock-keychain -p <pw> build.keychain-db` before invoking codesign. The keychain's password lives in the Mac-side scripts (`~/bin/dx-*.sh`), never on the Linux side and never in the repo. This is the same pattern used by Fastlane and GitHub Actions self-hosted runners.

## Free-tier provisioning profile bootstrap

Apple's developer portal doesn't let free-tier accounts download provisioning profiles. `dx` does have an auto-provisioning step, but it needs an existing profile for the bundle ID to extend.

> [!WARNING]
> To bootstrap a profile for a free-tier "Personal Team" Apple ID, you must use Xcode's GUI build flow at least once. Create a throwaway iOS App project with the exact bundle ID you want, set the team to Personal Team, enable automatic signing, pick a connected device as the run destination, and press Cmd+R. Xcode generates the provisioning profile during signing and saves it under `~/Library/Developer/Xcode/UserData/Provisioning Profiles/`. After that, `dx` can use it and even extend it to other paired devices automatically (one Xcode run for the iPhone was enough to also cover the iPad).

## USB cable and port reliability

Trying to get a physical Android phone visible to `adb` over USB on the dev workstation produced about 30 minutes of failures. Symptoms: `lsusb` saw the device, but `adb devices` did not, and `dmesg` showed:

```
usb 5-1.3: device descriptor read/64, error -32
usb 5-1.3: device not accepting address NN, error -71
usb 5-1-port3: attempt power cycle
```

Multiple cables produced the same `-71` and `-32` errors.

> [!WARNING]
> USB errors `-71` (EPROTO) and `-32` (EPIPE) during enumeration are textbook signs of a marginal physical link. Flaky cable, dirty connector, overloaded hub, or a buggy USB controller. The phone eventually enumerates enough for `lsusb` to read descriptors, but bulk transfers (which adb relies on) silently fail. Do not burn time on udev rules or adb restarts when you see these errors in `dmesg`. Swap cables, swap ports, or just switch to wireless adb (see below).

## Wireless adb is the better default

After the USB rabbit hole, wireless pairing worked first try:

```bash
# On phone: Developer Options -> Wireless debugging -> Pair device with pairing code
# (shows IP:port + 6-digit code)

adb pair <PAIR_IP>:<PAIR_PORT> <CODE>
```

Once paired, the phone auto-advertises via mDNS as `adb-<SERIAL>-<random>._adb-tls-connect._tcp` and `adb devices` picks it up without further action. Pairing is persistent. No need to re-pair after reboot, just re-enable "Wireless debugging" on the phone.

> [!NOTE]
> Wireless adb requires Android 11+. `adb reverse tcp:N tcp:N` works over wireless adb, so dev servers that need a reverse port-forward from device to host work too. Despite this, `dx serve --android` over wireless adb still crashes the app on the connected device (see the hot-reload warning above). Use `dx run` for now.

## `dx serve --android` builds for the wrong ABI

Without a `--device` argument, `dx serve --android` auto-boots the configured AVD and picks the build architecture from the AVD's ABI (x86_64 for a standard AVD). If your connected device is `arm64-v8a` (any modern real phone), the APK rejects with `INSTALL_FAILED_NO_MATCHING_ABIS`.

> [!WARNING]
> Always pass `--device <serial>` when you want to target a specific device. With `--device`, `dx` skips the AVD auto-boot AND picks the build target from the device's ABI. Example: `dx run --android --device <DEVICE_SERIAL>`.

## Tailscale and network paths

The original plan called for Tailscale to reach the Mac. The Linux dev box didn't have Tailscale installed. Once installed, the coordination service returned HTTP 502 errors during the `tailscale up` flow (transient backend issue confirmed by checking [status.tailscale.com](https://status.tailscale.com)). The fallback was direct LAN SSH.

> [!NOTE]
> If you can't reach the Mac over LAN (different networks, behind NAT), Tailscale is the obvious tool. For the same-LAN case, plain SSH to a static (or DHCP-reserved) IP is simpler and removes a dependency. Set up a DHCP reservation on your router so the Mac's IP doesn't drift.

## GNOME Keyring locked

An early attempt stored the Mac password in libsecret via `secret-tool` on Linux. The default keyring collection was locked and `gnome-keyring-daemon --unlock` with the login password didn't unlock it (Ubuntu keyring password can drift from the login password).

> [!NOTE]
> This turned out to be the wrong solution to the wrong problem anyway. The correct fix for SSH-driven codesign is a build keychain on the Mac, not credential storage on Linux. See the codesign warning above. If you ever need a secret on Linux for some other reason, a chmod-600 plain file is fine for a single-user dev box. `pass` (gpg-encrypted) is fine if you want it nicer.

---

# One-time setup

## Linux dev box

1. Rust toolchain (rustup).
2. Android targets: `rustup target add aarch64-linux-android armv7-linux-androideabi i686-linux-android x86_64-linux-android`
3. `sudo apt install openjdk-21-jdk libsecret-tools`
4. Android cmdline-tools to `~/Android/sdk/cmdline-tools/latest/` (downloaded from `https://dl.google.com/android/repository/commandlinetools-linux-<build>_latest.zip`. Get the current build number from <https://developer.android.com/studio>).
5. SDK components via `sdkmanager`: `platform-tools platforms;android-35 build-tools;35.0.1 ndk;28.2.13676358 emulator system-images;android-35;google_apis;x86_64`
6. AVD: `avdmanager create avd -n collection-test -k "system-images;android-35;google_apis;x86_64" -d pixel_7`
7. `sudo usermod -aG kvm $USER`. Needed for hardware-accelerated emulator. Group membership only applies to new processes. Use `sg kvm -c "..."` to invoke the emulator with kvm as primary group in an existing shell.
8. udev rule for adb non-root access at `/etc/udev/rules.d/51-android.rules`. Use the vendor ID of your phone manufacturer (find via `lsusb` after plugging in).
9. Env vars in `~/.bashrc`:

    ```bash
    export JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64
    export ANDROID_HOME="$HOME/Android/sdk"
    export ANDROID_SDK_ROOT="$ANDROID_HOME"
    export ANDROID_NDK_HOME="$ANDROID_HOME/ndk/28.2.13676358"
    export NDK_HOME="$ANDROID_NDK_HOME"
    export PATH="$JAVA_HOME/bin:$PATH:$ANDROID_HOME/cmdline-tools/latest/bin:$ANDROID_HOME/platform-tools:$ANDROID_HOME/emulator"
    ```

10. SSH key for the Mac at `~/.ssh/id_ed25519_mac`, `Host mac` entry in `~/.ssh/config`.

## Mac

1. Xcode from the App Store (about 40 GB, GUI install).
2. `sudo xcode-select -s /Applications/Xcode.app/Contents/Developer && sudo xcodebuild -license accept`
3. Rust toolchain via rustup.
4. iOS targets: `rustup target add aarch64-apple-ios x86_64-apple-ios aarch64-apple-ios-sim`
5. `cargo install dioxus-cli --version <X.Y.Z> --locked` (pin to match the dx on the Linux box).
6. Apple ID signed into Xcode. Settings -> Accounts -> add Apple ID -> Manage Certificates -> `+` -> Apple Development.
7. iOS Simulator: `xcrun simctl create "<sim name>" <devicetype> <runtime>` (list available with `xcrun simctl list devicetypes` and `xcrun simctl list runtimes`).
8. iPhone and iPad USB-paired with Mac, Developer Mode enabled on each.
9. Xcode -> Window -> Devices and Simulators -> tick "Connect via network" for each device. After this, USB cables aren't needed for deploys.
10. Throwaway Xcode iOS App project with your bundle ID, team Personal Team, automatic signing -> Cmd+R targeting one paired device. The provisioning profile is auto-generated and is auto-extended to other paired devices the first time dx builds for them.
11. Run `~/bin/setup-build-keychain.sh` in the Mac's local terminal. It needs the GUI prompt to export the dev cert from the login keychain to a build keychain.
12. Scripts `dx-run.sh`, `dx-deploy.sh`, `dx-serve-bg.sh` in `~/bin/`. They are invoked over SSH from the Linux box. Their content lives only on the Mac. NEVER commit them anywhere. They contain the build keychain password and the dev cert SHA1.

## Phone

- Developer Options unlocked. On Samsung One UI: tap `Build number` 7 times under `Software information`. On stock Android: tap it under `About phone`.
- `Wireless debugging` enabled.
- One-time `adb pair <IP>:<port> <code>` from the Linux box. Phone serial after pairing is the `--device` argument for dx.
