# On-Premises High Availability Test

## Objective

Verify that the on-premises environment remains operational when the primary Domain Controller (TW-DC01) becomes unavailable.

## Environment

| Component | IP Address | Role                     |
| --------- | ---------: | ------------------------ |
| TW-DC01   |   10.0.0.5 | AD DS / DNS / DHCP       |
| TW-DC02   |   10.0.0.6 | AD DS / DNS              |
| TW-CLIENT |  10.0.0.50 | Windows 11 Domain Client |

Domain: `techwork.local`

## Pre-Failure Validation

Before the failure simulation, the following checks were performed:

- DNS resolution for `techwork.local`
- Connectivity to TW-DC01
- Connectivity to TW-DC02
- Active Directory Domain Controller discovery
- Active Directory DNS SRV records

The environment was operating normally.

## Failure Simulation

TW-DC01 was intentionally shut down in VMware Workstation.

Expected state:

- TW-DC01: unavailable
- TW-DC02: available
- TW-CLIENT: available

## Result

The Windows client was able to discover and communicate with TW-DC02 after TW-DC01 was shut down.

This demonstrates that the second Domain Controller can provide continued Domain Controller availability when TW-DC01 is unavailable.

## Conclusion

The On-Premises High Availability test was successful.

The redundant Active Directory and DNS configuration provides continued Domain Controller availability in the tested failure scenario.

## Next Step

Start the Hybrid Connectivity phase and design the secure connection between the on-premises network and Microsoft Azure.
