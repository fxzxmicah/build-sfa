# Android Patches

Apply the numbered patches in lexical order. Maintain source changes as Git
commits and regenerate patches with `git format-patch`, rather than editing
their diff contents. This README documents all patches in this directory.

## 0001: Build Repository Updates

Adds `SFA_UPDATE_REPOSITORY` as the application's GitHub update repository.
The build supplies the destination repository instead of using upstream's
release assets. Updates use stable GitHub Releases only. F-Droid update and
mirror support, source and release-channel selectors, and their stored settings
are removed together with the corresponding navigation and update-check paths.
This includes the legacy variant and unused translations. APK selection follows
the device's supported ABIs, then the universal APK, rather than release-asset
ordering.

## 0002: Libbox Feature Alignment

Selects Android source sets and dependencies from `LIBBOX_TAGS`, so unavailable
Clash API, Tailscale, OpenVPN, OpenConnect, and USB/IP features are not compiled
into the UI. Ghostty dependencies follow Tailscale support.

Release resource shrinking removes resources left unused by feature selection.
Enabled modules retain localized entries, navigation transitions, unread/error
badges, and Tailscale's long-press SSH shortcuts. Taildrop deep links belong to
the Tailscale source set, so a build without it cannot navigate to its removed
screens. One subscription lifecycle switches targets and closes subscriptions
when the owning composition is disposed. Dashboard defaults have one source
of truth, filtered by the compiled feature set.

## 0003: Lockdown Integration

This patch adds SFA-specific Always-on VPN and Lockdown status, configuration
feedback, and an opt-in workaround for local incoming connections.

### Status and Settings

- The service queries `VpnService.isAlwaysOn` and `isLockdownEnabled` through
  its own context. Both must be true. Another app's VPN is not reported as SFA.
- MainActivity binds explicitly to this app's VPNService to query system policy
  and provides one shared Compose state to its screens. This binding is separate
  from the selected runtime service: it does not depend on `Settings.serviceMode`,
  a selected profile, or a running core. Binding creates the service object but
  does not invoke `onStartCommand`, start a foreground service, or establish a
  VPN interface. It does not register runtime-status callbacks.
  A dedicated local `PolicyBinder` is returned for `Action.VPN_POLICY`; policy
  queries do not extend the runtime `IService` interface or involve ProxyService.
- Policy is read when the binding connects and whenever the activity resumes.
  Stopping or switching the runtime service does not clear policy state. The
  policy binding is released when the activity is destroyed. An unbound query
  is unknown, not proof of disabled Lockdown. The app and services currently run
  in the same process; the added getter does not catch hypothetical
  remote-process errors. Screens do not create their own connections. Remote
  dashboards do not display the local indicator.
- Callback registration and removal follow normal service binding and
  unbinding, including reconnects, so a previous binding does not retain the UI.
- The dashboard lock icon opens a short status dialog, not system settings.
  Enterprise policy may prevent changing those settings.
- Dashboard item configuration has its own toolbar button. The remaining
  overflow menu is retained only for remote control.
- Bypass and per-app filtering cannot be edited while this VPN is locked down.
  Disabled controls use their normal disabled state without an explanation of
  why they are disabled. Related settings use the existing ROOT-settings
  convention: dim the entire row to 0.38 alpha and disable the interactive
  control. Both VPN switches use the same description/link layout;
  links use Compose's `LinkAnnotation` rather than manual character-offset
  handling.
  the local-ingress option briefly states that it relaxes ingress restrictions
  and links to AOSP source, not a nonexistent dedicated Android API document.
  The local-ingress switch stays visible on the Service
  settings page and is enabled only on Android 13+ while SFA is Always-on with
  Lockdown, matching the supported route submission path.
  Saved preferences and application lists are retained, not silently rewritten.
  This UI treats per-app filtering as a direct-access feature, not an application
  firewall. Existing profile-level application restrictions remain authored
  configuration and are not removed.
- Failed `protect(fd)` calls propagate an error instead of continuing with an
  unprotected socket. This correction also applies outside Lockdown.

### Default Route Option

`lockdown_split_default_route` defaults to false. When enabled, it replaces
IPv4 `0.0.0.0/0` with `0.0.0.0/1` and `128.0.0.0/1` only when SFA itself is
the Always-on VPN with Lockdown enabled. Both an automatic default route and
an explicitly configured `/0` pass through this same transformation.

On Android 13+, `VPNService.openTun` first selects the IPv4 routes using its
normal explicit-route and fallback behavior, then transforms any `/0` while
submitting routes to Android. It does not add a default route where the normal
selection would produce none. Other prefixes, route exclusions, IPv6 routes,
and the older Android route-range path are unchanged.
An exact-prefix exclusion is resolved before expanding included routes; in
particular, excluding `/0` must not leave newly generated `/1` routes active.
Exclusions are then submitted after the included IPv4 routes so more-specific
exclusions retain their original precedence.
Outside Lockdown the preference is retained but inactive. Changing the
preference uses the existing reload prompt and requires rebuilding the VPN
interface to take effect.

The IPv4 destination union is unchanged. However, AOSP Android 16's
`ConnectivityService.getVpnIsolationInterface` checks for literal IPv4 and IPv6
default routes, not equivalent prefix coverage. Without a literal IPv4 `/0`,
the VPN is assigned a wildcard incoming-interface allowance. The eBPF UID rule
then permits ingress on interfaces other than the VPN, subject to remaining
system filtering. This affects IPv6 ingress too, despite unchanged IPv6 routes.

This does not disable Lockdown's separate outgoing restrictions, but it does
relax incoming isolation for VPN-managed apps across all interfaces. It is not
a Wi-Fi Direct-only exception or a guarantee that every local service becomes
reachable. OEM changes or future route normalization may invalidate the
workaround. Validate local ingress, ordinary outgoing traffic, VPN teardown,
interface rebuilds, and network switches on target devices.

References:

- https://android.googlesource.com/platform/packages/modules/Connectivity/+/android16-release/service/src/com/android/server/ConnectivityService.java
- https://android.googlesource.com/platform/packages/modules/Connectivity/+/android16-release/service/src/com/android/server/BpfNetMaps.java
- https://android.googlesource.com/platform/packages/modules/Connectivity/+/android16-release/bpf/progs/netd.c

The patch is exported from the Android source repository's downstream commit
with `git format-patch`; this document is maintained separately.
