# Connecting to Windows Server EC2 Directly in AWS Browser (AWS Systems Manager Fleet Manager)

This detailed guide explains how to view and interact with your Windows Server desktop interface directly inside your web browser on the AWS Console without installing any local client software or opening port 3389.

---

## 📑 Table of Contents
1. [Prerequisites & Architecture](#prerequisites--architecture)
2. [Phase 1: Create the IAM Role for Systems Manager](#phase-1-create-the-iam-role-for-systems-manager)
3. [Phase 2: Attach the IAM Role to your EC2 Instance](#phase-2-attach-the-iam-role-to-your-ec2-instance)
4. [Phase 3: Connect via In-Browser Remote Desktop](#phase-3-connect-via-in-browser-remote-desktop)
5. [Troubleshooting](#troubleshooting)

---

## 🧠 Prerequisites & Architecture

- **AWS Windows Server AMI**: Pre-installed with AWS SSM Agent by default.
- **No Inbound Port Required**: You do **not** need to open port 3389 (RDP) on your Security Group. Communication happens outbound over secure HTTPS (port 443).
- **Public IP Not Required**: Works even with private subnets as long as outbound internet / VPC endpoints are reachable.

---

## Phase 1: Create the IAM Role for Systems Manager

### Step 1.1: Open IAM Roles Page
1. In the top search bar of the AWS Console, type **IAM** and press Enter.
2. In the left navigation menu, click **Roles**.
3. Click the orange **Create role** button (top right).

---

### Step 1.2: Select Trusted Entity Type & Use Case
1. Under **Trusted entity type**, select **AWS service**.
2. Under **Service or use case**, keep `EC2` selected.
3. Under **Use case**, select the first radio button:
   - 🔘 **EC2** *(Allows EC2 instances to call AWS services on your behalf)*.
4. Scroll to the bottom right and click the orange **Next** button.

---

### Step 1.3: Add Systems Manager Permissions Policy
1. In the **Filter policies by property or policy name** search box, type:
   ```text
   AmazonSSMManagedInstanceCore
   ```
2. Press Enter.
3. Check the checkbox **☑** next to `AmazonSSMManagedInstanceCore`.
4. Scroll to the bottom right and click **Next**.

---

### Step 1.4: Name and Create the Role
1. In the **Role name** field, enter:
   ```text
   EC2-SSM-Role
   ```
2. *(Optional)* Add a description (e.g., `Allows EC2 instances to communicate with AWS Systems Manager`).
3. Scroll all the way down to the bottom right and click **Create role**.

> **Verification:** A green banner will appear: *"Role EC2-SSM-Role created."*

---

## Phase 2: Attach the IAM Role to your EC2 Instance

### Step 2.1: Open EC2 Instances
1. In the top search bar, type **EC2** and click **EC2**.
2. In the left menu, click **Instances**.
3. Click the checkbox next to your running Windows instance.

---

### Step 2.2: Modify IAM Role
1. With your instance selected, click the **Actions** dropdown button in the top menu.
2. Hover over **Security** and click **Modify IAM role**.
3. In the **IAM role** dropdown list, select **`EC2-SSM-Role`**.
4. Click the orange **Update IAM role** button.

> **Verification:** A green banner appears confirming the IAM role was successfully updated.

---

### Step 2.3: Wait 2 to 3 Minutes
> ⏳ **Important:** The SSM Agent inside your Windows Server needs 2 to 3 minutes to register with AWS Systems Manager. Wait a few moments before moving to Phase 3.

---

## Phase 3: Connect via In-Browser Remote Desktop

### Step 3.1: Navigate to Fleet Manager
1. In the top AWS search bar, type **Systems Manager** and select it.
2. In the left sidebar under **Node Management**, click **Fleet Manager**.
3. Look at the **Managed nodes** table:
   - You should see your Windows instance ID listed.
   - The status indicator should show **Online**.
   *(If it does not appear yet, click the refresh 🔄 button in the table header and wait 1 minute)*.

---

### Step 3.2: Launch Remote Desktop Session
1. Select your Windows instance by clicking the radio button / checkbox next to it.
2. Click the **Node actions** dropdown button at the top of the table.
3. Click **Connect with Remote Desktop**.

---

### Step 3.3: Authenticate
1. In the **User name** field, type:
   ```text
   Administrator
   ```
2. Under **Authentication type**, choose one:
   - **Password**: Paste your decrypted Administrator password retrieved from AWS.
   - **Key Pair**: Upload your `.pem` key pair file directly to have AWS decrypt it automatically.
3. Click the orange **Connect** button.

> 🎉 **Verification:** A full-screen browser window opens showing the live Windows Server desktop and Server Manager!

---

## 🛠 Troubleshooting

| Problem | Cause | Solution |
| :--- | :--- | :--- |
| **Instance not showing in Fleet Manager Managed Nodes** | SSM Agent hasn't registered yet, or IAM role not attached. | 1. Ensure `EC2-SSM-Role` is attached.<br>2. Wait 3–5 minutes.<br>3. In EC2 Console &rarr; Reboot instance to force restart the SSM Agent. |
| **"Authentication Failed" when connecting** | Incorrect password or username typo. | 1. Ensure username is `Administrator`.<br>2. Re-decrypt password using your `.pem` key file in EC2 Console &rarr; Connect &rarr; RDP client. |
| **Browser pop-up blocked** | Browser settings blocked new tab. | Allow pop-ups for `*.console.aws.amazon.com`. |
