# 14.00 / Afterglow — Diagnostic report

Date: 2026-11-12  
Base branch: `main`  
Diagnostic branch: `diagnostics/14-00-cache-routing`

## Executive summary

The current public entry point for firmware 14.00 is routed to the legacy directory-level host, not the newer self-contained Afterglow host:

- The root `index.html` links to `14.00/index.html`.
- `14.00/index.html` advertises only 13.02, 13.04, 13.50 and 13.52; its `SUPPORTED` table likewise omits 14.00.
- That page navigates to `14.00/jb.html`.
- `14.00/jb.html` imports `./jb.js?v=19`, but `14.00/jb.js` is absent from the repository. The sibling legacy manifest also lists root-level `jb.js`, `core.js`, `mem.js`, `int64.js`, `ps4_offsets.js`, `rpc_worker.js` and payload/patch files that are not present at those root-level paths.

This is a concrete path/deployment inconsistency and is sufficient to explain why the legacy route cannot load its module chain. It is distinct from a proven defect in the Afterglow files themselves.

## Afterglow directory review

The following files exist under `14.00/afterglow/`:

- Entry pages: `index.html`, `jb.html`
- Entry logic and styles: `launch.js`, `style.css`
- Module files: `jb.js`, `core.js`, `mem.js`, `int64.js`, `ps4_offsets.js`, `afterglow.js`
- Binary assets: `hen.bin`, `goldhen.bin`, `patchset-1400.bin`, `patchset-1352.bin`
- Manifest generator: `make-manifest.sh`

The relative module imports inspected in `jb.js` resolve within this same directory. The landing page links to its sibling `jb.html`; the run page imports its sibling `jb.js`; the dynamic import of `afterglow.js` is also relative to `jb.js`. These paths are internally consistent.

## Manifest findings

`14.00/afterglow/cache.appcache` lists the HTML, CSS, JavaScript modules, image, patchsets and payload binaries expected by the checked-in manifest generator. Its entries are relative to the manifest directory, which is correct for the Afterglow pages.

The checked-in manifest says it was generated at `2026-11-04T04:40:34Z`. The generator computes the first 32 hexadecimal characters of SHA-256 for each listed file. I have not independently recomputed those hashes against the binary assets in this review, so hash freshness is **not verified**. Regenerate the manifest after any asset change and compare the generated file with the committed manifest before deployment.

The legacy `14.00/cache.appcache` belongs to the older directory-level host and must not be treated as the Afterglow manifest. Its file list points to root-level dependencies that are missing at those paths.

## Cache-loop interpretation

The legacy page waits on Application Cache events after entering its generic `checking cache...` branch, with no timeout/fallback if the expected event never arrives. That can leave the status message unchanged.

The Afterglow `launch.js` has a different policy: it blocks only while Application Cache is CHECKING or DOWNLOADING and otherwise navigates to the sibling run page. Therefore, diagnosing a persistent spinner requires confirming which URL the console actually opened and whether the browser is loading the legacy or Afterglow manifest.

## Recommended non-exploit deployment checks

1. Confirm the console's address bar is on the intended Afterglow landing URL, not the legacy `/14.00/` route.
2. Confirm the server returns HTTP 200 for every text asset listed in `14.00/afterglow/cache.appcache`.
3. Confirm the server returns the correct MIME types for JavaScript modules and the manifest, and that the manifest is served as `text/cache-manifest` where required by the console browser.
4. Regenerate and review `cache.appcache` whenever any listed file changes.
5. Clear the console browser's site data/Application Cache for this host before retesting, to avoid a stale manifest snapshot.
6. Capture the exact URL and the visible cache status/error message if the Afterglow route still stalls.

## Scope and safety

This branch intentionally contains this diagnostic report only. It does not alter the kernel exploit, exploit chain, firmware offsets, patchsets, payload binaries, or execution routing. Those components require separate validation on the actual target hardware; static repository inspection cannot establish that the exploit is safe or reliable to run.
