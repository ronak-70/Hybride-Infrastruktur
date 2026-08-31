# VMware Network Configuration

## Linux Host

The VMware environment is running on a Linux mini PC.

### Physical Network Interfaces

#### Wi-Fi

- Interface: `wlp3s0`
- IPv4: `192.168.0.110/24`
- Default Gateway: `192.168.0.1`
- Public IPv4: `134.130.119.93`

Public IP was verified with:

```bash
curl -4 ifconfig.me