# Releasing

Releases are published as **GitHub Releases** and built by CI. Binaries are not
committed to the repo — they're attached to the release as assets.

## Versioning

[Semantic Versioning](https://semver.org). Tags are prefixed with `v`. Beta
releases use a `-bN` suffix (e.g. `v1.4-b3`) and are automatically marked as
GitHub **pre-releases** (any tag containing a hyphen).

## Cutting a release

1. **Update the changelog.** In `CHANGELOG.md`, move the `[Unreleased]` items into
   a new version section dated today, and refresh the compare links at the bottom.
2. **Bump the firmware version** in `src/config.h` (`dk2mqtt_version`) to match.
3. **Commit and push** those changes.
4. **Tag and push the tag:**
   ```bash
   git tag v1.4.0          # or a beta: v1.4-b4
   git push origin v1.4.0
   ```
5. CI (`.github/workflows/release.yml`) builds the firmware and opens a **draft**
   GitHub Release for the tag, with `Daikin2MQTT_<tag>_OTA.bin` attached and an
   auto-generated **"What's Changed"** list.
6. **Edit the draft:** paste the new `CHANGELOG.md` section as the highlights at the
   top (the human "why it matters" summary), above the auto-generated list.
7. **Review and Publish.** Betas stay flagged as pre-release.

## Notes

- Auto-generated "What's Changed" groups merged PRs by label — see
  `.github/release.yml`. Direct commits won't be grouped, so the hand-written
  highlights from `CHANGELOG.md` are the primary notes.
- To install a release: OTA via `http://<device-ip>/upgrade`, or
  `pio run -e wifikit-serial-esp32-s3 --target upload --upload-port <device-ip>`.
