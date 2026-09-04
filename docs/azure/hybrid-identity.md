# Microsoft Entra ID Hybrid Identity

## Overview

Phase 7 of the project introduces hybrid identity between the
On-Premises Active Directory environment and Microsoft Entra ID.

The On-Premises Active Directory domain is `techwork.local`.
Microsoft Entra Cloud Sync is used to synchronize selected Active
Directory users with Microsoft Entra ID.

The goal of this phase is to allow selected On-Premises identities
to also exist in Microsoft Entra ID while keeping Active Directory
as the original identity source.

---

## Architecture

The Hybrid Identity architecture connects the existing
On-Premises Active Directory environment with Microsoft Entra ID.

`TW-DC02` is used as the Microsoft Entra Cloud Sync provisioning
server. The provisioning agent communicates with the local Active
Directory domain and sends synchronization data securely to
Microsoft Entra ID over HTTPS.

This design allows the On-Premises environment to remain responsible
for managing user identities while selected accounts are synchronized
to the cloud.

---

## Network Validation

Before configuring Microsoft Entra Cloud Sync, the required network
connectivity from `TW-DC02` was verified.

LDAP connectivity to the Active Directory environment was successfully
tested on TCP port `389`, and Global Catalog connectivity was verified
on TCP port `3268`.

Outbound HTTPS connectivity to Microsoft Entra ID was also successfully
tested on TCP port `443`.

DNS resolution for Microsoft Entra services was working correctly.
These tests confirmed that `TW-DC02` had the required connectivity for
the provisioning agent.

---

## Microsoft Entra Provisioning Agent

The Microsoft Entra Provisioning Agent was installed on `TW-DC02`.

The agent provides the connection between the local Active Directory
environment and Microsoft Entra Cloud Sync.

During configuration, the agent was registered with the Microsoft
Entra tenant using a dedicated cloud administrator account with the
Hybrid Identity Administrator role.

After registration, the agent status in Microsoft Entra ID was
successfully verified as `Active`.

---

## Service Account

During the provisioning agent setup, a Group Managed Service Account
(gMSA) was created for the Cloud Sync service.

The configured account is:

`techwork.local\provAgentgMSA`

This service account is used by the provisioning agent to communicate
securely with Active Directory without storing a normal user password
for the service.

The Domain Administrator account was only required during the initial
configuration process.

---

## Active Directory Scope

A dedicated Organizational Unit named `AzureSync` was created in
Active Directory.

The purpose of this OU is to control which Active Directory objects
are allowed to synchronize with Microsoft Entra ID.

The test user `Jolia Miller` was moved into this OU.

This design prevents built-in accounts, administrative users, and other
unnecessary Active Directory objects from being synchronized to the
cloud.

---

## Cloud Sync Configuration

A Microsoft Entra Cloud Sync configuration was created for the
`techwork.local` Active Directory domain.

Password Hash Synchronization was enabled so that synchronized users
can use their synchronized credentials with Microsoft Entra ID.

The synchronization scope was restricted to the `AzureSync`
Organizational Unit.

This ensures that only explicitly selected users are included in the
Hybrid Identity synchronization process.

---

## Attribute Mapping

The default Microsoft Entra Cloud Sync attribute mappings were reviewed
during configuration.

No custom mappings were required for the current project.

The standard mappings are sufficient for synchronizing the required
user information from Active Directory to Microsoft Entra ID.

Keeping the default mappings also reduces unnecessary complexity in
the lab environment.

---

## Provision on Demand Test

Before enabling automatic synchronization, the `Provision on demand`
feature was used to test the configuration with the user `Jolia Miller`.

The test successfully imported the Active Directory object, confirmed
that the user was inside the configured synchronization scope, matched
the source and target information, and successfully created the user
in Microsoft Entra ID.

This test confirmed that the provisioning agent, OU scope, attribute
mapping, and communication with Microsoft Entra ID were functioning
correctly.

---

## Current Status

The Microsoft Entra Cloud Sync environment has been successfully
prepared and tested.

The provisioning agent on `TW-DC02` is active, the
`techwork.local` domain is connected, the `AzureSync` OU is configured
as the synchronization scope, and Password Hash Synchronization is
enabled.

The Provision on Demand test for `Jolia Miller` completed successfully
and the user was created in Microsoft Entra ID.

The remaining step is to enable the full Cloud Sync configuration and
verify normal automatic synchronization and cloud sign-in.
