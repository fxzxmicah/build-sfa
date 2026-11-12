# sing-box for Android Build

Build signed sing-box for Android APKs with selected libbox features and
downstream patches. Run `./build` locally or use GitHub Actions to build and
publish a Release.

## Configuration

Edit `config.env` to select the upstream tag, libbox tags, Android flavor, and
build type. `SING_BOX_TAG` is the only manually maintained version.

The Android client comes from upstream `main` and must declare the same version
as the selected sing-box tag. Go modules and Gomobile follow that tag's
`go.mod`.

Android `main` can change without a version bump. Its patches must be rebased
when upstream changes; matching version names alone do not pin the source.

The default `otherRelease` variant uses GitHub updates, release signing and
minification, and builds all upstream ABI splits and the universal APK.
The Android API level follows upstream's main libbox build variant.

Default libbox tags retain gVisor, QUIC protocols, uTLS, and upstream
compatibility tags. Without `with_grpc`, sing-box uses gRPC-lite.
Other optional subsystems require their corresponding build tags.

Android source sets follow `with_clash_api`, `with_tailscale`,
`with_openvpn`, `with_openconnect`, and `with_usbip`. Absent features and their
UI are not compiled; Ghostty dependencies are included only with Tailscale.

## Local Build

Prepare a Linux build environment with Git, Go, Java 17, the Android SDK/NDK,
and standard GNU command-line tools. Use the Go version from the selected
upstream tag's `go.mod` and the NDK release from
`.github/workflows/build.yml` in that tag. Gradle uses the Android client's
wrapper; no separate Gradle installation is needed.

Place your keystore at `release.keystore` in this repository and create
`local.properties` alongside it:

```properties
KEYSTORE_PASS=...
ALIAS_NAME=...
ALIAS_PASS=...
```

Then run:

```sh
export ANDROID_HOME=/absolute/path/to/android-sdk
export ANDROID_NDK_HOME=/absolute/path/to/android-ndk
export SFA_UPDATE_REPOSITORY=owner/repository
./build
```

The script downloads fresh sources, applies patches, and builds all outputs.
Retain the signing identity for future APK updates.

Optional environment settings:

- `ANDROID_KEYSTORE`: absolute keystore path instead of `release.keystore`.
- `LOCAL_PROPERTIES`: base64-encoded signing properties, read directly by the
  Android client in preference to file properties.
- `OUTPUT_DIR`: absolute output directory; default is `dist/` in this repository.
- `BUILD_TMP`: existing temporary directory parent; default is `/dev/shm`.
- `GOCACHE`, `GOMODCACHE`, `GOPATH`, `GRADLE_USER_HOME`: existing cache/tool
  locations to reuse instead of temporary locations.

Temporary sources, module copies, and default caches are removed on exit;
externally configured caches are retained. Provide enough temporary space for
all-architecture builds, on a filesystem that permits executable files.

## GitHub Actions

Configure these repository secrets:

- `ANDROID_KEYSTORE_BASE64`: base64-encoded release keystore.
- `ANDROID_LOCAL_PROPERTIES`: base64-encoded signing properties shown above.

Actions prepares toolchains, caches, and signing, then runs `build` once.

Successful builds publish directly to the Release named by `SING_BOX_TAG`.
Rebuilding a version replaces its assets. App updates use stable GitHub
Releases from `SFA_UPDATE_REPOSITORY`, set to the current repository in Actions.

## Outputs

`dist/` contains:

- Signed ABI-specific and universal APKs.
- `libbox.aar`.
- `SFA-version-metadata.json`, generated from the Android client's version.
- `SHA256SUMS` with relative filenames.

## Patches

Maintain logical changes as commits on the source repository's `downstream`
branch and export them with `git format-patch`. Do not hand-edit exported
patches. Each directory's patches apply in lexical order:

```text
patches/sing-box/*.patch
patches/android/*.patch
patches/modules/<full-go-module-path>/*.patch
```

Individual component directories are optional. Patched Go modules are copied
to writable temporary directories and selected through `go mod replace`.
When updating the upstream tag, rebase affected downstream commits and
re-export their patches. Version mismatches and patch failures stop the build.
