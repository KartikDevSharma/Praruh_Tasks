# **File-Level Monitoring and Synchronization Setup Guide**

**Author:** Kartik Sharma  
**Date:** December 2024

---

## **Table of Contents**

1. [Introduction](#1-introduction)
2. [Prerequisites](#2-prerequisites)
3. [Setting Up the Source Server](#3-setting-up-the-source-server)
   - [3.1. Install Required Packages](#31-install-required-packages)
   - [3.2. Create Necessary Directories](#32-create-necessary-directories)
   - [3.3. Create and Configure the Python Synchronization Script](#33-create-and-configure-the-python-synchronization-script)
4. [Setting Up the Destination Server](#4-setting-up-the-destination-server)
   - [4.1. Install Required Packages](#41-install-required-packages)
   - [4.2. Create Necessary Directories](#42-create-necessary-directories)
   - [4.3. Create a Dedicated Synchronization User](#43-create-a-dedicated-synchronization-user)
5. [Configuring SSH Key-Based Authentication](#5-configuring-ssh-key-based-authentication)
6. [Installing Python Dependencies](#6-installing-python-dependencies)
7. [Configuring and Running the Synchronization Script](#7-configuring-and-running-the-synchronization-script)
   - [7.1. Adjust Script Configuration](#71-adjust-script-configuration)
   - [7.2. Make the Script Executable](#72-make-the-script-executable)
8. [Setting Up the `systemd` Service](#8-setting-up-the-systemd-service)
   - [8.1. Create the Service File](#81-create-the-service-file)
   - [8.2. Enable and Start the Service](#82-enable-and-start-the-service)
9. [Implementing Log Rotation](#9-implementing-log-rotation)
10. [Testing the Synchronization Setup](#10-testing-the-synchronization-setup)
    - [10.1. Initial Synchronization](#101-initial-synchronization)
    - [10.2. Modifying Files](#102-modifying-files)
    - [10.3. Deleting Files](#103-deleting-files)
    - [10.4. Verifying Efficient Synchronization](#104-verifying-efficient-synchronization)
11. [Monitoring and Maintenance](#11-monitoring-and-maintenance)
12. [Security Best Practices](#12-security-best-practices)
13. [Troubleshooting](#13-troubleshooting)
14. [Conclusion](#14-conclusion)

---

## **1. Introduction**

This guide provides a comprehensive step-by-step process to set up a **File-Level Monitoring and Synchronization** system using a Python script powered by `watchdog` and `rsync`. The system monitors specified directories for changes in `.rrd` and `.txt` files, handling creations, modifications, and deletions. It efficiently synchronizes only the changes to a remote server, ensuring optimal performance even with large files exceeding 20 GB.

---

## **2. Prerequisites**

Before beginning the installation, ensure you have the following:

- **Two Servers:**
  - **Source Server:** The server where the files reside and are monitored.
  - **Destination Server:** The server where files are synchronized to.

- **Operating System:** Both servers should be running a Linux distribution (e.g., CentOS, Ubuntu).

- **User Privileges:** Root or sudo access on both servers.

- **Network Connectivity:** Ensure both servers can communicate over SSH (typically on port 22).

- **Python 3 Installed:** Python 3.x should be installed on the source server.

- **Basic Knowledge of Linux Command-Line Operations**

---

## **3. Setting Up the Source Server**

The source server hosts the files to be monitored and synchronized. This section covers installing necessary packages, creating directories, and setting up the synchronization script.

### **3.1. Install Required Packages**

1. **Update System Packages:**

   ```bash
   sudo yum update -y
   ```

2. **Install Python 3 and `pip`:**

   ```bash
   sudo yum install -y python3 python3-pip
   ```

3. **Install `rsync`:**

   ```bash
   sudo yum install -y rsync
   ```

4. **Install Git (Optional):**

   If you plan to manage the script using version control.

   ```bash
   sudo yum install -y git
   ```

### **3.2. Create Necessary Directories**

1. **Create the Synchronization Directory Structure:**

   ```bash
   sudo mkdir -p /opt/opennms/rrd_data_move/snmp/1/opennms-jvm/
   sudo mkdir -p /opt/opennms/rrd_data_move/destination/snmp/2/
   ```

2. **Set Appropriate Permissions:**

   ```bash
   sudo chown -R $(whoami):$(whoami) /opt/opennms/rrd_data_move/
   sudo chmod -R 700 /opt/opennms/rrd_data_move/
   ```

   *Adjust ownership and permissions as needed, especially if using a dedicated synchronization user.*

### **3.3. Create and Configure the Python Synchronization Script**

1. **Navigate to the Synchronization Directory:**

   ```bash
   cd /opt/opennms/rrd_data_move/
   ```

2. **Create the Python Script:**

   ```bash
   sudo nano rrd_sync_new.py
   ```

3. **Paste the Final Script:**

   Insert the final optimized script provided earlier into this file. Ensure that the script includes enhanced logging and debouncing mechanisms for efficient synchronization.

4. **Save and Exit:**

   Press `CTRL + O` to save and `CTRL + X` to exit the editor.

---

## **4. Setting Up the Destination Server**

The destination server receives the synchronized files. This section involves installing required packages, creating directories, and setting up a dedicated synchronization user for enhanced security.

### **4.1. Install Required Packages**

1. **Update System Packages:**

   ```bash
   sudo yum update -y
   ```

2. **Install `rsync`:**

   ```bash
   sudo yum install -y rsync
   ```

### **4.2. Create Necessary Directories**

1. **Create the Destination Directory Structure:**

   ```bash
   sudo mkdir -p /opt/opennms/rrd_data_move/destination/snmp/2/
   ```

2. **Set Appropriate Permissions:**

   ```bash
   sudo chown -R root:root /opt/opennms/rrd_data_move/
   sudo chmod -R 700 /opt/opennms/rrd_data_move/
   ```

   *Adjust ownership and permissions based on the synchronization user setup.*

### **4.3. Create a Dedicated Synchronization User**

For enhanced security, it's recommended to create a dedicated non-root user to handle synchronization tasks.

1. **Create the User:**

   ```bash
   sudo adduser rrd_sync_user
   sudo passwd rrd_sync_user  # Set a strong password
   ```

2. **Grant Necessary Permissions:**

   - **Option 1: SSH Key-Based Access Only (Recommended)**
   
     Restrict the user to SSH key-based authentication and disable password logins.

   - **Option 2: Provide Limited Sudo Access (If Necessary)**
   
     *Not recommended unless specific sudo privileges are required.*

---

## **5. Configuring SSH Key-Based Authentication**

Secure SSH key-based authentication allows the source server to communicate with the destination server without using passwords, enhancing security and automation.

### **5.1. Generate SSH Keys on the Source Server**

1. **Switch to the Synchronization User (If Using a Dedicated User):**

   ```bash
   sudo su - rrd_sync_user
   ```

2. **Generate SSH Key Pair:**

   ```bash
   ssh-keygen -t rsa -b 4096 -C "rrd_sync@source" -f ~/.ssh/id_rsa_rrd_sync -N ""
   ```

   - **Parameters:**
     - `-t rsa`: Specifies the RSA algorithm.
     - `-b 4096`: Sets the key length to 4096 bits.
     - `-C "rrd_sync@source"`: Adds a comment for identification.
     - `-f ~/.ssh/id_rsa_rrd_sync`: Specifies the output file.
     - `-N ""`: Sets an empty passphrase for automation.

3. **Copy the Public Key to the Destination Server:**

   ```bash
   ssh-copy-id -i ~/.ssh/id_rsa_rrd_sync.pub rrd_sync_user@<DESTINATION_SERVER_IP>
   ```

   - **Example:**

     ```bash
     ssh-copy-id -i ~/.ssh/id_rsa_rrd_sync.pub rrd_sync_user@192.168.29.153
     ```

   - **Note:** Ensure that the destination server allows SSH access for `rrd_sync_user`.

4. **Verify SSH Connection:**

   ```bash
   ssh -i ~/.ssh/id_rsa_rrd_sync rrd_sync_user@192.168.29.153 "echo 'SSH connection successful!'"
   ```

   - **Expected Output:**

     ```
     SSH connection successful!
     ```

5. **Restrict SSH Access (Optional but Recommended):**

   On the destination server, restrict SSH access for `rrd_sync_user` by editing the SSH configuration.

   ```bash
   sudo nano /etc/ssh/sshd_config
   ```

   - **Add the Following at the End:**

     ```ini
     Match User rrd_sync_user
         PasswordAuthentication no
         PermitRootLogin no
         AllowTcpForwarding no
         X11Forwarding no
     ```

   - **Save and Exit.**

6. **Restart SSH Service on Destination Server:**

   ```bash
   sudo systemctl restart sshd
   ```

---

## **6. Installing Python Dependencies**

The synchronization script relies on the `watchdog` library to monitor filesystem events. Install it using `pip`.

1. **Ensure `pip` is Installed:**

   ```bash
   sudo yum install -y python3-pip
   ```

2. **Install `watchdog`:**

   ```bash
   sudo pip3 install watchdog
   ```

   - **Optional:** Install within a virtual environment for better package management.

     ```bash
     sudo pip3 install virtualenv
     mkdir ~/rrd_sync_env
     virtualenv ~/rrd_sync_env
     source ~/rrd_sync_env/bin/activate
     pip install watchdog
     ```

   - **Note:** If using a virtual environment, adjust the `systemd` service to activate it before running the script.

---

## **7. Configuring and Running the Synchronization Script**

This section involves adjusting script configurations to match your environment and ensuring it's executable.

### **7.1. Adjust Script Configuration**

1. **Open the Script for Editing:**

   ```bash
   sudo nano /opt/opennms/rrd_data_move/rrd_sync_new.py
   ```

2. **Update Configuration Variables:**

   - **Source and Destination Paths:**

     Ensure that `SOURCE_DIR` and `REMOTE_DEST_DIR` point to the correct directories.

     ```python
     SOURCE_DIR = "/opt/opennms/rrd_data_move/snmp/1/opennms-jvm/"
     REMOTE_DEST_DIR = "/opt/opennms/rrd_data_move/destination/snmp/2/"
     ```

   - **Remote Server Details:**

     Update `REMOTE_USER`, `REMOTE_HOST`, and `SSH_KEY_PATH` as per your setup.

     ```python
     REMOTE_USER = "rrd_sync_user"
     REMOTE_HOST = "192.168.29.153"  # Replace with your destination server's IP
     SSH_KEY_PATH = "/home/rrd_sync_user/.ssh/id_rsa_rrd_sync"
     ```

   - **Debounce Time:**

     Adjust `DEBOUNCE_TIME` if necessary, especially for very large files.

     ```python
     DEBOUNCE_TIME = 10  # seconds
     ```

   - **Save and Exit:**

     Press `CTRL + O` to save and `CTRL + X` to exit the editor.

### **7.2. Make the Script Executable**

1. **Set Executable Permissions:**

   ```bash
   sudo chmod +x /opt/opennms/rrd_data_move/rrd_sync_new.py
   ```

2. **Verify Execution:**

   ```bash
   /opt/opennms/rrd_data_move/rrd_sync_new.py
   ```

   - **Note:** Running the script directly will start the synchronization process. It's recommended to manage it via `systemd` for continuous operation.

---

## **8. Setting Up the `systemd` Service**

Managing the synchronization script as a `systemd` service ensures it runs continuously, restarts on failure, and starts automatically on system boot.

### **8.1. Create the Service File**

1. **Create the Service File:**

   ```bash
   sudo nano /etc/systemd/system/rrd_sync_new.service
   ```

2. **Add the Following Content:**

   ```ini
   [Unit]
   Description=RRD File-Level Synchronization Service
   After=network.target sshd.service

   [Service]
   Type=simple
   ExecStart=/usr/bin/python3 /opt/opennms/rrd_data_move/rrd_sync_new.py
   Restart=on-failure
   RestartSec=30
   User=rrd_sync_user
   Environment=PYTHONUNBUFFERED=1

   # Logging
   StandardOutput=append:/opt/opennms/rrd_data_move/rrd_sync_new.log
   StandardError=append:/opt/opennms/rrd_data_move/rrd_sync_new.log

   [Install]
   WantedBy=multi-user.target
   ```

   **Explanation of Service Parameters:**

   - **`Description`:** Brief description of the service.
   - **`After`:** Specifies service dependencies; ensures that networking and SSH are up before starting synchronization.
   - **`Type`:** Defines the process behavior; `simple` is suitable for most scripts.
   - **`ExecStart`:** Command to execute the synchronization script.
   - **`Restart` & `RestartSec`:** Automatically restarts the service on failure after a 30-second delay.
   - **`User`:** Specifies the user under which the service runs (`rrd_sync_user` in this case).
   - **`Environment`:** Sets environment variables; `PYTHONUNBUFFERED=1` ensures real-time logging.
   - **`StandardOutput` & `StandardError`:** Redirects both stdout and stderr to the specified log file.
   - **`WantedBy`:** Makes the service start during the system's multi-user runlevel.

3. **Save and Exit:**

   Press `CTRL + O` to save and `CTRL + X` to exit the editor.

### **8.2. Enable and Start the Service**

1. **Reload `systemd` Daemon to Recognize the New Service:**

   ```bash
   sudo systemctl daemon-reload
   ```

2. **Enable the Service to Start on Boot:**

   ```bash
   sudo systemctl enable rrd_sync_new.service
   ```

3. **Start the Service Immediately:**

   ```bash
   sudo systemctl start rrd_sync_new.service
   ```

4. **Verify Service Status:**

   ```bash
   sudo systemctl status rrd_sync_new.service
   ```

   - **Expected Output:**

     ```
     ● rrd_sync_new.service - RRD File-Level Synchronization Service
          Loaded: loaded (/etc/systemd/system/rrd_sync_new.service; enabled; vendor preset: enabled)
          Active: active (running) since [timestamp]
        Main PID: [pid] (python3)
           Tasks: [number]
          Memory: [memory usage]
             CPU: [cpu usage]
          CGroup: /system.slice/rrd_sync_new.service
                  └─[pid] /usr/bin/python3 /opt/opennms/rrd_data_move/rrd_sync_new.py
     ```

---

## **9. Implementing Log Rotation**

To prevent log files from consuming excessive disk space, implement log rotation using `logrotate`.

### **9.1. Create Logrotate Configuration**

1. **Create the Logrotate Configuration File:**

   ```bash
   sudo nano /etc/logrotate.d/rrd_sync_new
   ```

2. **Add the Following Content:**

   ```ini
   /opt/opennms/rrd_data_move/rrd_sync_new.log {
       daily
       missingok
       rotate 7
       compress
       delaycompress
       notifempty
       create 644 rrd_sync_user rrd_sync_user
       sharedscripts
       postrotate
           systemctl restart rrd_sync_new.service > /dev/null
       endscript
   }
   ```

   **Explanation of Configuration Parameters:**

   - **`daily`:** Rotate logs daily.
   - **`missingok`:** Do not issue an error if the log file is missing.
   - **`rotate 7`:** Keep seven rotated log files before deleting the oldest.
   - **`compress` & `delaycompress`:** Compress rotated logs to save space; delay compression by one rotation cycle.
   - **`notifempty`:** Do not rotate empty log files.
   - **`create`:** Create new log files with specified permissions and ownership after rotation.
   - **`sharedscripts`:** Run post-rotation scripts only once, even if multiple logs are rotated.
   - **`postrotate` Script:** Restart the synchronization service to continue logging seamlessly.

3. **Save and Exit:**

   Press `CTRL + O` to save and `CTRL + X` to exit the editor.

### **9.2. Test Logrotate Configuration**

1. **Run Logrotate in Debug Mode:**

   ```bash
   sudo logrotate -d /etc/logrotate.d/rrd_sync_new
   ```

   - **Explanation:** The `-d` flag runs logrotate in debug mode, showing what would happen without making actual changes.

2. **Force Log Rotation (If Needed):**

   ```bash
   sudo logrotate -f /etc/logrotate.d/rrd_sync_new
   ```

   - **Explanation:** The `-f` flag forces log rotation, useful for testing.

---

## **10. Testing the Synchronization Setup**

Thorough testing ensures that the synchronization setup operates as expected, efficiently handling large files and synchronizing only changes.

### **10.1. Initial Synchronization**

1. **Create a Large `.txt` File:**

   ```bash
   sudo -u rrd_sync_user dd if=/dev/zero of=/opt/opennms/rrd_data_move/snmp/1/opennms-jvm/large_test_file.txt bs=1M count=240
   ```

   - **Explanation:**
     - **`if=/dev/zero`:** Generates a file filled with zeros.
     - **`of=.../large_test_file.txt`:** Specifies the output file path.
     - **`bs=1M`:** Sets the block size to 1 Megabyte.
     - **`count=240`:** Creates a file of approximately 240 MB.

2. **Monitor Synchronization Logs:**

   ```bash
   sudo tail -f /opt/opennms/rrd_data_move/rrd_sync_new.log
   ```

   - **Expected Log Entries:**

     ```
     2024-12-12 15:05:40,908 - INFO - Syncing large_test_file.txt due to created event.
     2024-12-12 15:05:40,908 - INFO - rsync output for large_test_file.txt:
     >f+++++++++ large_test_file.txt
     sent 244,867 bytes  received 35 bytes  54,422.67 bytes/sec
     total size is 251,658,240  speedup is 1,027.59
     2024-12-12 15:05:40,908 - INFO - Synced large_test_file.txt successfully.
     ```

   - **Interpretation:**
     - **`>f+++++++++ large_test_file.txt`:** Indicates that the file was created and transferred.
     - **`sent 244,867 bytes`:** Represents the initial data transfer for the new file.

### **10.2. Modifying Files**

1. **Append Data to the Large `.txt` File:**

   ```bash
   sudo -u rrd_sync_user echo "Appending some data to test synchronization." >> /opt/opennms/rrd_data_move/snmp/1/opennms-jvm/large_test_file.txt
   ```

2. **Monitor Synchronization Logs:**

   ```bash
   sudo tail -f /opt/opennms/rrd_data_move/rrd_sync_new.log
   ```

   - **Expected Log Entries:**

     ```
     2024-12-12 15:06:38,818 - INFO - Syncing large_test_file.txt due to modified event.
     2024-12-12 15:06:45,413 - INFO - rsync output for large_test_file.txt:
     >f.st...... large_test_file.txt
     sent 200 bytes  received 111,143 bytes  14,845.73 bytes/sec
     total size is 251,658,287  speedup is 2,260.21
     2024-12-12 15:06:45,414 - INFO - Synced large_test_file.txt successfully.
     ```

   - **Interpretation:**
     - **`>f.st...... large_test_file.txt`:** Indicates that the file was modified.
     - **`sent 200 bytes`:** Only the changes (appended data) were transferred, confirming efficient synchronization.

### **10.3. Deleting Files**

1. **Delete the Large `.txt` File from Source:**

   ```bash
   sudo -u rrd_sync_user rm /opt/opennms/rrd_data_move/snmp/1/opennms-jvm/large_test_file.txt
   ```

2. **Monitor Synchronization Logs:**

   ```bash
   sudo tail -f /opt/opennms/rrd_data_move/rrd_sync_new.log
   ```

   - **Expected Log Entries:**

     ```
     2024-12-12 15:07:30,123 - INFO - Deleting large_test_file.txt from destination...
     2024-12-12 15:07:30,124 - INFO - rsync output for deletion:
     Number of files: 1 (reg: 1)
     Number of created files: 0
     Number of deleted files: 1
     Number of regular files transferred: 0
     Total file size: 251,658,287 bytes
     Total transferred file size: 0 bytes
     sent 200 bytes  received 111,143 bytes  14,845.73 bytes/sec
     total size is 251,658,287  speedup is 1,260.21
     2024-12-12 15:07:30,124 - INFO - Deleted large_test_file.txt from destination successfully.
     ```

   - **Interpretation:**
     - **`Number of deleted files: 1`:** Confirms that the deletion was synchronized.
     - **`Deleted large_test_file.txt from destination successfully.`**

### **10.4. Verifying Efficient Synchronization**

To confirm that only changes are being synchronized and not the entire file, analyze the `rsync` log outputs.

1. **Check Bytes Sent:**

   - **Initial Sync:** Larger bytes sent as the entire file is transferred.
   - **Modification Sync:** Minimal bytes sent corresponding to the changes made.

2. **Speedup Ratio:**

   - **Higher Speedup:** Indicates efficient synchronization, with only changes being transferred.

3. **Manual Checksum Verification:**

   ```bash
   # On Source Server
   md5sum /opt/opennms/rrd_data_move/snmp/1/opennms-jvm/large_test_file.txt

   # On Destination Server
   ssh -i /home/rrd_sync_user/.ssh/id_rsa_rrd_sync rrd_sync_user@192.168.29.153 "md5sum /opt/opennms/rrd_data_move/destination/snmp/2/large_test_file.txt"
   ```

   - **Expected Outcome:** Both checksums should match, confirming data integrity.

---

## **11. Monitoring and Maintenance**

Effective monitoring ensures the synchronization system remains reliable and performs optimally.

### **11.1. Continuous Log Monitoring**

1. **Real-Time Log Monitoring:**

   ```bash
   sudo tail -f /opt/opennms/rrd_data_move/rrd_sync_new.log
   ```

   - **Purpose:** Observe synchronization events and identify any issues promptly.

2. **Using `journalctl` (If Using `systemd`):**

   ```bash
   sudo journalctl -u rrd_sync_new.service -f
   ```

   - **Purpose:** View logs managed by `systemd` for the synchronization service.

### **11.2. Automated Alerts (Advanced)**

Integrate alerting mechanisms to notify you of synchronization failures or anomalies.

1. **Email Notifications (Example Using Python):**

   ```python
   import smtplib
   from email.mime.text import MIMEText

   def send_email(subject, body):
       msg = MIMEText(body)
       msg['Subject'] = subject
       msg['From'] = 'sync@example.com'
       msg['To'] = 'admin@example.com'

       with smtplib.SMTP('smtp.example.com') as server:
           server.login('username', 'password')
           server.send_message(msg)

   # Usage in the script:
   if result.returncode != 0:
       send_email(
           subject=f"Synchronization Failure: {relative_path}",
           body=f"Failed to sync {relative_path}. Error: {result.stderr.strip()}"
       )
   ```

   - **Note:** Replace SMTP server details and credentials accordingly.

2. **Integrate with Monitoring Tools:**

   Consider using tools like **Nagios**, **Prometheus**, or **Grafana** for advanced monitoring and alerting.

### **11.3. Regular Maintenance**

1. **Update Python Packages:**

   ```bash
   sudo pip3 install --upgrade watchdog
   ```

2. **System Updates:**

   ```bash
   sudo yum update -y
   ```

3. **Review and Rotate Logs:**

   Ensure that log rotation is functioning correctly and logs are maintained without consuming excessive disk space.

---

## **12. Security Best Practices**

Securing your synchronization setup protects against unauthorized access and potential data breaches.

### **12.1. Use a Dedicated Non-Root User**

1. **Create the User (If Not Already Done):**

   ```bash
   sudo adduser rrd_sync_user
   sudo passwd rrd_sync_user  # Set a strong password
   ```

2. **Assign Necessary Permissions:**

   ```bash
   sudo chown -R rrd_sync_user:rrd_sync_user /opt/opennms/rrd_data_move/
   sudo chmod -R 700 /opt/opennms/rrd_data_move/
   ```

### **12.2. Restrict SSH Access**

1. **Edit SSH Configuration on Destination Server:**

   ```bash
   sudo nano /etc/ssh/sshd_config
   ```

2. **Add the Following at the End:**

   ```ini
   Match User rrd_sync_user
       PasswordAuthentication no
       PermitRootLogin no
       AllowTcpForwarding no
       X11Forwarding no
   ```

3. **Restart SSH Service:**

   ```bash
   sudo systemctl restart sshd
   ```

### **12.3. Secure SSH Keys**

1. **Set Correct Permissions:**

   ```bash
   sudo chmod 600 /home/rrd_sync_user/.ssh/id_rsa_rrd_sync
   sudo chown rrd_sync_user:rrd_sync_user /home/rrd_sync_user/.ssh/id_rsa_rrd_sync
   ```

2. **Disable SSH Password Authentication (After Ensuring Key-Based Access):**

   ```bash
   sudo nano /etc/ssh/sshd_config
   ```

   - **Modify or Add:**

     ```ini
     PasswordAuthentication no
     ```

   - **Restart SSH Service:**

     ```bash
     sudo systemctl restart sshd
     ```

### **12.4. Limit File Permissions**

1. **Ensure Only Necessary Users Have Access:**

   ```bash
   sudo chown -R rrd_sync_user:rrd_sync_user /opt/opennms/rrd_data_move/
   sudo chmod -R 700 /opt/opennms/rrd_data_move/
   ```

### **12.5. Regular System Updates**

1. **Keep System Packages Updated:**

   ```bash
   sudo yum update -y
   ```

2. **Keep Python Packages Updated:**

   ```bash
   sudo pip3 install --upgrade watchdog
   ```

---

## **13. Troubleshooting**

Encountering issues is common during setup. Below are common problems and their solutions.

### **13.1. SSH Connection Issues**

- **Symptom:** Unable to establish SSH connection between source and destination servers.
  
- **Solution:**
  - Verify SSH key setup.
  - Ensure `rrd_sync_user` exists on the destination server.
  - Check SSH service status on the destination server.
  - Confirm network connectivity and firewall settings.

### **13.2. Synchronization Failures**

- **Symptom:** Files are not being synchronized as expected.
  
- **Solution:**
  - Check synchronization logs for errors.
  - Ensure `rsync` is installed on both servers.
  - Verify script permissions and ownership.
  - Confirm that the `systemd` service is running correctly.

### **13.3. High Resource Usage**

- **Symptom:** The synchronization process consumes excessive CPU or memory.
  
- **Solution:**
  - Monitor system resources using tools like `htop` or `top`.
  - Optimize `rsync` settings (e.g., adjust `--bwlimit`).
  - Increase debounce time to reduce synchronization frequency.

### **13.4. Log Rotation Not Working**

- **Symptom:** Log files grow indefinitely without rotation.
  
- **Solution:**
  - Verify `logrotate` configuration for `rrd_sync_new`.
  - Check `logrotate` service status.
  - Manually trigger log rotation to test.

    ```bash
    sudo logrotate -f /etc/logrotate.d/rrd_sync_new
    ```

### **13.5. File Permissions Issues**

- **Symptom:** Files are not being synchronized due to permission restrictions.
  
- **Solution:**
  - Ensure that `rrd_sync_user` has read access to source files.
  - Verify write permissions on the destination directory.
  - Adjust permissions as necessary.

---

## **14. Conclusion**

By following this comprehensive guide, you have successfully set up a robust **File-Level Monitoring and Synchronization** system capable of efficiently handling large files. The system ensures that only incremental changes are synchronized, optimizing bandwidth usage and maintaining data integrity. Enhanced logging provides clear visibility into synchronization activities, facilitating easy monitoring and troubleshooting.

**Key Highlights:**

- **Efficiency:** Only changes within files are synchronized, reducing unnecessary data transfers.
- **Reliability:** The debouncing mechanism prevents multiple rapid synchronization events.
- **Security:** Using a dedicated non-root user and SSH key-based authentication enhances the security of the synchronization process.
- **Maintainability:** Log rotation and comprehensive logging ensure that the system remains maintainable and easy to monitor.

**Next Steps:**

1. **Continuous Monitoring:** Regularly monitor synchronization logs to ensure the system operates as expected.
2. **Scalability:** Adjust configurations based on the growth of your data and synchronization needs.
3. **Security Audits:** Periodically review security settings to safeguard your synchronization setup.

Feel free to reach out for further assistance or clarification on any of the steps outlined in this guide.

---

**Disclaimer:**  
This guide is intended for informational purposes and assumes a certain level of familiarity with Linux system administration. Always back up important data before making significant changes to system configurations.
