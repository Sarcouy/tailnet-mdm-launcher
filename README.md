# tailnet-mdm-launcher

[![build](https://github.com/Sarcouy/tailnet-mdm-launcher/actions/workflows/build.yml/badge.svg)](https://github.com/Sarcouy/tailnet-mdm-launcher/actions/workflows/build.yml)
[![lint](https://github.com/Sarcouy/tailnet-mdm-launcher/actions/workflows/lint.yml/badge.svg)](https://github.com/Sarcouy/tailnet-mdm-launcher/actions/workflows/lint.yml)

A fork of [Headwind MDM](https://h-mdm.com)'s Android launcher
([`h-mdm/hmdm-android`](https://github.com/h-mdm/hmdm-android)) that adds one thing the stock
launcher cannot do: an **OS-level always-on VPN policy**. As the device owner, the launcher forces
a VPN app (e.g. Tailscale) on, so the tunnel comes up on every boot and — with lockdown enabled —
cannot be switched off by the user.

## Why

Headwind can install the VPN app and push user restrictions (`no_config_vpn`, …), but it never calls
`DevicePolicyManager.setAlwaysOnVpnPackage`. Without it the tunnel does not survive a reboot and the
user can disconnect at will; the restrictions only lock the settings UI. `setAlwaysOnVpnPackage`
requires being the **device owner**, and the launcher is the only device owner on an enrolled phone —
so the capability has to live inside the launcher itself.

`setAlwaysOnVpnPackage` is persistent: set once, the OS re-establishes the tunnel on every boot and
enforces the kill switch, so no boot receiver or watchdog is needed to keep it running.

## How it works

The policy is driven entirely from the panel, with **no server-side change**, by reusing the same
channel as the built-in proxy feature: *Application settings* pushed to the launcher's own package
(`com.hmdm.launcher`). On every configuration sync, `ConfigUpdater.updatePolicies()` reads these keys
and calls `Utils.setAlwaysOnVpn(...)`.

| Application setting (on the launcher package) | Effect |
| --- | --- |
| `always_on_vpn` | VPN package to force, e.g. `com.tailscale.ipn`. `0` or empty clears the policy. |
| `always_on_vpn_lockdown` | `false` disables the kill switch. Default: **enabled**. |
| `always_on_vpn_allowlist` | Comma-separated packages allowed to reach the network while the VPN is down. Default: **empty** (opt-in). |

The target VPN app must support always-on (it must not opt out via the
`android.net.VpnService.SUPPORTS_ALWAYS_ON` service meta-data). Tailscale does support it.

## Scope

This works only on phones where **this** launcher is the device owner, which is set at provisioning
time (`dpm set-device-owner` or QR enrolment) on a factory-reset device. It cannot be swapped onto a
phone already enrolled with the upstream launcher without a factory reset.

## Build

Requires JDK 17. The Android SDK is provisioned automatically in CI.

```bash
./gradlew assembleOpensourceDebug
```

Server URL, enrolment secret and signing credentials are **injected at build time** and never
committed. Provide them as Gradle properties (in `~/.gradle/gradle.properties`, via `-P`, or as
environment variables):

| Property / env var | Purpose |
| --- | --- |
| `MDM_BASE_URL` | Base URL of your Headwind server (scheme + host). |
| `MDM_SECONDARY_BASE_URL` | Fallback URL. Defaults to `MDM_BASE_URL`. |
| `MDM_REQUEST_SIGNATURE` | Shared secret expected by the server (`secure.enrollment`). |
| `MDM_LIBRARY_API_KEY` | API key for privileged library requests. |
| `RELEASE_STORE_FILE` | Path to your signing keystore. |
| `RELEASE_STORE_PASSWORD` / `RELEASE_KEY_ALIAS` / `RELEASE_KEY_PASSWORD` | Signing credentials. |

When `RELEASE_STORE_FILE` is absent the release build stays unsigned, so the project still configures
for contributors and CI without the key.

```bash
./gradlew assembleOpensourceRelease \
  -PMDM_BASE_URL=https://mdm.example.com \
  -PRELEASE_STORE_FILE=/path/to/release.jks
```

> **Signing matters.** A forked launcher is signed with your own key and becomes the device owner of
> every phone you provision with it — its signature is the trust anchor of the fleet. Keep the
> keystore safe and out of the repository.

## Relationship to upstream

This repository tracks `h-mdm/hmdm-android` as the `upstream` remote and keeps the change set minimal
so upstream updates can be merged. The always-on VPN feature lives in
[`Utils.setAlwaysOnVpn`](app/src/main/java/com/hmdm/launcher/util/Utils.java) and one call in
[`ConfigUpdater.updatePolicies`](app/src/main/java/com/hmdm/launcher/helper/ConfigUpdater.java).

## License

Apache License 2.0, inherited from the upstream project. See [LICENSE](LICENSE) and [NOTICE](NOTICE).
Headwind MDM is a product of Headwind Solutions LLC; this is an independent fork and is not affiliated
with or endorsed by them.
