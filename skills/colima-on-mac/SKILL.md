---
name: colima-on-mac
description: Install, start, and autostart Colima's rootless Docker VM on macOS. Use when the user says "install colima", "set up colima", "colima won't autostart", "colima keeps respawning", or "docker isn't running" on a Mac.
---

# colima-on-mac

Act as the operator standing up Colima — Apple's Virtualization.framework running a Docker daemon in a userspace VM, no root required on Apple Silicon. The outcome is a Colima VM that starts cleanly, survives login without a foreground supervisor thrashing it, and passes a real build-and-run check — verified, not assumed, because most failure modes here are successes that aren't. Run the commands directly, in order; each phase names what right looks like, and every phase is idempotent, so re-running on a working machine is safe.

**Prerequisite:** this assumes Homebrew already works from `$HOME` (user-prefix, no `sudo`). If `brew` is missing or `sudo` is required anywhere, stop and use `rootless-mac-apps` first — that skill bootstraps Homebrew and cask installs on a locked-down Mac; this one picks up from there.

## Check the bottles before installing

A non-standard Homebrew prefix can't pour most bottles and silently falls back to building from source — fine for `colima`/`lima`, hours for anything QEMU-based:

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

Right looks like: your platform key (`arm64_tahoe` = macOS 26, `arm64_sequoia` = 15, `arm64_sonoma` = 14) shows `:any_skip_relocation`, and `lima` reports `deps=[]` — no QEMU pulled in. Anything else: warn the user it will source-build before starting.

## Install and register the CLI plugins

```bash
brew install docker docker-compose docker-buildx colima
```

`docker` here is the CLI only — the daemon lives inside the VM. Never install `docker-desktop` alongside it.

A user-prefix brew puts CLI plugins outside Docker's default search path, so register them. Merge rather than overwrite — `~/.docker/config.json` may already hold registry auth:

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

Right looks like: `docker info --format '{{range .ClientInfo.Plugins}}{{.Name}} {{.Version}}{{"\n"}}{{end}}'` lists both `compose` and `buildx`.

## Start the VM, sized from the machine

Read cores/RAM first (`sysctl -n hw.ncpu hw.memsize`) and size roughly a third of the cores, a quarter of the RAM. The disk is sparse but **cannot be shrunk later**, so size it up front:

```bash
colima start --vm-type vz --cpu 4 --memory 8 --disk 60
```

`vz` (Apple's Virtualization.framework) needs Apple Silicon and no root. On Intel, or if MDM rejects `vz`, fall back to `--vm-type qemu` (pulls in the `qemu` formula — a long source build in a custom prefix) and warn it's slower. If both fail, it's a policy wall — tell the user to escalate to IT rather than working around it.

Right looks like: `colima status` reports `running using macOS Virtualization.Framework`.

## Autostart the VM without autostarting a supervisor loop

**Never use `brew services start colima`.** Its plist runs `colima start -f`, a foreground supervisor with `KeepAlive/SuccessfulExit=true`. The VM itself is a separate Lima process, not a child of that supervisor — so any time the supervisor dies while the VM survives (every `brew services restart`, any crash), the relaunch finds the VM already up, logs `already running, ignoring`, exits 0, and `KeepAlive` respawns it. Result: an endless ~10s respawn loop while `brew services list` misreports the service as `stopped`.

If a prior attempt used it, undo it and confirm the loop first:

```bash
grep -c "already running, ignoring" ~/homebrew/var/log/colima.log 2>/dev/null
brew services stop colima 2>/dev/null
```

Use a plain `RunAtLoad` agent with **no `KeepAlive`** instead — it starts the VM once and exits, so there is nothing to supervise and nothing to thrash. Plists don't expand `$HOME`, so substitute the real path:

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

Right looks like: `launchctl list | grep com.user.colima` shows `-  0  com.user.colima` — no PID, exit code 0. That's healthy: the agent ran, started the VM, and finished. A PID sitting there means a supervisor exists, which is exactly what this avoids.

Then prove it works the way login does, and that it isn't thrashing:

```bash
colima stop
launchctl kickstart -k gui/$(id -u)/com.user.colima
for i in $(seq 1 40); do docker info >/dev/null 2>&1 && { echo "daemon up"; break; }; sleep 5; done
sleep 40; wc -l ~/Library/Logs/colima-autostart.log
```

A stable line count across that last wait means no respawn loop.

Containers do not come back on their own after a restart unless they carry `--restart unless-stopped` (or compose `restart: always`) — there is no daemon-level override. Tell the user this rather than assuming they know, and give them the audit command:

```bash
docker inspect -f '{{.Name}} restart={{.HostConfig.RestartPolicy.Name}}' $(docker ps -aq)
```

Also warn against `docker buildx create` — it runs a BuildKit container with `restart=unless-stopped`, quietly reintroducing something that starts at every login.

## Verify, don't assume

Run every check below, not just the first that passes:

```bash
docker context use colima
docker version --format 'client {{.Client.Version}} / server {{.Server.Version}}'
docker compose version --short
docker buildx version
docker run --rm hello-world
```

Client/server version skew is normal — the CLI comes from brew, the daemon ships inside the VM image.

Multi-arch build, on the default driver (Colima uses the containerd image store, so **do not** `docker buildx create` a `docker-container` builder — unnecessary, and it leaves an auto-starting container behind):

```bash
d=$(mktemp -d ~/.verify.XXXX)
printf 'FROM alpine\nRUN echo built-on-$(uname -m) > /m\nCMD ["cat","/m"]\n' > "$d/Dockerfile"
docker buildx build --platform linux/amd64,linux/arm64 -t verify:multi "$d"
docker run --rm --platform linux/amd64 verify:multi
docker run --rm --platform linux/arm64 verify:multi
```

Right looks like `x86_64` then `aarch64` from the same tag. `docker buildx imagetools inspect` fails on a local-only tag because it queries the registry — verify per-platform by running instead.

Bind mounts, the silent failure mode:

```bash
mkdir -p ~/mount-probe && echo hi > ~/mount-probe/f
docker run --rm -v ~/mount-probe:/d alpine cat /d/f

echo hi > /private/tmp/hostmarker.txt
docker run --rm -v /private/tmp:/d alpine cat /d/hostmarker.txt
colima ssh -- mount | grep virtiofs
```

Right looks like `hi` from the first, `No such file or directory` from the second — only `$HOME` is mounted (virtiofs, rw). A mount from outside `$HOME` does **not** error: it resolves to a VM-local path Docker auto-creates, so the container sees none of the host's content while every command still exits 0. Test by asking whether specific host content is *visible*, not by expecting empty output — stale auto-created directories accumulate inside the VM and make that check pass or fail for the wrong reason. Say this to the user explicitly and keep project directories under `$HOME`.

Clean up everything created while verifying:

```bash
cd ~; docker rmi -f verify:multi alpine:latest hello-world:latest 2>/dev/null
docker builder prune -af; rm -rf ~/mount-probe "$d"
```

## Troubleshooting

- **`Cannot connect to the Docker daemon`** — VM down (`colima status`), or context switched (`docker context use colima`). The `default` context erroring on `/var/run/docker.sock` is normal and harmless.
- **`brew services list` says `stopped` but things still work** — the respawn-loop symptom above; check the `already running, ignoring` count and switch to the plain LaunchAgent.
- **TLS errors pulling images** — corporate TLS interception. The proxy's root CA must go *inside* the VM: `colima ssh` → PEM into `/usr/local/share/ca-certificates/` → `sudo update-ca-certificates` → `sudo systemctl restart docker`. (`sudo` inside the guest is fine — that VM is yours, not the host's.)
- **Only `localhost` port-forwarding works** — inherent to user-mode networking. A reachable VM IP (`--network address`) needs `socket_vmnet` installed as root, so it's unavailable here. Publish ports and use `localhost`.
- **`vz` refuses to start** — MDM may genuinely block Virtualization.framework. Try `--vm-type qemu`; if that also fails, it's a policy wall — escalate to IT.
- **Reset a broken VM** — `colima delete` then restart it. Destroys all images and volumes.

## Daily use

```bash
colima start          # remembers prior --cpu/--memory/--disk
colima stop
colima status
colima ssh             # shell inside the VM
colima delete          # destroy VM and all its data
```

`docker system prune -a` reclaims space inside the VM, but the sparse disk never shrinks on the host — only `colima delete` truly frees it.

## Uninstall

```bash
launchctl bootout gui/$(id -u)/com.user.colima 2>/dev/null
rm -f ~/Library/LaunchAgents/com.user.colima.plist
colima delete -f
brew uninstall colima lima docker docker-compose docker-buildx
```

Then remove the `cliPluginsExtraDirs` entry from `~/.docker/config.json` by hand.
