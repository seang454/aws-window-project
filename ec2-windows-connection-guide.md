# Connecting to AWS EC2 Windows Instance via Remote Desktop Connection (RDP)

This guide walks you through connecting to your Amazon EC2 Windows Server instance from a local Windows computer using the built-in Remote Desktop Connection (`mstsc`) client.

---

## 📋 Prerequisites

Before connecting, ensure you have gathered the following from the **AWS Management Console**:

1. **Public IPv4 Address**: Located in your EC2 instance details (e.g., `16.170.218.27`).
2. **Decrypted Administrator Password**:
   - In AWS Console &rarr; Select Instance &rarr; Click **Connect** &rarr; **RDP client**.
   - Click **Get password**, upload your `.pem` key pair file, and click **Decrypt password**.
3. **Security Group Inbound Rule**: Port `3389` (RDP) must be open to your IP address.

---

## 🚀 Step-by-Step Connection Guide

### Step 1: Open Remote Desktop Connection
*⏱ Estimated Time: ~10 seconds*

1. Press <kbd>Windows Key</kbd> + <kbd>R</kbd> on your keyboard to open the **Run** dialog box.
2. Type `mstsc` and press **Enter** (or search for **Remote Desktop Connection** in the Start menu).

> **Verification:** The **Remote Desktop Connection** window will appear on your screen.

---

### Step 2: Enter Instance Public IP & Username
*⏱ Estimated Time: ~30 seconds*

1. In the **Computer** field, enter your EC2 instance's **Public IPv4 address** (e.g., `16.170.218.27`).
2. Click the **Show Options** arrow (bottom-left corner) to expand settings.
3. In the **User name** field, type `Administrator`.
4. Click **Connect**.

> **Verification:** A credentials prompt or security warning prompt will appear.

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

> **Verification:** The connection window will expand to full screen, presenting the Windows Server desktop interface and opening **Server Manager**.

---

## 🛠 Troubleshooting Common Errors

| Error Message / Issue | Potential Cause & Solution |
| :--- | :--- |
| **"Remote Desktop can't connect to the remote computer"** | <ul><li>Check the **Security Group** attached to your instance.</li><li>Confirm an **Inbound Rule** exists allowing `Type: RDP`, `Protocol: TCP`, `Port: 3389`.</li><li>Ensure the source IP matches your current local public IP or allows access.</li><li>Verify the instance state is **Running** and **2/2 status checks** have passed.</li></ul> |
| **"User name or password incorrect"** | <ul><li>Ensure the username is exactly `Administrator`.</li><li>Check that there are no accidental trailing or leading spaces when pasting the decrypted password.</li><li>Verify that the correct `.pem` key pair was used to decrypt the password.</li></ul> |
