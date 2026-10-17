# NixOS on Raspberry Pi 5

A template for building a Pi 5 bootstrap SD image and deploying host
configurations over SSH.

## Repository layout

```text
hosts/       Example host configurations
images/      Bootstrap SD image configuration
modules/     Shared Pi user, SSH, kernel, and filesystem settings
```

## Outputs

| Output | Purpose |
| --- | --- |
| `images.pi5-bootstrap` | SD image with SSH access and hostname `pi5-bootstrap` |
| `nixosConfigurations.pi5-host-1` | First example host |
| `nixosConfigurations.pi5-host-2` | Second example host |

Both hosts use [`modules/pi5-common.nix`](modules/pi5-common.nix) and the
`nixos-hardware` Pi 5 module. The bootstrap image uses the shared module and
nixpkgs' generic aarch64 SD image module.

## Prerequisites

- Nix with the `nix-command` and `flakes` experimental features enabled.
- An `aarch64-linux` builder, either native, remote, or emulated.
- A microSD card and Ethernet connection for the Pi.

On an x86_64 NixOS build machine, enable ARM emulation and rebuild:

```nix
boot.binfmt.emulatedSystems = [ "aarch64-linux" ];
```

Emulation runs ARM binaries on the build machine; this is not cross-compilation.
The flashing commands below assume Linux.

## Bootstrap and deploy

### 1. Customize the template

Fork or copy the repository, then edit
[`modules/pi5-common.nix`](modules/pi5-common.nix):

- Replace the placeholder SSH public key with your own.
- Replace `myuser` in both `users.users` and `nix.settings.trusted-users` if
  using a different username.
- Set the timezone, locale, and keyboard layout.

Add services to the files under `hosts/`. If renaming a host, update its
hostname, filename, and corresponding output in `flake.nix`.

Run the remaining build and deployment commands from your checkout root.
The examples use `myuser` and `pi5-host-1`; substitute your configured names.

### 2. Build the image

```bash
nix build .#images.pi5-bootstrap
nix run nixpkgs#zstd -- -d result/sd-image/*.img.zst -o pi5-bootstrap.img
```

The flake tracks `nixos-unstable`. Its locked SD image module includes the
Pi 5 device trees, U-Boot, and firmware configuration needed for this image.

### 3. Flash the card

Use `lsblk` to identify the microSD card. Replace `/dev/sdX` with its device
path; the command overwrites that device:

```bash
sudo dd \
  if=pi5-bootstrap.img \
  of=/dev/sdX \
  bs=4M \
  status=progress \
  conv=fsync
sync
```

This guide boots from microSD. USB or NVMe storage needs its own boot and
filesystem setup; see the [Pi 5 storage notes][pi5-wiki].

### 4. Boot and connect

Insert the card, connect Ethernet, and power on the Pi. Find its address in
your router's DHCP leases. On the deployment machine, replace the example
address below with the Pi's address:

```bash
pi=192.0.2.10
ssh "myuser@$pi"
```

SSH accepts your configured key only; password authentication and root login
are disabled. `myuser` has passwordless sudo in both the bootstrap image and
the example hosts.

After an intentional reflash, remove an obsolete SSH host key if the client
reports that the host identity changed:

```bash
ssh-keygen -R "$pi"
```

### 5. Deploy a host

From the checkout on the deployment machine, using the same `pi` variable:

```bash
nix run nixpkgs#nixos-rebuild -- switch \
  --flake .#pi5-host-1 \
  --target-host "myuser@$pi" \
  --build-host "myuser@$pi" \
  --sudo
```

The Pi builds and activates the host configuration. Use `pi5-host-2` for the
other example. Repeat the command for later changes, updating the address if
needed. Reboot to test the new kernel and boot configuration:

```bash
ssh "myuser@$pi" sudo reboot
```

Passwordless sudo is set in the shared module, not just the bootstrap image.
Configure a user password or secret management before setting
`security.sudo.wheelNeedsPassword = true`.

## Kernel and board support

The deployed hosts import `nixos-hardware.nixosModules.raspberry-pi-5` for
extlinux boot configuration, Pi 5 device-tree filtering, and initrd drivers.
The profile selects the driver names appropriate to the chosen kernel.

The shared module overrides its Raspberry Pi vendor kernel with:

```nix
boot.kernelPackages = pkgs.linuxPackages;
```

This selects nixpkgs' mainline kernel, which can use the binary cache instead
of building the vendor kernel locally. To use the vendor kernel, remove the
override; check cache availability before expecting a quick build.

GPIO, display, HAT, and overlay support can differ between the kernels. Test
the peripherals needed by your host when changing the kernel or updating
the locked inputs.

## Firmware partition updates

The image writes the FAT firmware partition once. With the default template,
rebuilds update the kernel, initrd, and extlinux configuration on the root
filesystem; they do not refresh U-Boot, GPU firmware, or `config.txt` on FAT.

To manage that partition during activation, add the following to the deployed
host configuration:

```nix
fileSystems."/boot/firmware" = {
  device = "/dev/disk/by-label/FIRMWARE";
  fsType = "vfat";
};

hardware.raspberry-pi.firmware = {
  enable = true;
  uboot.enable = true;
};
```

The activation script requires the partition to be mounted. It rewrites
`config.txt` and prunes stale device trees and overlays, so keep those changes
in the Nix configuration rather than editing the FAT files manually.

## Validation

Format Nix files with `nixfmt` and check Markdown with `rumdl`:

```bash
nixfmt flake.nix hosts/*.nix images/*.nix modules/*.nix
rumdl check .
```

Evaluate the outputs without building or activating them:

```bash
nix eval --raw .#images.pi5-bootstrap.drvPath
nix eval --raw .#nixosConfigurations.pi5-host-1.config.system.build.toplevel.drvPath
nix eval --raw .#nixosConfigurations.pi5-host-2.config.system.build.toplevel.drvPath
```

[pi5-wiki]: https://wiki.nixos.org/wiki/NixOS_on_ARM/Raspberry_Pi_5
