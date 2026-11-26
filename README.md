# Mihomo MultiPanel

An Android companion app for managing multiple Mihomo controllers and switching between dashboard experiences.

## Platform target
- **Minimum Android:** Android 7.0 (API 24), subject to validation against the final Tauri 2 Android build and embedded panel compatibility.
- **Recommended:** Android 10+ for a more current WebView and platform security support.
- **Shell:** Tauri 2 (Android)
- **UI:** Vue 3 + TypeScript + Vite
- **Panels:** Zashboard, YACD, MetaCubeXD (managed as separately updateable assets)
- **Builds:** GitHub Actions

## Planned capabilities
- Multiple Mihomo controller profiles.
- Switch between dashboard experiences.
- Panel catalog with version tracking and separate panel updates.
- Automatic update checks, configurable and opt-in/out.
- Keep the current and previous panel version to enable rollback.
- Offline access to the last installed panel version.
- Secure controller credential storage.
- Material You-inspired Android UI.

## Status
This repository currently contains a frontend prototype only. The Tauri Android shell, Android API 24 configuration, actual controller/API connectivity, panel downloads, persistence, and CI APK builds are not implemented yet.

## Development
Install dependencies and run the web prototype with:
```sh
npm install
npm run dev
```
