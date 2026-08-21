# On-Premises Network

## Network Overview

The on-premises infrastructure is hosted on VMware Workstation.

The local network uses the `10.0.0.0/24` address space.

## IP Address Plan

| Device      | IP Address | Role               |
| ----------- | ---------: | ------------------ |
| TW-DC01     |   10.0.0.5 | AD DS / DNS / DHCP |
| TW-DC02     |   10.0.0.6 | AD DS / DNS        |
| TW-CLIENT01 |       DHCP | Windows Client     |

## Network

```text
On-Premises Network
10.0.0.0/24
        |
        +------------------+
        |                  |
     TW-DC01            TW-DC02
     10.0.0.5            10.0.0.6
        |
        |
   TW-CLIENT01
      DHCP
```
