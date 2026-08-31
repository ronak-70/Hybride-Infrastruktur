tcpdump:

192.168.0.110:500 → 134.112.209.110:500
134.112.209.110:500 → 192.168.0.110:500

dann:

- UDP/500 traffic is reaching the Azure VPN Gateway.
- Azure responds to IKEv2_INIT.
- IKE_SA_INIT is visible.
- No UDP/4500 traffic observed yet.
- Tunnel does not reach the connected state.
- IKE_AUTH has not been confirmed yet.