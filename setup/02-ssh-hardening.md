# 02 - SSH Hardening

## Goal

After SSH was working on the static IP, the next step was to harden it, changing the default password-based SSH login to key-based authentication only and disabling root login over SSH.

## Generating an SSH key pair

On my Windows host (not the VM), I generated a new key pair:

```powershell
ssh-keygen -t ed25519 -C "rain-linux-infra-lab"
```

I used `ed25519`, which is a modern, strong key type that creates two files:
- `id_ed25519`, which is the private key that stays on my host machine and is never shared
- `id_ed25519.pub`, this is the public key, shared openly and placed on the server

At that point, I set the private key with a passphrase so that the private key file could only be used if the passphrase was known as well, enhancing security.

## Copying the public key to the VM

```powershell
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh linux-infra-lab@192.168.0.50 "cat >> ~/.ssh/authorized_keys"
```

This appends my public key into `~/.ssh/authorized_keys` on the VM. `authorized_keys` is a list of public keys that SSH checks to see if it should allow logins for that user. No output was printed, which is as expected with a `cat` append.

## Testing key-based login before disabling passwords

I made sure the key was working by itself before touching any of the server-side config:

```powershell
ssh linux-infra-lab@192.168.0.50
```

It asked for the **passphrase** for my key instead of my account password, so I knew SSH was using my key. Still, I opened a new terminal window and tested it a couple of times to be sure before I made any changes that might lock me out of my account.

## Disabling password authentication and root login

Once I was confident the key would work reliably, I edited the SSH server config:

```bash
sudo nano /etc/ssh/sshd_config
```

Changed/added:
```
PasswordAuthentication no
PermitRootLogin no
```


Then restarted the service:
```bash
sudo systemctl restart ssh
```

**I also kept my original SSH session open** while testing this in a new window so that if anything broke, I would still have a working SSH session to fall back to.

I verified that the settings had applied:
```bash
grep -E "PasswordAuthentication|PermitRootLogin" /etc/ssh/sshd_config
```

## Why it is important (even though nothing changed for me personally)

Prior to this, SSH would accept **either** a valid key or a valid password to log in. As I, for one, always log in with my key, password-based login was still available as a fallback and anyone could have tried it (including the standard bruteforcers that scan the net for open SSH ports). Password auth should be disabled since passwords can be susceptible to brute-force attacks. With the implemented key-based authentication, an attacker would need access to the private key, and in my case, the private key is protected by a passphrase.

The same was true for root login, I wasn't using it regularly but it was still a possible target for attackers to try against my machine because root has complete control over the system.

## Result

- Password login has been disabled and only key-based login is accepted
- Root login over SSH is disabled
- Confirmed that the code works by connecting fresh and only asking for my key's passphrase

## Considered but not a priority

There are several other common hardening steps that I did not take at this time because the primary risk (automated brute force attempts) is already reduced by key-only authentication and disabled root login.

- `AllowUsers`, which would allow you to specify users who can SSH to the system. That would be a small extra layer of security. With only two accounts on this VM, this is less important at this stage.
- `MaxAuthTries`: maximum number of authentication attempts per connection (this is useful as an additional layer of protection but it is less significant when password auth is already disabled).
- I also considered changing the default SSH port and using `fail2ban`. Please see `03-firewall-config.md` for the reasoning since they were really for firewall/network-layer rather than for SSH.

I may get back into some of these later, especially if this lab gets connected with the AD lab, or is reachable from outside my home network.