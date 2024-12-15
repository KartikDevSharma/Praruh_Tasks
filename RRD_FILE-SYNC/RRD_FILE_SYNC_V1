A Step-by-step guide designed to help system administrators set up **RRD File Synchronization** between two RHEL-based servers. This documentation is crafted to be adaptable, allowing users to input custom source and destination addresses as needed.

---

# **RRD File Synchronization Setup Guide**

**Author:** Kartik Sharma
**Date:** 11-DEC-2024
**Version:** 1.0

---

## **Table of Contents**

1. [Introduction](#1-introduction)
2. [Prerequisites](#2-prerequisites)
3. [Server Configuration](#3-server-configuration)
    - [3.1. Configuring Source and Destination Servers](#31-configuring-source-and-destination-servers)
    - [3.2. Installing Required Software](#32-installing-required-software)
4. [Setting Up SSH Key-Based Authentication](#4-setting-up-ssh-key-based-authentication)
5. [Directory Structure Setup](#5-directory-structure-setup)
6. [Creating the Synchronization Script](#6-creating-the-synchronization-script)
7. [Configuring the systemd Service](#7-configuring-the-systemd-service)
8. [Implementing Log Rotation](#8-implementing-log-rotation)
9. [Transitioning to a Dedicated Non-Root User](#9-transitioning-to-a-dedicated-non-root-user)
10. [Testing the Setup](#10-testing-the-setup)
11. [Monitoring and Maintenance](#11-monitoring-and-maintenance)
12. [Troubleshooting](#12-troubleshooting)
13. [Security Best Practices](#13-security-best-practices)
14. [Conclusion](#14-conclusion)

---

## **1. Introduction**

This guide provides a detailed procedure to set up automated synchronization of RRD (Round-Robin Database) files from a **Source Server** to a **Destination Server** using `rsync` and `systemd`. The setup ensures real-time synchronization, reliability, and security. By following this guide, administrators can replicate the setup across different environments with customizable configurations.

---

## **2. Prerequisites**

Before initiating the setup, ensure the following:

- **Operating Systems:** Both Source and Destination servers are running **RHEL/CentOS 7 or 8**.
- **Network Connectivity:** Both servers can communicate over the network. Verify connectivity using `ping` or `ssh`.
- **User Privileges:** You have `root` or `sudo` access on both servers.
- **Python Environment:** Python 3 is installed on the Source server.
- **Firewall Access:** Necessary ports (e.g., SSH) are open on both servers.
- **SELinux Status:** SELinux is either configured correctly or set to permissive mode temporarily for setup.

---

## **3. Server Configuration**

### **3.1. Configuring Source and Destination Servers**

**Identify the roles:**

- **Source Server:** The server containing the original `.rrd` files to be synchronized.
- **Destination Server:** The server where the `.rrd` files will be replicated.

**Assign Custom IP Addresses:**

- **Source Server IP:** `192.168.29.106` (Example)
- **Destination Server IP:** `192.168.29.153` (Example)

*Note: Replace these IPs with your actual server IP addresses.*

#### **3.1.1. Configuring Network Settings**

**On Both Servers:**

1. **Identify Network Interfaces:**

   ```bash
   ip addr
   ```

   Note the primary network interface (e.g., `eth0`, `ens33`).

2. **Configure Static IP Address:**

   Edit the network configuration file corresponding to your interface.

   - **For RHEL/CentOS 7:**

     ```bash
     sudo vi /etc/sysconfig/network-scripts/ifcfg-<interface_name>
     ```

   - **For RHEL/CentOS 8:**

     ```bash
     sudo vi /etc/sysconfig/network-scripts/ifcfg-<interface_name>
     ```

   **Add/Modify the Following Entries:**

   ```ini
   DEVICE=<interface_name>
   BOOTPROTO=static
   ONBOOT=yes
   IPADDR=<SERVER_IP>
   NETMASK=255.255.255.0
   GATEWAY=<GATEWAY_IP>
   DNS1=8.8.8.8
   DNS2=8.8.4.4
   ```

   *Replace `<interface_name>`, `<SERVER_IP>`, and `<GATEWAY_IP>` with appropriate values.*

3. **Restart Network Service:**

   - **RHEL/CentOS 7:**

     ```bash
     sudo systemctl restart network
     ```

   - **RHEL/CentOS 8:**

     ```bash
     sudo systemctl restart network
     ```

4. **Verify IP Configuration:**

   ```bash
   ip addr show <interface_name>
   ```

### **3.2. Installing Required Software**

**On Both Servers:**

1. **Update System Packages:**

   ```bash
   sudo yum update -y
   ```

2. **Install `rsync` and OpenSSH Server:**

   ```bash
   sudo yum install rsync openssh-server -y
   ```

3. **Start and Enable SSH Service:**

   ```bash
   sudo systemctl start sshd
   sudo systemctl enable sshd
   ```

4. **Verify SSH Service Status:**

   ```bash
   sudo systemctl status sshd
   ```

---

## **4. Setting Up SSH Key-Based Authentication**

To enable password-less synchronization, set up SSH key-based authentication from the **Source Server** to the **Destination Server**.

### **4.1. Generate SSH Key Pair on Source Server**

**On Source Server (`192.168.29.106`):**

1. **Create a Dedicated SSH Key for Synchronization:**

   ```bash
   sudo ssh-keygen -t rsa -b 4096 -C "rrd_sync@source" -f /root/.ssh/id_rsa_rrd_sync -N ""
   ```

   - **Parameters:**
     - `-t rsa`: Specifies RSA key type.
     - `-b 4096`: Sets key length to 4096 bits.
     - `-C`: Adds a comment to the key.
     - `-f`: Specifies the file name.
     - `-N ""`: Sets an empty passphrase.

### **4.2. Copy Public Key to Destination Server**

**Option 1: Using `ssh-copy-id`**

```bash
sudo ssh-copy-id -i /root/.ssh/id_rsa_rrd_sync.pub root@192.168.29.153
```

**Option 2: Manual Method**

1. **Display Public Key:**

   ```bash
   cat /root/.ssh/id_rsa_rrd_sync.pub
   ```

2. **On Destination Server (`192.168.29.153`):**

   ```bash
   sudo mkdir -p /root/.ssh
   sudo chmod 700 /root/.ssh
   sudo vi /root/.ssh/authorized_keys
   ```

   - **Paste the Public Key Content.**

   ```bash
   sudo chmod 600 /root/.ssh/authorized_keys
   ```

### **4.3. Test SSH Connection**

**From Source Server:**

```bash
sudo ssh -i /root/.ssh/id_rsa_rrd_sync root@192.168.29.153 "echo 'SSH connection successful!'"
```

- **Expected Output:**

  ```
  SSH connection successful!
  ```

*If the output is as expected, SSH key-based authentication is functioning correctly.*

---

## **5. Directory Structure Setup**

### **5.1. Create Required Directories**

**On Source Server (`192.168.29.106`):**

```bash
sudo mkdir -p /opt/opennms/rrd_data_move/snmp/1/opennms-jvm/
```

**On Destination Server (`192.168.29.153`):**

```bash
sudo mkdir -p /opt/opennms/rrd_data_move/destination/snmp/1/opennms-jvm/
```

### **5.2. Set Ownership and Permissions**

**On Source Server:**

```bash
sudo chown -R root:root /opt/opennms/rrd_data_move/snmp/1/opennms-jvm/
sudo chmod -R 755 /opt/opennms/rrd_data_move/snmp/1/opennms-jvm/
```

**On Destination Server:**

```bash
sudo chown -R root:root /opt/opennms/rrd_data_move/destination/snmp/1/opennms-jvm/
sudo chmod -R 755 /opt/opennms/rrd_data_move/destination/snmp/1/opennms-jvm/
```

*Adjust ownership and permissions as per your security requirements.*

---

## **6. Creating the Synchronization Script**

Develop a Python script that monitors the source directory for `.rrd` file changes and synchronizes them to the destination server using `rsync`.

### **6.1. Install Python Dependencies**

**On Source Server:**

1. **Ensure Python 3 is Installed:**

   ```bash
   python3 --version
   ```

2. **Install `watchdog` Package:**

   ```bash
   sudo pip3 install watchdog
   ```

### **6.2. Develop the Synchronization Script**

**Create the Script:**

```bash
sudo vi /opt/opennms/rrd_data_move/rrd_sync_cross_server.py
```

**Paste the Following Content:**

```python
#!/usr/bin/env python3

import os
import sys
import json
import logging
import subprocess
from watchdog.observers import Observer
from watchdog.events import FileSystemEventHandler

# Configuration Variables
SOURCE_DIR = "/opt/opennms/rrd_data_move/snmp/1/opennms-jvm/"
REMOTE_USER = "root"  # Change if using a dedicated user
REMOTE_HOST = "192.168.29.153"  # Destination server IP
REMOTE_DEST_DIR = "/opt/opennms/rrd_data_move/destination/snmp/1/opennms-jvm/"
SSH_KEY_PATH = "/root/.ssh/id_rsa_rrd_sync"  # Path to the SSH private key
STATE_FILE = "/opt/opennms/rrd_data_move/rrd_sync_state.json"
LOG_FILE = "/var/log/rrd_sync.log"
ERROR_LOG_FILE = "/var/log/rrd_sync_error.log"

# Setup Logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler(LOG_FILE),
        logging.StreamHandler(sys.stdout)
    ]
)

error_logger = logging.getLogger('error_logger')
error_handler = logging.FileHandler(ERROR_LOG_FILE)
error_handler.setLevel(logging.ERROR)
error_logger.addHandler(error_handler)

# Load or Initialize State
def load_state():
    if os.path.exists(STATE_FILE):
        with open(STATE_FILE, 'r') as f:
            try:
                return json.load(f)
            except json.JSONDecodeError:
                logging.error("State file is corrupted. Initializing a new state.")
                return {}
    return {}

def save_state(state):
    with open(STATE_FILE, 'w') as f:
        json.dump(state, f)

# Synchronization Function
def sync_file(file_path):
    relative_path = os.path.relpath(file_path, SOURCE_DIR)
    destination_path = os.path.join(REMOTE_DEST_DIR, relative_path)
    dest_dir = os.path.dirname(destination_path)

    # Ensure destination directory exists
    subprocess.run(['ssh', '-i', SSH_KEY_PATH, f"{REMOTE_USER}@{REMOTE_HOST}", f"mkdir -p {dest_dir}"], check=True)

    # Execute rsync command
    rsync_command = [
        "rsync",
        "-avz",
        "--update",
        "--progress",
        "-e",
        f"ssh -i {SSH_KEY_PATH}",
        file_path,
        f"{REMOTE_USER}@{REMOTE_HOST}:{REMOTE_DEST_DIR}"
    ]

    try:
        result = subprocess.run(rsync_command, capture_output=True, text=True, check=True)
        logging.info(f"Synced {relative_path} successfully.")
        logging.debug(f"rsync output: {result.stdout}")
    except subprocess.CalledProcessError as e:
        error_message = f"Failed to sync {relative_path}. Error: {e.stderr}"
        error_logger.error(error_message)

# Event Handler Class
class RRDFileEventHandler(FileSystemEventHandler):
    def __init__(self, state):
        self.state = state

    def on_modified(self, event):
        if not event.is_directory and event.src_path.endswith('.rrd'):
            self.handle_event(event.src_path)

    def on_created(self, event):
        if not event.is_directory and event.src_path.endswith('.rrd'):
            self.handle_event(event.src_path)

    def handle_event(self, file_path):
        try:
            mtime = os.path.getmtime(file_path)
            if (file_path not in self.state) or (mtime > self.state[file_path]):
                logging.info(f"Syncing {os.path.basename(file_path)}...")
                sync_file(file_path)
                self.state[file_path] = mtime
                save_state(self.state)
        except Exception as e:
            error_logger.error(f"Error handling file {file_path}: {e}")

def initial_sync(state):
    logging.info("Starting initial synchronization of existing files...")
    for root, dirs, files in os.walk(SOURCE_DIR):
        for file in files:
            if file.endswith('.rrd'):
                file_path = os.path.join(root, file)
                try:
                    mtime = os.path.getmtime(file_path)
                    if (file_path not in state) or (mtime > state[file_path]):
                        logging.info(f"Syncing {file}...")
                        sync_file(file_path)
                        state[file_path] = mtime
                except Exception as e:
                    error_logger.error(f"Error during initial sync for {file_path}: {e}")
    logging.info("Initial synchronization completed.")

def main():
    state = load_state()
    initial_sync(state)

    event_handler = RRDFileEventHandler(state)
    observer = Observer()
    observer.schedule(event_handler, path=SOURCE_DIR, recursive=True)
    observer.start()
    logging.info("Started monitoring for file changes.")

    try:
        while True:
            pass  # Keep the script running
    except KeyboardInterrupt:
        observer.stop()
        logging.info("Stopping RRD synchronization script.")
    observer.join()

if __name__ == "__main__":
    main()
```

### **6.3. Make the Script Executable**

```bash
sudo chmod +x /opt/opennms/rrd_data_move/rrd_sync_cross_server.py
```

---

## **7. Configuring the systemd Service**

Create a `systemd` service to manage the synchronization script, ensuring it runs continuously and starts on system boot.

### **7.1. Create the Service File**

```bash
sudo nano /etc/systemd/system/rrd_sync.service
```

### **7.2. Add the Following Configuration**

```ini
[Unit]
Description=RRD File Synchronization Service
After=network.target sshd.service

[Service]
Type=simple
ExecStart=/usr/bin/python3 /opt/opennms/rrd_data_move/rrd_sync_cross_server.py
Restart=on-failure
RestartSec=30
User=root  # Change if using a dedicated non-root user

# Logging
StandardOutput=append:/var/log/rrd_sync.log
StandardError=append:/var/log/rrd_sync_error.log

# Environment
Environment=PYTHONUNBUFFERED=1

[Install]
WantedBy=multi-user.target
```

**Notes:**

- **`User=root`:**  
  For initial setup, running as `root` is acceptable. However, transitioning to a dedicated non-root user is recommended for enhanced security.

- **`StandardOutput` and `StandardError`:**  
  Redirects the script's standard output and error to log files for monitoring.

### **7.3. Configure Logging**

1. **Create Log Files:**

   ```bash
   sudo touch /var/log/rrd_sync.log /var/log/rrd_sync_error.log
   ```

2. **Set Appropriate Permissions:**

   ```bash
   sudo chmod 644 /var/log/rrd_sync.log /var/log/rrd_sync_error.log
   sudo chown root:root /var/log/rrd_sync.log /var/log/rrd_sync_error.log
   ```

### **7.4. Reload systemd and Enable the Service**

```bash
sudo systemctl daemon-reload
sudo systemctl enable rrd_sync.service
sudo systemctl start rrd_sync.service
```

### **7.5. Verify Service Status**

```bash
sudo systemctl status rrd_sync.service
```

**Expected Output:**

```
● rrd_sync.service - RRD File Synchronization Service
     Loaded: loaded (/etc/systemd/system/rrd_sync.service; enabled; vendor preset: disabled)
     Active: active (running) since Wed 2024-12-11 16:07:48 IST; 8s ago
   Main PID: 570144 (python3)
      Tasks: 4 (limit: 48775)
     Memory: 8.4M
        CPU: 174ms
     CGroup: /system.slice/rrd_sync.service
             └─570144 /usr/bin/python3 /opt/opennms/rrd_data_move/rrd_sync_cross_server.py
```

*If the service shows as active and running, proceed to the next steps.*

---

## **8. Implementing Log Rotation**

Prevent log files from growing indefinitely by setting up log rotation using `logrotate`.

### **8.1. Create a Logrotate Configuration File**

```bash
sudo nano /etc/logrotate.d/rrd_sync
```

### **8.2. Add the Following Content**

```ini
/var/log/rrd_sync.log /var/log/rrd_sync_error.log {
    daily
    missingok
    rotate 7
    compress
    delaycompress
    notifempty
    create 644 root root
    sharedscripts
    postrotate
        systemctl restart rrd_sync.service > /dev/null
    endscript
}
```

**Explanation:**

- **`daily`:** Rotate logs daily.
- **`rotate 7`:** Keep seven rotated logs.
- **`compress` and `delaycompress`:** Compress old logs to save space.
- **`notifempty`:** Do not rotate empty logs.
- **`create 644 root root`:** Create new log files with specified permissions.
- **`postrotate` Script:** Restarts the service to ensure it continues logging after rotation.

### **8.3. Test Logrotate Configuration**

```bash
sudo logrotate -d /etc/logrotate.d/rrd_sync
```

- **`-d`:** Debug mode; shows what would happen without making changes.

*If the output looks correct, proceed to perform actual rotation without the `-d` flag.*

---

## **9. Transitioning to a Dedicated Non-Root User**

For enhanced security, it's best practice to run services under a dedicated non-root user.

### **9.1. Create a Dedicated User on Source and Destination Servers**

**On Both Servers:**

```bash
sudo adduser rrd_sync_user
sudo passwd rrd_sync_user  # Set a strong password
```

### **9.2. Set Up SSH Key-Based Authentication for `rrd_sync_user`**

**On Source Server (`192.168.29.106`):**

1. **Switch to `rrd_sync_user`:**

   ```bash
   sudo su - rrd_sync_user
   ```

2. **Generate SSH Key Pair:**

   ```bash
   ssh-keygen -t rsa -b 4096 -C "rrd_sync@source" -f ~/.ssh/id_rsa_rrd_sync -N ""
   ```

3. **Copy Public Key to Destination Server:**

   ```bash
   ssh-copy-id -i ~/.ssh/id_rsa_rrd_sync.pub rrd_sync_user@192.168.29.153
   ```

   *Alternatively, use the manual method described earlier.*

4. **Test SSH Connection:**

   ```bash
   ssh -i ~/.ssh/id_rsa_rrd_sync rrd_sync_user@192.168.29.153 "echo 'SSH connection successful!'"
   ```

   - **Expected Output:**

     ```
     SSH connection successful!
     ```

### **9.3. Update Synchronization Script for Dedicated User**

**On Source Server:**

1. **Edit the Script:**

   ```bash
   sudo vi /opt/opennms/rrd_data_move/rrd_sync_cross_server.py
   ```

2. **Modify Configuration Variables:**

   ```python
   REMOTE_USER = "rrd_sync_user"  # Dedicated user
   REMOTE_HOST = "192.168.29.153"  # Destination server IP
   SSH_KEY_PATH = "/home/rrd_sync_user/.ssh/id_rsa_rrd_sync"  # Path to the SSH private key
   ```

3. **Save and Exit.**

4. **Set Correct Ownership and Permissions:**

   ```bash
   sudo chown rrd_sync_user:rrd_sync_user /opt/opennms/rrd_data_move/rrd_sync_cross_server.py
   sudo chmod 700 /opt/opennms/rrd_data_move/rrd_sync_cross_server.py
   ```

### **9.4. Update systemd Service to Use `rrd_sync_user`**

1. **Edit the Service File:**

   ```bash
   sudo nano /etc/systemd/system/rrd_sync.service
   ```

2. **Modify the `[Service]` Section:**

   ```ini
   [Service]
   Type=simple
   ExecStart=/usr/bin/python3 /opt/opennms/rrd_data_move/rrd_sync_cross_server.py
   Restart=on-failure
   RestartSec=30
   User=rrd_sync_user  # Change to dedicated user
   Environment=PYTHONUNBUFFERED=1

   # Logging
   StandardOutput=append:/var/log/rrd_sync.log
   StandardError=append:/var/log/rrd_sync_error.log
   ```

3. **Save and Exit.**

4. **Set Correct Permissions for the Script:**

   ```bash
   sudo chown rrd_sync_user:rrd_sync_user /opt/opennms/rrd_data_move/rrd_sync_cross_server.py
   sudo chmod 700 /opt/opennms/rrd_data_move/rrd_sync_cross_server.py
   ```

5. **Reload systemd Daemon:**

   ```bash
   sudo systemctl daemon-reload
   ```

6. **Restart and Enable the Service:**

   ```bash
   sudo systemctl restart rrd_sync.service
   sudo systemctl enable rrd_sync.service
   ```

7. **Verify Service Status:**

   ```bash
   sudo systemctl status rrd_sync.service
   ```

   **Expected Output:**

   ```
   ● rrd_sync.service - RRD File Synchronization Service
        Loaded: loaded (/etc/systemd/system/rrd_sync.service; enabled; vendor preset: enabled)
        Active: active (running) since Wed 2024-12-11 16:10:00 IST; 1s ago
      Main PID: 570200 (python3)
         Tasks: 4 (limit: 48775)
        Memory: 8.4M
           CPU: 174ms
        CGroup: /system.slice/rrd_sync.service
                └─570200 /usr/bin/python3 /opt/opennms/rrd_data_move/rrd_sync_cross_server.py
   ```

---

## **10. Testing the Setup**

Ensure that the synchronization process works as intended by performing both initial and real-time tests.

### **10.1. Initial Synchronization Test**

1. **Create a Test `.rrd` File on Source Server:**

   ```bash
   sudo -u rrd_sync_user touch /opt/opennms/rrd_data_move/snmp/1/opennms-jvm/TestFile1.rrd
   ```

2. **Monitor Logs on Source Server:**

   ```bash
   sudo tail -f /var/log/rrd_sync.log
   sudo tail -f /var/log/rrd_sync_error.log
   ```

   **Expected Log Entries:**

   ```
   Syncing TestFile1.rrd...
   Synced TestFile1.rrd successfully.
   ```

3. **Verify File on Destination Server:**

   ```bash
   sudo ls -l /opt/opennms/rrd_data_move/destination/snmp/1/opennms-jvm/
   ```

   **Expected Output:**

   ```
   -rw-r--r-- 1 root root 0 Dec 11 16:10 TestFile1.rrd
   ```

### **10.2. Real-Time Synchronization Test**

1. **Modify an Existing `.rrd` File on Source Server:**

   ```bash
   sudo -u rrd_sync_user touch /opt/opennms/rrd_data_move/snmp/1/opennms-jvm/TestFile2.rrd
   ```

2. **Monitor Logs:**

   ```bash
   sudo tail -f /var/log/rrd_sync.log
   sudo tail -f /var/log/rrd_sync_error.log
   ```

   **Expected Log Entries:**

   ```
   Syncing TestFile2.rrd...
   Synced TestFile2.rrd successfully.
   ```

3. **Verify File on Destination Server:**

   ```bash
   sudo ls -l /opt/opennms/rrd_data_move/destination/snmp/1/opennms-jvm/
   ```

   **Expected Output:**

   ```
   -rw-r--r-- 1 root root 0 Dec 11 16:15 TestFile2.rrd
   ```

---

## **11. Monitoring and Maintenance**

### **11.1. Continuous Log Monitoring**

Regularly monitor the synchronization logs to ensure the service operates smoothly.

```bash
sudo tail -f /var/log/rrd_sync.log
sudo tail -f /var/log/rrd_sync_error.log
```

### **11.2. Using `journalctl` for Detailed Logs**

Access detailed logs managed by `systemd`.

```bash
sudo journalctl -u rrd_sync.service -f
```

### **11.3. Verify Service Auto-Start on Reboot**

1. **Reboot Source Server:**

   ```bash
   sudo reboot
   ```

2. **After Reboot, Check Service Status:**

   ```bash
   sudo systemctl status rrd_sync.service
   ```

   **Expected Output:**

   ```
   ● rrd_sync.service - RRD File Synchronization Service
        Loaded: loaded (/etc/systemd/system/rrd_sync.service; enabled; vendor preset: enabled)
        Active: active (running) since Wed 2024-12-11 16:20:00 IST; 5s ago
      Main PID: 570300 (python3)
         Tasks: 4 (limit: 48775)
        Memory: 8.4M
           CPU: 200ms
        CGroup: /system.slice/rrd_sync.service
                └─570300 /usr/bin/python3 /opt/opennms/rrd_data_move/rrd_sync_cross_server.py
   ```

---

## **12. Troubleshooting**

If the service fails to start or behaves unexpectedly, follow these troubleshooting steps.

### **12.1. Check Service Logs**

```bash
sudo journalctl -u rrd_sync.service -b
```

**Common Errors:**

- **User Does Not Exist:**

  ```
  Failed to determine user credentials
  Failed at step USER spawning /usr/bin/python3: No such process
  ```

  **Solution:**  
  Ensure that the `User` specified in the service file exists on the Source Server.

- **Incorrect `ExecStart` Path:**

  ```
  ExecStart=/usr/bin/python3 /opt/opennms/rrd_data_move/rrd_sync_cross_server.py: No such file or directory
  ```

  **Solution:**  
  Verify the paths to Python and the synchronization script.

- **Script Errors:**

  ```
  rrd_sync_cross_server.py: error: some error message
  ```

  **Solution:**  
  Run the script manually to identify and fix errors.

### **12.2. Verify SSH Connectivity**

Ensure that the Source Server can SSH into the Destination Server without a password.

```bash
sudo ssh -i /home/rrd_sync_user/.ssh/id_rsa_rrd_sync rrd_sync_user@192.168.29.153 "echo 'SSH connection successful!'"
```

**Expected Output:**

```
SSH connection successful!
```

### **12.3. Check File Permissions**

Ensure that the `rrd_sync_user` has the necessary permissions on both servers.

```bash
# On Source Server
sudo ls -l /opt/opennms/rrd_data_move/snmp/1/opennms-jvm/

# On Destination Server
sudo ls -l /opt/opennms/rrd_data_move/destination/snmp/1/opennms-jvm/
```

### **12.4. Confirm Python Dependencies**

Ensure that all required Python packages are installed.

```bash
sudo pip3 list | grep watchdog
```

*If not installed, install using:*

```bash
sudo pip3 install watchdog
```

### **12.5. SELinux Considerations**

If SELinux is enforcing, it might block the synchronization script.

1. **Check SELinux Status:**

   ```bash
   sestatus
   ```

2. **Temporarily Set SELinux to Permissive Mode:**

   ```bash
   sudo setenforce 0
   sudo systemctl restart rrd_sync.service
   ```

   - **If Service Starts Successfully:**  
     Configure SELinux policies to allow the necessary operations.

3. **Re-enable Enforcing Mode:**

   ```bash
   sudo setenforce 1
   ```

### **12.6. Restarting Services**

After making changes, always reload `systemd` and restart the service.

```bash
sudo systemctl daemon-reload
sudo systemctl restart rrd_sync.service
sudo systemctl status rrd_sync.service
```

---

## **13. Security Best Practices**

Enhance the security of your synchronization setup by implementing the following best practices.

### **13.1. Use a Dedicated Non-Root User**

Running services as `root` can pose security risks. It's recommended to use a dedicated user with limited privileges.

1. **Create `rrd_sync_user` on Both Servers:**

   ```bash
   sudo adduser rrd_sync_user
   sudo passwd rrd_sync_user  # Set a strong password
   ```

2. **Set Ownership and Permissions:**

   ```bash
   sudo chown -R rrd_sync_user:rrd_sync_user /opt/opennms/rrd_data_move/
   sudo chmod -R 755 /opt/opennms/rrd_data_move/
   ```

3. **Update SSH Configuration:**

   On the Destination Server (`192.168.29.153`), restrict SSH access for `rrd_sync_user`.

   ```bash
   sudo nano /etc/ssh/sshd_config
   ```

   **Add the Following at the End:**

   ```ini
   Match User rrd_sync_user
       PasswordAuthentication no
       PermitRootLogin no
       AllowTcpForwarding no
       X11Forwarding no
   ```

   **Save and Exit, Then Restart SSH Service:**

   ```bash
   sudo systemctl restart sshd
   ```

### **13.2. Secure SSH Keys**

1. **Restrict Permissions on Private Keys:**

   ```bash
   sudo chmod 600 /home/rrd_sync_user/.ssh/id_rsa_rrd_sync
   sudo chown rrd_sync_user:rrd_sync_user /home/rrd_sync_user/.ssh/id_rsa_rrd_sync
   ```

2. **Disable SSH Password Authentication (After Ensuring Key-Based Access):**

   ```bash
   sudo nano /etc/ssh/sshd_config
   ```

   **Modify or Add:**

   ```ini
   PasswordAuthentication no
   ```

   **Save and Exit, Then Restart SSH Service:**

   ```bash
   sudo systemctl restart sshd
   ```

### **13.3. Limit File Permissions**

Ensure that only necessary users have access to synchronization directories and files.

```bash
sudo chmod -R 700 /opt/opennms/rrd_data_move/
sudo chown -R rrd_sync_user:rrd_sync_user /opt/opennms/rrd_data_move/
```

### **13.4. Regularly Update the System**

Keep both servers updated to protect against vulnerabilities.

```bash
sudo yum update -y
```

### **13.5. Implement Firewall Rules**

Ensure that only necessary ports are open.

**On Both Servers:**

1. **Check Current Firewall Rules:**

   ```bash
   sudo firewall-cmd --list-all
   ```

2. **Allow Only Required Services (e.g., SSH):**

   ```bash
   sudo firewall-cmd --permanent --add-service=ssh
   sudo firewall-cmd --reload
   ```

---

## **14. Conclusion**

By following this guide, administrators can set up a secure, reliable, and automated system for synchronizing `.rrd` files between RHEL-based servers. The use of `systemd` ensures that the synchronization service is managed efficiently, with automatic restarts in case of failures and persistence across system reboots. Implementing security best practices, such as using dedicated non-root users and securing SSH configurations, further fortifies the setup against potential threats.

**Key Takeaways:**

- **Automation:** `systemd` manages the synchronization script, ensuring continuous operation.
- **Security:** SSH key-based authentication and dedicated users minimize security risks.
- **Reliability:** Log rotation and monitoring ensure the system remains operational and manageable.
- **Scalability:** The guide can be adapted to various server configurations by customizing IP addresses and directories.

For any further assistance or advanced configurations, refer to the official documentation of the respective tools (`rsync`, `systemd`, `watchdog`) or consult with experienced system administrators.

---

# **Appendix**

## **A. Sample `systemd` Service File**

```ini
[Unit]
Description=RRD File Synchronization Service
After=network.target sshd.service

[Service]
Type=simple
ExecStart=/usr/bin/python3 /opt/opennms/rrd_data_move/rrd_sync_cross_server.py
Restart=on-failure
RestartSec=30
User=rrd_sync_user  # Change to root if not using a dedicated user

# Logging
StandardOutput=append:/var/log/rrd_sync.log
StandardError=append:/var/log/rrd_sync_error.log

# Environment
Environment=PYTHONUNBUFFERED=1

[Install]
WantedBy=multi-user.target
```

## **B. Sample Synchronization Script Requirements File**

If you prefer managing Python dependencies via a `requirements.txt` file, create one as follows:

**Create `requirements.txt`:**

```bash
sudo vi /opt/opennms/rrd_data_move/requirements.txt
```

**Add the Following Content:**

```
watchdog
```

**Install Dependencies:**

```bash
sudo pip3 install -r /opt/opennms/rrd_data_move/requirements.txt
```

---

# **Glossary**

- **RRD (Round-Robin Database):** A system to store and display time-series data like network bandwidth, temperatures, CPU load, etc.
- **`rsync`:** A utility for efficiently transferring and synchronizing files across computer systems.
- **`systemd`:** A system and service manager for Linux operating systems.
- **`watchdog`:** A Python API library and shell utilities to monitor file system events.
- **SELinux (Security-Enhanced Linux):** A set of kernel modifications and user-space tools that provide mandatory access control mechanisms.

---

# **References**

- [rsync Documentation](https://download.samba.org/pub/rsync/rsync.html)
- [systemd Manual](https://www.freedesktop.org/software/systemd/man/systemd.service.html)
- [watchdog Documentation](https://python-watchdog.readthedocs.io/en/stable/)
- [RHEL/CentOS Documentation](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/)

---

**End of Documentation**
