# RDLink

**A self-hosted remote desktop client for HarmonyOS NEXT and Windows, forked from RustDesk 1.4.9.**

![Licence: AGPL-3.0](https://img.shields.io/badge/licence-AGPL--3.0-blue)
![Platform: Android arm64](https://img.shields.io/badge/platform-Android%20arm64-3ddc84)
![Platform: Windows](https://img.shields.io/badge/platform-Windows-0078d4)

> [!WARNING]
> **Unofficial fork.** RDLink is not affiliated with, endorsed by, or supported by the RustDesk project. It is a rebranded derivative of RustDesk 1.4.9, distributed under the same licence (AGPL-3.0). Report RDLink-specific issues here; report upstream issues to [rustdesk/rustdesk](https://github.com/rustdesk/rustdesk).

> [!CAUTION]
> **Misuse disclaimer.** Neither the RDLink nor the RustDesk authors condone or support any unethical or illegal use of this software. Unauthorised access to, control of, or invasion of another person's devices or privacy is strictly prohibited. The authors accept no responsibility for misuse of this application.

---

## What is RDLink

RDLink is a remote desktop application built on **RustDesk 1.4.9** and released under the **AGPL-3.0** licence. It provides two kinds of endpoint that control each other over a network you own:

- an **Android client**, packaged specifically to run on **HarmonyOS NEXT** devices through the Zhuoyitong (卓易通) Android compatibility layer;
- a **Windows desktop client**, usable either as a controller or as an always-on agent that accepts inbound connections unattended.

RDLink is *self-hosted by design*. Upstream RustDesk defaults to a public cloud service; RDLink expects you to run your own **rendezvous (hbbs)** and **relay (hbbr)** servers and to authenticate peers with a server key you generate. There is no vendor account, no third-party telemetry and no dependency on anyone else's infrastructure. This repository ships neutral — it contains no server address, no key and no credentials — so every deployment is configured locally. See [Server setup](#server-setup).

## Features

**Remote control**
- Bidirectional screen control between Android (HarmonyOS NEXT) and Windows endpoints, over the network you operate.
- Direct peer-to-peer transport is preferred; a relay server is used automatically as a fallback when a direct path cannot be established.
- The inherited RustDesk capability set is retained: screen streaming with adjustable quality, remote keyboard and pointer/touch input, multi-monitor support on the Windows side, and clipboard and audio modes as supported by the target platform.

**Self-hosted infrastructure**
- Works against your own `hbbs` / `hbbr` deployment; a single server can serve the whole fleet.
- Server-key authentication: a client must present the key published by your server, so unauthenticated peers cannot register.
- Intranet rendezvous with public relay is a supported topology — endpoints can register internally while remote peers still reach them.

**Unattended operation**
- Permanent (fixed) password support for hands-free reconnection, so no one has to read a one-time password off the remote screen.
- The Windows agent runs as a system service: it starts at boot and stays reachable even when no user is logged in.
- Inbound firewall rules and a pinned server address make reconnection deterministic after a reboot.

**Open source**
- Full source under AGPL-3.0, including build scripts and packaging configuration.
- No hardcoded server information, no bundled credentials, and no proprietary binary blobs beyond the upstream RustDesk dependencies.

## Scope and limitations

- **HarmonyOS NEXT devices act as controllers only.** The Android/HarmonyOS endpoint can control a Windows machine, but a Windows machine cannot take control of a HarmonyOS NEXT phone: the platform does not permit an application to inject input. This is an operating-system restriction, not a build defect.
- **You must run a server.** RDLink has no bundled fallback to a public rendezvous service. Without a reachable `hbbs`/`hbbr` instance, clients cannot discover each other.
- **Security is your responsibility.** The server key and the unattended password are managed on your side. Treat both as credentials and keep them out of source control.
- **Rebranding status.** Application name, package identity and user-facing strings are rebranded to RDLink. The launcher icon assets are still the upstream artwork — supply your own in `flutter/android/app/src/main/res/mipmap-*/` if you redistribute a build.
- Documentation under `docs/` is inherited from upstream RustDesk and may describe features or upstream services that this fork does not provide.

---

## Build

### Prerequisites

| Component | Notes |
|---|---|
| Rust toolchain | Stable, via [rustup](https://rustup.rs). `cargo` **must be on `PATH`** — see the note below. |
| `cargo-ndk` | `cargo install cargo-ndk` |
| Android NDK | r25 or newer; export `ANDROID_NDK_HOME` |
| Android SDK | Export `ANDROID_HOME` |
| Flutter SDK | Stable channel; this project declares Dart `^3.1.0` |
| JDK | 17 (Gradle 7.6.4 with AGP 7.3.1; Java 8 source/target compatibility) |

Clone with the submodule, or the Rust build will fail:

```sh
git clone --recurse-submodules https://github.com/heyoutu-git/rdlink.git
cd rdlink
```

### 1. Build the Rust core for Android arm64

```sh
cd flutter
./ndk_arm64.sh
```

The script is a single `cargo-ndk` invocation; the equivalent command is:

```sh
cargo ndk --platform 21 --target aarch64-linux-android build --locked --release --features flutter,hwcodec
```

Make sure the resulting `librustdesk.so` ends up in `flutter/android/app/src/main/jniLibs/arm64-v8a/` (that directory is a build output and is not tracked; `cargo-ndk` writes to `jniLibs` by default, so copy it if your working directory differs).

### 2. Bundle the C++ runtime

`librustdesk.so` links against the NDK's shared C++ runtime. It is **not** pulled in automatically, and the app crashes on launch if it is missing. Copy it next to the Rust library:

```sh
cp "$ANDROID_NDK_HOME/toolchains/llvm/prebuilt/<host-tag>/sysroot/usr/lib/aarch64-linux-android/libc++_shared.so" \
   flutter/android/app/src/main/jniLibs/arm64-v8a/
```

`<host-tag>` is `windows-x86_64`, `linux-x86_64` or `darwin-x86_64` depending on your build machine.

### 3. Build the APK

```sh
cd flutter
flutter build apk --release --target-platform android-arm64
```

Result: `flutter/build/app/outputs/flutter-apk/app-release.apk`

### Troubleshooting

- **`rustls-platform-verifier-android package not found in cargo metadata`** — Gradle runs `cargo metadata` while configuring the build. If `cargo` is not on `PATH` (or `CARGO_HOME` / `RUSTUP_HOME` are wrong), the build stops here. Add the Rust toolchain's `bin` directory to `PATH`.
- **Application exits immediately on start** — almost always the missing `libc++_shared.so` from step 2. Verify both libraries sit side by side in `jniLibs/arm64-v8a/`.
- **`did not receive expected object` when pushing** — this repository's history may be shallow if you cloned with `--depth`. Fetch the full history before pushing, or publish from a squashed root commit.
- **Deploying to HarmonyOS NEXT** — install the arm64 APK inside the Zhuoyitong (卓易通) Android container. No separate HarmonyOS build is required.

---

## Server setup

RDLink talks to [rustdesk-server](https://github.com/rustdesk/rustdesk-server) (the open-source `hbbs` / `hbbr` pair). Any host that the clients can reach — a VPS, a home server or an internal VM — will do. A stable address (static IP or DNS name) is recommended so clients do not need reconfiguration.

### Ports

| Port | Protocol | Component | Purpose | Required |
|---|---|---|---|---|
| 21114 | TCP | hbbs | Web console (Pro edition only) | no |
| 21115 | TCP | hbbs | NAT type test | **yes** |
| 21116 | TCP | hbbs | TCP hole punching and connection service | **yes** |
| 21116 | UDP | hbbs | ID registration and heartbeat | **yes** |
| 21117 | TCP | hbbr | Relay service | **yes** |
| 21118 | TCP | hbbs | Web client (WebSocket) | no |
| 21119 | TCP | hbbr | Web client (WebSocket) | no |

Ports **21115/TCP, 21116/TCP, 21116/UDP and 21117/TCP are the minimum**. Note that 21116 must be open for *both* TCP and UDP — clients cannot register without UDP.

### Option A — Docker

```sh
docker image pull rustdesk/rustdesk-server
docker run --name hbbs -v ./data:/root -td --net=host --restart unless-stopped rustdesk/rustdesk-server hbbs
docker run --name hbbr -v ./data:/root -td --net=host --restart unless-stopped rustdesk/rustdesk-server hbbr
```

`--net=host` is Linux-only, and it lets the servers observe real client IP addresses instead of container IPs. On other platforms drop `--net=host` and publish the ports explicitly with `-p`.

### Option B — native binaries

Download the `hbbs` and `hbbr` binaries from the rustdesk-server releases, then run each as a long-lived service: a systemd unit on Linux, or a service wrapper such as NSSM on Windows (set both to start automatically). Keep a persistent working directory — the server writes its key pair there.

### Get the server key

On first start, `hbbs` generates a key pair in its working directory:

- `id_ed25519` — the private key. **Keep it on the server; never publish or commit it.**
- `id_ed25519.pub` — the public key. Its **contents** are what clients use as the `Key`.

For a Docker deployment the pair lands in the mounted `./data` directory. For a service-based deployment, check the account the service runs under — the file is written to *that* account's profile or working directory, which is the most common reason a key "cannot be found".

### Point clients at your server

On **every** endpoint — controller and controlled alike — open **Settings → Network → ID/Relay Server** and set:

| Field | Value |
|---|---|
| ID Server | `<your-server-host>`, optionally `<your-server-host>:21116` |
| Relay Server | leave empty to derive it from the ID server, or set `<your-server-host>:21117` explicitly |
| Key | the contents of `id_ed25519.pub` |

The same values live in `RustDesk2.toml` as `custom-rendezvous-server`, `relay-server` and `key`. If a client registers with the wrong server address, delete stale values rather than adding a second one — a leftover `rendezvous_server` entry silently takes precedence.

For unattended access, set a **permanent password** on the controlled endpoint and enable unattended access. On Windows, also allow the agent through the firewall for inbound connections.

### Security notes

- **Keep 21118 and 21119 closed unless you actually use the web client.** When they are open, `hbbs`/`hbbr` trust the `X-Real-IP` / `X-Forwarded-For` headers of incoming WebSocket connections without validating them, so anyone able to reach those ports directly can spoof an arbitrary IP address — bypassing IP-based rate limiting and blocking, and falsifying the addresses recorded in logs. If you need the web client, expose those ports only through a reverse proxy that sets the headers itself, and firewall them so only the proxy can connect.
- Expose the server to the public internet only over the ports listed above; front everything else with a firewall.
- The public key identifies your server but is not a secret. The private key is — treat it like any other server credential.
- Never commit server addresses, keys or passwords into this repository. Keep deployment configuration local.

---

## Licence

RDLink is distributed under the **GNU Affero General Public License, version 3** — see [LICENCE](LICENCE), retained verbatim from upstream.

- **Upstream:** this is a fork of [rustdesk/rustdesk](https://github.com/rustdesk/rustdesk) 1.4.9 (upstream commit `6c57829`). All RustDesk copyright notices and the AGPL-3.0 licence are preserved.
- **Modifications:** application rebranding to RDLink (application label, Android package ID `com.rdlink.remote`, user-facing strings), build configuration adjustments, repository cleanup, and this documentation.
- **Network distribution:** because AGPL-3.0 covers software offered over a network, if you run a modified RDLink as a network service you must offer the corresponding source to the users of that service.
- **Third-party components:** bundled dependencies of the upstream project remain under their own licences — refer to `Cargo.lock`, `vcpkg.json` and the Flutter plugin manifests.

Upstream documentation, the wiki and the FAQ at [rustdesk.com](https://rustdesk.com) apply to the unmodified project and are useful background, but they are not maintained by this fork.
