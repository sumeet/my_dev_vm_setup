# Research

## User

i was thinking of having a way to have like.... idk like nested forking
linux VMs or something like this... basically i just want to have a
thing where i can launch as many instances of my dev environment
as i want for running as many instances of claude code as i want
and connecting to them through screen or maybe like zellij or
something

i was thinking of using like nixos. i wasn't sure if you can do like VM
branching or... any other way of setting this up, can you help me
think through this?

i might also want to be able to access the dev environments via
sshing in from my phone. don't want it to be too complicated or
require too much prior setup but just be able to ssh in from a
regular client.

the whole idea shouldn't require too much setup i think

## Assistant

Short version: don't nest VMs. Use branchable QCOW2 overlays (for "forks") or just containers. Add
Tailscale once so your phone can SSH in like normal. Inside each instance, auto-attach to zellij and
run your Claude Code loop(s).

Option A - Branchable VMs (NixOS + KVM/libvirt)

0) Host prep (Arch-ish)

```bash
sudo pacman -S --needed qemu-full libvirt virt-install edk2-ovmf dnsmasq bridge-utils ipta
sudo systemctl enable --now libvirtd
sudo usermod -aG libvirt $USER
newgrp libvirt
```

1) Minimal NixOS VM config (reproducible)

ccvm.nix:

```nix
{ config, pkgs, ... }:
{
  users.users.dev = {
    isNormalUser = true;
    extraGroups = [ "wheel" ];
    shell = pkgs.zsh;
    openssh.authorizedKeys.keys = [ "ssh-ed25519 AAAA...YOUR_KEY" ];
  };

  services.openssh = {
    enable = true;
    settings = { PasswordAuthentication = false; PermitRootLogin = "no"; };
  };

  # Optional: direct-from-phone SSH without port fiddling
  services.tailscale.enable = true;

  programs.zellij.enable = true;

  programs.zsh.interactiveShellInit = ''
    if [ -n "$SSH_CONNECTION" ] && command -v zellij >/dev/null; then
      zellij attach -c || true
    fi
  '';
  services.getty.autologinUser = "dev";
  services.qemuGuest.enable = true;

  # lets `virsh domifaddr` find the IP
  networking.firewall.allowedTCPPorts = [ 22 ];
  time.timeZone = "America/Los_Angeles";
}
```

Build a base QCOW2 once (using nixos-generators):

```bash
nix run github:nix-community/nixos-generators -- \
  --format qcow2 --configuration ./ccvm.nix -o ~/vms/base.qcow2
```

2) "Forkable" overlays = cheap clones

Scripts (drop in `~/bin/devctl`, `chmod +x`):

```bash
#!/usr/bin/env bash
set -euo pipefail
POOL="${POOL:-$HOME/vms}"
BASE="$POOL/base.qcow2"
usage(){ echo "devctl {new NAME|fork FROM TO|ls|ip NAME|ssh NAME|rm NAME}"; exit 1; }
case "${1:-}" in
  new)
    NAME="${2:?}"; mkdir -p "$POOL"
    qemu-img create -f qcow2 -b "$BASE" "$POOL/$NAME.qcow2"
    virt-install --name "$NAME" --memory 4096 --vcpus 4 \
      --disk path="$POOL/$NAME.qcow2",bus=virtio,format=qcow2 \
      --import --os-variant generic --network network=default,model=virtio \
      --graphics none --noautoconsole
    ;;
  fork)
    FROM="${2:?}"; TO="${3:?}"
    qemu-img create -f qcow2 -b "$POOL/$FROM.qcow2" "$POOL/$TO.qcow2"
    virt-install --name "$TO" --memory 4096 --vcpus 4 \
      --disk path="$POOL/$TO.qcow2",bus=virtio,format=qcow2 \
      --import --os-variant generic --network network=default,model=virtio \
      --graphics none --noautoconsole
    ;;
  ls)
    virsh list --all
    ;;
  ip)
    NAME="${2:?}"
    virsh domifaddr "$NAME" | awk '/ipv4/ {print $4}' | cut -d/ -f1
    ;;
  ssh)
    NAME="${2:?}"
    IP=$( "$0" ip "$NAME" )
    exec ssh dev@"$IP"
    ;;
  rm)
    NAME="${2:?}"
    virsh destroy "$NAME" 2>/dev/null || true
    virsh undefine "$NAME"
    rm -f "$POOL/$NAME.qcow2"
    ;;
  *)
    usage
    ;;
esac
```

Usage:

```bash
devctl new cc1
# first VM

devctl fork cc1 cc2
# branch/fork from cc1

devctl ip cc2
# get IP

devctl ssh cc2
# SSH (auto-attaches zellij)
```
