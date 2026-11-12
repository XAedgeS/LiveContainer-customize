# Issue #729 – Hide status bar / home indicator for fullscreen multitask apps

Upstream: https://github.com/LiveContainer/LiveContainer/issues/729 (dup: #1197, YouTube fullscreen).

## Root cause

In virtual-window multitask mode the guest app is hosted as an `FBScene` inside
LiveContainer's own window, with `settings.statusBarDisabled = YES`
(`MultitaskSupport/AppSceneViewController.m`). The status bar and home indicator
on screen therefore belong to the **host** (LiveContainer's root SwiftUI
`UIHostingController`), which never hides them. The guest's
`prefersStatusBarHidden` / `prefersHomeIndicatorAutoHidden` only end up in the
guest scene's client settings, which nothing reads.

## Fix

1. `AppSceneViewController` exposes the guest preferences read from
   `presenter.scene.clientSettings` (`statusBarHidden`,
   `homeIndicatorAutoHidden`), guarded with `respondsToSelector:`.
2. `MultitaskDockManager.updateHostSystemOverlays()` decides what the host should
   hide. It hides only when the frontmost virtual window is maximized and the
   dock is collapsed (title bar already hidden = "full screen"). A 0.3 s
   timer re-evaluates while apps run, because client settings have no stable
   public change callback across iOS 16–27.
3. `LCHostSystemOverlayState` (ObservableObject) publishes the result. The root
   view in `LiveContainerSwiftUIApp` applies `.statusBarHidden(_:)` and
   `.persistentSystemOverlays(.hidden)` (iOS 16+, auto-hides home indicator).
4. When the state changes, maximized windows refresh their scene frame and
   safe area, because the host safe area can change when the status bar hides.
5. A new setting under Settings › Multitask, *Fullscreen Status Bar*
   (`LCFullscreenStatusBarMode`, app-group defaults):
   - `0` Follow App (default): hide only when the guest app asks to.
   - `1` Always Hide.
   - `2` Always Show (the old behaviour).

6. Native-window mode (`MultitaskAppWindow.swift`): each guest runs in its own
   system window whose status bar is owned by the host SwiftUI view. A
   per-window `NativeWindowOverlayState` polls the same guest preferences and
   applies `.statusBarHidden` / `.persistentSystemOverlays` to that window.
   The same setting applies. The iPadOS 26 system menu bar (shown on pointer
   hover or swipe down) is system-controlled; there's no API to hide it.

## Build

Unsigned archive, then package `Payload/` into an `.ipa` (same layout as
`.github/build_github.sh`, without the SideStore variant). Sideload with
SideStore/AltStore/etc., which re-sign it.

```sh
xcodebuild archive -project LiveContainer.xcodeproj -scheme LiveContainer \
  -configuration Release -destination 'generic/platform=iOS' \
  -archivePath build/LiveContainer -derivedDataPath build/DerivedData \
  CODE_SIGNING_ALLOWED=NO
mkdir -p build/ipa && cp -R build/LiveContainer.xcarchive/Products/Applications build/ipa/Payload
rm -rf build/ipa/Payload/LiveContainer.app/Frameworks/SideStoreSupport.framework
# Ad-hoc sign with resolved entitlements (build/entitlements/*.plist: $(APP_GROUP_*),
# $(AppIdentifierPrefix), $(DEVELOPMENT_TEAM) replaced with Release values from Global.xcconfig).
# Without this, SideStore registers no app groups and LiveContainer shows "App Group accessible: No".
A=build/ipa/Payload/LiveContainer.app
find $A -name "*.dylib" -type f -exec codesign -f -s - {} \;
for fw in $A/Frameworks/*.framework; do codesign -f -s - "$fw"; done
for ex in LiveProcess ShareExtension LaunchAppExtension; do codesign -f -s - --entitlements build/entitlements/$ex.plist $A/PlugIns/$ex.appex; done
codesign -f -s - --entitlements build/entitlements/LiveContainer.plist $A
(cd build/ipa && zip -qry ../LiveContainer-issue729.ipa Payload -x "._*" -x "*.DS_Store")
```

Requirements: `git submodule update --init --recursive`, and with Xcode 26 an
installed iOS simulator runtime (`xcodebuild -downloadPlatform iOS`). Without
it, `actool` can't compile colorsets/imagesets into `Assets.car`, and the scheme
reports "iOS 26.x is not installed".

## Test on device

1. Settings › Multitask: mode = Virtual Window, Fullscreen Status Bar = Follow App.
2. Launch a game/video app in multitask, maximize it, collapse the dock.
3. Status bar and home indicator should disappear when the app hides them
   (e.g. YouTube landscape video). With *Always Hide* they hide for any app.
4. Restore the window or expand the dock: status bar comes back.
