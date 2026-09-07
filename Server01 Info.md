
Admin: cindy_d
Hostname: ubuntu-srv01

Server Type
Ubuntu 26.04.1 LTS (Resolute Racooon)

CPU Architecture: x86_64
AMD Ryzen 9 3900X 12-Core Processor

3.3Gb RAM available

sda2 40Gb space

Network Interface: enp0s3
Network Mode: NAT
SSH Service: OpenSSH Server
SSH Guest Port: TCP 22
Host Forwarding: 127.0.0.1:2222 → 10.0.2.15:22

# Topology

Internet 
│ 
Home Router 
│
Windows 11 PC
│ 
VirtualBox NAT 
│ 
Ubuntu VM 10.0.2.15

Setting up the Port Forwarding to access server on Win11 Host
Used port 2222 for Host Port
10.0.2.15 for Guest IP
22 for the Guest Port

The Ubuntu guest's OpenSSH service listens on TCP port 22. Because the VM uses VirtualBox NAT, a port-forwarding rule maps TCP port 2222 on the Windows host (`127.0.0.1:2222`) to TCP port 22 on the Ubuntu guest (`10.0.2.15:22`). This allows the Windows host to initiate SSH connections to the otherwise NAT-isolated guest.

# Day 1 Milestone

```bash
Channel7 :: C:\Users\DJ ::  ssh -p 2222 cindy_d@127.0.0.1
The authenticity of host '[127.0.0.1]:2222 ([127.0.0.1]:2222)' can't be established.
ED25519 key fingerprint is SHA256:p47pPJ0oxoWkgwj2TtNiXLR5ke6I+4ULVXdavbZJboE.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '[127.0.0.1]:2222' (ED25519) to the list of known hosts.
cindy_d@127.0.0.1's password:
Welcome to Ubuntu 26.04.1 LTS (GNU/Linux 7.0.0-31-generic x86_64)

 * Documentation:  https://docs.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

 System information as of Mon Sep  7 10:24:58 PM UTC 2026

  System load:             0.69
  Usage of /:              38.4% of 18.53GB
  Memory usage:            7%
  Swap usage:              0%
  Processes:               119
  Users logged in:         0
  IPv4 address for enp0s3: 10.0.2.15
  IPv6 address for enp0s3: fd17:625c:f037:2:a00:27ff:fec0:1edd


Expanded Security Maintenance for Applications is not enabled.

0 updates can be applied immediately.

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status


cindy_d@ubuntu-srv01:~$
```
We are now able to login to the server via ssh from my host machine
This creates a remote administrator setup for the lab
![[Pasted image 20260907172522.png]]

During verification, `systemctl is-enabled ssh` returned `disabled`, despite SSH functioning correctly. Checked `ssh.socket` and confirmed it was both `enabled` and `active`, establishing that Ubuntu was using systemd socket activation rather than a permanently enabled SSH service.

**SSH configuration:** Ubuntu uses systemd socket activation for OpenSSH. `ssh.service` itself is disabled, while `ssh.socket` is enabled and active. The socket listens for SSH connections and activates the SSH service as required. Therefore, `ssh.service` does not need to be manually enabled.