# 01 - Base Server Setup

## Goal

My goal was to set up an Ubuntu VM which would be the foundation for this lab and make it reachable over SSH from my host machine (Windows) and make sure its network address remains stable across reboots.

## What I built

A VM in Oracle VirtualBox running Ubuntu 26.04.1 LTS. I initially thought I was installing Ubuntu Server until I found out halfway through that I had actually downloaded Ubuntu Desktop. I initially continued with it since everything worked for the lab's goals, but later rebuilt the VM with Ubuntu Server for a more realistic headless infrastructure setup.

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

```
ssh: connect to host 192.168.0.71 port 22: Connection timed out
```

My first thought was that SSH itself had stopped or it had been blocked by the firewall. But after checking with `ip addr` inside the VM, it showed a different IP address (`192.168.0.37`) than the one I'd been using (`192.168.0.71`).
What had actually happened: the VM was using DHCP and my router had given it a new IP after a reboot which meant that I was trying to SSH into an address that no longer existed.

## Fix: static IP

Instead of relying on DHCP, I gave the VM a fixed address to avoid this issue in the future. I found the VM was using NetworkManager (not systemd-networkd, as I initially expected for an Ubuntu install, but probably because I had used Desktop instead of Server).
I then edited `/etc/netplan/01-network-manager-all.yaml`:

```
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

## Update: rebuilt with Ubuntu Server

As planned above, I rebuilt the VM using Ubuntu Server rather than Ubuntu Desktop. I prefer Server due to it being a more realistic and lightweight setup for infrastructure work. I deleted the old Desktop VM as soon as I confirmed that Server was working.

## Differences on Ubuntu Server

A few things were different on Server compared to Desktop.
**SSH was already installed.** During the Server installation, I was asked whether I wanted to install OpenSSH server. I said yes, and it was already running after installation. Unlike Desktop, where I had to manually enable and start it.
**The netplan config was different.** The netplan config was located at `/etc/netplan/00-installer-config.yaml` instead of `01-network-manager-all.yaml` in Server. More importantly, it had no `renderer:` line at all:

```
network:
  ethernets:
    enp0s3:
      dhcp4: true
      dhcp6: true
      match:
        macaddress: 08:00:27:67:e2:15
      set-name: enp0s3
  version: 2
```

I initially thought this meant that no renderer was being used, but it just means that there's no `renderer:` line at all, which makes netplan use its default renderer. On Server, it's `systemd-networkd`, and the config of my previous Desktop installation explicitly assigned `NetworkManager` to be the renderer. The filename and renderer are not the same: the filename is a name given to the configuration file by the installer, while the renderer determines which networking backend will use it.
I edited this to add a static IP and retain the existing MAC address:

```
network:
  ethernets:
    enp0s3:
      dhcp4: false
      addresses:
        - 192.168.0.50/24
      routes:
        - to: default
          via: 192.168.0.1
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
      match:
        macaddress: 08:00:27:67:e2:15
      set-name: enp0s3
  version: 2
```

## Problem: SSH host key warning after rebuild

However, when I had set up the static IP and reconnected to the server, I got a security warning:

```
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!
Someone could be eavesdropping on you right now (man-in-the-middle attack)!
```

At first, this looked quite worrying, but then it all made sense. `192.168.0.50` was the IP of my old Desktop VM, which I had deleted, and the new Server VM had picked up the same IP. But it's a different machine install entirely, so it has a completely different SSH key. My host machine, noticing that the machine at that IP address no longer matched the SSH host key it had previously cached, flagged it. This is exactly what the warning is designed to catch: an unexpected change in a server's identity, which could indicate a man-in-the-middle attack. In this case, however, I knew the cause because I had deliberately deleted and rebuilt the VM.
This was fixed by removing the old, now-invalid key entry:

```
ssh-keygen -R 192.168.0.50
```

Then I reconnected and accepted the new host key, just as I did the first time I connected.
This still felt worth understanding, rather than just running the fix and moving on, because a warning like this is exactly what would catch a real man-in-the-middle attempt too. So understanding *why* it appears (and confirming why it was safe to bypass in this specific case) is likely to be more useful than the command to clear it.
