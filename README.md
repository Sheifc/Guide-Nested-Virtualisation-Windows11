# Guide: Nested Virtualization on Windows (UEFI)

## Prerequisites (strongly recommended)

1. **Check if your system is UEFI or Legacy**

   * Win+R → `msinfo32` → Enter → look for **“BIOS Mode.”**

     * **UEFI** → follow this guide.
     * **Legacy/CSM** → *skip step 6 (UEFI lock)*; the rest still applies.

2. **Safety**

   * Sign in with an **Administrator** account.
   * Keep your **BitLocker recovery key** handy.
   * (Optional) Create a **restore point** or export the registry keys you’ll modify:

     * Win+R → `regedit` → Enter → right-click each key → **Export**.

---

## Single, streamlined flow for Windows with UEFI

1. **Turn off Windows features that load the hypervisor**

   * “Turn Windows features on or off”: uncheck
     **Hyper-V**, **Virtual Machine Platform (VMP)**, and **Windows Subsystem for Linux (WSL)**.
   * Click OK and **restart** if prompted; if not, you’ll restart later.

2. **Disable “Memory Integrity” (HVCI/Core Isolation)**

   * Settings → **Privacy & security** → **Windows Security** → **Open Windows Security** →
     **Device security** → **Core isolation details** → **Memory integrity: Off**.

3. **Prevent hypervisor/VBS from loading at boot** (CMD **Admin**)

   ```bat
   bcdedit /set hypervisorlaunchtype off
   bcdedit /set vsmlaunchtype off
   ```

4. **Registry: disable Credential Guard and clear VBS bits** (CMD **Admin**)

   ```bat
   reg add "HKLM\SYSTEM\CurrentControlSet\Control\Lsa" /v LsaCfgFlags /t REG_DWORD /d 0 /f
   reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\DeviceGuard" /v LsaCfgFlags /t REG_DWORD /d 0 /f

   reg delete "HKLM\SYSTEM\CurrentControlSet\Control\DeviceGuard" /v EnableVirtualizationBasedSecurity /f
   reg delete "HKLM\SYSTEM\CurrentControlSet\Control\DeviceGuard" /v RequirePlatformSecurityFeatures /f
   ```

5. **Group Policy (VBS) → Disabled**

   * Win+R → `gpedit.msc` → Enter.
   * **Computer Configuration** → **Administrative Templates** → **System** → **Device Guard** →
     **Turn On Virtualization Based Security** = **Disabled**.
   * (Optional) CMD **Admin**: `gpupdate /force`.

6. **UEFI lock (only if your firmware is UEFI)**

   * CMD **Admin**:

   ```bat
   mountvol X: /s
   copy %WINDIR%\System32\SecConfig.efi X:\EFI\Microsoft\Boot\SecConfig.efi /Y

   bcdedit /create {0cb3b571-2f2e-4343-a879-d86a476d7215} /d "DebugTool" /application osloader
   bcdedit /set {0cb3b571-2f2e-4343-a879-d86a476d7215} path "\EFI\Microsoft\Boot\SecConfig.efi"
   bcdedit /set {0cb3b571-2f2e-4343-a879-d86a476d7215} device partition=X:
   bcdedit /set {0cb3b571-2f2e-4343-a879-d86a476d7215} loadoptions DISABLE-LSA-ISO,DISABLE-VBS
   bcdedit /set {bootmgr} bootsequence {0cb3b571-2f2e-4343-a879-d86a476d7215}

   mountvol X: /d
   ```

   * **Restart** → when the UEFI prompt appears, press **F3** and then **Enter** to confirm.

7. **Quick verification**

   * Win+R → `msinfo32` → **Virtualization-based security: Not enabled/Off**.
   * CMD **Admin** → `bcdedit /enum {current}` → should show
     **hypervisorlaunchtype Off** and **vsmlaunchtype Off**.
   * PowerShell **Admin**:

     ```powershell
     Get-CimInstance -Namespace root\Microsoft\Windows\DeviceGuard -ClassName Win32_DeviceGuard |
       Select VirtualizationBasedSecurityStatus, SecurityServicesConfigured, SecurityServicesRunning
     ```

     * Expected: `VirtualizationBasedSecurityStatus = 0` and empty/0 lists for Services.

8. **VMware Workstation: enable nested virtualization (once)**

   * With the VM **powered off** → **Settings → Processors → Virtualize Intel VT-x/EPT**.
   * If it’s missing, edit the `.vmx`: `vhv.enable = "TRUE"`.
   * Assign **4–8 vCPU** and **8–16 GB RAM**.
   * Networking: **NAT** or **Host-only** (avoid **Bridged** for malware labs).
   * In a Linux guest:

     ```bash
     egrep -o 'vmx|svm' /proc/cpuinfo | head   # Expect 'vmx' (Intel) or 'svm' (AMD)
     sudo apt update
     sudo apt install -y cpu-checker qemu-kvm libvirt-daemon-system
     kvm-ok   # "KVM acceleration can be used"
     ```

---

## Optional extra (diagnostics)

* Win+R → `eventvwr.msc` → **Windows Logs → System**: confirm there are **no** **VBS/HVCI** errors at boot.

---

## Final mini-checklist

* [ ] Hyper-V / VMP / WSL **unchecked** in *Windows Features*.
* [ ] **Memory Integrity** (HVCI) **Off**.
* [ ] `bcdedit`: **hypervisorlaunchtype off** and **vsmlaunchtype off**.
* [ ] **VBS policy** = **Disabled** under Device Guard.
* [ ] (UEFI) **UEFI lock** applied and confirmed with **F3**.
* [ ] `msinfo32`: **Virtualization-based security: Not enabled**.
* [ ] VMware: **Virtualize Intel VT-x/EPT** (or `vhv.enable="TRUE"`).


