# Godot installation security record

Install date: 2026-10-19

## Official source

- Release: `4.7.2-stable`
- Release URL: https://github.com/godotengine/godot/releases/tag/4.7.2-stable
- Release state: published, not draft, not prerelease
- Engine ZIP: `Godot_v4.7.2-stable_macos.universal.zip`
- Engine SHA-256: `c58a24e31d720be9d62f60cb5627c4e695fb72f21b0cfe1bc9ccaa9a3b3ba63e`
- Engine SHA-512: `38aa16e5bba2083941fc5b3e54be0089bd4cc35e32415f5b9fd9a8a6a7b9818255d44532ea8ef94b5aef56c4b407c2d634fa4f657e4ebe681ebbf59b7bac69ca`
- Export templates: `Godot_v4.7.2-stable_export_templates.tpz`
- Templates SHA-256: `f298490b8d44d934be425a5a65a51bf15f422428b229a06a6e11d9ffea248011`
- Templates SHA-512: `ca4d71c4d7b81dfc15d1a98baa07534aa95b03fdda78a0075b06672e1648d2e5f40980c9adc28d23e1b92e732ee7bf3461997aa804af74ec2fcd7a93ccb84079`

The SHA-256 values matched GitHub's Release asset metadata. The SHA-512 values
matched the release's official `SHA512-SUMS.txt`.

## CuaDriver

- Project: https://github.com/trycua/cua
- Installed release: `cua-driver-rs-v0.35.4`
- Installed app: `/Applications/CuaDriver.app`
- Downloaded archive SHA-256 was verified against the release's official
  `SHA256SUMS`
- `codesign --verify --deep --strict`: valid on disk and satisfies its
  Designated Requirement
- `spctl --assess --type execute`: accepted as a Notarized Developer ID app
- Persisted CuaDriver telemetry was disabled with `cua-driver telemetry disable`

## macOS trust checks

- `codesign --verify --deep --strict`: valid on disk and satisfies its
  Designated Requirement
- `spctl --assess --type execute`: accepted
- Gatekeeper source: Notarized Developer ID
- Binary: universal `x86_64` and `arm64`
- Installed engine version: `4.7.2.stable.official.ed1daf0bf`

No API keys, credentials, or Godot token values are stored in this repository.
Native test builds are validated by executable and PCK SHA-256 before launch.

## Isolation boundary

The CuaDriver daemon is intended to run as a non-admin `godot-test` macOS
account. The main account publishes builds and jobs to a shared directory. The
test account owns result files and must explicitly grant Accessibility, Screen
Recording, and direct-capture consent to `CuaDriver.app`.

Install or repair the bridge from the `godot-test` account with:

```bash
/Users/Shared/godot-playtest/runtime/install-bridge-login.command
```

This single installer requests the CuaDriver permissions and installs the
LaunchAgent that starts the bridge at login. The bridge accepts background jobs
only and executes `cua-playtest-cards.sh` directly.

Do not grant Full Disk Access or administrator rights to the test account.
