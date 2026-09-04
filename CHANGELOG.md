# Changelog

## v1.0.0 (2026-09-03)

Initial release. Verified on KernelSU + ZygiskNext + LSPosed v2.1.1 (Android 16).

- Amap 16.23: splash / real-time fetch / home banner / background push / search template splash — 14 hook points, `ok=14 miss=0`
- Baidu Maps 21.18: splash (3 channels) / mediation loaders ×6 / BMAd providers / mid banner / floating promo bar — 34 hook points, `ok=34 miss=0`, fast entry via `F()/z()` gate
- Tencent Maps 11.4: GDT init kill / splash pipeline tasks ×3 / home banner binding / POI ad cards (view layer) — 8 hook points + ViewKiller
- Generic ViewKiller sweep on `Activity.onResume` (class + resource-id patterns)
