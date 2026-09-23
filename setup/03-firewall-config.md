# 03 - Firewall Configuration

## Goal

With SSH now hardened, the next step was to set up a firewall so that nothing could reach my VM except what I explicitly needed, which was the SSH.

## Checking the starting state

```bash
sudo ufw status
```

This showed 'inactive', meaning Ubuntu does not have a firewall enabled by default.

## Setting default policies

I set the default behaviour for traffic for enabling anything:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

This means nothing can reach into the VM unless explicitly allowed, but the VM does have outgoing access to the Internet for updates, etc.

## Allowing SSH before enabling the firewall

This should happen **before enabling ufw**, because enabling the firewall to "deny incoming" connections by default (with no exception for SSH) meant that I'd be locked out and only able to get back in via the VirtualBox console.

```bash
sudo ufw allow 22/tcp
```

## Enabling the firewall

```bash
sudo ufw enable
```

This was a warning that it might disrupt any existing SSH connections (as I had been applying these firewall rules while on an SSH connection). As port 22 was already allowed, I verified it.

## Verifying the rules

```bash
sudo ufw status verbose
```

The output confirmed that the firewall was active and that both IPv4 and IPv6 SSH traffic was active:

```text
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)
New profiles: skip
To                         Action      From
---
22/tcp                     ALLOW IN    Anywhere
22/tcp (v6)                ALLOW IN    Anywhere (v6)
```

To be safe, I pulled up a new terminal window and checked that I could still SSH into the server with the firewall running (I could).

## Adding rate-limiting on SSH

While key-based authentication already stops anyone getting in with a guessed password, it doesn't stop repeated connection attempts from being made in the first place, and so I switched the SSH rule from a plain allow to a rate-limited rule:

```bash
sudo ufw limit 22/tcp
```

Therefore, this will automatically slow down an IP that tries to connect several times in a row.

Checked it applied correctly:

```bash
sudo ufw status verbose
```

Port 22 now shows `LIMIT IN` instead of `ALLOW IN` for both IPv4 and IPv6.

## Logging

'ufw' was logging blocked connection attempts to /var/log/ufw.log (Logging: on (low) in the output of status) but I did not enable this. It is useful to know that it is there if you will want to check what is hitting your firewall.

## Result

* firewall active, default deny on all incoming traffic
* SSH (port 22) explicitly allowed and rate-limited, both IPv4 and IPv6
* All other incoming traffic is blocked by default
* Outbound traffic unrestricted
* Confirmed that SSH connection works after enabling the firewall

## Considered but not a priority

* **Changing the SSH port** - this eliminates most of the automated scanner noise, but it is not a defense against a dedicated attacker, and does little for this lab.
* **fail2ban** ,  bans IPs after repeated failures, similar to the rate-limiting I already added via ufw limit. This is only valuable if this VM is ever exposed beyond my home network. For now, I have ufw limit, which effectively covers the most immediate risk.

The only service running on this VM is SSH, so there is no other port to allow for now. Any future services you deploy (a web server, monitoring software, etc.) will require a specific ufw allow rule before they can be accessed.
