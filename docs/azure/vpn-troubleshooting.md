# VPN Troubleshooting

## Current Status

The Azure Site-to-Site VPN connection is currently not connected.

Connection type:

- Site-to-Site VPN
- IKEv2
- IPsec

## Network Configuration

### On-Premises

Public IP:

134.130.119.93

Internal network:

10.0.0.0/24

pfSense:

- WAN: em0
- LAN: em1
- LAN IPv4: 10.0.0.1/24

### Azure

Azure VPN Gateway public IP:

134.112.209.110

Azure workload network:

10.10.1.0/24

Azure management network:

10.10.2.0/24

Azure GatewaySubnet:

10.10.10.0/24

## IKEv2 Packet Test

The following tcpdump command was used:

```bash
sudo tcpdump -ni any 'udp port 500 or udp port 4500'
```
Observed traffic:

192.168.0.110.500 > 134.112.209.110.500
134.112.209.110.500 > 192.168.0.110.500

## IKEv2 Configuration

### Phase 1

- Authentication: Mutual PSK
- My Identifier: IP address
- Local public IP: 134.130.119.93
- Peer Identifier: Peer IP address
- Remote Gateway: 134.112.209.110
- Encryption: AES-256
- Hash: SHA-256
- DH Group: 14
- Lifetime: 28800 seconds
- DPD: Enabled
- DPD Delay: 45 seconds

### Phase 2

- Mode: Tunnel IPv4
- Local Network: 10.0.0.0/24
- Remote Network: 10.10.1.0/24
- Protocol: ESP
- Encryption: AES-256
- Hash: SHA-256
- PFS: Group 14
- Lifetime: 3600 seconds

## Test Result

The VPN gateway responds to IKEv2 initiation packets.

However, the IPsec connection does not reach the Connected state.

The observed traffic currently confirms IKE_SA_INIT communication, but a successful completed IPsec tunnel has not been confirmed.