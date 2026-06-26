# Intune-BitLocker-Remediation

Fix for Intune BitLocker remediation script failures caused by user privilege and UAC constraints.

# Microsoft Intune: Resolving BitLocker Script Remediation Failures (UAC Block)

---

## 🛠️ About Microsoft Intune

[Microsoft Intune](https://www.microsoft.com/en-us/security/business/microsoft-intune) is a cloud-based unified endpoint management (UEM) service that functions as the central security and configuration engine for modern corporate fleets. It allows organizations to enforce rules, deploy applications, and manage security posture across Windows, macOS, iOS, and Android devices remotely.

### Core Architecture Pillars

1. **Mobile Device Management (MDM):** Complete, hardware-level control over organization-owned devices. This allows administrators to push global security configurations, monitor hardware baselines, and require full disk encryption (such as BitLocker).
2. **Mobile Application Management (MAM):** Sandboxing and data protection at the individual application layer (e.g., securing corporate data inside Microsoft Teams or Outlook without touching a user's personal files on a BYOD device).
3. **Identity & Conditional Access Integration:** Intune continuously feeds real-time device health and compliance telemetry back to Microsoft Entra ID. If a device drops out of compliance, access to company resources is instantly gated.

### High-Level Service Architecture

The diagram below outlines how the core components of Microsoft Intune interface with Microsoft Entra ID and cloud infrastructure to manage and secure modern device endpoints:

[![Microsoft Intune Architecture Diagram](https://github.com/Essie-Wabomba-dmd/Intune-BitLocker-Remediation/raw/main/intune-architecture.png)](/Essie-Wabomba-dmd/Intune-BitLocker-Remediation/blob/main/intune-architecture.png)

*Image reference: Overview of cloud-native device enrollment and endpoint configuration workflows.*

### The Role of Remediation Scripts

While Intune natively tracks compliance, environments frequently experience synchronization or policy evaluation gaps. Custom PowerShell remediation scripts allow administrators to detect and silently fix client-side discrepancies in the background. Ensuring these scripts execute under the correct permission context (such as `NT AUTHORITY\SYSTEM`) ensures that automated maintenance succeeds without requiring end-user interaction or administrative elevation.

## 📌 Problem Overview

When deploying PowerShell scripts via Microsoft Intune to remediate BitLocker encryption or back up recovery keys on endpoints, deployments can encounter widespread failures. For example, initial dashboard metrics can show massive execution errors across a targeted fleet.

Although endpoints have physical encryption active, the remediation script repeatedly fails to execute or fetch critical BitLocker metadata.

---

## 💼 Real-World Case Study

This solution documents a specific, recurring issue I resolved in a production environment as a Cloud Delivery Engineer.

### The Symptom

- **Environment:** Endpoints managed entirely via Microsoft Intune.
- **The Issue:** Randomly throughout the day or every morning, a subset of 1 to 3 machines would flag a persistent **"Remediation Failed"** compliance error.
- **Target Behavior:** This issue specifically targets **random, existing devices in the fleet during routine check-ins**, rather than newly enrolled or "first-time" setups which typically pass initial deployment checks without issue.
- **The Paradox:** BitLocker was confirmed active on the affected devices — the compliance error was a permissions/reporting issue, not an actual encryption gap.

### Previous Manual Workarounds

Before implementing the system-context fix, support teams had to rely on temporary administrative overhead to clear the error:

1. Manually initiating a device sync from the backend.
2. Renaming the device inside Microsoft Intune to force policy re-evaluation.

### The Automated Resolution

Instead of relying on these manual syncs, shifting the script execution context away from user credentials to **`NT AUTHORITY\SYSTEM`** (as documented below) completely automates the process. It ensures the background agent can seamlessly escrow keys and evaluate compliance every single time without administrative intervention.

## 💻 Deployed PowerShell Code

The following automated scripts are executed sequentially on the target endpoints to fetch active BitLocker key protectors and securely escalate them to Azure Active Directory (AAD):

```powershell
# Step 1: Identify and target the unique ID of the Recovery Password key protector
$KeyPair = (Get-BitLockerVolume -MountPoint "C:").KeyProtector | Where-Object {$_.KeyProtectorType -eq 'RecoveryPassword'}

# Step 2: Push the identified BitLocker recovery key into Entra ID / Azure AD
BackupToAAD-BitLockerKeyProtector -MountPoint "C:" -KeyProtectorId $KeyPair.KeyProtectorId
```

---

## 🔍 Backend Process & Permissions Context Diagram

The backend architecture diagram below clarifies how Intune interacts with the Windows subsystem and why the permission context makes or breaks this specific deployment.

[![Backend Architecture Diagram](https://github.com/Essie-Wabomba-dmd/Intune-BitLocker-Remediation/raw/main/bitlocker-remediation-permissions-context.png)](/Essie-Wabomba-dmd/Intune-BitLocker-Remediation/blob/main/bitlocker-remediation-permissions-context.png)

### 📋 Detailed Diagram Explanation

The diagram splits the deployment behavior into two distinct tracks to show the direct correlation between your Intune script settings and the endpoint's operating system behavior:

#### **Scenario A: Deployment Failure (UAC Block)**

- **The Configuration:** This track represents the environment when **"Run this script using the logged on credentials"** is set to **Yes**.
- **The Security Barrier:** Even if the targeted end-user has local administrator rights, Windows launches background Intune scripts inside a non-elevated user token (Standard Integrity Level).
- **The Silent Block:** Because the script executes silently in the background, it cannot trigger a visible User Account Control (UAC) prompt ("Run as Administrator"). As a result, the script is denied access when attempting to query the high-integrity **BitLocker Configuration Service Provider (CSP)** layer, ending in a "Remediation Failed" error.

#### **Scenario B: Deployment Success (SYSTEM Context)**

- **The Configuration:** This track represents the optimal setup where **"Run this script using the logged on credentials"** is set to **No**.
- **Bypassing the User:** By disabling the user context, the Intune Management Extension agent shifts execution entirely to the machine's local system profile (**`NT AUTHORITY\SYSTEM`**), climbing straight to a High Integrity Level.
- **Seamless Escalation:** Operating as SYSTEM bypasses all user-level UAC blocks natively. The script instantly penetrates the system core, gains access to the BitLocker modules, extracts the keys, and successfully uploads them to the cloud—resulting in full device compliance.

---

## 🛠️ The Fix: Intune Script Settings Configuration

To implement Scenario B and resolve the failures, adjust the **Script Settings** within the Microsoft Intune dashboard to the following parameters:

| Setting                                             | Configuration | Technical Purpose                                                                                                             |
| ---------------------------------------------------- | ------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| **Run this script using the logged on credentials** | **No**        | Forces the script to bypass the user context entirely and run within the elevated **`NT AUTHORITY\SYSTEM`** security context. |
| **Enforce script signature check**                  | **No**        | Allows execution of custom remediation scripts without requiring internal CA code-signing certificates.                       |
| **Run script in 64-bit PowerShell Host**             | **Yes**       | Ensures native 64-bit BitLocker cmdlets load properly, preventing Windows SysWOW64 file system redirection errors.            |

---

## 📈 Final Results

After saving these settings, devices re-evaluated the policy on their next sync cycle and compliance errors cleared fleet-wide.
