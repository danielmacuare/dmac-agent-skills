---
name: rootless-mac-docker
description: Rootless installation and management of developer tooling — Homebrew, Docker, Colima — on a locked-down, MDM-managed macOS machine where `sudo` is blocked and the user is not an admin. Use when the user asks to install Homebrew, Docker, Colima, Lima, or any brew formula or cask and lacks admin rights; when `sudo` returns "blocked by IT"; when Docker Desktop is unavailable or disallowed; when a cask fails on `/usr/bin/sudo -- /bin/cp -pR ... /Applications`; when `brew install` tries to build everything from source; or when Colima needs starting, stopping, resetting, or made to survive reboots without autostarting containers.
---

# Rootless dev tooling on a managed Mac

Corporate Macs (MDM plus a privilege-management agent — BeyondTrust/Avecto, Jamf, AirWatch/Workspace ONE) leave the user a standard user with `sudo` hard-blocked. Standard Homebrew and Docker Desktop both need admin. Everything here runs rootless, in `$HOME`, as the logged-in user.

Run the commands below directly, in order, except where marked as a branch. Most failures here are **false greens** — a command exits 0 having done the wrong thing — and **false reds**, where a healthy system looks broken. Each phase's "Right looks like" is the check that tells them apart; never infer a phase passed from the absence of an error message.

Every phase is idempotent; re-running on a working machine is safe.

## Ground rules

- Substitute the real home path for `$HOME` when writing files that need absolute paths — plists don't expand `$HOME`. Get it with `echo $HOME`.
- Every command here runs as the logged-in user. If a step seems to need `sudo`, that step is wrong for this machine.
- Ask before the first download. Flag the policy question once, plainly: working in `$HOME` isn't bypassing a security control, but whether a container runtime is sanctioned on a managed endpoint is the user's call, not yours. Then proceed if they confirm.
- Prepend `export PATH="$HOME/homebrew/bin:$PATH"` to brew/docker/colima commands until Phase 1 has persisted it, since each Bash call is a fresh shell.

## Phase 0 — Diagnose

```bash
uname -m; sw_vers -productVersion
sudo -n true 2>&1 | head -2
id -Gn | tr ' ' '\n' | grep -x admin || echo "NOT admin"
command -v brew docker colima limactl
find ~ -maxdepth 4 -path "*/homebrew/bin/brew" 2>/dev/null
scutil --proxy | head -5
sysctl -n hw.memsize hw.ncpu
```

Read the result carefully:

- **A user-prefix Homebrew often already exists but isn't on `PATH`** — `command -v brew` finds nothing while `~/homebrew/bin/brew` works fine. This is why the `find` is there. Missing it causes a pointless reinstall; skip Phase 1's clone if it's present.
- `sudo` printing an IT block message is the expected state, not a problem to solve. If the user *is* admin, this skill is the wrong tool — standard Homebrew and Docker Desktop will work better.
- Note the arch. `vz` needs Apple Silicon; on Intel, plan on `--vm-type qemu`, warn the user it's slower, and note Homebrew is moving Intel macOS to lower-priority support over time.
- Note RAM/cores for Phase 3 sizing, and whether a proxy exists (see TLS in Troubleshooting).

**Right looks like:** you can state five things before moving on — architecture, whether the user is admin, the path to an existing user-prefix brew or "none", proxy present or not, and the exact `--cpus/--memory/--disk` numbers Phase 3 will use.

## Phase 1 — Homebrew in the user's home directory

Skip the clone if Phase 0 found `~/homebrew/bin/brew` already there. The official installer targets `/opt/homebrew` and calls `sudo`, so clone instead:

```bash
git clone --depth 1 https://github.com/Homebrew/brew ~/homebrew
```

Persist PATH and the cask setting, guarded so re-runs don't duplicate lines:

```bash
mkdir -p ~/Applications
grep -q 'homebrew/bin/brew shellenv' ~/.zshrc || cat >> ~/.zshrc <<'EOF'

# Homebrew (user-prefix install — no admin rights on this Mac)
eval "$($HOME/homebrew/bin/brew shellenv)"
export HOMEBREW_CASK_OPTS="--appdir=$HOME/Applications"
EOF
```

**Right looks like:** `~/homebrew/bin/brew --version` prints a version. `brew config` reporting `Core tap: N/A` is normal — Homebrew resolves formulae through its internal JSON API. `brew doctor` will flag the non-default prefix as a support-tier finding; that's expected here, not actionable. Tell the user to open a new shell (or `source ~/.zshrc`) since the current one won't have the PATH.

### Installing GUI apps (casks) — branch, only if the user wants one

A cask copies its `.app` into `/Applications` via `sudo`, so an unpatched cask install dies like this:

```
Sudo Command Blocked: This Sudo Command has been blocked by IT Support.
Error: iterm2: Failure while executing; `/usr/bin/sudo -E -- /bin/cp -pR
  ~/homebrew/Caskroom/iterm2/3.6.11/iTerm.app /Applications/iTerm.app` exited with 1.
```

Retarget the cask at a directory the user owns:

```bash
brew install --cask iterm2 --appdir="$HOME/Applications"
```

The `HOMEBREW_CASK_OPTS` line from above makes that the default in new shells, so once it's persisted the flag is redundant. `~/Applications` is Spotlight-indexed and appears in Launchpad, and notarized apps still pass Gatekeeper from there. A failed cask install leaves nothing behind, so just retry with the flag.

**Right looks like:** `spctl -a -vvv ~/Applications/<App>.app` reports `accepted` and `source=Notarized Developer ID`.

## Phase 2 — Install Docker and Colima

A non-standard prefix can't pour most bottles, so Homebrew silently falls back to building from source — slow for a Go toolchain, hours for anything C++ (QEMU). Check every formula's bottle before installing, plus its direct dependencies — one source-only dep poisons the whole install:

```bash
for f in colima lima docker docker-compose docker-buildx; do
  curl -s "https://formulae.brew.sh/api/formula/$f.json" | python3 -c "
import json,sys
d=json.load(sys.stdin)
files=d['bottle']['stable']['files']
arm={k:v.get('cellar') for k,v in files.items() if k.startswith('arm64_')}
print(f\"$f {d['versions']['stable']} deps={d['dependencies']} {arm}\")"
done
```

**Right looks like:** your platform key shows `:any_skip_relocation` — bottles tagged this way embed no prefix paths and pour fine regardless of prefix — and `lima` reports `deps=[]`, no QEMU. Platform keys, newest first: `arm64_golden_gate` (macOS 27), `arm64_tahoe` (26), `arm64_sequoia` (15), `arm64_sonoma` (14, oldest still supported). A brand-new macOS version often has no bottle key yet — Homebrew then pours the newest older-compatible arm64 bottle, so no `golden_gate` key today still means `arm64_tahoe` pours fine on macOS 27. If a formula shows anything else, warn the user it will source-build before you start it, and use `FALLBACK-BINARIES.md` instead — it installs the same five tools as prebuilt binaries with no Homebrew involved, then rejoins this skill at Phase 3.

Then install:

```bash
brew install colima lima docker docker-compose docker-buildx
```

`docker` is the CLI only — the daemon lives inside the VM Colima runs. `lima` arrives as a Colima dependency. Colima runs rootless on Apple Silicon: Apple's Virtualization.framework (`vz`) with user-mode networking, entirely userspace — which is also why only `localhost` port-forwarding works; a reachable VM IP needs `colima start --network-address`, which requires `sudo` for Colima's own vmnet daemon, so it stays out of reach here.

A user-prefix brew puts CLI plugins outside Docker's search path, so register them. Merge rather than overwrite — `config.json` may already hold registry auth:

```bash
mkdir -p ~/.docker
python3 - <<'EOF'
import json, pathlib
p = pathlib.Path.home()/".docker/config.json"
try: cfg = json.loads(p.read_text())
except Exception: cfg = {}
d = str(pathlib.Path.home()/"homebrew/lib/docker/cli-plugins")
dirs = cfg.setdefault("cliPluginsExtraDirs", [])
if d not in dirs: dirs.append(d)
p.write_text(json.dumps(cfg, indent=2)+"\n")
print(p.read_text())
EOF
```

**Right looks like:** this lists both plugins —

```bash
docker info --format '{{range .ClientInfo.Plugins}}{{.Name}} {{.Version}}{{"\n"}}{{end}}'
```

## Phase 3 — Start the VM

Size cores and memory from the Phase 0 numbers: roughly a third of the cores, a quarter of the RAM. The disk is sparse — it consumes only what it uses, but cannot be shrunk short of `colima delete`, so size it up front.

```bash
colima start --vm-type vz --cpus 4 --memory 8 --disk 60
```

This is the one step MDM can veto. If `vz` is rejected, try `--vm-type qemu` (needs the `qemu` formula — a long source build in a custom prefix) and if that fails too, it's a policy wall: tell the user to escalate to IT rather than working around it.

**Right looks like:** `colima status` reports `running using macOS Virtualization.Framework`.

## Phase 4 — Autostart the VM without autostarting containers

Use a plain `RunAtLoad` LaunchAgent with **no `KeepAlive`** — it starts the VM once at login and exits, so there's nothing to supervise and nothing to thrash.

**Do not use `brew services start colima`.** Its plist runs `colima start -f`, a foreground supervisor, with `KeepAlive/SuccessfulExit=true`. But the VM is a separate Lima process, *not* a child of that supervisor. Any time the supervisor dies while the VM survives — every `brew services restart`, or any crash — the relaunch finds the VM already up, logs `already running, ignoring`, exits 0, and `KeepAlive` respawns it. The result is an endless ~10s respawn loop while `brew services list` misreports the service as `stopped` — a false green riding on a false red.

If a previous attempt used it, undo it first and confirm the loop:

```bash
grep -c "already running, ignoring" ~/homebrew/var/log/colima.log 2>/dev/null
brew services stop colima 2>/dev/null
```

Write the plist with the real home path substituted (plists don't expand `$HOME`, hence the `sed`):

```bash
mkdir -p ~/Library/LaunchAgents ~/Library/Logs
cat > ~/Library/LaunchAgents/com.user.colima.plist <<'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>Label</key>
	<string>com.user.colima</string>
	<key>ProgramArguments</key>
	<array>
		<string>__HOME__/homebrew/bin/colima</string>
		<string>start</string>
	</array>
	<key>EnvironmentVariables</key>
	<dict>
		<key>PATH</key>
		<string>__HOME__/homebrew/bin:/usr/bin:/bin:/usr/sbin:/sbin</string>
	</dict>
	<key>RunAtLoad</key>
	<true/>
	<key>ProcessType</key>
	<string>Background</string>
	<key>WorkingDirectory</key>
	<string>__HOME__</string>
	<key>StandardOutPath</key>
	<string>__HOME__/Library/Logs/colima-autostart.log</string>
	<key>StandardErrorPath</key>
	<string>__HOME__/Library/Logs/colima-autostart.log</string>
</dict>
</plist>
EOF
sed -i '' "s|__HOME__|$HOME|g" ~/Library/LaunchAgents/com.user.colima.plist
plutil -lint ~/Library/LaunchAgents/com.user.colima.plist
launchctl bootout gui/$(id -u)/com.user.colima 2>/dev/null
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.user.colima.plist
```

**Right looks like:** `launchctl list | grep com.user.colima` shows `-  0  com.user.colima` — no PID, exit code 0. That is healthy here: the agent ran, started the VM, and finished. A PID would mean a supervisor is sitting there, which is what we're avoiding.

Then prove it works the way login does, and that it isn't thrashing. Give the VM real time to boot — this can run past a default 2-minute command timeout, so allow for that:

```bash
colima stop
launchctl kickstart -k gui/$(id -u)/com.user.colima
for i in $(seq 1 40); do docker info >/dev/null 2>&1 && { echo "daemon up"; break; }; sleep 5; done
wc -l ~/Library/Logs/colima-autostart.log
sleep 40; wc -l ~/Library/Logs/colima-autostart.log
```

Two matching line counts means no respawn loop; a rising count means the thrash is back.

### Containers stay down unless asked

The daemon restarts only containers carrying a restart policy, and the default is `no` — so nothing comes back on its own unless a container opted in (`--restart unless-stopped`, or compose `restart: always`); there is no daemon-level switch to override policies. Tell the user this rather than assuming they know, and give them the audit command:

```bash
docker inspect -f '{{.Name}} restart={{.HostConfig.RestartPolicy.Name}}' $(docker ps -aq)
```

`docker buildx create` builds a BuildKit container with `restart=unless-stopped` — that quietly adds a container that starts at every login. Phase 5 explains why you shouldn't need it.

## Phase 5 — Verify

Run every command below in sequence; the phase is done only once all five have reported a result:

```bash
docker context use colima
docker version --format 'client {{.Client.Version}} / server {{.Server.Version}}'
docker compose version --short
docker buildx version
docker run --rm hello-world
```

Client/server version skew is normal and fine — the CLI comes from brew, the daemon ships inside the VM image. A `default` context erroring on `/var/run/docker.sock` is likewise normal — another false red, since that context is simply unused here.

Builds, including cross-platform:

```bash
d=~/.verify-build
mkdir -p "$d"
printf 'FROM alpine\nRUN echo built-on-$(uname -m) > /m\nCMD ["cat","/m"]\n' > "$d/Dockerfile"
docker buildx build -t verify:x "$d"
docker buildx build --platform linux/amd64,linux/arm64 -t verify:multi "$d"
docker run --rm --platform linux/amd64 verify:multi
docker run --rm --platform linux/arm64 verify:multi
```

**Right looks like:** `x86_64` then `aarch64` from the same tag. Multi-arch works on the default `docker` driver because Colima enables the containerd image store by default — don't `docker buildx create` a `docker-container` builder; it's unnecessary and leaves the auto-starting container from Phase 4 behind. (If Colima's docker config was ever hand-edited via `colima start --edit`, Colima skips enabling the containerd snapshotter and this build fails — re-run `colima start` without a custom `features` block.) `docker buildx imagetools inspect` fails on a local-only tag regardless — a false red, since it queries the registry — so verify by running each platform instead, as above.

Bind mounts — the silent one:

```bash
mkdir -p ~/mount-probe && echo hi > ~/mount-probe/f
docker run --rm -v ~/mount-probe:/d alpine cat /d/f

echo hi > /private/tmp/hostmarker.txt
docker run --rm -v /private/tmp:/d alpine cat /d/hostmarker.txt
colima ssh -- mount | grep virtiofs
```

**Right looks like:** `hi` from the first, and `No such file or directory` from the second. Only `$HOME` is mounted (virtiofs, rw).

A bind mount from outside `$HOME` is the flagship false green: it does not fail. It resolves to a VM-local path that Docker auto-creates, so the container sees none of the host's content while every command still exits 0. A compose bind mount from `/tmp` appears to work and silently serves nothing. Keep project dirs under `$HOME` and say this to the user explicitly.

Test it by asking whether specific host content is *visible*, as above. Do not test it by expecting `ls` to print nothing — those auto-created directories persist inside the VM and accumulate, so a stale one makes an empty-output check pass or fail for the wrong reason.

Compose, end to end (under `$HOME`, per the above):

```bash
mkdir -p ~/verify-compose/html && cd ~/verify-compose
printf 'services:\n  web:\n    image: nginx:alpine\n    ports: ["8098:80"]\n    volumes: ["./html:/usr/share/nginx/html:ro"]\n' > compose.yaml
echo '<h1>ok</h1>' > html/index.html
docker compose up -d && sleep 3 && curl -s http://localhost:8098
docker compose down
```

Then clean up everything created while verifying:

```bash
cd ~; d=~/.verify-build
docker rmi -f verify:x verify:multi alpine:latest nginx:alpine hello-world:latest 2>/dev/null
docker builder prune -af; rm -rf ~/mount-probe ~/verify-compose "$d"
docker ps -a; docker images
```

**Right looks like:** an empty container list, and none of `verify:*`, `alpine`, `nginx:alpine`, or `hello-world` left in `docker images` — the machine is back the way it was found.

## Troubleshooting

- **TLS errors pulling images, or downloading Colima/Lima itself** — corporate TLS interception. For pulls inside the VM, the proxy's root CA must go inside the VM: `colima ssh` → PEM into `/usr/local/share/ca-certificates/` → `sudo update-ca-certificates` → `sudo systemctl restart docker` (`sudo` inside the guest is fine; that VM is yours). For Colima's own downloads on the host, add `--downloader curl` to `colima start` so it uses the system CA store.
- **Reset a broken VM** — `colima delete` then redo Phase 3. Destroys all images and volumes.
- **Bad ownership under `~/homebrew`** — the usual `sudo chown` advice doesn't apply; use `chown -R "$(whoami)" ~/homebrew`.

## Daily use

`colima start` remembers the `--cpus/--memory/--disk` from Phase 3, so a normal restart needs no flags. `docker system prune -a` reclaims space inside the VM, but the sparse disk (Phase 3) never shrinks on the host — only `colima delete` truly frees it.

## Uninstall

```bash
launchctl bootout gui/$(id -u)/com.user.colima 2>/dev/null
rm -f ~/Library/LaunchAgents/com.user.colima.plist
colima delete -f
brew uninstall colima lima docker docker-compose docker-buildx
```

Then remove the block appended to `~/.zshrc` and the `cliPluginsExtraDirs` entry in `~/.docker/config.json` by hand.
