# Figma composition reference

Detail for Stage 3. Read this when implementing the Figma-side composition, or when deciding
whether to use Figma at all.

## The two-tool workflow (official server)

The official Figma MCP server splits the work across two tools because they do different jobs:

| Tool | Role | Handles images? |
|------|------|-----------------|
| `use_figma` | Writes native Figma structure (frames, auto layout, components, variables, text) by executing Plugin API code. Builds the template. | **No** — emits placeholders where images should go. |
| `upload_assets` | Uploads PNG/JPG/GIF/WebP into the file; with a target node, sets the image as that node's **fill**. | **Yes** — this is the fill mechanism. |

`upload_assets` behavior: given a URL/asset and a target node ID, it uploads the image as a
fill to that node. Given no target node, it creates new frames with the images as fills. For
this pipeline you always want the **targeted** form so the screenshot lands in the right
placeholder inside your template.

Underneath, both ultimately drive the Plugin API image primitive
(`figma.createImageAsync` → set the node's `.fills` with the returned `imageHash`), which
supports PNG/JPG/GIF up to 4096×4096.

## Recommended sequence

1. **Template** (`use_figma`): per screen, build background + device-frame shape + localized
   caption text layer + an empty rectangle placeholder for the screenshot. Reuse the app's
   design-system components/variables where they exist.
2. **Naming convention**: name each placeholder node to encode locale + screen unambiguously,
   e.g. `shot@de-DE@01-home`. The fill step matches on this, so keep it strict and consistent.
3. **Fan out** (`use_figma`): duplicate the template across locales × screens, swapping in the
   localized caption copy per locale.
4. **Fill** (`upload_assets`): for each placeholder node, upload the matching
   `screenshots/<locale>/<screen>.png` targeting that node ID as a fill.

## Constraints to design around

- **Full seat required** to write to canvas (outside drafts), plus edit permission on the file.
  Dev seats are read-only outside their own drafts.
- **10 MB per asset** cap on `upload_assets`. A 1320×2868 PNG of a dense screen can get close.
  Verify captures are under the cap; if not, export optimized PNG or JPG.
- **Fonts must be uploaded to the Figma account** to be available when writing to canvas —
  locally installed fonts won't resolve. Critical for CJK, Arabic, Hebrew, Thai, Devanagari
  caption text across locales; missing fonts render as tofu (□□□).
- **Beta quality**: the write tools are actively evolving and output may need manual cleanup.
  Test on a duplicated library or a draft file before running against the real template.
- **Rate limits** apply to read tools; some write tools are exempt, but don't assume unlimited
  throughput when fanning out across many locales × screens.

## Alternative: skip Figma, compose in code

If keeping Figma as the design source of truth doesn't matter, composing finished frames in
code is the lowest-moving-parts path and never touches the beta write tooling:

- Compose device frame + background + localized caption with SwiftUI/Core Graphics (natural
  given an iOS codebase) or a Pillow-style image compositor, emitting flat store-ready PNGs at
  exact App Store Connect dimensions.
- Watch **font fallback** for non-Latin caption text — same tofu risk as above; ensure the
  compositor has fonts covering every locale's script.

Trade-off: you lose editable, on-brand, component-based output (and easy design review), but
you gain a fully headless, dependency-light step. Offer this when the user signals they just
want finished images.

## Alternative: community server (`figma-console-mcp`)

Has image-fill via its write tools, but routes through a **Desktop Bridge plugin** running in
Figma Desktop: AI client → cloud relay → Desktop Bridge plugin → Figma. Remote SSE without
that pairing is read-only. So it always needs a live Figma Desktop client in the loop, which
is awkward for a headless/repeatable pipeline. Reach for it only if the user separately wants
its token-sync / design-system tooling and doesn't mind keeping Figma Desktop open. For a
plain screenshot pipeline, the official `use_figma` + `upload_assets` path is cleaner.
