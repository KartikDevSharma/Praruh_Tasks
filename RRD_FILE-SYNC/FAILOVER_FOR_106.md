
---

## **Table of Contents**

1. [Introduction](#1-introduction)
2. [Objectives](#2-objectives)
3. [Prerequisites](#3-prerequisites)
4. [Step 1: Setting Up Secondary Directories](#step-1-setting-up-secondary-directories)
5. [Step 2: Creating Sample Data](#step-2-creating-sample-data)
6. [Step 3: Generating SSH Key Pair for Secondary Synchronization](#step-3-generating-ssh-key-pair-for-secondary-synchronization)
7. [Step 4: Copying Public Key to Destination Server (Server 106)](#step-4-copying-public-key-to-destination-server-server-106)
8. [Step 5: Updating the Secondary Configuration File](#step-5-updating-the-secondary-configuration-file)
9. [Step 6: Developing the Secondary Python Synchronization Script](#step-6-developing-the-secondary-python-synchronization-script)
10. [Step 7: Configuring the Secondary `systemd` Service](#step-7-configuring-the-secondary-systemd-service)
11. [Step 8: Testing the Secondary Synchronization Service](#step-8-testing-the-secondary-synchronization-service)
12. [Step 9: Switching Roles in Case of Failure](#step-9-switching-roles-in-case-of-failure)
13. [Step 10: Implementing Failover Switching Script](#step-10-implementing-failover-switching-script)
14. [Additional Recommendations](#step-14-additional-recommendations)
15. [Conclusion](#15-conclusion)

---

## **1. Introduction**

Ensuring **continuous availability** of data across critical systems is paramount in any IT infrastructure. While your existing synchronization setup efficiently handles data replication from **Server 106** to **Server 254**, introducing a **failover mechanism** ensures that operations remain uninterrupted even if the primary server encounters issues. This guide details the process of establishing a **secondary synchronization service** that allows **Server 254** to take over as the primary source, thereby maintaining data integrity and availability.

---

## **2. Objectives**

By the end of this guide, you will:

- **Establish a secondary synchronization service** that can reverse roles between Server 106 and Server 254.
- **Ensure seamless failover** without disrupting existing synchronization processes.
- **Maintain data consistency** across both servers during role transitions.
- **Automate the failover process** for quick response during failures.

---

## **3. Prerequisites**

Before proceeding, ensure you have the following:

- **Access to Both Servers:**
  - **Source Server (Primary):** Server 106
  - **Destination Server:** Server 254

- **User Permissions:**
  - **Root Access** on both servers.

- **Installed Software:**
  - **Python 3.x**
  - **`rsync`**
  - **`systemd`**
  - **Python Packages:**
    - `watchdog` (Installed via `pip3 install watchdog`)

- **Network Configuration:**
  - **SSH Access** from Server 254 to Server 106 using SSH keys for password-less authentication.

---

## **4. Step 1: Setting Up Secondary Directories**

To prevent interference with your existing synchronization setup, create **secondary directories** dedicated to the failover synchronization process.

### **On Server 254 (Destination Server):**

1. **Create Secondary Source and Destination Directories:**

   ```bash
   sudo mkdir -p /opt/opennms/rrd_data_move_secondary/source/
   sudo mkdir -p /opt/opennms/rrd_data_move_secondary/destination/
   ```

2. **Set Permissions:**

   ```bash
   sudo chown -R root:root /opt/opennms/rrd_data_move_secondary/
   sudo chmod -R 750 /opt/opennms/rrd_data_move_secondary/
   ```

   *Since you're operating as `root`, ownership is assigned to `root`.*

### **On Server 106 (Destination Server for Failover):**

1. **Create Secondary Source and Destination Directories:**

   ```bash
   sudo mkdir -p /opt/opennms/rrd_data_move_secondary/source/
   sudo mkdir -p /opt/opennms/rrd_data_move_secondary/destination/
   ```

2. **Set Permissions:**

   ```bash
   sudo chown -R root:root /opt/opennms/rrd_data_move_secondary/
   sudo chmod -R 750 /opt/opennms/rrd_data_move_secondary/
   ```

---

## **5. Step 2: Creating Sample Data**

Populate the **secondary source directory** on **Server 254** with sample files to test the synchronization process.

### **On Server 254:**

1. **Navigate to the Secondary Source Directory:**

   ```bash
   cd /opt/opennms/rrd_data_move_secondary/source/
   ```

2. **Create Sample Files:**

   ```bash
   sudo touch sample1.rrd
   sudo touch sample2.txt
   sudo touch sample3.properties
   sudo touch sample4.meta
   ```

3. **Add Content to Sample Files (Optional):**

   ```bash
   sudo echo "Sample data for RRD file" > sample1.rrd
   sudo echo "Sample text content" > sample2.txt
   sudo echo "property=value" > sample3.properties
   sudo echo "meta information" > sample4.meta
   ```

---

## **6. Step 3: Generating SSH Key Pair for Secondary Synchronization**

Generate an **SSH key pair** on **Server 254** to enable password-less SSH access to **Server 106**.

### **On Server 254:**

1. **Verify `.ssh` Directory Exists for Root:**

   ```bash
   ls -ld /root/.ssh
   ```

   **If the directory does not exist:**

   ```bash
   sudo mkdir -p /root/.ssh
   sudo chmod 700 /root/.ssh
   ```

2. **Generate the SSH Key Pair:**

   ```bash
   sudo ssh-keygen -t rsa -b 4096 -C "rrd_sync_secondary" -f /root/.ssh/id_rsa_rrd_sync_secondary -N ""
   ```

   **Explanation of Flags:**

   - **`-t rsa`**: Specifies the RSA algorithm.
   - **`-b 4096`**: Sets the key length to 4096 bits.
   - **`-C "rrd_sync_secondary"`**: Adds a comment for identification.
   - **`-f /root/.ssh/id_rsa_rrd_sync_secondary`**: Specifies the filename for the key.
   - **`-N ""`**: Sets an empty passphrase for automated scripts.

3. **Verify Key Generation:**

   ```bash
   ls -l /root/.ssh/id_rsa_rrd_sync_secondary
   ls -l /root/.ssh/id_rsa_rrd_sync_secondary.pub
   ```

   **Expected Output:**

   ```
   -rw------- 1 root root 3243 Dec 26 21:30 /root/.ssh/id_rsa_rrd_sync_secondary
   -rw-r--r-- 1 root root  744 Dec 26 21:30 /root/.ssh/id_rsa_rrd_sync_secondary.pub
   ```

---

## **7. Step 4: Copying Public Key to Destination Server (Server 106)**

To allow **Server 254** to SSH into **Server 106** without a password, append the public key to **Server 106**'s `authorized_keys`.

### **On Server 254:**

1. **Copy Public Key Using `ssh-copy-id`:**

   ```bash
   sudo ssh-copy-id -i /root/.ssh/id_rsa_rrd_sync_secondary.pub root@192.168.29.106
   ```

   **Replace:**

   - **`root@192.168.29.106`**: With the actual username and IP address of **Server 106**.

   **Note:** If `ssh-copy-id` is not available or you encounter issues, use the manual method below.

2. **Alternatively, Manually Append the Public Key:**

   ```bash
   sudo cat /root/.ssh/id_rsa_rrd_sync_secondary.pub | ssh root@192.168.29.106 "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
   ```

3. **Verify SSH Key is Added on Server 106:**

   SSH into **Server 106** and confirm the key is present.

   ```bash
   ssh root@192.168.29.106 "grep 'rrd_sync_secondary' ~/.ssh/authorized_keys"
   ```

   **Expected Output:**

   ```
   ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQC... rrd_sync_secondary
   ```

---

## **8. Step 5: Updating the Secondary Configuration File**

Ensure that the **secondary configuration file** on **Server 254** correctly references the newly generated SSH key.

### **On Server 254:**

1. **Open the Secondary Configuration File:**

   ```bash
   sudo nano /opt/opennms/rrd_monitoring_secondary/config_rrdPyScript_secondary.json
   ```

2. **Update the `ssh_key_path`:**

   Ensure that the `ssh_key_path` points to `/root/.ssh/id_rsa_rrd_sync_secondary`.

   **Example Configuration:**

   ```json
   {
       "source_dir": "/opt/opennms/rrd_data_move_secondary/source/",
       "state_file": "/opt/opennms/rrd_monitoring_secondary/rrdPyScript_secondary.json",
       "remote_sync": {
           "remote_user": "root",
           "remote_host": "192.168.29.106",
           "remote_dest_dir": "/opt/opennms/rrd_data_move_secondary/destination/",
           "ssh_key_path": "/root/.ssh/id_rsa_rrd_sync_secondary"
       },
       "logging": {
           "log_file": "/opt/opennms/rrd_monitoring_secondary/rrdPyScript_secondary.log",
           "log_level": "INFO"
       },
       "debounce_time": 5,
       "monitored_extensions": [
           ".rrd",
           ".txt",
           ".properties",
           ".meta"
       ],
       "rsync_options": {
           "archive": true,
           "compress": true,
           "verbose": true,
           "update": true,
           "checksum": true,
           "partial": true,
           "bwlimit": 1000,
           "itemize_changes": true,
           "stats": true
       }
   }
   ```

3. **Save and Exit:**

   - **In `nano`:** Press `CTRL + O` to save, `ENTER` to confirm, and `CTRL + X` to exit.

---

## **9. Step 6: Developing the Secondary Python Synchronization Script**

Create a **secondary Python script** tailored to use the secondary configuration for failover synchronization.

### **On Server 254:**

1. **Create the Synchronization Script:**

   ```bash
   sudo nano /opt/opennms/rrd_monitoring_secondary/rrdPyScript_secondary.py
   ```

2. **Populate the Python Script:**

   **Option 1: Duplicate and Modify Existing Script**

   - **Copy the Existing Script:**

     ```bash
     sudo cp /opt/opennms/rrd_monitoring/rrdPyScript.py /opt/opennms/rrd_monitoring_secondary/rrdPyScript_secondary.py
     ```

   - **Edit the Secondary Script:**

     ```bash
     sudo nano /opt/opennms/rrd_monitoring_secondary/rrdPyScript_secondary.py
     ```

   - **Update the Configuration File Path:**

     Find the line:

     ```python
     CONFIG_FILE = "/opt/opennms/rrd_monitoring/config_rrdPyScript.json"
     ```

     **Change it to:**

     ```python
     CONFIG_FILE = "/opt/opennms/rrd_monitoring_secondary/config_rrdPyScript_secondary.json"
     ```

   **Option 2: Write a New Script**

   If you prefer to have separate scripts, ensure that all paths and configurations point to the secondary setup.

3. **Make the Script Executable:**

   ```bash
   sudo chmod +x /opt/opennms/rrd_monitoring_secondary/rrdPyScript_secondary.py
   ```

4. **Install Python Dependencies (If Not Already Installed):**

   ```bash
   sudo pip3 install watchdog
   ```

---

## **10. Step 7: Configuring the Secondary `systemd` Service**

Set up a **`systemd` service** to manage the secondary synchronization script, ensuring it runs continuously and restarts on failure.

### **On Server 254:**

1. **Create the `systemd` Service File:**

   ```bash
   sudo nano /etc/systemd/system/rrdPyScript_secondary.service
   ```

2. **Populate the Service File:**

   ```ini
   [Unit]
   Description=RRD File-Level Synchronization Service Secondary
   After=network.target
   
   [Service]
   Type=simple
   ExecStart=/usr/bin/python3 /opt/opennms/rrd_monitoring_secondary/rrdPyScript_secondary.py
   Restart=on-failure
   RestartSec=30
   User=root
   Environment=PYTHONUNBUFFERED=1
   
   # Logging
   StandardOutput=append:/opt/opennms/rrd_monitoring_secondary/rrdPyScript_secondary.log
   StandardError=append:/opt/opennms/rrd_monitoring_secondary/rrdPyScript_secondary.log
   
   [Install]
   WantedBy=multi-user.target
   ```

   **Important Considerations:**

   - **`User=root`:** Since you're operating as `root`, set the user to `root`.
   - **Logging Paths:** Ensure that the log file path points to the **secondary log directory**.

3. **Set Permissions for the Log File:**

   ```bash
   sudo touch /opt/opennms/rrd_monitoring_secondary/rrdPyScript_secondary.log
   sudo chown root:root /opt/opennms/rrd_monitoring_secondary/rrdPyScript_secondary.log
   sudo chmod 644 /opt/opennms/rrd_monitoring_secondary/rrdPyScript_secondary.log
   ```

4. **Reload `systemd` to Recognize the New Service:**

   ```bash
   sudo systemctl daemon-reload
   ```

5. **Enable the Secondary Service to Start on Boot:**

   ```bash
   sudo systemctl enable rrdPyScript_secondary.service
   ```

6. **Start the Secondary Service:**

   ```bash
   sudo systemctl start rrdPyScript_secondary.service
   ```

---

## **11. Step 8: Testing the Secondary Synchronization Service**

Before integrating the secondary service into your main directories, ensure it functions correctly.

### **8.1. Verify Service Status**

```bash
sudo systemctl status rrdPyScript_secondary.service
```

**Expected Output:**

```
● rrdPyScript_secondary.service - RRD File-Level Synchronization Service Secondary
     Loaded: loaded (/etc/systemd/system/rrdPyScript_secondary.service; enabled; vendor preset: enabled)
     Active: active (running) since Thu 2024-12-26 21:45:00 UTC; 1min ago
   Main PID: 12346 (python3)
      Tasks: 1 (limit: 4915)
     Memory: 50.0M
        CPU: 0.10s
     CGroup: /system.slice/rrdPyScript_secondary.service
             └─12346 /usr/bin/python3 /opt/opennms/rrd_monitoring_secondary/rrdPyScript_secondary.py
```

### **8.2. Monitor Service Logs**

```bash
sudo tail -f /opt/opennms/rrd_monitoring_secondary/rrdPyScript_secondary.log
```

**Look For:**

- **Startup Messages:** Indicating that the script has initiated.
- **Synchronization Attempts:** Logs showing attempts to sync files.
- **Error Messages:** Any issues or failures during synchronization.

### **8.3. Test Synchronization**

1. **Create a New File in the Secondary Source Directory:**

   ```bash
   sudo touch /opt/opennms/rrd_data_move_secondary/source/test_secondary.txt
   ```

2. **Verify Synchronization on Destination (Server 106):**

   SSH into **Server 106** and check if the file exists in the destination directory.

   ```bash
   ssh root@192.168.29.106 "ls /opt/opennms/rrd_data_move_secondary/destination/"
   ```

   **Expected Output:**

   ```
   test_secondary.txt
   ```

3. **Modify an Existing File:**

   ```bash
   sudo echo "Secondary sync test" >> /opt/opennms/rrd_data_move_secondary/source/test_secondary.txt
   ```

4. **Verify Modification on Destination:**

   ```bash
   ssh root@192.168.29.106 "cat /opt/opennms/rrd_data_move_secondary/destination/test_secondary.txt"
   ```

   **Expected Output:**

   ```
   Secondary sync test
   ```

5. **Delete a File:**

   ```bash
   sudo rm /opt/opennms/rrd_data_move_secondary/source/test_secondary.txt
   ```

6. **Verify Deletion on Destination:**

   ```bash
   ssh root@192.168.29.106 "ls /opt/opennms/rrd_data_move_secondary/destination/"
   ```

   **Expected Output:**

   ```
   # test_secondary.txt should no longer be listed
   ```

   **Note:** Ensure your Python script handles deletions appropriately based on your implementation. If deletions are not being synchronized, review the deletion logic in your script.

---

## **12. Step 9: Switching Roles in Case of Failure**

In the event that **Server 106** (primary source) experiences a failure, you can **activate the secondary synchronization service** to make **Server 254** the new primary source. Follow these steps to ensure a **seamless transition**.

### **9.1. Understanding the Setup**

- **Primary Setup:**
  - **Source:** Server 106
  - **Destination:** Server 254

- **Secondary Setup:**
  - **Source:** Server 254
  - **Destination:** Server 106

### **9.2. Steps to Activate Secondary Synchronization**

1. **Ensure Server 106 is Down or Unreachable:**

   Before switching, verify that **Server 106** is indeed down to prevent synchronization conflicts.

   ```bash
   ping -c 4 192.168.29.106
   ```

   **Expected Outcome:**

   - **Successful Ping:** Server 106 is reachable.
   - **Unsuccessful Ping:** Indicates Server 106 is down.

2. **Stop the Primary Synchronization Service on Server 106 (If Partially Functional):**

   If Server 106 is still operational but you want to switch roles, stop its synchronization service to prevent data conflicts.

   ```bash
   ssh root@192.168.29.106 "systemctl stop rrdPyScript.service"
   ```

3. **Start the Secondary Synchronization Service on Server 254:**

   Activate the secondary service to begin synchronizing from **Server 254** to **Server 106**.

   ```bash
   sudo systemctl start rrdPyScript_secondary.service
   ```

4. **Enable the Secondary Service to Start on Boot (If Not Already Enabled):**

   ```bash
   sudo systemctl enable rrdPyScript_secondary.service
   ```

5. **Verify the Secondary Service is Running:**

   ```bash
   sudo systemctl status rrdPyScript_secondary.service
   ```

   **Expected Output:**

   ```
   ● rrdPyScript_secondary.service - RRD File-Level Synchronization Service Secondary
        Loaded: loaded (/etc/systemd/system/rrdPyScript_secondary.service; enabled; vendor preset: enabled)
        Active: active (running) since Thu 2024-12-26 21:10:00 UTC; 1min ago
      Main PID: 12346 (python3)
         Tasks: 1 (limit: 4915)
        Memory: 50.0M
           CPU: 0.10s
        CGroup: /system.slice/rrdPyScript_secondary.service
                └─12346 /usr/bin/python3 /opt/opennms/rrd_monitoring_secondary/rrdPyScript_secondary.py
   ```

6. **Monitor Synchronization:**

   Ensure that files are being synchronized correctly by monitoring logs.

   ```bash
   sudo tail -f /opt/opennms/rrd_monitoring_secondary/rrdPyScript_secondary.log
   ```

7. **Notify Stakeholders:**

   Inform relevant personnel about the role switch to prevent confusion and ensure coordinated actions.

---

## **13. Step 10: Implementing Failover Switching Script**

To streamline the **role-switching process**, create a **shell script** that automates stopping the primary service and starting the secondary service.

### **On Server 254:**

1. **Create the Switching Script:**

   ```bash
   sudo nano /root/switch_sync_roles.sh
   ```

2. **Populate the Script:**

   ```bash
   #!/bin/bash

   PRIMARY_SERVICE="rrdPyScript.service"
   SECONDARY_SERVICE="rrdPyScript_secondary.service"

   echo "Stopping primary synchronization service on Server 106..."
   ssh root@192.168.29.106 "systemctl stop $PRIMARY_SERVICE"

   echo "Starting secondary synchronization service on Server 254..."
   sudo systemctl start $SECONDARY_SERVICE

   echo "Synchronization roles have been switched successfully."
   ```

3. **Make the Script Executable:**

   ```bash
   sudo chmod +x /root/switch_sync_roles.sh
   ```

4. **Usage:**

   Whenever you need to switch roles (e.g., Server 106 fails), execute the script:

   ```bash
   sudo /root/switch_sync_roles.sh
   ```

   **Note:** Ensure that you have SSH access to **Server 106** from **Server 254** without password prompts, as configured earlier.

---

## **14. Step 11: Additional Recommendations**

To maintain a robust and secure synchronization environment, consider the following best practices:

### **11.1. Regularly Monitor Service Status**

Implement monitoring to ensure both primary and secondary services are running as expected.

```bash
sudo systemctl status rrdPyScript.service
sudo systemctl status rrdPyScript_secondary.service
```

### **11.2. Implement Logging and Alerts**

- **Centralized Logging:** Utilize tools like `journalctl` or log aggregation services to monitor logs efficiently.
- **Alerts:** Configure alerts for service failures or synchronization errors using tools like **Nagios**, **Prometheus**, or **Mail Alerts**.

### **11.3. Schedule Regular Failover Tests**

Periodically simulate server failures to ensure the failover mechanism operates smoothly. This proactive approach helps in identifying and resolving potential issues before actual failures occur.

### **11.4. Secure Your SSH Keys**

Ensure that your SSH private keys are **securely stored** and **access is restricted**.

```bash
chmod 600 /root/.ssh/id_rsa_rrd_sync_secondary
chown root:root /root/.ssh/id_rsa_rrd_sync_secondary
```

### **11.5. Backup Configuration and State Files**

Regularly back up your configuration and state files to prevent data loss.

```bash
cp /opt/opennms/rrd_monitoring_secondary/config_rrdPyScript_secondary.json /backup/config_rrdPyScript_secondary.json.bak
cp /opt/opennms/rrd_monitoring_secondary/rrdPyScript_secondary.json /backup/rrdPyScript_secondary.json.bak
```

### **11.6. Review and Update `rsync` Options**

Periodically review your `rsync` options to ensure they align with your synchronization requirements, optimizing performance and security.

---

## **15. Conclusion**

By integrating this **failover mechanism** into your existing file synchronization setup, you've significantly enhanced the **reliability** and **resilience** of your data replication process between **Server 106** and **Server 254**. This dual-service approach ensures that your data remains consistent and available, even in the face of server failures.

**Key Takeaways:**

- **Redundancy:** Establishing a secondary synchronization service provides a safety net against primary server failures.
- **Automation:** Leveraging `systemd` services ensures that synchronization scripts run reliably and restart automatically on failures.
- **Security:** Proper SSH key management and permissions are crucial for secure and seamless synchronization.
- **Testing:** Regularly testing both primary and secondary services ensures operational readiness during actual failover events.

**Next Steps:**

- **Enhance Monitoring:** Implement advanced monitoring solutions to gain real-time insights into your synchronization processes.
- **Documentation:** Maintain up-to-date documentation of your synchronization setup and failover procedures for reference and onboarding.
- **Scalability:** Consider scaling your synchronization setup to accommodate additional servers or more complex data replication needs as your infrastructure grows.

---
