---
name: containerlab-arm64-lab
description: Set up containerlab and Arista cEOS network-device labs on Apple Silicon (M1/M2/M3) via Colima. Use when the user asks to set up containerlab, run network OS containers (cEOS, SR Linux, etc.) on a Mac, hits "Cpu virtualization support is required" from containerlab, gets a mislabeled/wrong-architecture container image, or asks about running VM-based network appliances (vEOS, Cisco n9kv, any vrnetlab-based image) on Apple Silicon.
---

# containerlab on Apple Silicon

Everything here comes from a real, working build (Arista cEOS pair, arm64-native, confirmed by reading ELF bytes, not by trusting labels). Follow it in order — each step exists because skipping it produced a specific, confirmed failure.

## Ground truth up front

- **Container-kind network OSes (cEOS, SR Linux) can work great on Apple Silicon, IF the vendor publishes a real arm64 build.** Verify this yourself; see "The architecture-label trap" below.
- **VM-kind network OSes via vrnetlab (vEOS-lab, Cisco n9kv, and every other `vr-*` containerlab kind) do not work on Apple Silicon**, full stop, as of this writing. Two independent, confirmed dead ends — see "Why VM-based nodes don't work" below. Don't spend time trying; move that work to real x86_64 hardware (a Proxmox box, a cloud VM, an Intel machine) instead.

## 1. Set up Colima

containerlab has to run inside a Linux VM, not on macOS directly — it manipulates Linux network namespaces and veth interfaces the macOS kernel doesn't have. Colima provides that VM.

```sh
colima start --cpu 8 --memory 20 --disk 100
```

Size for whatever nodes you're running. A VM-kind node like Cisco n9kv wants ~10GB RAM alone; container-kind nodes like cEOS are much lighter (a few hundred MB each). `colima stop` / `colima start --cpu X --memory Y --disk Z` resizes an existing instance — it does not delete data, but the VM must be stopped first.

## 2. Install containerlab inside the VM

```sh
colima ssh -- bash -c "curl -sL https://get.containerlab.dev | sudo bash"
colima ssh -- sudo usermod -aG clab_admins \$USER
```

Every `containerlab` invocation from here on routes through `colima ssh --`. A wrapper in `~/.zshrc` makes this feel native:

```sh
containerlab() {
  colima ssh -- containerlab "$@"
}
alias clab=containerlab
```

Colima mounts your Mac home directory into the VM at the identical path, so topology files under `~/...` resolve the same on both sides — no path translation needed.

## 3. The architecture-label trap (read this before loading any image)

**`docker inspect --format '{{.Architecture}}'` and `docker import`'s platform tag are not proof of the real architecture.** `docker import` sets the platform label from whatever you specify, or the host's platform by default — it does not inspect the tarball's actual binaries. Loading an x86_64 tarball with a bare `docker import` on an arm64 Mac will silently tag it `arm64`, and containerlab will deploy it without complaint. It only surfaces at boot time, as a vendor-software-specific failure that looks unrelated to architecture (e.g. Arista EOS: `Dependency failed for Platform Arc[hitecture check]... an unsupported OS Arch is booted`, tracing back to `ProcMgr warmstart` failing).

**Verify the real architecture from the ELF header before trusting any label or filename:**
```sh
docker run --rm --entrypoint sh <image> -c "od -An -tx1 -N20 /usr/bin/id"
```
- Byte 5 (`EI_CLASS`): `02` = 64-bit, `01` = 32-bit.
- Bytes 19-20 (`e_machine`, little-endian): `b7 00` = `EM_AARCH64` (arm64, what you want), `3e 00` = `EM_X86_64`, `03 00` = `EM_386` (32-bit x86).

Vendor naming is also a trap. For Arista specifically: `cEOS-lab` and `cEOS64-lab` are both x86_64 (the "64" means 64-bit x86, not ARM). Only **`cEOSarm-lab`** is genuinely arm64, and it may only be published for specific versions (often as an EFT/early-access release) — check the download portal for that exact filename, don't assume every EOS version has one.

## 4. Load and deploy a verified arm64 image

```sh
xz -dk cEOSarm-lab-<version>.tar.xz
docker import cEOSarm-lab-<version>.tar ceos:<version>
rm -f cEOSarm-lab-<version>.tar
# verify per step 3 before trusting it
```

Reference it in a topology file with `kind: arista_ceos`, then:
```sh
colima ssh -- containerlab deploy -t <file>.clab.yml
```

Confirm it's real, not just "running": `docker exec <container> Cli -c "show version"` should report `Architecture: aarch64`.

### Default credentials (Arista cEOS)

Containerlab auto-generates a `secret sha512` hash in the startup-config per deploy, but the plaintext default login is:
- Username: `admin`
- Password: `admin`

Quickest access with no credentials at all — exec straight into the container's own namespace:
```sh
colima ssh -- docker exec -it <container-name> Cli
```

Full SSH (the lab's docker network is internal to the Colima VM, not reachable from macOS directly — go through `colima ssh` first, or use `sshpass` from the Mac in one line):
```sh
colima ssh -- sshpass -p admin ssh -o StrictHostKeyChecking=no admin@<node-ip>
```

## 5. Why VM-based (vrnetlab) nodes don't work on Apple Silicon

Two separate, confirmed walls. Both were diagnosed from source/registry, not guessed.

**Wall 1 — containerlab's own preflight check is wrong on non-x86 hosts.** `virt/virt.go`'s `VerifyVirtSupport` greps `/proc/cpuinfo` for `vmx` (Intel) or `svm` (AMD) — flags that will never exist on any arm64 CPU. It fails "Cpu virtualization support is required for node ..." unconditionally on Apple Silicon, even though vrnetlab itself (`common/vrnetlab.py`) already checks for `/dev/kvm` and falls back to `-accel tcg` (software emulation) gracefully on its own. This check can be patched:

```go
// virt/virt.go, top of VerifyVirtSupport()
if runtime.GOARCH != "amd64" {
    return true // vmx/svm are x86-only flags; meaningless check on this arch
}
```
Cross-compile and swap into the VM:
```sh
git clone --branch <tag matching your installed version> https://github.com/srl-labs/containerlab
cd containerlab
# apply the patch above to virt/virt.go
GOOS=linux GOARCH=arm64 go build -o containerlab-patched .
colima ssh -- sudo cp <path>/containerlab-patched /usr/bin/containerlab
```
This needs redoing after any containerlab upgrade inside the VM (`get.containerlab.dev` or `apt upgrade` overwrites it with the unpatched binary).

**Wall 2 — `vrnetlab-base` has no arm64 build at all**, confirmed directly against the registry:
```sh
curl -s "https://ghcr.io/v2/srl-labs/vrnetlab-base/tags/list"   # lists all published tags
# then for each tag:
TOKEN=$(curl -s "https://ghcr.io/token?scope=repository:srl-labs/vrnetlab-base:pull" | python3 -c 'import json,sys;print(json.load(sys.stdin)["token"])')
curl -s "https://ghcr.io/v2/srl-labs/vrnetlab-base/manifests/<tag>" -H "Authorization: Bearer $TOKEN" -H "Accept: application/vnd.oci.image.index.v1+json,application/vnd.docker.distribution.manifest.list.v2+json"
# a plain "manifest.v2+json" response (not a manifest list/index) means single-arch only
```
Patching past Wall 1 lets the deploy proceed, but the *entire* vrnetlab container — not just the emulated x86 guest — then runs under cross-architecture binfmt emulation, because there's no arm64 image to run natively. It fails during its own networking setup, before the guest OS even loads: `Cannot talk to rtnetlink: Operation not supported` from the `tc-tap-ifup` script. This is not a performance problem and not fixable by more patching — it needs `vrnetlab-base` rebuilt from source for arm64, which nobody has published as of this writing (checked GitHub issues/discussions, containerlab's own macOS docs, and the registry directly).

**Don't chase this further on Apple Silicon.** Move VM-kind network OS work to real x86_64 hardware — the same vrnetlab/containerlab tooling works natively there with hardware-accelerated KVM, no emulation, no architecture games.

## Makefile pattern for a lab directory

A `make fast-up` / `make status` / `make down-all` wrapper keeps every `colima ssh --` invocation out of muscle memory. See a working example's `labs/swim/Makefile` structure (targets: deploy/destroy per topology, `inspect`/`status`, plus an image-load helper that enforces the architecture check from step 3 before importing).
