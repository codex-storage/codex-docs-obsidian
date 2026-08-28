# Agent Context

## Quartz Deployment

- This vault is deployed to GitHub Pages with Quartz v5.
- GitHub Pages URL: `https://logos-storage.github.io/logos-storage-docs-obsidian/`.
- The workflow clones Quartz from `jackyzha0/quartz` branch `v5` and copies committed config from `.quartz/` into the cloned Quartz checkout.
- `.quartz/quartz.lock.json` pins the community plugin revisions. The workflow copies the lock file into the Quartz checkout and restores plugins with `npx quartz plugin install --clean`.
- The pinned `syntax-highlighting` plugin declares Shiki as a peer dependency but does not install it for its isolated build. The workflow installs the exact `shiki@4.4.3` dependency in that plugin and rebuilds it after the general plugin restore. Remove this repair when the pinned plugin revision no longer needs it.
- Keep Quartz config explicit in this repo under `.quartz/`; avoid dynamic CI patching unless there is a concrete reason.
- The workflow applies `.quartz/patch-crawl-links-object-data.mjs` after `npx quartz plugin install` because the current `quartz-community/crawl-links` plugin resolves `img[src]` but not SVG embed `object[data]` URLs.
- The crawl-links patch makes basename SVG embeds such as `![[diagram.svg]]` resolve to emitted asset paths like `../90-extras/92-assets/diagram.svg` instead of broken note-relative URLs.
- `.quartz/custom.scss` constrains SVG embeds rendered as `<object type="image/svg+xml">` to the article width with `width: 100%`, `max-width: 100%`, and `height: auto`; do not use hardcoded per-file aspect ratios unless a specific SVG needs a targeted exception.
- `baseUrl` in `.quartz/quartz.config.yaml` should remain `logos-storage.github.io/logos-storage-docs-obsidian` with no protocol and no trailing slash.
- The Quartz build should use `--baseDir "/logos-storage-docs-obsidian"` for correct GitHub Pages project-site routing.
- Dates are based on latest git commit mtimes, set by the workflow before building.

## Math Rendering

- MathJax is required. KaTeX does not support some expressions used in this vault, especially `\enclose{horizontalstrike}{...}`.
- `.quartz/quartz.config.yaml` configures the Quartz latex plugin with `renderEngine: mathjax` and loads MathJax packages including `ams`, `color`, `configmacros`, and `enclose`.
- Wide display equations should scroll horizontally instead of overflowing the article column.
- Inline MathJax must remain inline. Do not apply block layout, width, or overflow scrolling rules to inline `mjx-container` elements.

## MathJax CSS Fix

- The working custom CSS is in `.quartz/custom.scss`.
- Display equations are identified by MathJax with `mjx-container.MathJax[display="true"]`; target this attribute for scroll behavior.
- Inline equations do not have `display="true"`; keep them inline and only reset wrapping:

```scss
article mjx-container.MathJax[display="true"] {
  box-sizing: border-box;
  display: block !important;
  width: 100%;
  max-width: 100%;
  overflow-x: auto;
  overflow-y: hidden;
  padding-bottom: 0.25rem;
}

article mjx-container.MathJax[display="true"] > svg {
  display: block;
  margin-left: auto !important;
  margin-right: auto !important;
  max-width: none;
}

article mjx-container.MathJax:not([display="true"]) {
  overflow-wrap: normal;
  white-space: nowrap;
  word-break: normal;
}
```

- Do not use broad selectors such as `article .markdown-rendered > mjx-container.MathJax` or `article p > mjx-container.MathJax` for block behavior; those broke inline formulas.
- Centering for display math that fits depends on `margin-left: auto` and `margin-right: auto` on the SVG.
- Oversized display math still scrolls because the container is width-constrained and the SVG has `max-width: none`.

## Excalidraw Publishing

- Treat `.excalidraw.md` files as Obsidian plugin source artifacts, not publishable Markdown.
- `.quartz/quartz.config.yaml` ignores `**/*.excalidraw.md` so standalone Excalidraw source pages are not generated.
- Preserve `.excalidraw.md` embeds in notes when they are needed for Obsidian editing/viewing.
- For public Quartz rendering, add a short blockquote note immediately before the embed, such as: `> The original diagram is an Obsidian Excalidraw drawing. To view it, open this vault in Obsidian with the Excalidraw plugin installed and enabled.`
- Do not remove Obsidian Excalidraw embeds just to fix Quartz output; suppress the rendered Excalidraw source in Quartz instead.
- `.quartz/custom.scss` hides unresolved Excalidraw transclude placeholders with `article blockquote.transclude[data-url$=".excalidraw"]`.
- The unresolved Excalidraw transclude may still be present in generated HTML, but it should not be visible because the bundled CSS hides it.
- If an Obsidian note embeds a generated `.excalidraw.svg` by basename, the crawl-links patch should rewrite the generated `<object data="...">` URL to the emitted asset path under `90-extras/92-assets`.
- Verification for Excalidraw pages should confirm: the source note still contains the Obsidian embed, raw sections such as `Excalidraw Data`, `Text Elements`, and `Embedded Files` are absent from generated HTML, no standalone `*excalidraw*` HTML page is emitted, and the explanatory blockquote remains visible.

## Local Verification

- A CI-like local build can be run by cloning Quartz v5 into `/tmp/opencode`, copying the vault into `content`, copying `.quartz/quartz.config.yaml` and `.quartz/custom.scss`, installing plugins, patching crawl-links with `QUARTZ_CHECKOUT="/tmp/opencode/<build>/quartz" node .quartz/patch-crawl-links-object-data.mjs`, and running `npx quartz build --baseDir "/logos-storage-docs-obsidian"` from the Quartz checkout.
- When serving a local build, use:

```bash
python3 -m http.server 8080 --directory "/tmp/opencode/<build>/quartz/public"
```

- Do not start local preview servers in the background. Give the foreground command to run instead so the server is easy to stop with Ctrl-C.
- Plain `python3 -m http.server` does not rewrite Quartz extensionless URLs locally; open local pages with the explicit `.html` suffix, such as `http://localhost:8080/10-notes/sphinx-header-processing-infographic.html`.

- Check `10-notes/sphinx-header-processing-example.html` for regressions:
- Inline math inside prose and blockquotes must stay inline.
- Display equations that fit must be centered.
- Wide display equations must scroll horizontally.

## Worktree Notes

- During this session, unrelated local modifications were observed in `.obsidian/appearance.json` and `10 Notes/Sphinx Header Processing Example.md`.
- Do not revert or stage those files unless explicitly requested.

## Current Session State

- The user visually checked the latest local Quartz build and said the SVG rendering looks good.
- The latest verified local build was under `/tmp/opencode/quartz-build.Cy3CNu/quartz/public`; if rebuilding, use a fresh `/tmp/opencode` build directory rather than relying on that path existing.
- Intended repo changes from this Quartz work include `.github/workflows/deploy.yml`, `.quartz/quartz.config.yaml`, `.quartz/custom.scss`, `.quartz/patch-crawl-links-object-data.mjs`, and `AGENTS.md`.
- SVG embeds currently render as `<object type="image/svg+xml">`, so clicking the diagram itself does not open the raw SVG as a separate file.
- Possible future improvement for SVG click-through is to add a small explicit `Open SVG` link near SVG embeds or an overlay link; wrapping `<object>` directly in `<a>` may be inconsistent because embedded SVG documents can capture pointer events.
- The current choice is to leave SVG opening as-is unless the user asks to implement separate-file links.
