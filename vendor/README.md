# Endorfy protocol dependency

This fork installs `@openmouse/protocol` from
`openmouse-protocol-endorfy.tgz` so Endorfy LIV Plus Wireless works after
`npm ci`, without a sibling checkout or an upstream protocol release.
The archive is an npm package built from the protocol library at base commit
`beef2df996836dbc6a488b8c2e38a67603ec6102` plus `endorfy-protocol.patch`.
The source patch includes the Endorfy codec, driver, discovery entries,
automated tests and hardware notes. Its local package version is `0.1.0`;
this is a fork snapshot, not an npm release.

The driver exposes RGB light-strip effects, color, ten brightness levels and
ten speed levels through the shared Lighting panel. Every lighting write is
read back and checked; it preserves neighboring DPI and debounce settings.
The hardware notes include the vendor-software evidence and receiver results.

`endorfy-testing.md` records the tested receiver and write/read-back results.
Only the `3299:00a7` 2.4 GHz receiver is enabled. The source stays in the
external protocol library; the app uses its existing controls.

To refresh the package from the matching sibling `mouse-protocol` checkout:

```bash
cd ../mouse-protocol
npm run check
git add --intent-to-add src/endorfy src/drivers/endorfy docs/endorfy-testing.md
git diff beef2df996836dbc6a488b8c2e38a67603ec6102 --binary > ../openmouse/vendor/endorfy-protocol.patch
cp docs/endorfy-testing.md ../openmouse/vendor/endorfy-testing.md
npm pack --ignore-scripts --pack-destination ../openmouse/vendor
mv ../openmouse/vendor/openmouse-protocol-0.1.0.tgz ../openmouse/vendor/openmouse-protocol-endorfy.tgz
cd ../openmouse
npm install --ignore-scripts ./vendor/openmouse-protocol-endorfy.tgz
npm run check
```

The lockfile pins the archive's integrity, and CI permits this exact package
path. Replace the archive dependency with a stable npm release if that release
contains the driver, then remove these snapshot files.
