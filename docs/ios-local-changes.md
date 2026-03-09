# Local iOS Configuration Notes

This note documents the current local, unstaged iOS-specific changes in the
workspace as of March 9, 2026.

It is based on the diff between `HEAD` and the current local files under
`ios/`. The goal is to preserve context before any of these changes are
committed, reverted, or reapplied later.

## Summary

The local iOS changes fall into four groups:

1. Signing and bundle identifier rewrites in the Xcode project.
2. App Group removal from app, share extension, and widget entitlements.
3. Share extension plist cleanup plus display-name changes.
4. A CocoaPods lockfile update that adds `path_provider_foundation`.

Some of these are clearly machine- or developer-specific. Others look risky and
should be reviewed before they are committed as product behavior.

## File-by-file notes

### `ios/Runner.xcodeproj/project.pbxproj`

Observed local changes:

- `DEVELOPMENT_TEAM` changes from `X2662V5DT2` to `AGV53XD2D4`.
- Main app, test target, widget extension, and share extension bundle IDs are
  rewritten from `app.cogwheel.conduit...` to `app.ra-develop.conduit...`.
- `EXCLUDED_ARCHS[sdk=iphonesimulator*] = x86_64` is added for several targets.
- `INFOPLIST_KEY_CFBundleDisplayName` for `ShareExtension` changes from
  `ShareExtension` to `Ask Conduit`.
- macOS code signing identity entries are added for the test target.

Interpretation:

- The team and bundle ID rewrites are local signing changes for this machine
  and Apple developer account.
- The simulator architecture exclusion is likely a workaround for arm64-only
  Flutter engine or pod linkage behavior on Apple Silicon.
- The share extension display-name change is user-facing and could be worth
  keeping if intentional.

Risk:

- The team ID and bundle IDs are not portable across contributors and should be
  treated as local environment overrides unless the project is intentionally
  being rebranded or moved to a new Apple account.

### `ios/Runner/Info.plist`

Observed local changes:

- `AppGroupId` is moved near the top of the plist, but the value remains
  `group.app.cogwheel.conduit`.

Interpretation:

- This is effectively a key reorder, not a functional value change.

Risk:

- None by itself.

### `ios/Runner/Runner.entitlements`

Observed local changes:

- `com.apple.security.application-groups` changes from:
  `group.app.cogwheel.conduit`
  to an empty array.

Interpretation:

- The main app no longer declares membership in the shared App Group.

Risk:

- High. This can break shared storage and data exchange with the widget and
  share extension.

### `ios/ShareExtension/Info.plist`

Observed local changes:

- The file is reformatted and comment blocks are removed.
- `AppGroupId` remains `group.app.cogwheel.conduit`.
- The extension activation rules remain broadly the same.
- The file no longer explicitly contains `CFBundleDisplayName` in the plist
  body; the display name is instead being set from the Xcode project build
  setting as `Ask Conduit`.

Interpretation:

- This is mostly cleanup plus a shift from plist-managed display name to
  project-managed display name.

Risk:

- Low, assuming the Xcode build setting remains the source of truth.

### `ios/ShareExtension/ShareExtension.entitlements`

Observed local changes:

- `com.apple.security.application-groups` changes from:
  `group.app.cogwheel.conduit`
  to an empty array.

Interpretation:

- The share extension no longer declares the shared App Group.

Risk:

- High. This can break app-extension-to-app data passing and any shared
  container usage.

### `ios/ConduitWidgetExtension.entitlements`

Observed local changes:

- `com.apple.security.application-groups` changes from:
  `group.app.cogwheel.conduit`
  to an empty array.

Interpretation:

- The widget extension no longer declares the shared App Group.

Risk:

- High. This can break widget access to shared data or app-group-backed storage.

### `ios/ConduitWidget/Info.plist`

Observed local changes:

- Only trailing blank lines were removed.

Interpretation:

- No functional change.

Risk:

- None.

### `ios/Podfile.lock`

Observed local changes:

- `path_provider_foundation` is added to the resolved pods and external sources.

Interpretation:

- A pod resolution changed locally, likely because Flutter regenerated iOS pod
  dependencies after plugin resolution changed.

Risk:

- Usually low, but lockfile changes should be reviewed together with the Dart
  dependency changes that caused them.

## Most important follow-up questions

Before committing the local iOS changes themselves, these are the key decisions
to make:

1. Should the project keep the local Apple team and bundle ID rewrites, or are
   they only for one developer machine?
2. Was removing App Group membership intentional?
3. If App Groups were removed only to get local signing working, should they be
   restored properly with matching capabilities and provisioning profiles?
4. Is `Ask Conduit` the intended public-facing name for the share extension?

## Recommendation

Treat the following as local-only until explicitly validated:

- `DEVELOPMENT_TEAM`
- `PRODUCT_BUNDLE_IDENTIFIER`
- `EXCLUDED_ARCHS[sdk=iphonesimulator*]` workarounds

Treat the following as needing product review before merge:

- Empty App Group entitlements in:
  - `ios/Runner/Runner.entitlements`
  - `ios/ShareExtension/ShareExtension.entitlements`
  - `ios/ConduitWidgetExtension.entitlements`

If widget or share functionality is expected to use a shared container, those
entitlements should likely be restored rather than kept empty.
