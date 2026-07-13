---
name: localized-appstore-screenshots
description: >-
  End-to-end pipeline for generating localized App Store screenshots for an iOS app:
  capture screens across locales with native Xcode tooling, extract them, and compose
  finished marketing screenshots in Figma. Use this whenever the user wants to automate,
  build, or set up App Store / marketing screenshots across multiple languages or locales,
  mentions localizing screenshots, "screenshot pipeline", per-locale captures, Xcode Test
  Plan screenshots, XCUITest/Swift Testing screenshots, or dropping app screenshots into
  Figma device frames. Trigger even when the user only describes part of the flow (e.g.
  "capture screenshots in 10 languages" or "fill these screenshots into my Figma template") —
  the skill covers each stage independently and they usually want the whole chain.
---

# Localized App Store Screenshots

This skill builds a repeatable pipeline that turns one UI-navigation test into finished,
localized App Store marketing screenshots across many locales. It has three stages that are
each independently runnable:

1. **Capture** — a single Swift Testing UI test drives the app; an Xcode Test Plan runs it
   once per locale, producing a screenshot per screen per locale.
2. **Extract** — screenshots are pulled out of the opaque `.xcresult` bundle into a clean
   per-locale folder tree.
3. **Compose** — the raw screenshots are placed into a marketing template in Figma (device
   frame, background, localized caption) via the Figma MCP server.

Stages 1–2 use **only native Xcode components** plus one extraction tool (`xcparse`). No
third-party screenshot framework and no Ruby/gem tooling. Stage 3 uses the **official Figma
MCP server** (`use_figma` + `upload_assets`).

Before building, inspect the target project to learn its structure: find or create the UI
test target, read the app's localization setup (the actual `Localizable` catalog / supported
languages), and identify the screens and elements to capture. Where the desired on-screen
state or the specific screens aren't clear from the codebase, ask the user focused questions
rather than guessing.

---

## Stage 1 — Capture (native Xcode)

**Write one UI test, run it many times.** The locale matrix comes from the Test Plan, not
from test code. The test should be written once and run unchanged across every locale.

### The UI test

- Use **Swift Testing** (Xcode 26) for the test body, with the `XCUIApplication` capture
  layer inside a UI test target. (Capture inherently needs the XCUITest host — Swift Testing
  is the authoring/attachment layer on top, not a replacement for it.)
- Capture with `Attachment.record(screenshot.pngRepresentation, named: ...)`. Use stable,
  descriptive names per screen (e.g. `"01-home"`, `"02-detail"`) so extracted files are
  identifiable and order predictably.
- **Locate elements exclusively by accessibility identifier — never by visible/localized
  text.** Localized labels break the moment the locale changes; accessibility identifiers are
  language-independent. Define the identifiers in a structure shared between the app target
  and the UI test target so there's a single source of truth. If the app doesn't already set
  these on the relevant elements, add them.
- Keep the flow deterministic: wait for elements/state explicitly (`waitForExistence(timeout:)`
  or XCTest expectations) before each capture, so no loading spinners or partial states get
  shot.

Example shape:

```swift
import Testing
import XCTest

@MainActor
@Test func captureAppStoreFlow() async throws {
    let app = XCUIApplication()
    app.launch()

    let home = app.otherElements[A11y.homeScreen]
    #expect(home.waitForExistence(timeout: 10))
    Attachment.record(app.screenshot().pngRepresentation, named: "01-home")

    app.buttons[A11y.firstItem].tap()
    let detail = app.otherElements[A11y.detailScreen]
    #expect(detail.waitForExistence(timeout: 10))
    Attachment.record(app.screenshot().pngRepresentation, named: "02-detail")
}
```

### The Test Plan (this is where locales live)

- Create a **dedicated** screenshots Test Plan (`.xctestplan`) containing only the screenshot
  test(s). Do not let it auto-include other tests, or it will get polluted over time.
- Add **one configuration per locale**. In each configuration set the application language and
  region. Adding or removing a locale is then a change to the Test Plan only — never to test
  code. Make it obvious in the file how to add another.
- Set attachment preservation so screenshots are **kept** and not deleted after the run
  (`preferredScreenCaptureFormat` / attachment lifetime configured to keep always).
- If the app needs a specific state for screenshots (seeded demo data, suppressed
  subscription/permission/ad dialogs, a logged-in user), pass launch arguments or environment
  variables through the Test Plan's **shared** settings and have the app read them. Ask the
  user what state they want on screen if it isn't inferable from the codebase.

See `references/xcode-native.md` for the Test Plan JSON structure and configuration details.

### Capture environment

- Set a clean status bar before capture via
  `xcrun simctl status_bar <device> override --time "9:41" --dataNetwork wifi --wifiMode active --wifiBars 3 --cellularMode notSupported --batteryState charged --batteryLevel 100`.
- Target the required **6.9" iPhone** simulator at minimum. Make the device list a variable in
  the run script so the 13" iPad can be added if the app ships an iPad build. (Apple downscales
  the 6.9" set for smaller iPhones automatically, so you don't need a set per device.)

---

## Stage 2 — Extract (`xcparse`)

Native capture lands screenshots **inside the `.xcresult` bundle**, which is opaque; Xcode has
no built-in way to get them out in the open. `xcparse` (install via `brew install chargepoint/xcparse/xcparse`)
does exactly this.

The run + extract script should:

- Apply the status-bar override.
- Run `xcodebuild test` against the screenshots Test Plan with a **known derived data path**
  (`-derivedDataPath`) so the `.xcresult` location is predictable, and the screenshots scheme.
- Run `xcparse screenshots --os --model --test-plan-config <path>.xcresult <out>` — the
  `--test-plan-config` flag names output by the Test Plan configuration (i.e. the locale), so
  the per-locale organization falls out for free.
- Reorganize into a clean `screenshots/<locale>/<screen>.png` tree at the exact pixel
  dimensions App Store Connect expects.
- Be re-runnable (clear/overwrite previous output) and parameterized for locales and devices
  at the top.

See `references/xcode-native.md` for the full script skeleton.

---

## Stage 3 — Compose in Figma (official MCP server)

The goal is to place each locale's raw screenshot into a marketing template (device frame,
background, localized caption) and end with store-ready frames. This uses **two** official
Figma MCP tools in sequence, because they do different jobs:

- `use_figma` — writes native Figma structure (frames, auto layout, components, variables,
  text). It builds the template but **cannot place raster images yet** — it emits placeholders
  where images should go.
- `upload_assets` — uploads a PNG/JPG/GIF/WebP into the file; when given a target node it sets
  the image as that node's **fill**. This is what actually drops the screenshots in.

So the workflow is: build the template with named placeholder nodes via `use_figma`, then fill
each placeholder with the right locale's screenshot via `upload_assets`.

**High-level sequence:**

1. With `use_figma`, build a marketing-screenshot template per screen: background, device-frame
   shape, a localized caption text layer, and an **empty rectangle placeholder** where the
   screenshot goes. Use the app's existing design-system components/variables where available.
   Name placeholder nodes with a strict convention encoding locale + screen, e.g.
   `shot@de-DE@01-home`, so the fill step can match unambiguously.
2. Duplicate the template across all locales × screens (still via `use_figma`), substituting the
   localized caption copy for each locale.
3. For each placeholder node, call `upload_assets` targeting that node's ID with the matching
   `screenshots/<locale>/<screen>.png`, setting it as a fill.

**Important constraints** (details in `references/figma-mcp.md`):

- Writing to canvas needs a **Full seat**, and edit permission on the file.
- `upload_assets` caps at **10 MB per asset**. A 1320×2868 PNG of a busy screen can approach
  this — verify captures land under the cap, or export optimized PNG/JPG.
- Fonts must be **uploaded to the Figma account** to be usable when writing to canvas; locally
  installed fonts won't work. This matters for CJK/Arabic/etc. caption coverage across locales.
- The write tools are beta — output may need review. Test on a duplicate/draft first.

If keeping Figma as the design source of truth is *not* important to the user, note the
alternative: compose device-frame + caption entirely in code (e.g. SwiftUI/Core Graphics or a
Pillow-style compositor) producing flat finished PNGs, and skip Figma. Offer this if the user
signals they just want finished images with minimal moving parts. See `references/figma-mcp.md`
for the trade-offs and the community-server option.

---

## Deliverables

- The Swift Testing UI test file(s) and the shared accessibility-identifier structure.
- The `.xctestplan` with all locale configurations.
- The run/extract shell script.
- A Figma-composition step: either an agent runbook/prompt that drives `use_figma` +
  `upload_assets`, or (if chosen) a code compositor.
- A short README: how to run it, how to add a locale, how to add a screen.

## Guardrails

- Native Xcode + `xcparse` only for stages 1–2. No screenshot framework, no Ruby/gems.
- Stage 1–2 stop at raw extracted PNGs. Don't composite marketing framing in those stages —
  that's Stage 3's job, kept separate so each stage is independently runnable.
- Inspect the project and ask focused questions about on-screen state and which screens to
  capture before writing code.
