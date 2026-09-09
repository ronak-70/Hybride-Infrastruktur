# Backup and Recovery

## Overview

Phase 12 of the Hybrid Infrastructure project focuses on backup and recovery capabilities for the Azure virtual machine workload.

Azure Backup was configured for the `tw-app-01` Linux virtual machine using a Recovery Services vault.

The implementation covered:

```text
Recovery Services Vault deployment
VM backup policy configuration
Azure VM backup enablement
On-demand backup execution
Recovery point validation
File Recovery testing
Recovery troubleshooting
Outbound connectivity hardening
```

---

## Recovery Services Vault

A dedicated Recovery Services vault was created for the project.

```text
Vault name: rsv-hybrid-prod
Resource group: rg-hybrid-infrastructure
Region: Poland Central
Protected workload: tw-app-01
```

The vault is located in the same Azure region as the protected virtual machine.

---

## Backup Policy

A dedicated Standard Azure VM backup policy was created.

```text
Policy name: policy-daily-vm-backup
Policy type: Standard
Backup frequency: Daily
Backup time: 02:30 UTC
Daily retention: 30 days
Instant Restore retention: 2 days
```

The policy was assigned to:

```text
Virtual machine: tw-app-01
```

Future disks were configured to be included in backup protection.

---

## Backup Enablement

Azure Backup protection was successfully enabled for `tw-app-01`.

The backup configuration completed successfully.

```text
Backup item: tw-app-01
Recovery Services vault: rsv-hybrid-prod
Backup policy: policy-daily-vm-backup
Backup Pre-Check: Passed
```

An on-demand backup was triggered to validate the configuration without waiting for the scheduled backup window.

---

## Recovery Point Validation

Recovery points were successfully created for the protected virtual machine.

Observed recovery points included:

```text
File-system consistent recovery point
Crash-consistent recovery point
Snapshot recovery point
Snapshot and Vault-Standard recovery point
```

A file-system consistent recovery point was selected for recovery testing.

This confirmed that:

```text
Azure Backup protection: Working
Backup policy assignment: Working
Recovery point creation: Working
Recovery Services vault integration: Working
```

---

## File Recovery Test

Azure Backup File Recovery was tested using a recovery point from `tw-app-01`.

For Linux virtual machines, Azure Backup generates a Python recovery script together with a temporary recovery password.

The recovery script was downloaded from the Azure portal.

Because `tw-app-01` does not have a public IP address and the Site-to-Site VPN was unavailable due to the external network environment, the script could not be transferred directly using SSH or SCP.

A temporary Azure Storage container was therefore used to transfer the recovery script to the VM.

The workflow was:

```text
Azure Backup Recovery Point
        |
        v
Generate Linux Recovery Script
        |
        v
Temporary Azure Storage Blob
        |
        v
Azure VM Run Command
        |
        v
tw-app-01
```

The script was stored on the VM at:

```text
/home/azureuser/recovery/file-recovery.py
```

---

## Recovery Prerequisite Validation

The Linux VM was checked for the prerequisites required by the Azure Backup File Recovery script.

The following components were validated:

```text
Operating system: Ubuntu 22.04
Python: 3.10.12
open-iscsi: Installed
iscsiadm: Available
iscsid service: Active
lshw: Installed
TLS 1.2: Supported
```

The recovery script was executable and successfully started on the VM.

---

## Outbound Connectivity Issue

During initial recovery testing, the VM was unable to reach required external endpoints.

The following connectivity failures were observed:

```text
download.microsoft.com:443 - Timeout
Azure Backup Recovery Service:3260 - Blocked
Ubuntu package repository:80 - Timeout
```

DNS resolution was working correctly.

The effective route table showed:

```text
0.0.0.0/0 -> Internet
```

The effective Network Security Group configuration also contained:

```text
AllowInternetOutBound
```

This confirmed that the NSG and Azure route configuration were not directly blocking outbound connectivity.

---

## NAT Gateway Implementation

To provide explicit outbound connectivity for `snet-workload`, an Azure NAT Gateway was deployed.

Configuration:

```text
NAT Gateway: nat-hybrid-workload
Subnet: snet-workload
Virtual network: vnet-hybrid-prod
Public IP: 134.112.209.110
```

The public IP was an existing Standard Public IP that had previously been used by the removed VPN Gateway and was no longer associated with another resource.

After the NAT Gateway was attached to `snet-workload`, outbound connectivity was successfully validated.

The VM reported:

```text
Public outbound IP: 134.112.209.110
HTTPS TCP/443: Working
Azure Backup TCP/3260: Open
```

This confirmed that the NAT Gateway successfully restored explicit outbound connectivity for the workload subnet.

---

## File Recovery Troubleshooting

After network connectivity was corrected, the File Recovery script progressed further and successfully started the Azure Backup Secure TCP tunnel.

The recovery logs confirmed:

```text
SecureTCPTunnel started successfully
Recovery Service DNS resolution successful
TCP 3260 connectivity available
Recovery password length accepted
```

However, the iSCSI discovery process failed during target authentication.

The recovery log contained:

```text
iscsiadm: Login failed to authenticate with target
iscsiadm: discovery login to 127.0.0.1 rejected: initiator failed authorization
iSCSI login failed due to authorization failure
```

Additional troubleshooting was performed.

Existing recovery sessions were cleaned using the Azure Backup recovery script.

The local iSCSI state was verified:

```text
iscsiadm: No active sessions.
```

Old Secure TCP tunnel sessions and Azure Backup ILR state were also investigated.

Fresh recovery points, newly generated recovery scripts, and newly generated temporary recovery passwords were tested.

Despite these actions, the recovery script continued to fail during iSCSI target authentication.

---

## Recovery Test Result

The Azure VM backup implementation was successfully validated through creation of usable recovery points.

The File Recovery workflow was also tested through the network, authentication preparation, Secure TCP tunnel, and iSCSI discovery stages.

The final file mount operation did not complete because the Azure Backup iSCSI target rejected authentication during discovery.

Final result:

```text
Recovery Services Vault: Successful
Backup policy: Successful
VM backup protection: Successful
On-demand backup: Successful
Recovery point creation: Successful
File-system consistent recovery point: Available
Vault-Standard recovery point: Available

Recovery script generation: Successful
Recovery script transfer: Successful
Linux prerequisites: Validated
Outbound HTTPS connectivity: Successful
Azure Backup TCP/3260 connectivity: Successful
Secure TCP tunnel: Successful

iSCSI target authentication: Failed
Recovery volume mount: Not completed
Individual file restore: Not completed
```

The recovery limitation was documented rather than presenting the restore operation as successful.

---

## Backup and Recovery Architecture

```text
tw-app-01
    |
    | Azure Backup
    v
rsv-hybrid-prod
    |
    v
Recovery Points
    |
    +---------------------------+
    |                           |
    v                           v
VM / Disk Restore          File Recovery
                                |
                                v
                       Linux Recovery Script
                                |
                                v
                         Secure TCP Tunnel
                                |
                                v
                           iSCSI Target
                                |
                                X
                     Authentication Failure
```

---

## Security Considerations

Temporary recovery passwords and SAS URLs were treated as sensitive credentials and were not stored in the GitHub repository.

The Azure VM itself remained without a public IP address.

Outbound access was provided through the NAT Gateway rather than by exposing the VM directly to the internet.

SSH inbound access remained restricted by the existing Network Security Group configuration.

Temporary recovery artifacts should be removed after testing, including:

```text
Temporary recovery scripts
Temporary SAS tokens
Recovery mount sessions
Unused recovery files in Azure Storage
```

---

## Production Recommendations

For a production environment, the following additional controls should be considered:

```text
Regular restore testing
Automated backup health monitoring
Backup job alerting
Documented Recovery Time Objective (RTO)
Documented Recovery Point Objective (RPO)
Periodic full VM or disk restore tests
Protected recovery operations
Backup vault security monitoring
Centralized diagnostic logging
Separate recovery environment for restore validation
```

A backup should not be considered fully validated solely because backup jobs succeed. Periodic restore testing should also be part of the operational recovery process.

---

## Current Status

Phase 12 backup implementation status:

```text
Recovery Services Vault: Deployed
VM backup policy: Configured
tw-app-01 backup protection: Enabled
On-demand backup: Executed
Recovery points: Created and verified
NAT Gateway outbound connectivity: Configured and validated

File Recovery test: Attempted
Recovery script: Successfully executed
iSCSI discovery: Authentication failure
File mount: Not completed
Restore limitation: Documented
```

Phase 12 is therefore considered complete for the lab scope with a documented File Recovery limitation.

A full VM or disk restore remains a recommended future validation activity.
