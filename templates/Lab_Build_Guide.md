# AD Detection Lab --- Step-by-Step Build Guide (All Machines)

**Hypervisor:** VMware Workstation Pro 26H1 · **Domain:** corp.lab ·
**Telemetry:** Trend Vision One XDR

Screen-by-screen build for every machine, from empty VM to
enrolled-and-snapshotted, then the pre-pentest configuration. Follow it
literally --- every wizard screen, dropdown, and checkbox is listed.

### Naming key

  -----------------------------------------------------------------------------------
  Hostname       Role           IP               RAM         Disk        Guest OS
  -------------- -------------- ---------------- ----------- ----------- ------------
  **ZION**       Domain         192.168.150.10   4 GB        40 GB       Server 2025
                 Controller +                                            
                 DNS                                                     

  **NEB**        Member server  192.168.150.20   4 GB        40 GB       Server 2025
                 (MSSQL)                                                 

  **WIN11-BASE → Workstation /  192.168.150.21   4 GB        64 GB       Windows 11
  NEO**          patient zero                                            

  **WIN11-BASE → Workstation 2  192.168.150.22   4 GB        64 GB       Windows 11
  TRINITY**                                                              

  **MORPHEUS**   Kali           192.168.150.50   4 GB        ~40 GB     Kali
                 (attacker)                                              (imported)
  -----------------------------------------------------------------------------------

### Before you start (once)

-   VMware Workstation Pro **26H1** installed (personal-use, no key).
-   **VMnet8 (NAT)** set: subnet `192.168.150.0/24`, gateway `.2`, DHCP
    **disabled** (Edit → Virtual Network Editor).
-   ISOs ready: Server 2025, Windows 11 Enterprise, Kali VMware image,
    Vision One agent installer.

### Vision One agent --- settings that enroll cleanly (every Windows host)

Installer package · Endpoint group **Sensor Only** · OS **Windows** ·
Architecture **64-bit (x86-64)** · Agent installer proxy **Direct
connect**. Run the **full** package as admin **to completion**; verify
config.json` under
`C:\Program Files (x86)\`Trend Micro\Endpoint Basecamp\`, service
Running, host **online** in Endpoint Inventory.

> **Build order:** ZION → WIN11-BASE → clone to NEO & TRINITY → NEB →
> MORPHEUS → Part 7 config.

# SECTION A --- VM Creation Wizard (screen by screen)

Follow these screens for ZION, NEB, and WIN11-BASE. Per-VM values come
from the table at the end of this section. Start: **File → New Virtual
Machine.**

1.  **Welcome:** select **Custom (advanced)** → Next.

1.  **Hardware Compatibility:** leave default (Workstation 17.x/26H1) →
    Next.
2.  **Guest OS Installation:** select **"I will install the operating
    system later"** → Next. *(Skips Easy Install, which can
    auto-configure things you don't want.)*
3.  **Select a Guest Operating System:** **Microsoft Windows**, then in
    the Version dropdown choose **Windows Server 2025** (servers) or
    **Windows 11 x64** (workstation base). → Next.
4.  **Name the Virtual Machine:** type the name (e.g. `ZION`), pick a
    folder on your NVMe → Next.
5.  **(Windows 11 only) Configure Encryption:** select **"Only the
    files needed to support a virtual TPM (.nvram, .vmss, .vmem,
    .vmsn)"** (not "All the files"). Set a **Password** (record it
    --- you can't run the VM without it), leave **"Remember the
    password... in Credential Manager"** ticked → Next. *(This is what
    gives Win 11 its required TPM 2.0.)*
6.  **Firmware Type:** select **UEFI** for everything. **Windows 11:**
    also tick **"Enable secure boot."** **Servers (ZION, NEB):** leave
    **Secure Boot unchecked** (the screen defaults to it ticked ---
    uncheck it). → Next. *(If this screen doesn't appear, set it later
    via VM → Settings → Options → Advanced.)*
7.  **Processor Configuration:** Number of processors **2**, cores per
    processor **1** (Total = **2**) → Next.
8.  **Memory:** set **4096 MB** → Next.
9.  **Network Type:** select **NAT** ("Use network address
    translation") → Next.
10. **Select I/O Controller Types:** leave **LSI Logic SAS
    (Recommended)** → Next. *(Irrelevant once the disk is NVMe, but this
    is the safe default.)*
11. **Select a Disk Type:** choose **NVMe (Recommended)** → Next.
12. **Select a Disk:** **Create a new virtual disk** → Next.
13. **Specify Disk Capacity:** Maximum disk size = **40 GB** (servers)
    or **64 GB** (Win 11). **Leave "Allocate all disk space now"
    UNticked.** Select **"Split virtual disk into multiple files."** →
    Next.
14. **Specify Disk File:** accept the default name (or match the VM
    name) → Next.
15. **Ready to Create:** click **Customize Hardware**:
    -   **CD/DVD (SATA):** select **"Use ISO image file"** →
        **Browse** → choose the correct ISO → ensure **"Connect at
        power on"** is ticked.

    -   Confirm **Memory = 4096**, **Network Adapter = NAT**.

    -   **(Win 11)** confirm a **Trusted Platform Module** is listed and
        firmware is **UEFI + Secure Boot**.

    -   **Close** → **Finish**.

**Per-VM parameters for Section A:**

  --------------------------------------------------------------------------
  VM           Step 4      Step 6        Step 7      Step 9 RAM  Step 14
               Guest OS    Encrypt/TPM   Firmware                Disk
  ------------ ----------- ------------- ----------- ----------- -----------
  ZION         Windows     (skip --- no  UEFI        4096        40 GB
               Server 2025 TPM)                                  

  NEB          Windows     (skip --- no  UEFI        4096        40 GB
               Server 2025 TPM)                                  

  WIN11-BASE   Windows 11  Yes ---       UEFI +      4096        64 GB
               x64         "only TPM    Secure Boot             
                           files" +                             
                           password                              
  --------------------------------------------------------------------------

# SECTION B --- Windows installation (screen by screen)

After the VM is created and the ISO mounted, **power on** (green ▶). If
it boots past into a "no OS / PXE" screen, power off and confirm the
ISO is connected, then power on again.

1.  **Press a key** when prompted to boot from the disc.

1.  **Language / time / keyboard** → Next.

1.  Click **Install now**.

1.  **Select edition:**
    -   Servers: **Windows Server 2025 Standard (Desktop Experience)**
        --- the GUI one (not the plain "Standard" Core).

    -   Workstation: **Windows 11 Enterprise**. → Next.

1.  **Accept the license terms** → Next.

1.  Choose **Custom: Install Windows only (advanced)**.

1.  Select the **unallocated disk** (40/64 GB) → Next. *(VMware's NVMe
    driver is built into setup --- no driver load needed.)*

1.  Let it install and reboot. On the setup screens, set a strong
    **Administrator password** (and for Win 11, complete OOBE) → Finish.

1.  To log in, send Ctrl+Alt+Del via **VM menu → Send Ctrl+Alt+Delete**
    (or Ctrl+Alt+Insert), then sign in.

# SECTION C --- VMware Tools (every Windows VM)

1.  **VM menu → Install VMware Tools** (mounts the Tools ISO).

1.  In the guest: open **File Explorer → the DVD drive → run**
    `setup64.exe` → **Typical** → finish → **reboot**.

1.  Disable Tools time sync (so it doesn't fight the clock):

-   & "C:\Program Files\VMware\VMware Tools\VMwareToolboxCmd.exe" timesync disable

# PART 1 --- ZION (Domain Controller)

1.  **Create the VM:** Section A with the **ZION** row (Server 2025,
    UEFI, 4096, 40 GB).

1.  **Install OS:** Section B (edition = Server 2025 Standard **Desktop
    Experience**).

1.  **VMware Tools:** Section C.

1.  **Static IP + rename** --- elevated PowerShell:

    -   `Get-NetAdapter` (if "not recognized":
        `Import-Module NetAdapter`). Note the name (`Ethernet0`).

    -   If it holds DHCP:
        `Set-NetIPInterface` -`InterfaceAlias` "Ethernet0" -`Dhcp` Disabled`

-   New-NetIPAddress -InterfaceAlias "Ethernet0" -IPAddress 192.168.150.10 -PrefixLength 24 -DefaultGateway 192.168.150.2
        Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses 127.0.0.1
        Rename-Computer -NewName ZION -Restart

1.  **Fix time (before enrolling the agent)** --- wrong time breaks
    agent TLS and Kerberos:

-   w32tm /config /manualpeerlist:"time.windows.com,0x9 time.google.com,0x9" /syncfromflags:manual /reliable:yes /update
        Restart-Service w32time
        w32tm /resync /rediscover
        w32tm /query /status

-   Confirm the displayed time is correct, and that the **host** clock
    is right too.

1.  **Promote to a new forest:**

-   Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
        Install-ADDSForest -DomainName corp.lab -InstallDns -SafeModeAdministratorPassword (Read-Host -AsSecureString)

    -   DSRM password must meet **complexity**
        (upper+lower+number+symbol). Confirm with **Y**. Yellow
        DNS-delegation warning is normal. Reboots; login becomes
        **CORP\\Administrator**.

1.  **DNS forwarders + verify:**

-   Add-DnsServerForwarder -IPAddress 1.1.1.1,8.8.8.8
        nslookup corp.lab          # → 192.168.150.10
        nslookup www.google.com    # → public IPs
        Get-ADDomain

1.  **Seed the directory (do all of this here, on ZION --- it's the
    DC).** Create the OU, the service account + SPN, the per-workstation
    users, and a privileged account. Use the **inline password** form so
    you don't fight the blind masked prompt:

-   # OU to hold the lab objects
        New-ADOrganizationalUnit -Name "Lab" -Path "DC=corp,DC=lab"

        # Kerberoastable service account (SPN host = NEB, the member server)
        New-ADUser -Name "svc_sql" -SamAccountName svc_sql -AccountPassword (ConvertTo-SecureString "<LAB_SQL_PASSWORD>" -AsPlainText -Force) -Enabled $true -Path "OU=Lab,DC=corp,DC=lab"
        setspn -S MSSQLSvc/neb.corp.lab:1433 corp\svc_sql

        # Per-workstation users (realistic logons for the exercise)
        New-ADUser -Name "Lab User 1" -SamAccountName labuser1 -UserPrincipalName labuser1@corp.lab -AccountPassword (ConvertTo-SecureString "<LAB_USER_PASSWORD>" -AsPlainText -Force) -Enabled $true -Path "OU=Lab,DC=corp,DC=lab"
        New-ADUser -Name "Lab User 2"    -SamAccountName labuser2     -UserPrincipalName labuser2@corp.lab     -AccountPassword (ConvertTo-SecureString "<LAB_USER_PASSWORD>" -AsPlainText -Force) -Enabled $true -Path "OU=Lab,DC=corp,DC=lab"

        # Privileged account to seed/harvest later
        New-ADUser -Name "Domain Admin (lab)" -SamAccountName dadmin -AccountPassword (ConvertTo-SecureString "<LAB_ADMIN_PASSWORD>" -AsPlainText -Force) -Enabled $true -Path "OU=Lab,DC=corp,DC=lab"
        Add-ADGroupMember "Domain Admins" dadmin

        # (Optional) bulk-populate the directory so discovery has data to find
        #   download BadBlood, then run it here on ZION

        # Verify
        Get-ADUser -Filter * -SearchBase "OU=Lab,DC=corp,DC=lab" | Select-Object Name,SamAccountName,Enabled

-   **Account → host mapping** (use these as the logon users on each
    workstation):

  -----------------------------------------------------------------------
  Host                    Logon user              Role in the story
  ----------------------- ----------------------- -----------------------
  NEO                     `corp\labuser1`    User who "opens the
                                                  payload" --- initial
                                                  access

  TRINITY                 `corp\labuser2`        Second workstation the
                                                  attacker pivots to

  NEB / dump target       `corp\dadmin`       Privileged credential
                          session                 to harvest
  -----------------------------------------------------------------------

-   Passwords must meet complexity (upper+lower+number+symbol). "The
    > specified account already exists" *after* a complexity failure
    > means it **was** created on a prior try --- reset with
    > `Set-ADAccountPassword` <user> -Reset -`NewPassword` (`ConvertTo-SecureString` "..." -`AsPlainText` -Force)`.

1.  **Install the Vision One agent:**

    -   Console → Endpoint Inventory → **Agent Installer** → set
        **Installer package / Sensor Only / Windows / 64-bit / Direct
        connect** → click the **download** icon.

    -   Copy the package to ZION, **run as Administrator, let it
        finish**.

    -   Verify config.json` exists under
        `C:\Program Files (x86)\`Trend Micro\Endpoint Basecamp\`, the
        **Endpoint Basecamp** service is Running, and **ZION shows
        online** in the console.

1.  **Snapshot:** **shut down gracefully**, then **VM → Snapshot → Take
    Snapshot** → name `ZION - clean + enrolled`.

# PART 2 --- WIN11-BASE (clone source)

> Build it clean --- **no domain join, no agent** --- so each clone gets
> a unique sensor identity.

1.  **Create the VM:** Section A with the **WIN11-BASE** row --- pay
    attention to step 6 (Encryption: "only TPM files" + password) and
    step 7 (**UEFI + Secure Boot**). At step 16 confirm a **TPM** device
    is present.

1.  **Install OS:** Section B (edition = **Windows 11 Enterprise**).

1.  **VMware Tools:** Section C (including the timesync-disable
    command).

1.  **Do NOT** join the domain or install the agent.

1.  **Shut down** (optional: run
    `sysprep` /generalize /`oobe` /shutdown` first for a cleaner
    clone).

1.  **Clone twice:**
    -   **VM → Manage → Clone** → Next.

    -   Clone source: **The current state in the virtual machine** →
        Next.

    -   Clone type: **Create a full clone** → Next. *(Linked clone is
        greyed out --- encrypted VMs can't be linked-cloned, and Win
        11's vTPM makes this one encrypted. Full clone is correct; each
        is independent, ~25--30 GB thin.)*

    -   Name **NEO**, pick a location → **Finish** → Close.

    -   Repeat for **TRINITY**.

# PART 3 --- NEO (workstation / patient zero, .21)

1.  **Power on NEO**, sign in, open **PowerShell as Administrator**.

1.  **Static IP + join (renames + reboots):**

-   New-NetIPAddress -InterfaceAlias "Ethernet0" -IPAddress 192.168.150.21 -PrefixLength 24 -DefaultGateway 192.168.150.2
        Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses 192.168.150.10
        Add-Computer -DomainName corp.lab -NewName NEO -Credential corp\Administrator -Restart

-   *(Time now syncs from ZION automatically.)*

1.  **Confirm the join** (after the reboot):

-   (Get-WmiObject Win32_ComputerSystem).Domain   # → corp.lab
        Test-ComputerSecureChannel                     # → True (trust is healthy)

1.  **Install the Vision One agent** (Sensor Only / Direct connect). Run
    to completion; confirm a **distinct** online endpoint named NEO.

1.  **Log in as the exercise user.** The local "thomas" account cloned
    from the base is harmless, but use a **domain user** for realistic
    telemetry: sign out → **Other user** → `corp\labuser1` →
    password. Confirm with `whoami` → `corp\labuser1`. *(Create
    labuser1/labuser2 first --- Part 7 identity plan.)*

1.  **Snapshot:** `NEO - clean + joined + enrolled`.

# PART 4 --- TRINITY (workstation 2, .22)

1.  Identical to Part 3 with the second address/name:

-   New-NetIPAddress -InterfaceAlias "Ethernet0" -IPAddress 192.168.150.22 -PrefixLength 24 -DefaultGateway 192.168.150.2
        Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses 192.168.150.10
        Add-Computer -DomainName corp.lab -NewName TRINITY -Credential corp\Administrator -Restart

1.  **Confirm the join:**
    `(Get-`WmiObject` Win32_ComputerSystem`).Domain` → `corp.lab`;
    `Test-ComputerSecureChannel` → `True`.

1.  **Install the agent** (distinct endpoint).

1.  **Log in for the exercise** as `corp\labuser2` (Other user →
    `corp\labuser2`); confirm `whoami` → `corp\labuser2`.

1.  **Snapshot:** `TRINITY - clean + joined + enrolled`.

> The duplicate local "thomas" user inherited from the clone is
> harmless --- the unique hostname, AD computer account, and
> (post-clone) sensor identity are what matter. Logging in as domain
> users (labuser1/labuser2) just makes the telemetry read like a real org.

# PART 5 --- NEB (member server, .20)

> **Build NEB directly --- do NOT clone ZION.** ZION is a domain
> controller; a clone of it is a *second DC* (AD database, DNS, SYSVOL,
> duplicate identity), which conflicts on the network and can't be
> cleanly demoted to a member server. If you want a reusable shortcut,
> clone a **clean, unpromoted, unjoined** Server 2025 base instead ---
> never the DC.

1.  **Create the VM:** Section A with the **NEB** row (Server 2025,
    UEFI, 4096, 40 GB).

1.  **Install OS:** Section B (Server 2025 Desktop Experience). **VMware
    Tools:** Section C.

1.  **Static IP + join:**

-   New-NetIPAddress -InterfaceAlias "Ethernet0" -IPAddress 192.168.150.20 -PrefixLength 24 -DefaultGateway 192.168.150.2
        Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses 192.168.150.10
        Add-Computer -DomainName corp.lab -NewName NEB -Credential corp\Administrator -Restart

1.  **(Optional)** install **SQL Server Express**; set the SQL service
    to log on as **corp\\svc_sql** (matches the
    `MSSQLSvc`/neb.corp.lab:1433` SPN).

1.  **Install the Vision One agent** → online. **Snapshot:**
    `NEB - clean + joined + enrolled`.

# PART 6 --- MORPHEUS (Kali attacker, .50)

1.  **Import:** **File → Open** → browse to the downloaded Kali `.`vmx`
    → Import.

1.  **VM → Settings:** Memory **4096 MB**, Processors **2**, Network
    Adapter **NAT** → OK.

1.  **Power on**, log in (default kali/kali, change it). *(If the mouse
    cursor is invisible, it's a cosmetic VMware/Kali rendering glitch
    --- everything below is keyboard/terminal, so just carry on.)*

1.  **Static networking --- modern Kali uses NetworkManager, NOT**
    `/`etc`/network/interfaces` (that file holds only loopback, which
    confirms NetworkManager owns the interface). Set it with `nmcli`:

-   # Identify the interface + connection profile name
        nmcli device status        # interface is usually  eth0
        nmcli connection show      # profile is usually  "Wired connection 1"

        # Apply static config to that profile (use the exact name shown above)
        sudo nmcli connection modify "Wired connection 1" \
          ipv4.method manual \
          ipv4.addresses 192.168.150.50/24 \
          ipv4.gateway 192.168.150.2 \
          ipv4.dns 192.168.150.10

        # Re-activate so it takes effect
        sudo nmcli connection down "Wired connection 1"
        sudo nmcli connection up "Wired connection 1"

-   If no wired profile exists, create one instead:

-   sudo nmcli connection add type ethernet ifname eth0 con-name lab-static \
          ipv4.method manual ipv4.addresses 192.168.150.50/24 \
          ipv4.gateway 192.168.150.2 ipv4.dns 192.168.150.10
        sudo nmcli connection up lab-static

-   This persists across reboots (unlike a temporary
    `ipaddr` add`).

1.  **Verify networking:**

-   ip addr show eth0        # → inet 192.168.150.50/24
        ip route                 # → default via 192.168.150.2
        ping -c3 192.168.150.10  # ZION reachable
        ping -c3 1.1.1.1         # internet via NAT
        nslookup corp.lab        # resolves via ZION

1.  **Update + confirm C2:**

-   sudo apt update && sudo apt full-upgrade -y
        msfconsole -q            # Metasploit (FIN6-faithful C2) — confirm it launches, then 'exit'

1.  **Snapshot:** `MORPHEUS - ready`.

# PART 7 --- Pre-pentest configuration (before the emulation)

Do these in order, top to bottom. This assumes the choices already made
for this lab: **the firewall is turned OFF, and the attacker
authenticates as the domain admin** `corp\dadmin`**.** Leave the
Vision One sensor running on every machine the whole time --- it is the
telemetry source.

## Step 1 --- On ZION: confirm the accounts exist

The accounts were created during the ZION build (Part 1, Stage 8). Open
**PowerShell as Administrator on ZION** and run:

    Get-ADUser -Filter * -SearchBase "OU=Lab,DC=corp,DC=lab" | Select-Object Name,SamAccountName,Enabled

You should see `svc_sql`, `labuser1`, `labuser2`, and `dadmin`, all with
**Enabled : True**. If one is missing or disabled, re-create or enable
it before continuing. Nothing else to do in this step.

## Step 2 --- On NEO: run the target-prep block

First, in the NEO system tray: **Windows Security → Virus & threat
protection → Manage settings → turn Tamper Protection OFF** (the
Defender commands below won't apply otherwise).

Then open **PowerShell as Administrator on NEO** and run this whole
block:

    # Windows Update OFF
    Stop-Service wuauserv -Force
    Set-Service wuauserv -StartupType Disabled
    New-Item -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU" -Force | Out-Null
    Set-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU" -Name NoAutoUpdate -Value 1

    # Microsoft Defender: exclude C:, stop blocking, no cloud submission
    Add-MpPreference -ExclusionPath "C:"
    Set-MpPreference -DisableRealtimeMonitoring $true
    Set-MpPreference -MAPSReporting Disabled
    Set-MpPreference -SubmitSamplesConsent NeverSend

    # Windows Firewall OFF (all profiles)
    Set-NetFirewallProfile -Profile Domain,Private,Public -Enabled False

    # Enable Remote Desktop
    Set-ItemProperty -Path "HKLM:\System\CurrentControlSet\Control\Terminal Server" -Name fDenyTSConnections -Value 0

    # Turn UP logging (so the attack is visible to the trainees)
    auditpol /set /subcategory:"Process Creation" /success:enable
    New-Item -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\Audit" -Force | Out-Null
    Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\Audit" -Name ProcessCreationIncludeCmdLine_Enabled -Value 1
    New-Item -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging" -Force | Out-Null
    Set-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging" -Name EnableScriptBlockLogging -Value 1

    # Never sleep / hibernate / display-off
    powercfg /change standby-timeout-ac 0
    powercfg /change standby-timeout-dc 0
    powercfg /change hibernate-timeout-ac 0
    powercfg /change monitor-timeout-ac 0

## Step 3 --- On TRINITY: run the exact same target-prep block

Turn Tamper Protection OFF on TRINITY (same menu as Step 2), then open
**PowerShell as Administrator on TRINITY** and run the identical block:

    # Windows Update OFF
    Stop-Service wuauserv -Force
    Set-Service wuauserv -StartupType Disabled
    New-Item -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU" -Force | Out-Null
    Set-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU" -Name NoAutoUpdate -Value 1

    # Microsoft Defender
    Add-MpPreference -ExclusionPath "C:"
    Set-MpPreference -DisableRealtimeMonitoring $true
    Set-MpPreference -MAPSReporting Disabled
    Set-MpPreference -SubmitSamplesConsent NeverSend

    # Windows Firewall OFF
    Set-NetFirewallProfile -Profile Domain,Private,Public -Enabled False

    # Enable Remote Desktop
    Set-ItemProperty -Path "HKLM:\System\CurrentControlSet\Control\Terminal Server" -Name fDenyTSConnections -Value 0

    # Logging UP
    auditpol /set /subcategory:"Process Creation" /success:enable
    New-Item -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\Audit" -Force | Out-Null
    Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\Audit" -Name ProcessCreationIncludeCmdLine_Enabled -Value 1
    New-Item -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging" -Force | Out-Null
    Set-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging" -Name EnableScriptBlockLogging -Value 1

    # Never sleep
    powercfg /change standby-timeout-ac 0
    powercfg /change standby-timeout-dc 0
    powercfg /change hibernate-timeout-ac 0
    powercfg /change monitor-timeout-ac 0

## Step 4 --- On NEB: run the exact same target-prep block

Turn Tamper Protection OFF on NEB, then open **PowerShell as
Administrator on NEB** and run the identical block:

    # Windows Update OFF
    Stop-Service wuauserv -Force
    Set-Service wuauserv -StartupType Disabled
    New-Item -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU" -Force | Out-Null
    Set-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU" -Name NoAutoUpdate -Value 1

    # Microsoft Defender
    Add-MpPreference -ExclusionPath "C:"
    Set-MpPreference -DisableRealtimeMonitoring $true
    Set-MpPreference -MAPSReporting Disabled
    Set-MpPreference -SubmitSamplesConsent NeverSend

    # Windows Firewall OFF
    Set-NetFirewallProfile -Profile Domain,Private,Public -Enabled False

    # Enable Remote Desktop
    Set-ItemProperty -Path "HKLM:\System\CurrentControlSet\Control\Terminal Server" -Name fDenyTSConnections -Value 0

    # Logging UP
    auditpol /set /subcategory:"Process Creation" /success:enable
    New-Item -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\Audit" -Force | Out-Null
    Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\Audit" -Name ProcessCreationIncludeCmdLine_Enabled -Value 1
    New-Item -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging" -Force | Out-Null
    Set-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging" -Name EnableScriptBlockLogging -Value 1

    # Never sleep
    powercfg /change standby-timeout-ac 0
    powercfg /change standby-timeout-dc 0
    powercfg /change hibernate-timeout-ac 0
    powercfg /change monitor-timeout-ac 0

    Download the Commpany Data from 
    https://powerbox-file.trend.org/publicdownload?share_id=SKDMiADIH21dgrlEkak6YLGQj_2FYOgbhvN_2Fytrft_2F9Qhv9gvd_2FB6STf4SCXc0j43X
    And put it in C:\Company_Data
    These contains the data that needs to be exfiltrated and encrypted

## Step 5 --- Seed the credential the attack will steal

On **NEB** (or NEO): at the Windows login screen click **Other user**,
log in as `corp\dadmin` with its password, and **leave that session
logged in** (do not sign out). This puts a Domain Admin's credentials
in memory for the attack to harvest. This is the whole point of the
credential-access stage --- don't skip it.

## Step 6 --- Confirm telemetry before attacking

In the Vision One console → Endpoint Inventory, confirm **all four
Windows endpoints are online with unique names**: ZION, NEB, NEO,
TRINITY. If any is missing, fix its agent before continuing (see the
troubleshooting table).

## Step 7 --- Take the `emulation-ready` snapshot on each box

For each VM (ZION, NEB, NEO, TRINITY, MORPHEUS), in VMware: **VM →
Snapshot → Take Snapshot → name it** `emulation-ready`**.** Take these
**while the machines are running** so the seeded `dadmin` session from
Step 5 is preserved. This is the state you revert to before each new
cohort.

> The Metasploit C2 listener on MORPHEUS is started at the **start of
> the attack run**, not here --- it's the first step of the emulation
> guide, so there's nothing to do for it in Part 7.

## Optional extras (skip these for your first run)

These add more attacker techniques but are not required. Do them only if
you want the extra detections:

-   **AS-REP roasting target** (on ZION):
    `Set-ADAccountControlsvc_sql` -`DoesNotRequirePreAuth` $true`

-   **Pass-the-hash setup:** create a local administrator account with
    the **same password** on NEO and TRINITY, so a hash stolen from one
    works on the other.

-   **Bulk AD population:** download and run **BadBlood** on ZION to
    flood the directory with users/groups/OUs for richer discovery.

# Appendix --- Troubleshooting quick reference

  ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
  Symptom                                 Fix
  --------------------------------------- ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
  `Get-NetAdapter` / `New-`NetIPAddressImport-Module NetAdapter`, or open a fresh elevated PowerShell
  "not recognized"                      

  `New-NetIPAddress` says address/route `Set-NetIPInterface` -`InterfaceAlias` "Ethernet0" -`Dhcp` Disabled` first
  exists                                  

  Quoted path won't run                  Prefix with the call operator: `& "C:\path\tool.exe" `args`
  (`"C:...\VMwareToolboxCmd.exe" ...`)   

  VM time wrong / drifting                NTP config on the DC; disable VMware Tools timesync; fix the host clock

  AD promotion / `New-ADUser`           Password needs upper+lower+number+symbol. Tip: use the inline form `(`ConvertTo-SecureString` "Pw#2026" -`AsPlainText` -Force)` to skip the blind prompt
  complexity error                        

  "The specified account already         It **was** created on the prior attempt --- reset its password (`Set-ADAccountPassword`) instead of recreating
  exists" after a complexity error       

  Read-Host password prompt shows nothing Expected (masked). Type it and press Enter, or use the inline `ConvertTo-SecureString` form
  as you type                             

  Domain login fails: "no logon          ZION powered on, workstation DNS = `192.168.150.10`, and `Test-ComputerSecureChannel` = True
  servers" / "trust relationship        
  failed"                                

  Two commands pasted on one line fail    Run each on its **own** line

  Win 11 setup: "TPM 2.0 required"      VM must be UEFI + Secure Boot + TPM (encrypt the VM at creation --- Section A step 6)

  Guest-OS list shows only old versions   Old VMware build --- use 26H1
  (e.g. Server 2019)                      

  Boots to "no OS / PXE" instead of     Power off; confirm CD/DVD = ISO with "Connect at power on"; power on again
  installer                               

  **Agent installed but not in Vision     Check in order: clock (TLS), DNS + 443 reachable, services running, then the Endpoint Basecamp log
  One**                                   

  **Log: "config file not exist / no     Incomplete install --- uninstall (uninstall tool), reboot, reinstall the **full** package to completion
  connection information"; no**          
  config.json`                           

  **Agent can't register in the NAT      Agent installer proxy = **Direct connect**
  lab**                                   

  Clone shows as the same sensor          Agent installed before cloning; clone clean, enroll per-VM

  `New-NetIPAddress`: "Instance        Adapter already has an IP (inherited from the clone). Clear it first:
  MSFT_NetIPAddress already exists"      `Remove-NetIPAddress` -`InterfaceAlias` "Ethernet0" -`Confirm:$`false; Remove-NetRoute -`InterfaceAlias` "Ethernet0" -`DestinationPrefix` 0.0.0.0/0 -`Confirm:$`false`,
                                          then re-add

  `Add-Computer`: "already in that       It was joined earlier but not renamed. Just rename: `Rename-Computer -`NewName` <NAME> -`DomainCredentialcorp`\Administrator -Restart` (run `Test-ComputerSecureChannel` first;
  domain" but hostname is still the      if False, `-Repair` it or unjoin/rejoin)
  default                                 

  Kali (imported appliance): invisible    Upgrade the VM's virtual hardware: **VM → Manage → Change Hardware Compatibility → latest version → "Alter this virtual machine."** (Older imported appliances lag several hardware
  mouse cursor / display glitches         versions behind Workstation 26H1.) Cursor is cosmetic --- everything on Kali is terminal-driven regardless

  Tempted to clone ZION for a second      Don't --- ZION is a DC, so a clone is a second DC. Build member servers directly, or clone a clean unpromoted/unjoined server base
  server                                  

  "Linked clone" greyed out in the      The VM is encrypted (Win 11 vTPM) --- linked clones aren't allowed; use **full clone** (expected, not an error)
  Clone wizard                            

  PsExec "network path not found"       SMB/firewall --- enable "File and Printer Sharing"

  PsExec "access denied"                Use a domain local-admin; for local accounts set `LocalAccountTokenFilterPolicy`=1`

  PsExec service binary vanishes          Defender quarantined it --- add a path exclusion on the target
  ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
