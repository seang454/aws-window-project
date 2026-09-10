# Windows Server: Local User Management & Active Directory (AD DS) Setup

This guide covers:
1. **Option A:** Managing Local Users (Workgroup Mode)
2. **Option B:** Installing Active Directory Domain Services (AD DS) & Promoting to Domain Controller

---

## 📑 Table of Contents
1. [Option A: Creating Local Users (Standalone Server)](#option-a-creating-local-users-standalone-server)
   - [Method 1: GUI (Computer Management)](#method-1-gui-computer-management)
   - [Method 2: PowerShell](#method-2-powershell)
   - [How Teammates Connect via RDP](#how-teammates-connect-via-rdp)
2. [Option B: Installing Active Directory & Promoting to Domain Controller](#option-b-installing-active-directory--promoting-to-domain-controller)
   - [Step 1: Install AD DS Role](#step-1-install-ad-ds-role)
   - [Step 2: Promote Server to Domain Controller](#step-2-promote-server-to-domain-controller)
   - [Step 3: Create Active Directory Domain Users](#step-3-create-active-directory-domain-users)
3. [Important Differences Summary](#important-differences-summary)

---

# Option A: Creating Local Users (Standalone Server)

*Use this when your server is not a Domain Controller and you want to give teammates separate logins.*

### Method 1: GUI (Computer Management)

1. Connect to your Windows Server via RDP as `Administrator`.
2. Open **Server Manager** (click Start or search Server Manager).
3. In the top right corner, click **Tools** &rarr; **Computer Management**.
4. In the left panel, expand **Local Users and Groups** &rarr; click on **Users**.
5. Right-click in the empty white space &rarr; click **New User...**.
6. Fill in the user information:
   - **User name**: (e.g., `sarah`)
   - **Full name**: (e.g., `Sarah Smith`)
   - **Password**: Enter a temporary password (e.g., `Pass1234!@#$`)
   - **Confirm password**: Re-enter password
   - [x] Check **User must change password at next logon**
7. Click **Create**, then click **Close**.
8. **Grant Remote Desktop Permissions**:
   - Double-click the newly created user (`sarah`).
   - Go to the **Member Of** tab.
   - Click **Add...**.
   - In the text box, type: `Remote Desktop Users`
   - *(Optional: If this user should be a full server administrator, type `Administrators` instead)*.
   - Click **Check Names**, then click **OK**.
   - Click **Apply** and **OK**.

---

### Method 2: PowerShell (Fastest)

Open **PowerShell as Administrator** and run:

```powershell
# 1. Define username and temporary password
$Username = "sarah"
$Password = ConvertTo-SecureString "Pass1234!@#$" -AsPlainText -Force

# 2. Create the local user (forces password change on first logon)
New-LocalUser -Name $Username -Password $Password -FullName "Sarah Smith" -PasswordNeverExpires $false

# 3. Add user to Remote Desktop Users group
Add-LocalGroupMember -Group "Remote Desktop Users" -Member $Username

# (Optional: Add to Administrators group if they need admin rights)
Add-LocalGroupMember -Group "Administrators" -Member $Username
```

---

### How Teammates Connect via RDP:
1. Open **Remote Desktop Connection (`mstsc`)** on their computer.
2. In **Computer**, enter: `Instance-Public-IP` (e.g., `16.170.218.27`)
3. In **User name**, enter: `sarah` (or `.\sarah`)
4. In **Password**, enter: `Pass1234!@#$`
5. When connecting for the first time, Windows will prompt:
   > *"The user's password must be changed before signing in."*
6. The user sets their own private password and logs in!

---

# Option B: Installing Active Directory & Promoting to Domain Controller

*Use this if your Year 4 class requires setting up an enterprise Active Directory Domain (e.g., `lab.local` or `rupp.edu.kh`).*

### Step 1: Install AD DS Role

1. In **Server Manager**, click **Manage** (top right) &rarr; **Add Roles and Features**.
2. Click **Next** until you reach **Server Roles**.
3. Check the box **☑ Active Directory Domain Services**.
4. In the popup dialog, click **Add Features**, then click **Next**.
5. Click **Next** through *Features* and *AD DS*.
6. On the *Confirmation* screen, check **Restart the destination server automatically if required**.
7. Click **Install**. Wait ~2–3 minutes until installation succeeds.

---

### Step 2: Promote Server to Domain Controller

1. In **Server Manager**, click the **Flag notification icon (🚩)** with a yellow warning at the top right.
2. Click **Promote this server to a domain controller**.
3. In the Deployment Configuration wizard:
   - Select **Add a new forest**.
   - **Root domain name**: Enter your domain (e.g., `rupp.local` or `class.internal`).
   - Click **Next**.
4. Set **Directory Services Restore Mode (DSRM) password**:
   - Enter a secure password (e.g., `RestorePass123!`).
   - Click **Next** through *DNS Options* and *Additional Options* (NetBIOS name).
5. Click **Next** through *Paths* and *Review Options*.
6. Once the **Prerequisites Check** passes (green checkmark), click **Install**.
7. The server will automatically restart.

---

### Step 3: Create Active Directory Domain Users

*After the server restarts and becomes a Domain Controller, Local Users are replaced by AD Users.*

#### Method 1: Using GUI (ADUC)
1. In **Server Manager**, click **Tools** &rarr; **Active Directory Users and Computers**.
2. Expand your domain (e.g., `rupp.local`) &rarr; click **Users**.
3. Right-click in the empty area &rarr; **New** &rarr; **User**.
4. Enter First name, Last name, and **User logon name** (e.g., `sarah`).
5. Set password and check **User must change password at next logon**.
6. Click **Finish**.
7. Double-click `sarah` &rarr; **Member Of** tab &rarr; add to **Remote Desktop Users** (or **Domain Admins**).

#### Method 2: PowerShell
```powershell
$Password = ConvertTo-SecureString "Pass1234!@#$" -AsPlainText -Force
New-ADUser -Name "sarah" -GivenName "Sarah" -Surname "Smith" -SamAccountName "sarah" -UserPrincipalName "sarah@rupp.local" -AccountPassword $Password -Enabled $true -ChangePasswordAtLogon $true
Add-ADGroupMember -Identity "Remote Desktop Users" -Members "sarah"
```

---

## ⚖️ Important Differences Summary

| Feature | Local Users (Option A) | Active Directory (Option B) |
| :--- | :--- | :--- |
| **Login Format** | `.\sarah` or `sarah` | `sarah@rupp.local` or `RUPP\sarah` |
| **Management Tool** | Computer Management (`lusrmgr.msc`) | AD Users and Computers (`dsa.msc`) |
| **Setup Time** | Immediate (1 minute) | ~10 minutes (requires DC promotion & reboot) |
| **Best For** | Standalone EC2 server | Enterprise domain, multi-server network, school labs |
