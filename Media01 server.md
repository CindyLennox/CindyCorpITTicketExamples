
This is my media server being hosted from my Fedora Workstation permanently

//media01/Media

I set up a Reserved DHCP for the Fedora Machine so its IP stays stable, but still uses DHCP

Then I set a Samba share that shares the external drive connected that has all my downloaded media

It is now accessible everywhere on my home network but requires credentials to access

We then set up a custom DNS for the server as well so it is easier to type when I want to access it from our home network

Also had to configure firewalld to allow for outside connections via SMB

Tested first locally on the machine the SMB server then set it up remote

Finally easy access and can watch the media on any computer or SMB capable device in the house

## Technical Details

**Server**
- OS: Fedora 44 Workstation
- Hostname: media01
- IPv4: 192.168.254.25
- Addressing: DHCP with router-side reservation
- DNS: media01.home → 192.168.254.25

**Storage**
- Filesystem: NTFS (ntfs-3g)
- Persistent mount: /srv/media
- Shared directory: /srv/media/見て
- Mount configured in /etc/fstab by UUID

**SMB**
- Share name: Media
- Network path: \\media01\Media
- Protocol: SMB
- Port: TCP 445
- Authentication required: Yes
- User: cindyjones
- Service: smb.service

**Fedora Security**
- firewalld: samba service allowed
- SELinux: Enforcing
- samba_share_fusefs: enabled

## Troubleshooting / Validation

1. Tested Samba locally with smbclient.
2. Tested Windows access by IP:
   \\192.168.254.25\Media
3. IP access worked, but hostname initially failed.
4. nslookup showed media01 did not exist in DNS.
5. Added local DNS entry on router.
6. Confirmed access using:
   \\media01\Media