# sing-box Android build

This repository builds signed SFA APKs from version-matched upstream sources.
The libbox and Android application choices are both explicit in `config.env`.

## Version model

`config.env` contains the only manually maintained upstream version:
`SING_BOX_TAG`.

- The Android client comes from its `main` branch and must declare the same
  version as `SING_BOX_TAG`. This avoids stale submodule pointers while refusing
  to combine different released versions.
- The sing-tun version comes from that tag's `go.mod`.
- The gomobile version also comes from that tag's `go.mod`.
- The Android NDK release comes from the pinned sing-box build workflow.
- The Go toolchain comes from the pinned sing-box `go.mod`.

Updating sing-box therefore requires changing only `SING_BOX_TAG`.

When run locally, the build obtains a clean upstream tree and keeps the source,
Go caches, Gomobile installation, and Gradle cache under
`${BUILD_TMPDIR:-/dev/shm}`. The temporary tree is removed on exit; only
`dist` persists. GitHub Actions continues to use its runner and managed caches.

## Build scope

The libbox build and non-legacy Android flavors follow the API selected by
upstream's main libbox variant. The build targets every Android architecture,
and its tags are deliberately smaller than the upstream all-feature Android
defaults:

- `with_gvisor` keeps the mixed TUN stack used when the current configuration
  does not specify `stack`;
- `with_quic` provides Hysteria, Hysteria 2, and TUIC;
- `with_utls` provides uTLS and is required by Reality clients;
- `badlinkname` and `tfogo_checklinkname0` are the upstream compatibility tags.

The build intentionally excludes the optional NaiveProxy, WireGuard, Clash
API, Tailscale, OpenVPN, OpenConnect, DHCP, ACME, Cloudflared, and USB/IP
subsystems. Standard gRPC is also excluded: sing-box automatically uses its
built-in gRPC-lite transport when `with_grpc` is absent. Protocols registered
without optional build tags remain available. Adding `with_tailscale`,
`with_openvpn`, `with_openconnect`, or `with_usbip` to `LIBBOX_TAGS` also
selects that feature's Android source set; absent features and their UI are not
compiled. Ghostty dependencies are likewise limited to Tailscale builds.

The SFA build uses the `otherRelease` variant: `other` selects the GitHub
distribution rather than Google Play or legacy Android, while `release`
enables minification and release signing. It produces the client's standard
four ABI splits and universal APK. Updates always use stable GitHub Releases
from the current Actions repository, or `SFA_UPDATE_REPOSITORY` for a local
build; source and release-channel selectors are removed.

Every successful GitHub Actions build publishes the generated files directly
to the Release named by `SING_BOX_TAG`. Rebuilding the same version replaces
its assets instead of creating another Release. The build reads both metadata
values from the selected Android client and generates this file automatically:

```json
{
  "version_code": 730,
  "version_name": "1.14.0"
}
```

The platform command backend does not require `with_clash_api`: SFA still gets
logs, status, connection statistics, and outbound groups without exposing a
Clash API server. If a configuration explicitly requests Clash API while the
tag is absent, sing-box retains its normal missing-feature error.

The GitHub Release contains:

- the signed ABI-specific and universal APKs;
- `libbox.aar`;
- `SHA256SUMS`.

## Signing

Create one keystore and retain it permanently. Configure these GitHub Actions
secrets:

- `ANDROID_KEYSTORE_BASE64`: base64 of the keystore;
- `ANDROID_LOCAL_PROPERTIES`: base64 of a properties file containing:

```properties
KEYSTORE_PASS=...
ALIAS_NAME=...
ALIAS_PASS=...
```

The Android client decodes `ANDROID_LOCAL_PROPERTIES` itself. The signing
identity must remain unchanged for Android to accept later APKs as updates.

## Patches

Maintain each logical fix as a commit on the corresponding source repository's
`downstream` branch, then export it with `git format-patch`. Patch files are
derived artifacts and should not be edited manually. They are applied in
lexical order:

```text
patches/sing-box/0001-description.patch
patches/android/0001-description.patch
patches/modules/github.com/sagernet/sing-tun/0001-description.patch
```

For example:

```sh
git format-patch -1 <commit> --output-directory \
  /path/to/patches/modules/github.com/sagernet/sing-tun
```

Go module patch directories use their complete module paths. The build
automatically copies every patched module into temporary memory, applies its
patch series, and adds the corresponding `go mod replace`; adding another
module therefore requires no build-script change.

The sing-tun patch changes the system stack's Android TCP forwarders to listen
on the dual-stack wildcard address. This avoids binding them to a TUN address
shared by separate per-user VPN instances; UDP is unaffected because the mixed
stack handles it through gVisor instead of these listeners. Other platforms
retain sing-tun's address-specific listeners.

If an upstream update makes a patch inapplicable, source preparation fails
instead of silently building without it.
