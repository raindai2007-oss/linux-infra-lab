# 01 - Base Server Setup

## Goal

My goal was to set up an Ubuntu VM which would be the foundation for this lab and make it reachable over SSH from my host machine (Windows) and make sure its network address remains stable across reboots.

## What I built

A VM in Oracle VirtualBox running Ubuntu 26.04.1 LTS. I first thought I was installing Ubuntu Server until I found out halfway through that I had actually downloaded Ubuntu Desktop instead. I didn't bother rebuilding since everything still works the same for this lab's goals (SSH, firewall, monitoring) but I'll probably end up rebuilding with Server later on for a more realistic headless setup.

## Networking

I set it up so that the VM's network adapter is Bridged so that it is able to get a real IP address on my home network instead of it being behind VirtualBox's NAT. The VM was also accessible directly from my host without setting up port forwarding, which was a simpler setup for my needs.

## Enabling SSH

SSH was installed but it wasn't running and I confirmed this with:

* `sudo systemctl status ssh`, which showed it was inactive.

I then ran:

* `sudo systemctl enable ssh`
* `sudo systemctl start ssh`, which enabled and started it.

This makes sure that SSH starts automatically on every reboot, not just on this session.

## Problem: SSH connection had timed out

Having connected once, I came back later and got:

```text
ssh: connect to host 192.168.0.71 port 22: Connection timed out
```

My first thought was that SSH itself had stopped or it had been blocked by the firewall. But after checking with `ip addr` inside the VM, it showed a different IP address (`192.168.0.37`) than the one I'd been using (`192.168.0.71`).

What had actually happened: the VM was using DHCP and my router had given it a new IP after a reboot which meant that I was trying to SSH into an address that no longer existed.

## Fix: static IP

Instead of relying on DHCP, I gave the VM a fixed address to avoid this issue in the future. I found the VM was using NetworkManager (not systemd-networkd, as I initially expected for an Ubuntu install, but probably because I had used Desktop instead of Server).

I then edited `/etc/netplan/01-network-manager-all.yaml`:

```yaml
network:
  version: 2
  renderer: NetworkManager
  ethernets:
    enp0s3:
      dhcp4: no
      addresses:
        - 192.168.0.50/24
      routes:
        - to: default
          via: 192.168.0.1
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
```

Before choosing `192.168.0.50` to be my static IP, I checked it wasn't already in use on the network with a quick ping (in which I received a response of "Destination Host Unreachable," which confirmed nothing was there).

I ran `sudo netplan apply` which briefly dropped my SSH session so then I managed to reconnect successfully on the new IP address.

## Small mistake along the way

When I first tried to open the netplan config, I accidentally typed an extra space in the file path `nano /etc/netplan/ 01-...yaml` instead of `nano /etc/netplan/01-...yaml`, nano would then try to create a new blank file in the wrong location instead of opening the real config. I then exited without saving and reran the command correctly.

It's a small thing but it serves as a good reminder to double check paths before editing system config files.

## Result

* VM was reachable via SSH at a fixed IP: `192.168.0.50`
* SSH persists across reboots (enabled as a service)
* IP no longer changes on reboot

## What I'd do differently

Installing Ubuntu Server instead of Desktop would make this more of a realistic, lightweight infrastructure setup. I'm likely to do this before starting the firewall/hardening stage.
