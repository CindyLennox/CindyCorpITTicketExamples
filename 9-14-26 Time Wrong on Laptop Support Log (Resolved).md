
## Issue

After the computer was left asleep overnight, the system clock was approximately two minutes slow.

**System:** Dell Inspiron 7353  
**OS:** Fresh Arch Linux installation

## Hypothesis

Network Time Protocol (NTP) synchronization may be disabled.

## Test

Ran:

```bash
timedatectl
```

The output showed that the NTP service was disabled.

## Resolution

Enabled NTP synchronization:

```bash
sudo timedatectl set-ntp true
```

Verified the system time afterward. The clock synchronized correctly.

## Result

Resolved. System time is now synchronized automatically using NTP.