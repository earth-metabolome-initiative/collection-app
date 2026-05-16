# collection-app: Cross-Platform Setup Plan

Plan for getting a trivial Dioxus 0.7 hello-world running on:

- Physical Android phone (built from Linux dev box)
- Physical iPhone (built from macOS over SSH)
- Physical iPad (built from macOS over SSH)
- Plus emulators/simulators on the respective host OSes

Each phase is a hard checkpoint. Finish a phase, verify the deliverable, then continue.

## Context

- **Linux dev box**: primary development. Has `rustc` nightly (1.97), `dx` 0.7.7, OpenJDK 21. No Android SDK/NDK yet.
- **macOS system**: reachable via SSH (to be configured). Required for iOS. Apple does not permit iOS compilation or signing from Linux.
- **Apple account**: free Apple ID tier. Implications:
  - Signed apps expire on devices after 7 days. Must be re-signed and redeployed weekly.
  - Max 3 free provisioning profiles at a time.
  - No TestFlight, no App Store. Device install requires USB pairing with the signing Mac.
  - Upgrade to the paid Apple Developer Program ($99/yr) if weekly re-signing becomes a friction point. It enables TestFlight (wireless install, longer validity).
- **Dioxus version**: pinned by the locally installed `dx` at 0.7.7. The Mac's `dx` should match.

## Phase A: Linux: scaffold + Android

### A1. Scaffold the Dioxus project into this repo

Generate the Dioxus 0.7 hello-world template into the current directory while preserving `.git`, `LICENSE`, `.gitignore`, `README.md`.

Approach: generate in a temp directory with `dx new`, then move the generated files into this repo. Commit the scaffold as a discrete commit so future diffs are legible.

**Deliverable**: scaffolded project committed, `cargo check` succeeds.

### A2. Desktop smoke test

```bash
dx serve --desktop
```

Confirms the scaffold is sane before we start wrestling with NDK paths.

**Deliverable**: desktop window opens with the hello-world UI.

### A3. Install Android SDK + NDK (headless)

Install via Google's `cmdline-tools` archive. Avoids the full Android Studio install (about 5 GB vs 20 GB and a GUI we don't need).

Components to install via `sdkmanager`:

- `platform-tools` (for `adb`)
- `platforms;android-34` (or current API level)
- `build-tools;34.0.0`
- `ndk;26.x.x` (latest 26 series)
- `emulator`
- `system-images;android-34;google_apis;x86_64`

Export to shell profile (choice of `~/.profile` vs `~/.bashrc` to be made at this step):

```bash
export ANDROID_HOME="$HOME/Android/sdk"
export ANDROID_SDK_ROOT="$ANDROID_HOME"
export NDK_HOME="$ANDROID_HOME/ndk/<version>"
export ANDROID_NDK_HOME="$NDK_HOME"
export PATH="$PATH:$ANDROID_HOME/platform-tools:$ANDROID_HOME/emulator:$ANDROID_HOME/cmdline-tools/latest/bin"
```

**Deliverable**: `adb --version`, `sdkmanager --list_installed`, and `ls "$NDK_HOME"` all succeed in a fresh shell.

### A4. Install Rust Android targets + create an AVD

```bash
rustup target add \
  aarch64-linux-android \
  armv7-linux-androideabi \
  i686-linux-android \
  x86_64-linux-android
```

Create an x86_64 AVD (emulates fastest on this Linux host since CPU is x86_64):

```bash
avdmanager create avd -n collection-test -k "system-images;android-34;google_apis;x86_64" -d pixel_7
```

**Deliverable**: `rustup target list --installed` shows the four Android targets. `emulator -list-avds` shows `collection-test`.

### A5. Run on the Android emulator

```bash
emulator -avd collection-test &
dx serve --android
```

**Deliverable**: hello-world UI renders inside the emulator window.

### A6. Run on the physical Android phone

**Manual user steps**:

1. On the phone: Settings -> About Phone -> tap "Build number" 7 times to unlock Developer Options.
2. Settings -> Developer Options -> enable "USB debugging".
3. Plug the phone into the Linux box via USB.
4. On the phone: tap "Allow" on the "Allow USB debugging?" prompt. Check "Always allow from this computer".

Verify with `adb devices`. The phone should appear with status `device`. Then:

```bash
dx serve --android
```

(With both an emulator and a physical device attached, `dx` and `adb` may need a target hint. Addressed at runtime.)

**Deliverable**: hello-world UI renders on the physical phone.

## Phase B: SSH to Mac + iOS Simulator

### B1. Configure SSH to the Mac

**Manual user input needed**:

- Mac's hostname or IP (and whether it's reachable from the Linux box's network, or needs a tunnel)
- SSH username on the Mac
- SSH key status: existing key reused, or generate a new one for this purpose

Add a `Host mac` entry to `~/.ssh/config`. Verify with `ssh mac 'uname -a; sw_vers'`.

**Deliverable**: `ssh mac` connects without password prompt.

### B2. Verify Xcode on the Mac

Over SSH:

```bash
ssh mac 'xcode-select -p; xcodebuild -version'
```

If Xcode is missing or stub-installed, the user must install it from the Mac App Store directly on the Mac. It cannot be installed over SSH (about 40 GB, GUI-gated). After install, accept the license:

```bash
ssh mac 'sudo xcodebuild -license accept'
```

Also ensure command-line tools are present:

```bash
ssh mac 'xcode-select --install || true'
```

**Deliverable**: `xcodebuild -version` reports a recent Xcode (16.x+ recommended).

### B3. Install Rust + iOS targets + dx on the Mac

```bash
ssh mac 'curl --proto "=https" --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y'
ssh mac 'source ~/.cargo/env && rustup target add aarch64-apple-ios aarch64-apple-ios-sim'
ssh mac 'source ~/.cargo/env && cargo install dioxus-cli --version 0.7.7 --locked'
```

**Deliverable**: `ssh mac 'dx --version'` reports `dioxus 0.7.7`.

### B4. Sync the repo to the Mac

Two options, decision deferred to this step:

- **Option 1, GitHub**: push this repo to GitHub, clone on the Mac. Clean and reproducible, but exposes the repo (private repo OK).
- **Option 2, rsync over SSH**: `rsync -av --exclude target ./ mac:~/collection-app/`. Stays local, no third party.

**Deliverable**: project tree present on the Mac under `~/collection-app/`.

### B5. iOS Simulator

```bash
ssh mac 'cd ~/collection-app && dx serve --ios'
```

The simulator window appears on the Mac's display. To verify the hello-world renders, the user needs visual access to the Mac at this step: physical screen, Screen Sharing (built-in macOS VNC), or another remote-desktop tool.

**Deliverable**: hello-world UI renders in the iOS Simulator on the Mac.

## Phase C: Physical iPhone + iPad

### C1. Sign into Xcode with Apple ID (one-time, manual at the Mac)

On the Mac: Xcode -> Settings -> Accounts -> "+" -> Apple ID -> sign in.

Cannot be automated. Apple's auth flow is GUI-only and may trigger 2FA on another device.

After signing in, note the **Team ID** (visible under the account, "Manage Certificates", or via `security find-identity -v -p codesigning` over SSH).

**Deliverable**: an "Apple Development" signing certificate exists in Keychain on the Mac.

### C2. Pair iPhone with the Mac (one-time, manual)

1. Plug iPhone into the Mac via USB.
2. On the iPhone: tap "Trust this computer", enter passcode.
3. iOS 16+: Settings -> Privacy & Security -> Developer Mode -> on. Phone reboots. Confirm "Turn On" after reboot.
4. In Xcode: Window -> Devices and Simulators -> confirm iPhone appears as a "Connected" development device.

**Deliverable**: `xcrun devicectl list devices` (over SSH) shows the iPhone.

### C3. Configure iOS signing for the project

Edit the project's iOS configuration (Dioxus generates either an Xcode project or `Info.plist` + entitlements depending on template version). Set:

- A unique bundle identifier in reverse-DNS form, e.g. `com.example.collectionapp`. Must be Java-valid (no hyphens) for Android compatibility.
- Development team to the Team ID from C1
- Provisioning style: automatic (Xcode manages free-tier provisioning profiles)

These edits are file-based and can be done over SSH from the Linux box once the team ID is known.

**Deliverable**: `xcodebuild -showBuildSettings` reports the expected `DEVELOPMENT_TEAM` and `PRODUCT_BUNDLE_IDENTIFIER`.

### C4. Deploy hello-world to iPhone

```bash
ssh mac 'cd ~/collection-app && dx serve --ios --device'
```

(Exact flag depends on `dx` 0.7. `--device` vs `--target=device` to be verified.)

**Deliverable**: hello-world launches on the physical iPhone. The first launch may require the user to: Settings -> General -> VPN & Device Management -> trust the developer certificate.

**Caveat**: free-tier signing means the app stops launching after 7 days. To use it again, re-run this step from the Mac.

### C5. Repeat for iPad

Same procedure as C2 through C4 with the iPad. Note: free tier allows up to about 3 active provisioning profiles total across devices.

**Deliverable**: hello-world launches on the physical iPad.

## Phase D: Workflow documentation

### D1. Add a development workflow doc

Capture the per-platform commands and the recurring manual steps (Android USB debug toggle, weekly iOS re-signing) so neither future-you nor future-me has to rediscover them. Likely a `DEVELOPMENT.md` or expanded section in `README.md`.

### D2. Final commit

Single commit closing out the milestone, tag optional.

## Known friction points

- **Disk usage**: Android SDK + NDK + 1 system image is about 10 to 20 GB on Linux. Xcode is about 40 GB on the Mac. Confirm Mac free space before B2.
- **Apple free tier**: 7-day app expiration on devices is the single biggest ongoing annoyance. The remedy is the paid Developer Program ($99/yr) which unlocks TestFlight and longer-lived signing.
- **Mac display access**: iOS Simulator (B5) and Xcode GUI steps (C1, C2) require visual access to the Mac. Plan for Screen Sharing or physical access at those steps.
- **Network reachability**: SSH to the Mac assumes the Linux box can reach it. If they're on different networks, plan for either a VPN or Tailscale-style mesh. To be addressed at B1 if it comes up.
- **Dioxus version drift**: pin `dx` to 0.7.7 on both machines. Mismatched versions across hosts can produce subtly different builds.

## Manual user steps summary

These items require the user. I cannot script them.

| Step | Where | What |
|------|-------|------|
| A3 | Linux | Pick shell profile target for env vars (`~/.profile` vs `~/.bashrc`) |
| A6 | Phone | Enable Developer Options + USB debugging. Plug into Linux. |
| B1 | (any) | Provide Mac hostname/user/SSH key info |
| B2 | Mac | Install Xcode from the App Store (if not already) |
| B4 | (any) | Decide GitHub vs rsync for repo sync |
| B5 | Mac | Have visual access to the Mac's display |
| C1 | Mac | Sign into Xcode with Apple ID (GUI) |
| C2 | Mac + iPhone | Plug iPhone into Mac, trust computer, enable Developer Mode |
| C4 | iPhone | Trust developer certificate on first launch |
| C5 | Mac + iPad | Repeat C2/C4 for the iPad |
| weekly | Mac | Re-run `dx serve --ios --device` to refresh expired signing |
