# Deployment Guide: Action1 Linux Agent Rollout & Wazuh Telemetry Verification

## Overview
This guide documents the end-to-end deployment of the **Action1** endpoint management agent on the lab's Ubuntu Linux target (`ubuntuvictim` / Wazuh agent `deb64linux_agent`, `192.168.126.50`) from the management node (`Kali-AttackerNSiem`, `192.168.126.25`), followed by automated patch remediation and SIEM log verification in **Wazuh**.

---

## Prerequisites & Network Preparation
- **Dual-Adapter Configuration:** Ensure **Adapter 1 (NAT)** is temporarily enabled on `Kali-AttackerNSiem` (and `ubuntuvictim` for package/update downloads) to reach the Action1 cloud console and Ubuntu repositories while keeping **Adapter 2 (`intnet` – `192.168.126.0/24`)** active for internal SSH/SCP transfers and Wazuh SIEM telemetry.
- **Active SIEM Monitoring:** Keep the Wazuh Manager dashboard open on `Kali-AttackerNSiem` to observe real-time authentication, privilege escalation, package installation, and lifecycle events generated on `deb64linux_agent`.

---

## Step 1: Download the Action1 `.deb` Installer on Kali
1. Log into the Action1 console on `Kali-AttackerNSiem`. Prior to enrollment, the dashboard reflects only the existing Windows Server endpoint (`1` endpoint) and pending baseline metrics.
   ![Action1 Initial Dashboard](../screenshots/01-action1-initial-dashboard.png)
2. Click **+ Install Agent** in the top-right corner (or **Endpoints > Add Agent**) and select the **Linux (Debian/Ubuntu `.deb`)** package to download `action1_agent(<ORG_ID>).deb` to `~/Downloads`.

---

## Step 2: Transfer the Installer via `scp` to Ubuntu
1. Open a terminal on `Kali-AttackerNSiem`, navigate to `~/Downloads`, and confirm connectivity to `ubuntuvictim` (`192.168.126.50`).
2. Transfer the `.deb` package to `/tmp/` on `ubuntuvictim` using `scp` (escaping the parentheses in the filename):
   ```bash
   cd ~/Downloads
   scp ./action1_agent\(<ORG_ID>\).deb ubuntuvictim@192.168.126.50:/tmp/
   ```
3. Accept the host key prompt (`yes`) and enter the `ubuntuvictim` account password to complete the transfer.
   ![Kali SCP Transfer and Ping Verification](../screenshots/02-kali-scp-transfer.png)
4. **Wazuh SIEM Verification (`sshd`):** In Wazuh Threat Hunting (`deb64linux_agent`), verify that the incoming SCP session triggered **Rule `5715`** (`sshd: authentication success`, mapped to **MITRE ATT&CK `T1078` & `T1021`** – *Valid Accounts, Remote Services*), logging source IP `192.168.126.25` authenticating as `ubuntuvictim`.
   ![Wazuh SSHD Authentication Success Log](../screenshots/03-wazuh-sshd-auth-success.png)
   ![Wazuh SSHD Rule 5715 Metadata](../screenshots/03b-wazuh-sshd-auth-success-rule.png)

---

## Step 3: Stage & Install the Action1 Agent on Ubuntu
1. Switch to `ubuntuvictim`, verify the package arrived in `/tmp/`, and move it to `~/Downloads`:
   ```bash
   cd /tmp/
   ls
   mv /tmp/action1_agent\(<ORG_ID>\).deb ~/Downloads
   ls
   ```
   ![Ubuntu Staging Installer from /tmp to Downloads](../screenshots/04-ubuntu-move-deb-tmp.png)
2. Update the local `apt` package index and install the Action1 `.deb` package:
   ```bash
   cd ~/Downloads
   sudo apt update
   sudo apt install ./action1_agent\(<ORG_ID>\).deb
   ```
3. Confirm that `dpkg` unpacks `action1-agent` (`1.67.82.1`) and enables/starts `action1_agent.service`.
   ![Ubuntu Action1 Agent Installation Output](../screenshots/05-ubuntu-apt-install-action1.png)
4. **Wazuh SIEM Verification (`sudo` & `dpkg`):** Inspect the Wazuh alerts generated during staging and installation:
   - **Rule `5402` (`Successful sudo to ROOT executed`, mapped to MITRE ATT&CK `T1548.003` – *Sudo and Sudo Caching*):** Captures `ubuntuvictim` invoking `/usr/bin/apt update` with `root` privileges.
     ![Wazuh Sudo Apt Update Log](../screenshots/06-wazuh-sudo-apt-update.png)
     ![Wazuh Sudo Apt Update Rule 5402 Metadata](../screenshots/06b-wazuh-sudo-apt-update-rule.png)
   - **Rule `2902` (`New dpkg (Debian Package) installed.`):** Parses `/var/log/dpkg.log` to confirm `status installed action1-agent:amd64 1.67.82.1`.
     ![Wazuh DPKG Action1 Install Log](../screenshots/07-wazuh-dpkg-action1-installed.png)
     ![Wazuh DPKG Rule 2902 Metadata](../screenshots/07b-wazuh-dpkg-action1-installed-rule.png)

---

## Step 4: Verify Endpoint Enrollment & Deploy Automated Updates
1. Return to the Action1 console on `Kali-AttackerNSiem` and navigate to **Endpoints**. `ubuntuvictim` (`Ubuntu 26.04.1 LTS (Resolute Raccoon)`) now appears as **Connected**, reporting **7 non-critical** missing updates and **Reboot: Required**.
   ![Action1 Endpoints Showing Ubuntu Enrolled and 7 Missing Updates](../screenshots/08-action1-ubuntu-endpoint-enrolled.png)
2. Select `ubuntuvictim` and click **Deploy Updates** to launch the `"Deploy Updates: All"` automation. Monitor **Automation > History** as Action1 queues (`Pending`), downloads, and installs the missing packages (`gir1.2-javascriptcoregtk-4.1`, `gir1.2-webkit2-4.1`, `libjavascriptcoregtk`, and `ubuntu-drivers-common`).
   ![Action1 Automation History Pending](../screenshots/09-action1-automation-pending.png)
   ![Action1 Automation History Completed](../screenshots/10-action1-automation-completed.png)
3. **Wazuh Vulnerability Detector Correlation:** During the patching window, Wazuh's vulnerability detector module simultaneously flags kernel-level CVEs (such as `CVE-2026-99227`, base score `7.8`, affecting `linux-image-7.0.0-44-generic`) on `deb64linux_agent`, demonstrating dual visibility between Action1 patch management and Wazuh vulnerability scanning.
   ![Wazuh Vulnerability Detector CVE Alert](../screenshots/11-wazuh-cve-vulnerability-alert.png)
4. **Local `apt` History Verification:** On `ubuntuvictim`, inspect `/var/log/apt/history.log` (`tail -n 30 /var/log/apt/history.log`) to confirm both the manual `action1-agent` installation and the automated `--only-upgrade` commands executed by Action1.
   ![Ubuntu APT History Log Verification](../screenshots/12-ubuntu-apt-history-log.png)

---

## Step 5: Manual Reboot & Post-Boot Verification
1. After the updates complete, `ubuntuvictim` shows `Missing Updates: Up-to-date` while retaining `Reboot: Required`. Attempting to trigger a reboot directly from the Action1 console displays *"Reboot is currently not supported on Linux endpoints."*
   ![Action1 Linux Reboot Not Supported Modal](../screenshots/13-action1-reboot-not-supported.png)
2. Execute the reboot manually on `ubuntuvictim` (`sudo reboot`), wait for the VM to restart, and refresh the Action1 **Endpoints** page to confirm **Status: Connected**, **Reboot: Not required**, and **Missing Updates: Up-to-date**.
   ![Action1 Endpoint Up-to-Date and Reboot Not Required](../screenshots/14-action1-post-reboot-clean.png)
3. **Wazuh SIEM Reboot & Agent Lifecycle Verification:** Confirm the reboot sequence across Wazuh alerts on `deb64linux_agent`:
   - **Rule `5402` (`Successful sudo to ROOT executed`, mapped to MITRE ATT&CK `T1548.003` – *Sudo and Sudo Caching*):** Logs `ubuntuvictim` executing `/usr/sbin/reboot -i now` as `root`.
     ![Wazuh Sudo Reboot Command Log](../screenshots/15-wazuh-sudo-reboot-log.png)
     ![Wazuh Sudo Reboot Rule 5402 Metadata](../screenshots/15b-wazuh-sudo-reboot-log-rule.png)
   - **Agent Lifecycle & Baseline Checks:** The Threat Hunting event stream records `Wazuh agent stopped`, followed ~58 seconds later by `Wazuh agent started`, `Listened ports status (netstat) changed`, and post-boot `Host-based anomaly detection event (rootcheck)` checks, verifying the endpoint came back online cleanly.
     ![Wazuh Agent Stop Start and Rootcheck Timeline](../screenshots/16-wazuh-agent-reboot-lifecycle.png)
