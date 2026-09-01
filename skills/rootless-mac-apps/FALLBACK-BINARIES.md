# Fallback: official release binaries (no Homebrew, or bottles won't pour)

Use this instead of Phase 2 when the bottle check shows a source build ahead, or Homebrew itself is unavailable. Everything installs into `~/.local/bin` (usually already on `PATH`) — rootless, no elevation, same as everywhere else in this skill. When done, resume at Phase 3 in `SKILL.md`; Colima and the Docker CLI behave the same regardless of how they were installed.

| Component | Source |
|---|---|
| `colima` | `github.com/abiosoft/colima` release, `colima-Darwin-arm64` |
| `lima` | `github.com/lima-vm/lima` release, `lima-<ver>-Darwin-arm64.tar.gz` (+ signed `SHA256SUMS`) |
| `docker` CLI | `download.docker.com/mac/static/stable/aarch64/docker-<ver>.tgz` |
| `docker compose` | `github.com/docker/compose` release, `docker-compose-darwin-aarch64` (+ `.sha256`) |
| `docker buildx` | `github.com/docker/buildx` release, `buildx-v<ver>.darwin-arm64` (ships provenance/SBOM only — no plain checksum file) |

Pair Colima and Lima at the **same versions Homebrew currently ships** (read them off `formulae.brew.sh`) — Colima tracks Lima's CLI closely and a mismatch breaks VM startup confusingly. Verify every checksum that's published. Keep Lima's tarball tree intact (`bin/` beside `share/`; `limactl` resolves guest agents relative to itself) — the main tarball carries only native-arch guest agents; a foreign-arch VM needs the separate `lima-additional-guestagents-*` tarball too. Symlink `limactl` into `~/.local/bin`. Compose and buildx go in `~/.docker/cli-plugins/` as `docker-compose` and `docker-buildx`, `chmod +x`.

`curl`-downloaded files carry no quarantine attribute, so Gatekeeper stays quiet. If `vz` fails on entitlements, ad-hoc re-sign (no admin needed):

```bash
codesign --force --sign - --entitlements /dev/stdin ~/.local/bin/limactl <<'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0"><dict>
  <key>com.apple.security.virtualization</key><true/>
</dict></plist>
EOF
```
