# Xcode-native capture & extraction reference

Detail for Stages 1–2. Read this when implementing the UI test target, the Test Plan, or the
run/extract script.

## Test Plan (`.xctestplan`) structure

A Test Plan is a JSON file referenced from a scheme. It has three parts: test targets, shared
`defaultOptions`, and `configurations` that override the defaults. Localization lives in the
configurations — each configuration overrides the application language and region, and the
plan runs every enabled test once per configuration.

Skeleton with one configuration per locale:

```json
{
  "configurations": [
    {
      "id": "EN-US",
      "name": "en-US",
      "options": {
        "language": "en",
        "region": "US"
      }
    },
    {
      "id": "DE-DE",
      "name": "de-DE",
      "options": {
        "language": "de",
        "region": "DE"
      }
    }
  ],
  "defaultOptions": {
    "targetForVariableExpansion": {
      "containerPath": "container:MyApp.xcodeproj",
      "identifier": "SCREENSHOT_UITEST_TARGET_ID",
      "name": "MyAppScreenshotUITests"
    },
    "commandLineArgumentEntries": [
      { "argument": "-uiTestScreenshotMode YES" }
    ],
    "environmentVariableEntries": [
      { "key": "SCREENSHOT_MODE", "value": "1" }
    ],
    "preferredScreenCaptureFormat": "screenshot",
    "userAttachmentLifetime": "keepAlways"
  },
  "testTargets": [
    {
      "target": {
        "containerPath": "container:MyApp.xcodeproj",
        "identifier": "SCREENSHOT_UITEST_TARGET_ID",
        "name": "MyAppScreenshotUITests"
      }
    }
  ],
  "version": 1
}
```

Notes:
- `name` on each configuration is what `xcparse --test-plan-config` uses to label output, so
  name them exactly as the locale folder you want (`en-US`, `de-DE`, ...).
- To add a locale: append one object to `configurations`. Nothing else changes.
- `userAttachmentLifetime: keepAlways` keeps screenshots after the run — without this Xcode
  may discard attachments from passing tests.
- Pass screenshot-only state through `commandLineArgumentEntries` /
  `environmentVariableEntries` in `defaultOptions` (shared across all configs), and read them
  in the app to seed demo data or suppress dialogs.
- Field names have shifted slightly across Xcode versions; if a key is rejected, create a
  throwaway plan in the Xcode Test Plan editor and diff its JSON to confirm the current schema.

## Suggested starter locale set

`en-US, en-GB, de-DE, fr-FR, es-ES, it-IT, ja, ko, zh-Hans, pt-BR` — reconcile against the
app's actual supported languages and flag any mismatch. `zh-Hans` uses language `zh` with the
appropriate script; verify script/region handling for the CJK locales specifically.

## Accessibility identifiers

Shared structure, referenced from both app and UI test target:

```swift
enum A11y {
    static let homeScreen = "screen.home"
    static let detailScreen = "screen.detail"
    static let firstItem = "home.item.first"
}
```

In the app, set `.accessibilityIdentifier(A11y.homeScreen)` (SwiftUI) or
`view.accessibilityIdentifier = A11y.homeScreen` (UIKit) on the elements the test navigates by.

## Run + extract script skeleton

```bash
#!/usr/bin/env bash
set -euo pipefail

# --- parameters ---
SCHEME="MyAppScreenshots"
TEST_PLAN="Screenshots"
DEVICES=("iPhone 17 Pro Max")          # add "iPad Pro 13-inch (M4)" if shipping iPad
DERIVED="$PWD/.screenshot-deriveddata"
OUT="$PWD/screenshots"

rm -rf "$DERIVED" "$OUT"
mkdir -p "$OUT"

for DEVICE in "${DEVICES[@]}"; do
  # boot + clean status bar (9:41, full battery/signal)
  xcrun simctl boot "$DEVICE" 2>/dev/null || true
  xcrun simctl status_bar "$DEVICE" override \
    --time "9:41" \
    --dataNetwork wifi --wifiMode active --wifiBars 3 \
    --cellularMode notSupported \
    --batteryState charged --batteryLevel 100

  xcodebuild test \
    -scheme "$SCHEME" \
    -testPlan "$TEST_PLAN" \
    -destination "platform=iOS Simulator,name=$DEVICE" \
    -derivedDataPath "$DERIVED"

  # newest .xcresult for this run
  XCRESULT=$(find "$DERIVED/Logs/Test" -name "*.xcresult" -maxdepth 1 | sort | tail -n 1)

  # extract, named by locale (test-plan config), model and OS
  xcparse screenshots --os --model --test-plan-config "$XCRESULT" "$OUT/_raw"
done

# reorganize _raw into screenshots/<locale>/<screen>.png as needed
# (xcparse groups by config name = locale; flatten/rename to taste here)
```

Adjust the `find` path and the final reorganization to match the actual xcparse output layout
(inspect `$OUT/_raw` after the first run — layout varies by xcparse version).
