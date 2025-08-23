# vm-images

**Prebuilt and reproducible virtual machine images for CopperlineOS (Phase‑0).**  
This repo builds and ships ready‑to‑run VMs so you can demo or develop CopperlineOS on **any host OS** without touching your main system. Images boot a minimal Linux with the Copperline **host stack** pre‑installed (`compositord`, `copperd`, `blitterd`, `audiomixerd`, tools) and a small “first run” guide on the desktop.

> TL;DR: download a QCOW2/OVA, start the VM, run the sprite demo in 60 seconds.

---

## What you get

- **QEMU/KVM** (`.qcow2`) with **virtio‑gpu + virgl** for GL/Vulkan in guests.  
- **VirtualBox** (`.ova` / `.vdi`).  
- **VMware** (`.ova` / `.vmdk`).  
- Optional **cloud images** (cloud‑init) for CI runners.

Inside the VM:
- Base OS: **Debian 12** or **Ubuntu 24.04** minimal.  
- User: `copper` (password: set on first boot) with passwordless `sudo` (can be disabled).  
- Copperline services are installed and managed via **systemd --user** (see `host-linux`).  
- `examples`, `tools` and SDKs pre‑cloned in `~/CopperlineOS/` for hands‑on demos.

---

## Repository layout

```
vm-images/
├─ packer/
│  ├─ debian-12.pkr.hcl          # main template
│  ├─ ubuntu-24.04.pkr.hcl
│  └─ http/                      # cloud-init seed, preseed/autoinstall files
├─ qemu/
│  ├─ run.sh                     # convenience runner (gtk+virgl)
│  └─ README.md
├─ vagrant/                      # optional Vagrant boxes
│  └─ Vagrantfile
├─ cloud/
│  ├─ user-data                  # cloud-init user-data
│  └─ meta-data                  # cloud-init meta-data
├─ scripts/
│  ├─ build.sh                   # wrapper: Packer → images/ artifacts
│  ├─ provision.sh               # installs stack, configs user, pulls repos
│  └─ shrink.sh                  # zero+shrink for small uploads
├─ images/                       # output artifacts (gitignored)
└─ README.md
```

---

## Quick start (QEMU/KVM)

### 1) Build (optional) or download

- **Download** a release artifact (`.qcow2`) from the Releases page **or**  
- **Build** your own with Packer (see below).

### 2) Run with virgl (Linux host)

```bash
# Requires: qemu-system-x86_64, virglrenderer, KVM enabled
./qemu/run.sh images/copperlineos-debian12.qcow2
```

`qemu/run.sh` roughly executes:

```bash
qemu-system-x86_64 \
  -enable-kvm -cpu host -smp 4 -m 4096 \
  -device virtio-gpu-pci \
  -display gtk,gl=on \
  -device virtio-keyboard-pci -device virtio-mouse-pci \
  -device ich9-intel-hda -device hda-duplex \
  -netdev user,id=n0 -device virtio-net-pci,netdev=n0 \
  -drive file="$1",if=virtio,cache=none,discard=unmap
```

This gives you a guest window with OpenGL through **virgl** (good enough for demos) and working audio.

> Tip: on macOS/Windows hosts without KVM, use UTM/VirtualBox/VMware builds below.

---

## First run inside the VM

1. Log in as `copper` (password prompt appears on first boot).  
2. Start the stack (if not already running):
   ```bash
   clctl up && clctl status && clctl doctor
   ```
3. Generate a test sprite and animate it:
   ```bash
   portctl /run/copperline/compositord.sock '{"cmd":"create_layer"}'
   portctl /run/copperline/compositord.sock      '{"cmd":"bind_image","id":1,"path":"/tmp/sprite.rgba","w":128,"h":128,"format":"RGBA8"}'
   portctl /run/copperline/copperd.sock      '{"cmd":"load","program":{"version":1,"program":[{"op":"WAIT","vsync":true}]}}'
   portctl /run/copperline/copperd.sock '{"cmd":"start","id":1}'
   ```
4. Watch timing:
   ```bash
   timeline-inspect --socket /run/copperline/compositord.sock --events vsync
   ```

The VM ships with the `host-linux` scripts (`clctl`, profiles, systemd user units).

---

## Building images with Packer

### Prereqs

- **Packer** ≥ 1.9  
- QEMU, VirtualBox, or VMware installed (depending on builders you need)  
- Host Linux or macOS (QEMU on macOS uses HVF accel; slower but OK)

### Build a Debian 12 QEMU image

```bash
cd packer
packer init .
PACKER_LOG=1 packer build -only=qemu.debian12 -var 'username=copper' -var 'password=changeme' debian-12.pkr.hcl
# Result: ../images/copperlineos-debian12.qcow2
```

### Build a VirtualBox OVA

```bash
packer build -only=virtualbox-iso.debian12 debian-12.pkr.hcl
# Result: ../images/copperlineos-debian12.ova
```

### Build a VMware OVA

```bash
packer build -only=vmware-iso.debian12 debian-12.pkr.hcl
# Result: ../images/copperlineos-debian12.ova
```

**Provisioning** runs `scripts/provision.sh`, which:
- installs system deps (Mesa, Vulkan loader, ALSA, rtkit),  
- installs Copperline packages (or builds from source) via `host-linux`,  
- creates user `copper`, sets up **systemd --user** units and profiles,  
- places short README and launchers on the desktop.

---

## Cloud images (CI / headless)

We provide a **cloud-init** profile under `cloud/`. Example `user-data` highlights:

```yaml
#cloud-config
users:
  - name: copper
    groups: [sudo, video, audio]
    shell: /bin/bash
    sudo: ALL=(ALL) NOPASSWD:ALL
package_update: true
packages: [git, curl, jq, mesa-utils, vulkan-tools]
runcmd:
  - [ bash, -lc, "git clone https://github.com/CopperlineOS/host-linux ~/host-linux" ]
  - [ bash, -lc, "~/host-linux/scripts/build.sh --release || true" ]
  - [ bash, -lc, "systemctl --user enable --now copperline.target" ]
```

Great for cloud CI and local KVM clouds (libvirt, Proxmox).

---

## Limitations & tips

- **GPU acceleration:** QEMU virgl provides decent GL/Vulkan for demos, but not full performance or timing accuracy. For most deterministic tests, use **Wayland/X11 host mode** in the guest’s `compositord`.  
- **KMS in VM:** direct KMS may be limited; prefer host mode unless using virtio‑gpu scanout paths.  
- **Audio latency:** set `AUDIOMIXERD_PERIOD` to `256` or `512` if you hear xruns in VMs.  
- **Clipboard/file‑sharing:** enable VirtIO‑FS or host hypervisor integrations if needed.

---

## Checksums & provenance

Release artifacts include SHA256 checksums and a manifest file:
```
images/
  copperlineos-debian12.qcow2
  copperlineos-debian12.qcow2.sha256
  manifest.json  # base OS, package versions, git SHAs
```

You can regenerate `manifest.json` with `scripts/build.sh --manifest`.

---

## Troubleshooting

- **Black window / software rendering**: verify virgl is active (`glxinfo | grep renderer`).  
- **No sound**: try HDA instead of AC97 (default in our QEMU run script).  
- **Permission errors**: inside the guest, ensure user is in `video`/`audio` and `copperline` groups; re‑login.  
- **Very slow GL on macOS**: HVF is slower than KVM; try smaller window or VirtualBox/VMware builds.

---

## License

Packer templates, scripts, and docs are dual‑licensed under **Apache‑2.0 OR MIT**.

---

## See also

- Host scripts & units: [`host-linux`](https://github.com/CopperlineOS/host-linux)  
- Demos: [`examples`](https://github.com/CopperlineOS/examples)  
- Packaging: [`packaging`](https://github.com/CopperlineOS/packaging)
