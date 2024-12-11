Data sync
Data sync
# **Enhancing File-Level Monitoring to Handle Deletions**

**Author:** Kartik Sharma  
**Date:** 12-DEC-2024  
**Version:** 1.1

----------

## **Table of Contents**

1.  [Understanding the Issue](https://chatgpt.com/c/6759d00d-7400-8004-b608-4c4e829ce5fe#1-understanding-the-issue)
2.  [Key Enhancements Needed](https://chatgpt.com/c/6759d00d-7400-8004-b608-4c4e829ce5fe#2-key-enhancements-needed)
3.  [Modifying the Synchronization Script](https://chatgpt.com/c/6759d00d-7400-8004-b608-4c4e829ce5fe#3-modifying-the-synchronization-script)
    -   [3.1. Adding Deletion Event Handling](https://chatgpt.com/c/6759d00d-7400-8004-b608-4c4e829ce5fe#31-adding-deletion-event-handling)
    -   [3.2. Updating the `rsync` Command](https://chatgpt.com/c/6759d00d-7400-8004-b608-4c4e829ce5fe#32-updating-the-rsync-command)
    -   [3.3. Enhancing Logging for Deletions](https://chatgpt.com/c/6759d00d-7400-8004-b608-4c4e829ce5fe#33-enhancing-logging-for-deletions)
4.  [Testing the Enhanced Script](https://chatgpt.com/c/6759d00d-7400-8004-b608-4c4e829ce5fe#4-testing-the-enhanced-script)
5.  [Ensuring Robustness](https://chatgpt.com/c/6759d00d-7400-8004-b608-4c4e829ce5fe#5-ensuring-robustness)
6.  [Final Script Example](https://chatgpt.com/c/6759d00d-7400-8004-b608-4c4e829ce5fe#6-final-script-example)
7.  [Conclusion](https://chatgpt.com/c/6759d00d-7400-8004-b608-4c4e829ce5fe#7-conclusion)

----------

## **1. Understanding the Issue**

Your current **File-Level Monitoring** setup successfully detects and synchronizes file additions and modifications. However, it doesn't handle file deletions, meaning that when a file is removed from the **Source Server**, it remains on the **Destination Server**. This can lead to inconsistencies between the two servers.

**Objectives:**

-   Detect when files are deleted on the **Source Server**.
-   Ensure that corresponding files are deleted on the **Destination Server**.
-   Log deletion events for auditing and troubleshooting purposes.

----------

## **2. Key Enhancements Needed**

To address the issue of unsynchronized deletions, the following enhancements are necessary:

1.  **Detect Deletion Events:**
    
    -   Utilize `watchdog` to monitor deletion events (`on_deleted`).
2.  **Synchronize Deletions:**
    
    -   Modify the `rsync` command to handle deletions using the `--delete` flag.
3.  **Log Deletion Events:**
    
    -   Ensure that deletions are recorded in the log files for monitoring and auditing.
4.  **Maintain Separate State Management:**
    
    -   Keep track of deletions to prevent unnecessary synchronization attempts.

----------

## **3. Modifying the Synchronization Script**

Assuming your current synchronization script resembles the one provided in your initial documentation, we'll focus on enhancing it to handle deletions.

### **3.1. Adding Deletion Event Handling**

**Step 1: Update the Event Handler Class**

Extend the `FileSystemEventHandler` to handle deletion events by implementing the `on_deleted` method.

```python
# Inside your synchronization script

class RRDFileEventHandler(FileSystemEventHandler):
    def __init__(self, state):
        self.state = state

    def on_modified(self, event):
        if not event.is_directory and event.src_path.endswith('.rrd'):
            self.handle_event(event.src_path, event_type='modified')

    def on_created(self, event):
        if not event.is_directory and event.src_path.endswith('.rrd'):
            self.handle_event(event.src_path, event_type='created')

    def on_deleted(self, event):
        if not event.is_directory and event.src_path.endswith('.rrd'):
            self.handle_event(event.src_path, event_type='deleted')

    def handle_event(self, file_path, event_type):
        if event_type in ['modified', 'created']:
            try:
                mtime = os.path.getmtime(file_path)
                if (file_path not in self.state) or (mtime > self.state[file_path]):
                    logging.info(f"{event_type.capitalize()} {os.path.basename(file_path)}...")
                    sync_file(file_path, event_type)
                    self.state[file_path] = mtime
                    save_state(self.state)
            except Exception as e:
                error_logger.error(f"Error handling file {file_path}: {e}")
        elif event_type == 'deleted':
            try:
                logging.info(f"Deleted {os.path.basename(file_path)}...")
                sync_file(file_path, event_type)
                if file_path in self.state:
                    del self.state[file_path]
                    save_state(self.state)
            except Exception as e:
                error_logger.error(f"Error handling deletion of {file_path}: {e}")

```

**Explanation:**

-   **`on_deleted` Method:** Detects when a `.rrd` file is deleted and invokes the `handle_event` method with `event_type='deleted'`.
-   **`handle_event` Method:** Differentiates between `created`, `modified`, and `deleted` events and processes them accordingly.

### **3.2. Updating the `rsync` Command**

To synchronize deletions, modify the `sync_file` function to handle different event types. For deletions, use the `--delete` flag to remove files from the **Destination Server**.

```python
def sync_file(file_path, event_type):
    relative_path = os.path.relpath(file_path, SOURCE_DIR)
    destination_path = os.path.join(REMOTE_DEST_DIR, relative_path)
    dest_dir = os.path.dirname(destination_path)

    if event_type in ['created', 'modified']:
        # Ensure destination directory exists
        subprocess.run([
            'ssh', '-i', SSH_KEY_PATH, f"{REMOTE_USER}@{REMOTE_HOST}",
            f"mkdir -p {dest_dir}"
        ], check=True)

        # Execute rsync command without --delete for creation/modification
        rsync_command = [
            "rsync",
            "-avz",
            "--progress",
            "-e",
            f"ssh -i {SSH_KEY_PATH}",
            file_path,
            f"{REMOTE_USER}@{REMOTE_HOST}:{REMOTE_DEST_DIR}"
        ]

    elif event_type == 'deleted':
        # Execute rsync command with --delete for deletions
        rsync_command = [
            "rsync",
            "-avz",
            "--delete",
            "--progress",
            "-e",
            f"ssh -i {SSH_KEY_PATH}",
            SOURCE_DIR,
            f"{REMOTE_USER}@{REMOTE_HOST}:{REMOTE_DEST_DIR}"
        ]

    try:
        result = subprocess.run(rsync_command, capture_output=True, text=True, check=True)
        if event_type in ['created', 'modified']:
            logging.info(f"Synced {relative_path} successfully.")
        elif event_type == 'deleted':
            logging.info(f"Synchronized deletion of {relative_path} successfully.")
        logging.debug(f"rsync output: {result.stdout}")
    except subprocess.CalledProcessError as e:
        if event_type in ['created', 'modified']:
            error_message = f"Failed to sync {relative_path}. Error: {e.stderr}"
        elif event_type == 'deleted':
            error_message = f"Failed to synchronize deletion of {relative_path}. Error: {e.stderr}"
        error_logger.error(error_message)

```

**Explanation:**

-   **For `created` and `modified` Events:**
    -   Ensures the destination directory exists.
    -   Executes `rsync` without the `--delete` flag to synchronize the specific file.
-   **For `deleted` Events:**
    -   Executes `rsync` with the `--delete` flag to remove the deleted file from the **Destination Server**.
    -   Synchronizes the entire source directory to ensure deletions are propagated.

**Note:** When handling deletions, it's important to synchronize the entire directory structure to effectively remove the deleted file from the destination. The `--delete` flag ensures that any files present in the destination but not in the source are removed.

### **3.3. Enhancing Logging for Deletions**

Ensure that deletion events are properly logged for monitoring and auditing purposes.

```python
# Setup Logging (ensure this section includes appropriate handlers)
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

```

**Explanation:**

-   **Logging Messages:** The `handle_event` method logs both synchronization and deletion events, differentiating them by their event types.
-   **Log Files:** Ensure that both `rrd_sync.log` and `rrd_sync_error.log` capture relevant information about deletions and any errors encountered.

----------

## **4. Testing the Enhanced Script**

After implementing the above modifications, it's essential to test the script to ensure that deletions are handled correctly.

### **4.1. Test File Deletion**

1.  **Delete a Test `.rrd` File on Source Server:**
    
    ```bash
    sudo -u rrd_sync_user rm /opt/opennms/rrd_data_move/snmp/1/opennms-jvm/TestFileToDelete.rrd
    
    ```
    
2.  **Monitor Logs on Source Server:**
    
    ```bash
    sudo tail -f /var/log/rrd_sync.log
    sudo tail -f /var/log/rrd_sync_error.log
    
    ```
    
    **Expected Log Entries:**
    
    ```
    Deleted TestFileToDelete.rrd...
    Synchronized deletion of TestFileToDelete.rrd successfully.
    
    ```
    
3.  **Verify Deletion on Destination Server:**
    
    ```bash
    sudo ls -l /opt/opennms/rrd_data_move/destination/snmp/1/opennms-jvm/
    
    ```
    
    **Expected Output:**
    
    ```
    ls: cannot access '/opt/opennms/rrd_data_move/destination/snmp/1/opennms-jvm/TestFileToDelete.rrd': No such file or directory
    
    ```
    

### **4.2. Test Logging for Deletions**

Ensure that deletion events are accurately recorded in the log files.

```bash
sudo grep "Deleted TestFileToDelete.rrd" /var/log/rrd_sync.log
sudo grep "Synchronized deletion of TestFileToDelete.rrd" /var/log/rrd_sync.log

```

**Expected Output:**

```
[Timestamp] Deleted TestFileToDelete.rrd...
[Timestamp] Synchronized deletion of TestFileToDelete.rrd successfully.

```

----------

## **5. Ensuring Robustness**

To ensure that your enhanced synchronization script remains robust and reliable, consider the following best practices:

### **5.1. Handling Multiple Deletion Events**

If multiple files are deleted in quick succession, ensure that the script can handle batch deletions efficiently without overwhelming the system or the network.

### **5.2. Error Handling Enhancements**

Enhance error handling to manage scenarios where deletions cannot be synchronized due to network issues or permission problems.

```python
except subprocess.CalledProcessError as e:
    # Detailed error logging
    if event_type in ['created', 'modified']:
        error_message = f"Failed to sync {relative_path}. Error: {e.stderr}"
    elif event_type == 'deleted':
        error_message = f"Failed to synchronize deletion of {relative_path}. Error: {e.stderr}"
    error_logger.error(error_message)
    
    # Optional: Implement retry mechanisms or alerting systems

```

### **5.3. Security Considerations**

Ensure that the synchronization process adheres to security best practices:

-   **SSH Key Security:** Restrict permissions on SSH keys to prevent unauthorized access.
    
    ```bash
    sudo chmod 600 /root/.ssh/id_rsa_rrd_sync
    sudo chown root:root /root/.ssh/id_rsa_rrd_sync
    
    ```
    
-   **Dedicated User Accounts:** Consider using dedicated non-root users for synchronization tasks to minimize security risks.
    

----------

## **6. Final Script Example**

Below is the **enhanced synchronization script** incorporating the modifications to handle deletions.

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
def sync_file(file_path, event_type):
    relative_path = os.path.relpath(file_path, SOURCE_DIR)
    destination_path = os.path.join(REMOTE_DEST_DIR, relative_path)
    dest_dir = os.path.dirname(destination_path)

    if event_type in ['created', 'modified']:
        # Ensure destination directory exists
        subprocess.run([
            'ssh', '-i', SSH_KEY_PATH, f"{REMOTE_USER}@{REMOTE_HOST}",
            f"mkdir -p {dest_dir}"
        ], check=True)

        # Execute rsync command without --delete for creation/modification
        rsync_command = [
            "rsync",
            "-avz",
            "--progress",
            "-e",
            f"ssh -i {SSH_KEY_PATH}",
            file_path,
            f"{REMOTE_USER}@{REMOTE_HOST}:{REMOTE_DEST_DIR}"
        ]

    elif event_type == 'deleted':
        # Execute rsync command with --delete for deletions
        rsync_command = [
            "rsync",
            "-avz",
            "--delete",
            "--progress",
            "-e",
            f"ssh -i {SSH_KEY_PATH}",
            SOURCE_DIR,
            f"{REMOTE_USER}@{REMOTE_HOST}:{REMOTE_DEST_DIR}"
        ]

    try:
        result = subprocess.run(rsync_command, capture_output=True, text=True, check=True)
        if event_type in ['created', 'modified']:
            logging.info(f"Synced {relative_path} successfully.")
        elif event_type == 'deleted':
            logging.info(f"Synchronized deletion of {relative_path} successfully.")
        logging.debug(f"rsync output: {result.stdout}")
    except subprocess.CalledProcessError as e:
        if event_type in ['created', 'modified']:
            error_message = f"Failed to sync {relative_path}. Error: {e.stderr}"
        elif event_type == 'deleted':
            error_message = f"Failed to synchronize deletion of {relative_path}. Error: {e.stderr}"
        error_logger.error(error_message)

# Event Handler Class
class RRDFileEventHandler(FileSystemEventHandler):
    def __init__(self, state):
        self.state = state

    def on_modified(self, event):
        if not event.is_directory and event.src_path.endswith('.rrd'):
            self.handle_event(event.src_path, event_type='modified')

    def on_created(self, event):
        if not event.is_directory and event.src_path.endswith('.rrd'):
            self.handle_event(event.src_path, event_type='created')

    def on_deleted(self, event):
        if not event.is_directory and event.src_path.endswith('.rrd'):
            self.handle_event(event.src_path, event_type='deleted')

    def handle_event(self, file_path, event_type):
        if event_type in ['created', 'modified']:
            try:
                mtime = os.path.getmtime(file_path)
                if (file_path not in self.state) or (mtime > self.state[file_path]):
                    logging.info(f"{event_type.capitalize()} {os.path.basename(file_path)}...")
                    sync_file(file_path, event_type)
                    self.state[file_path] = mtime
                    save_state(self.state)
            except Exception as e:
                error_logger.error(f"Error handling file {file_path}: {e}")
        elif event_type == 'deleted':
            try:
                logging.info(f"Deleted {os.path.basename(file_path)}...")
                sync_file(file_path, event_type)
                if file_path in self.state:
                    del self.state[file_path]
                    save_state(self.state)
            except Exception as e:
                error_logger.error(f"Error handling deletion of {file_path}: {e}")

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
                        sync_file(file_path, event_type='created' if file_path not in state else 'modified')
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

**Key Points in the Final Script:**

-   **Deletion Handling:** The `on_deleted` method detects when a `.rrd` file is deleted and triggers the synchronization process with the `--delete` flag in `rsync`.
-   **Logging:** Both creation/modification and deletion events are logged distinctly for clarity.
-   **State Management:** The state file is updated to remove entries corresponding to deleted files, preventing unnecessary synchronization attempts.
-   **Error Handling:** Enhanced error logging captures issues related to synchronization failures.

----------

## **7. Conclusion**

By implementing the above enhancements, your **File-Level Monitoring** script will now effectively handle file deletions, ensuring that the **Destination Server** remains an accurate mirror of the **Source Server**. This not only maintains data consistency but also ensures that your monitoring system remains reliable and trustworthy.

**Key Takeaways:**

-   **Event Handling:** Properly handling different file system events (`created`, `modified`, `deleted`) is essential for comprehensive synchronization.
-   **Rsync Configuration:** Utilizing the `--delete` flag in `rsync` ensures that deletions are propagated to the destination.
-   **Logging:** Detailed logging of all synchronization activities, including deletions, facilitates easier monitoring and troubleshooting.
-   **State Management:** Maintaining an accurate state file prevents redundant synchronization efforts and ensures efficiency.

**Recommendations:**

-   **Regular Monitoring:** Continuously monitor your log files to ensure that synchronization operates as expected.
-   **Performance Considerations:** Be mindful of the performance implications of using the `--delete` flag, especially in environments with large datasets.
-   **Security Practices:** Continue adhering to security best practices, such as using dedicated non-root users and securing SSH keys.


Enhancing File-Level Monitoring to Handle Deletions
Author: Kartik Sharma
Date: 12-DEC-2024
Version: 1.1

Table of Contents
Understanding the Issue
Key Enhancements Needed
Modifying the Synchronization Script
3.1. Adding Deletion Event Handling
3.2. Updating the rsync Command
3.3. Enhancing Logging for Deletions
Testing the Enhanced Script
Ensuring Robustness
Final Script Example
Conclusion
1. Understanding the Issue
Your current File-Level Monitoring setup successfully detects and synchronizes file additions and modifications. However, it doesn’t handle file deletions, meaning that when a file is removed from the Source Server, it remains on the Destination Server. This can lead to inconsistencies between the two servers.

Objectives:

Detect when files are deleted on the Source Server.
Ensure that corresponding files are deleted on the Destination Server.
Log deletion events for auditing and troubleshooting purposes.
2. Key Enhancements Needed
To address the issue of unsynchronized deletions, the following enhancements are necessary:

Detect Deletion Events:

Utilize watchdog to monitor deletion events (on_deleted).
Synchronize Deletions:

Modify the rsync command to handle deletions using the --delete flag.
Log Deletion Events:

Ensure that deletions are recorded in the log files for monitoring and auditing.
Maintain Separate State Management:

Keep track of deletions to prevent unnecessary synchronization attempts.
3. Modifying the Synchronization Script
Assuming your current synchronization script resembles the one provided in your initial documentation, we’ll focus on enhancing it to handle deletions.

3.1. Adding Deletion Event Handling
Step 1: Update the Event Handler Class

Extend the FileSystemEventHandler to handle deletion events by implementing the on_deleted method.

# Inside your synchronization script

class RRDFileEventHandler(FileSystemEventHandler):
    def __init__(self, state):
        self.state = state

    def on_modified(self, event):
        if not event.is_directory and event.src_path.endswith('.rrd'):
            self.handle_event(event.src_path, event_type='modified')

    def on_created(self, event):
        if not event.is_directory and event.src_path.endswith('.rrd'):
            self.handle_event(event.src_path, event_type='created')

    def on_deleted(self, event):
        if not event.is_directory and event.src_path.endswith('.rrd'):
            self.handle_event(event.src_path, event_type='deleted')

    def handle_event(self, file_path, event_type):
        if event_type in ['modified', 'created']:
            try:
                mtime = os.path.getmtime(file_path)
                if (file_path not in self.state) or (mtime > self.state[file_path]):
                    logging.info(f"{event_type.capitalize()} {os.path.basename(file_path)}...")
                    sync_file(file_path, event_type)
                    self.state[file_path] = mtime
                    save_state(self.state)
            except Exception as e:
                error_logger.error(f"Error handling file {file_path}: {e}")
        elif event_type == 'deleted':
            try:
                logging.info(f"Deleted {os.path.basename(file_path)}...")
                sync_file(file_path, event_type)
                if file_path in self.state:
                    del self.state[file_path]
                    save_state(self.state)
            except Exception as e:
                error_logger.error(f"Error handling deletion of {file_path}: {e}")

Explanation:

on_deleted Method: Detects when a .rrd file is deleted and invokes the handle_event method with event_type='deleted'.
handle_event Method: Differentiates between created, modified, and deleted events and processes them accordingly.
3.2. Updating the rsync Command
To synchronize deletions, modify the sync_file function to handle different event types. For deletions, use the --delete flag to remove files from the Destination Server.

def sync_file(file_path, event_type):
    relative_path = os.path.relpath(file_path, SOURCE_DIR)
    destination_path = os.path.join(REMOTE_DEST_DIR, relative_path)
    dest_dir = os.path.dirname(destination_path)

    if event_type in ['created', 'modified']:
        # Ensure destination directory exists
        subprocess.run([
            'ssh', '-i', SSH_KEY_PATH, f"{REMOTE_USER}@{REMOTE_HOST}",
            f"mkdir -p {dest_dir}"
        ], check=True)

        # Execute rsync command without --delete for creation/modification
        rsync_command = [
            "rsync",
            "-avz",
            "--progress",
            "-e",
            f"ssh -i {SSH_KEY_PATH}",
            file_path,
            f"{REMOTE_USER}@{REMOTE_HOST}:{REMOTE_DEST_DIR}"
        ]

    elif event_type == 'deleted':
        # Execute rsync command with --delete for deletions
        rsync_command = [
            "rsync",
            "-avz",
            "--delete",
            "--progress",
            "-e",
            f"ssh -i {SSH_KEY_PATH}",
            SOURCE_DIR,
            f"{REMOTE_USER}@{REMOTE_HOST}:{REMOTE_DEST_DIR}"
        ]

    try:
        result = subprocess.run(rsync_command, capture_output=True, text=True, check=True)
        if event_type in ['created', 'modified']:
            logging.info(f"Synced {relative_path} successfully.")
        elif event_type == 'deleted':
            logging.info(f"Synchronized deletion of {relative_path} successfully.")
        logging.debug(f"rsync output: {result.stdout}")
    except subprocess.CalledProcessError as e:
        if event_type in ['created', 'modified']:
            error_message = f"Failed to sync {relative_path}. Error: {e.stderr}"
        elif event_type == 'deleted':
            error_message = f"Failed to synchronize deletion of {relative_path}. Error: {e.stderr}"
        error_logger.error(error_message)

Explanation:

For created and modified Events:
Ensures the destination directory exists.
Executes rsync without the --delete flag to synchronize the specific file.
For deleted Events:
Executes rsync with the --delete flag to remove the deleted file from the Destination Server.
Synchronizes the entire source directory to ensure deletions are propagated.
Note: When handling deletions, it’s important to synchronize the entire directory structure to effectively remove the deleted file from the destination. The --delete flag ensures that any files present in the destination but not in the source are removed.

3.3. Enhancing Logging for Deletions
Ensure that deletion events are properly logged for monitoring and auditing purposes.

# Setup Logging (ensure this section includes appropriate handlers)
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

Explanation:

Logging Messages: The handle_event method logs both synchronization and deletion events, differentiating them by their event types.
Log Files: Ensure that both rrd_sync.log and rrd_sync_error.log capture relevant information about deletions and any errors encountered.
4. Testing the Enhanced Script
After implementing the above modifications, it’s essential to test the script to ensure that deletions are handled correctly.

4.1. Test File Deletion
Delete a Test .rrd File on Source Server:

sudo -u rrd_sync_user rm /opt/opennms/rrd_data_move/snmp/1/opennms-jvm/TestFileToDelete.rrd

Monitor Logs on Source Server:

sudo tail -f /var/log/rrd_sync.log
sudo tail -f /var/log/rrd_sync_error.log

Expected Log Entries:

Deleted TestFileToDelete.rrd...
Synchronized deletion of TestFileToDelete.rrd successfully.

Verify Deletion on Destination Server:

sudo ls -l /opt/opennms/rrd_data_move/destination/snmp/1/opennms-jvm/

Expected Output:

ls: cannot access '/opt/opennms/rrd_data_move/destination/snmp/1/opennms-jvm/TestFileToDelete.rrd': No such file or directory

4.2. Test Logging for Deletions
Ensure that deletion events are accurately recorded in the log files.

sudo grep "Deleted TestFileToDelete.rrd" /var/log/rrd_sync.log
sudo grep "Synchronized deletion of TestFileToDelete.rrd" /var/log/rrd_sync.log

Expected Output:

[Timestamp] Deleted TestFileToDelete.rrd...
[Timestamp] Synchronized deletion of TestFileToDelete.rrd successfully.

5. Ensuring Robustness
To ensure that your enhanced synchronization script remains robust and reliable, consider the following best practices:

5.1. Handling Multiple Deletion Events
If multiple files are deleted in quick succession, ensure that the script can handle batch deletions efficiently without overwhelming the system or the network.

5.2. Error Handling Enhancements
Enhance error handling to manage scenarios where deletions cannot be synchronized due to network issues or permission problems.

except subprocess.CalledProcessError as e:
    # Detailed error logging
    if event_type in ['created', 'modified']:
        error_message = f"Failed to sync {relative_path}. Error: {e.stderr}"
    elif event_type == 'deleted':
        error_message = f"Failed to synchronize deletion of {relative_path}. Error: {e.stderr}"
    error_logger.error(error_message)
    
    # Optional: Implement retry mechanisms or alerting systems

5.3. Security Considerations
Ensure that the synchronization process adheres to security best practices:

SSH Key Security: Restrict permissions on SSH keys to prevent unauthorized access.

sudo chmod 600 /root/.ssh/id_rsa_rrd_sync
sudo chown root:root /root/.ssh/id_rsa_rrd_sync

Dedicated User Accounts: Consider using dedicated non-root users for synchronization tasks to minimize security risks.

6. Final Script Example
Below is the enhanced synchronization script incorporating the modifications to handle deletions.

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
def sync_file(file_path, event_type):
    relative_path = os.path.relpath(file_path, SOURCE_DIR)
    destination_path = os.path.join(REMOTE_DEST_DIR, relative_path)
    dest_dir = os.path.dirname(destination_path)

    if event_type in ['created', 'modified']:
        # Ensure destination directory exists
        subprocess.run([
            'ssh', '-i', SSH_KEY_PATH, f"{REMOTE_USER}@{REMOTE_HOST}",
            f"mkdir -p {dest_dir}"
        ], check=True)

        # Execute rsync command without --delete for creation/modification
        rsync_command = [
            "rsync",
            "-avz",
            "--progress",
            "-e",
            f"ssh -i {SSH_KEY_PATH}",
            file_path,
            f"{REMOTE_USER}@{REMOTE_HOST}:{REMOTE_DEST_DIR}"
        ]

    elif event_type == 'deleted':
        # Execute rsync command with --delete for deletions
        rsync_command = [
            "rsync",
            "-avz",
            "--delete",
            "--progress",
            "-e",
            f"ssh -i {SSH_KEY_PATH}",
            SOURCE_DIR,
            f"{REMOTE_USER}@{REMOTE_HOST}:{REMOTE_DEST_DIR}"
        ]

    try:
        result = subprocess.run(rsync_command, capture_output=True, text=True, check=True)
        if event_type in ['created', 'modified']:
            logging.info(f"Synced {relative_path} successfully.")
        elif event_type == 'deleted':
            logging.info(f"Synchronized deletion of {relative_path} successfully.")
        logging.debug(f"rsync output: {result.stdout}")
    except subprocess.CalledProcessError as e:
        if event_type in ['created', 'modified']:
            error_message = f"Failed to sync {relative_path}. Error: {e.stderr}"
        elif event_type == 'deleted':
            error_message = f"Failed to synchronize deletion of {relative_path}. Error: {e.stderr}"
        error_logger.error(error_message)

# Event Handler Class
class RRDFileEventHandler(FileSystemEventHandler):
    def __init__(self, state):
        self.state = state

    def on_modified(self, event):
        if not event.is_directory and event.src_path.endswith('.rrd'):
            self.handle_event(event.src_path, event_type='modified')

    def on_created(self, event):
        if not event.is_directory and event.src_path.endswith('.rrd'):
            self.handle_event(event.src_path, event_type='created')

    def on_deleted(self, event):
        if not event.is_directory and event.src_path.endswith('.rrd'):
            self.handle_event(event.src_path, event_type='deleted')

    def handle_event(self, file_path, event_type):
        if event_type in ['created', 'modified']:
            try:
                mtime = os.path.getmtime(file_path)
                if (file_path not in self.state) or (mtime > self.state[file_path]):
                    logging.info(f"{event_type.capitalize()} {os.path.basename(file_path)}...")
                    sync_file(file_path, event_type)
                    self.state[file_path] = mtime
                    save_state(self.state)
            except Exception as e:
                error_logger.error(f"Error handling file {file_path}: {e}")
        elif event_type == 'deleted':
            try:
                logging.info(f"Deleted {os.path.basename(file_path)}...")
                sync_file(file_path, event_type)
                if file_path in self.state:
                    del self.state[file_path]
                    save_state(self.state)
            except Exception as e:
                error_logger.error(f"Error handling deletion of {file_path}: {e}")

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
                        sync_file(file_path, event_type='created' if file_path not in state else 'modified')
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

Key Points in the Final Script:

Deletion Handling: The on_deleted method detects when a .rrd file is deleted and triggers the synchronization process with the --delete flag in rsync.
Logging: Both creation/modification and deletion events are logged distinctly for clarity.
State Management: The state file is updated to remove entries corresponding to deleted files, preventing unnecessary synchronization attempts.
Error Handling: Enhanced error logging captures issues related to synchronization failures.
7. Conclusion
By implementing the above enhancements, your File-Level Monitoring script will now effectively handle file deletions, ensuring that the Destination Server remains an accurate mirror of the Source Server. This not only maintains data consistency but also ensures that your monitoring system remains reliable and trustworthy.

Key Takeaways:

Event Handling: Properly handling different file system events (created, modified, deleted) is essential for comprehensive synchronization.
Rsync Configuration: Utilizing the --delete flag in rsync ensures that deletions are propagated to the destination.
Logging: Detailed logging of all synchronization activities, including deletions, facilitates easier monitoring and troubleshooting.
State Management: Maintaining an accurate state file prevents redundant synchronization efforts and ensures efficiency.
Recommendations:

Regular Monitoring: Continuously monitor your log files to ensure that synchronization operates as expected.
Performance Considerations: Be mindful of the performance implications of using the --delete flag, especially in environments with large datasets.
Security Practices: Continue adhering to security best practices, such as using dedicated non-root users and securing SSH keys.
Markdown 20266 bytes 1838 words 524 lines Ln 6, Col 0HTML 14337 characters 1740 words 359 paragraphs
