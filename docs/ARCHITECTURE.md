# Architecture decisions

## Android compatibility
We target **Android 7.0 (API 24)** as the intended minimum version. This target is not considered validated until the Tauri Android build and the chosen WebView/panel assets have been tested on API 24 devices or emulators.

## Updateable dashboard assets
The dashboards are intended to be packaged/downloaded as independent static web assets rather than coupled to app APK releases.

Planned updater responsibilities:
1. Read a trusted panel catalog and version metadata.
2. Validate catalog and asset integrity before installation.
3. Download to a staging directory.
4. Switch to the new version only after validation.
5. Retain a previous version for rollback.
6. Check periodically and on app startup, with a user setting to disable automatic checks.
7. Keep the last known-good version usable offline.

## Security notes
- Do not persist controller secrets in ordinary web storage.
- Use native secure storage for secret tokens when the Android shell is implemented.
- Validate downloaded assets and use HTTPS for catalog/release sources.
- Do not blindly execute remote panel code without a controlled update and integrity model.
