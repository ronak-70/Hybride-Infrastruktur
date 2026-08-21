# TW-CLIENT — Windows Domain Client

## Overview

TW-CLIENT is the Windows 11 client used to validate the on-premises Active Directory environment.

The client was successfully joined to the `techwork.local` domain.

## Configuration

| Property           | Value                 |
| ------------------ | --------------------- |
| Hostname           | TW-CLIENT             |
| Operating System   | Windows 11 Enterprise |
| Domain             | techwork.local        |
| IPv4 Address       | 10.0.0.50             |
| Address Assignment | DHCP                  |
| DNS 1              | 10.0.0.5              |
| DNS 2              | 10.0.0.6              |
| Gateway            | 10.0.0.1              |

## DNS Configuration

The client uses the internal DNS servers provided by the Domain Controllers:

- TW-DC01 — 10.0.0.5
- TW-DC02 — 10.0.0.6

Using the internal DNS servers is required for reliable Active Directory domain discovery.

## Domain Join

The client was successfully joined to:

`techwork.local`

After joining the domain, the system was restarted and domain functionality was verified.

## Validation

The following commands can be used to verify the configuration:

```powershell
ipconfig /all
nslookup techwork.local
whoami
```
