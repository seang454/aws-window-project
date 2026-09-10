# AWS EC2 Windows Server: Launch & RDP Connection Guide

A complete step-by-step guide on how to create, configure, launch an Amazon EC2 Windows Server instance, decrypt administrator credentials, and connect via Remote Desktop Connection (RDP).

---

## 📑 Table of Contents
1. [Part 1: Launching the Windows EC2 Instance](#part-1-launching-the-windows-ec2-instance)
2. [Part 2: Post-Launch & Retrieving Administrator Password](#part-2-post-launch--retrieving-administrator-password)
3. [Part 3: Connecting via Remote Desktop Connection (mstsc)](#part-3-connecting-via-remote-desktop-connection-mstsc)
4. [Part 4: Troubleshooting Common Errors](#part-4-troubleshooting-common-errors)

---

## Part 1: Launching the Windows EC2 Instance

### Step 1: Open Launch Instance Page
*⏱ Estimated Time: 1 click*

1. Open the [AWS Management Console](https://console.aws.amazon.com/ec2/).
2. In the **EC2 Dashboard**, click the orange **Launch instances** button in the top-right corner.

> **Verification:** You will be redirected to the **Launch an instance** setup page.

---

### Step 2: Set Name & Choose Windows OS
*⏱ Estimated Time: ~1 minute*

1. In the **Name** field under *Name and tags*, enter a name for your server (e.g., `MyWindowsServer`).
2. Under **Application and OS Images (Amazon Machine Image)**, click the **Windows** icon.
3. Keep the default AMI (e.g., *Microsoft Windows Server 2022 Base* or *2025 Base*).

> **Verification:** The **Windows** card is selected and displays a **Free tier eligible** badge.

---

### Step 3: Select Instance Type
*⏱ Estimated Time: ~30 seconds*

1. Under **Instance type**, select `t3.micro` (or `t2.micro` depending on your account region/requirements).

> **Verification:** The instance type box shows `t3.micro` (2 vCPU, 1 GiB Memory).

---

### Step 4: Create and Download Key Pair (Crucial)
*⏱ Estimated Time: ~1 minute*

> ⚠️ **Important:** Do NOT skip this step! Without a key pair, you will not be able to retrieve the Windows Administrator password.

1. Under **Key pair (login)**, click **Create new key pair**.
2. In the popup dialog:
   - **Key pair name**: Enter a name (e.g., `win-key`).
   - **Key pair type**: Select **RSA**.
   - **Private key file format**: Select **`.pem`**.
3. Click the orange **Create key pair** button.

> **Verification:** A file named `win-key.pem` will automatically download to your computer's `Downloads` folder.

---

### Step 5: Configure Firewall / Network Rules
*⏱ Estimated Time: ~1 minute*

1. Under **Network settings**, configure the firewall rules:
   - [x] Check **Allow RDP traffic from** &rarr; Select **My IP** (recommended for security) or **Anywhere** (`0.0.0.0/0`).
   - [x] Check **Allow HTTP traffic from the internet**.
   - [x] Check **Allow HTTPS traffic from the internet**.

> **Verification:** All three checkboxes (**RDP**, **HTTP**, **HTTPS**) are checked with blue checkmarks.

---

### Step 6: Launch the Instance
*⏱ Estimated Time: 1 click*

1. Review your settings in the Summary panel on the right.
2. Click the orange **Launch instance** button at the bottom right.

> **Verification:** A green banner saying **"Successfully initiated launch of instance"** will appear with a direct link to your new instance ID.

---

## Part 2: Post-Launch & Retrieving Administrator Password

*⏱ Estimated Time: ~4–5 minutes*

1. **Wait for Instance Initialization**:
   - Go to **EC2 &rarr; Instances**.
   - Wait ~4 to 5 minutes until:
     - **Instance state** = `Running`
     - **Status check** = `2/2 checks passed`
2. **Retrieve Public IPv4 Address**:
   - Select your instance and copy the **Public IPv4 address** (e.g., `16.170.218.27`).
3. **Decrypt Administrator Password**:
   - Select the instance and click the **Connect** button at the top.
   - Select the **RDP client** tab.
   - Click **Get password**.
   - Click **Upload private key file** and choose the `win-key.pem` file from your Downloads folder.
   - Click **Decrypt password**.
   - Copy the decrypted password generated on the screen.

---

## Part 3: Connecting via Remote Desktop Connection (mstsc)

### Step 1: Open Remote Desktop Connection
*⏱ Estimated Time: ~10 seconds*

1. Press <kbd>Windows Key</kbd> + <kbd>R</kbd> on your keyboard to open the **Run** dialog box.
2. Type `mstsc` and press **Enter** (or search for **Remote Desktop Connection** in the Windows Start menu).

> **Verification:** The **Remote Desktop Connection** client window opens.

---

### Step 2: Enter Instance Public IP & Username
*⏱ Estimated Time: ~30 seconds*

1. In the **Computer** field, enter your EC2 instance's **Public IPv4 address** (e.g., `16.170.218.27`).
2. Click **Show Options** in the bottom-left corner to expand settings.
3. In the **User name** field, type `Administrator`.
4. Click **Connect**.

> **Verification:** A credentials prompt or security warning dialog appears.

---

### Step 3: Enter Decrypted Password
*⏱ Estimated Time: ~30 seconds*

1. In the **Windows Security** credentials popup, paste the **decrypted Administrator password** you retrieved from AWS.
2. Click **OK**.

> **Verification:** A certificate warning popup titled *"The identity of the remote computer cannot be verified"* will appear.

---

### Step 4: Accept the Certificate Warning
*⏱ Estimated Time: ~5 seconds*

1. Check the box **"Don't ask me again for connections to this computer"**.
2. Click **Yes**.

> **Verification:** The connection window expands to full screen, displaying the Windows Server desktop interface and opening **Server Manager**.

---

## Part 4: Troubleshooting Common Errors

| Error Message / Issue | Potential Cause & Solution |
| :--- | :--- |
| **"Remote Desktop can't connect to the remote computer"** | <ul><li>**Security Group:** Verify that an Inbound Rule exists allowing `Type: RDP`, `Protocol: TCP`, `Port: 3389`.</li><li>**Source IP:** If set to *My IP*, check if your local public IP address has changed.</li><li>**Instance Status:** Ensure the instance state is **Running** and status checks show **2/2 passed**.</li></ul> |
| **"User name or password incorrect"** | <ul><li>Ensure the username is typed exactly as `Administrator` (case-sensitive).</li><li>Check for accidental leading/trailing spaces when copying and pasting the decrypted password.</li><li>Verify that the `.pem` key pair used for decryption matches the key pair selected when launching the instance.</li></ul> |
| **"Password is not available yet"** | <ul><li>AWS requires ~4 minutes after instance launch to generate the password. Wait a moment and try decrypting again.</li></ul> |
