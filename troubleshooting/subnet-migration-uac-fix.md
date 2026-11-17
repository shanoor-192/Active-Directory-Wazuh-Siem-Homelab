# Troubleshooting: RFC-1918 Subnet Migration & UAC Domain-Trust Error

## Environment Context
* **Hypervisor:** Oracle VirtualBox (Isolated Internal Network)
* **Previous Subnet:** `10.x.x.x` (Class A RFC-1918 range)
* **Target Subnet:** `192.168.126.0/24` (Class C RFC-1918 range)
* **Affected Systems:** Windows Server 2022 (`NJ-DC-01` – `192.168.126.60`) and domain-joined Windows 11 Pro workstations (`Desktop01` – `192.168.126.61`, `Desktop02` – `192.168.126.62`)

## Symptom
During a manual network migration from a `10.x.x.x` subnet to `192.168.126.0/24`, attempting to modify the static IPv4 settings on the domain-joined Windows 11 Pro workstations triggered a User Account Control (UAC) elevation prompt. Entering valid Domain Admin credentials failed with an authentication error indicating the domain could not be contacted or the workstation was not part of the domain.

## Root Cause
The static IP migration was performed on the **Windows Server 2022 Domain Controller (`NJ-DC-01`) first**. Once the Domain Controller moved to `192.168.126.60`, the Windows 11 workstations—still configured on the legacy `10.x.x.x` subnet—lost Layer 3 reachability to the Domain Controller and Active Directory DNS. Because UAC relies on network connectivity to the Domain Controller to validate Domain Admin credentials, elevation requests timed out and failed domain-trust verification.

## Resolution
1. Authenticated at the UAC elevation prompt using each workstation's **Local Administrator** account (`.\Administrator`), which validates against the local Security Account Manager (SAM) database without requiring Domain Controller reachability.
2. Opened **Network Connections (`ncpa.cpl`)** and updated the static IPv4 address (`192.168.126.61` / `.62`), subnet mask (`255.255.255.0`), and preferred DNS server (`192.168.126.60`).
3. Verified Layer 3 connectivity and Active Directory DNS resolution using `ping` and `nslookup` against `NJ-DC-01`.
4. Confirmed domain credential authentication and Group Policy processing (`gpupdate /force`) were fully restored so Wazuh agent deployment could proceed.

## Key Takeaway
When migrating static IP subnets in an Active Directory environment, member endpoints should either be migrated before the Domain Controller, or local administrator credentials must be accessible to remediate isolated network interfaces when domain trust is temporarily severed.
