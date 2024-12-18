
1. **Understanding the Need for Change Tracking**
2. **Designing the Change Tracking Mechanism**
3. **Implementing Change Tracking in the Python Script**
4. **Handling Different File Types**
5. **Enhancing Logs for Better Readability**
6. **Summary and Best Practices**

---

## **1. Understanding the Need for Change Tracking**

While your current script effectively monitors and synchronizes file changes, it doesn’t provide detailed insights into **what exactly changed** within those files. Implementing change tracking allows you to:

- **Audit Changes:** Know who changed what and when.
- **Debug Issues:** Quickly identify problematic changes.
- **Maintain History:** Keep a record of file evolution over time.

---

## **2. Designing the Change Tracking Mechanism**

To effectively track changes, especially within `.txt` files, follow this approach:

1. **Backup Previous Versions:**
   - **Purpose:** To compare the current file with its previous state.
   - **Implementation:** Maintain a local backup directory where copies of synchronized files are stored.

2. **Generate Diffs for Text Files:**
   - **Purpose:** To capture the exact changes (additions, deletions, modifications) within text files.
   - **Implementation:** Use Python's `difflib` module or Unix `diff` to generate human-readable diffs.

3. **Log Change Details:**
   - **Purpose:** To store the diffs for auditing and review.
   - **Implementation:** Append diffs to a dedicated change log file with timestamps and file identifiers.

4. **Handle Binary Files Appropriately:**
   - **Purpose:** Binary files (e.g., `.rrd`) can’t be diffed meaningfully.
   - **Implementation:** Log that the file was modified without detailing the content changes.

---

## **3. Implementing Change Tracking in the Python Script**

Below are detailed steps and code modifications to incorporate change tracking into your existing `rrd_sync_new.py` script.

### **3.1. Update Configuration File (`config.json`)**

First, update your `config.json` to include configurations for change tracking.

```json
{
    "source_dir": "/opt/opennms/rrd_data_move/snmp/1/opennms-jvm/",
    "state_file": "/opt/opennms/rrd_data_move/rrd_sync_new.json",
    "remote_sync": {
        "remote_user": "rrd_sync_user",
        "remote_host": "192.168.29.153",
        "remote_dest_dir": "/opt/opennms/rrd_data_move/destination/snmp/2/",
        "ssh_key_path": "/home/rrd_sync_user/.ssh/id_rsa_rrd_sync"
    },
    "logging": {
        "log_file": "/opt/opennms/rrd_data_move/rrd_sync_new.log",
        "log_level": "INFO"
    },
    "debounce_time": 5,
    "monitored_extensions": [".rrd", ".txt"],
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
    },
    "change_tracking": {
        "backup_dir": "/opt/opennms/rrd_data_move/backup/",
        "change_log_file": "/opt/opennms/rrd_data_move/change_log.txt",
        "diff_tool": "difflib"  # Options: "difflib" or "diff"
    }
}
```

### **3.2. Modify the Python Script**

Here’s the updated script with change tracking implemented.

```python
#!/usr/bin/env python3

import os
import json
import subprocess
import logging
import time
from watchdog.observers import Observer
from watchdog.events import FileSystemEventHandler
from threading import Lock, Timer
from pathlib import Path
import difflib  # For generating diffs

# ---------------------------
# Load Configuration
# ---------------------------

CONFIG_FILE = "/opt/opennms/rrd_data_move/config.json"

def load_config(config_path):
    """Load configuration from a JSON file."""
    if not os.path.exists(config_path):
        raise FileNotFoundError(f"Configuration file {config_path} does not exist.")
    with open(config_path, 'r') as f:
        try:
            config = json.load(f)
            return config
        except json.JSONDecodeError as e:
            raise ValueError(f"Error parsing JSON configuration: {e}")

config = load_config(CONFIG_FILE)

# ---------------------------
# Configuration Variables
# ---------------------------

SOURCE_DIR = config.get("source_dir", "/opt/opennms/rrd_data_move/snmp/1/opennms-jvm/")
STATE_FILE = config.get("state_file", "/opt/opennms/rrd_data_move/rrd_sync_new.json")

REMOTE_SYNC = config.get("remote_sync", {})
REMOTE_USER = REMOTE_SYNC.get("remote_user", "root")
REMOTE_HOST = REMOTE_SYNC.get("remote_host", "192.168.29.153")
REMOTE_DEST_DIR = REMOTE_SYNC.get("remote_dest_dir", "/opt/opennms/rrd_data_move/destination/snmp/2/")
SSH_KEY_PATH = REMOTE_SYNC.get("ssh_key_path", "/root/.ssh/id_rsa_rrd_sync")

LOGGING_CONFIG = config.get("logging", {})
LOG_FILE = LOGGING_CONFIG.get("log_file", "/opt/opennms/rrd_data_move/rrd_sync_new.log")
LOG_LEVEL = LOGGING_CONFIG.get("log_level", "INFO").upper()

DEBOUNCE_TIME = config.get("debounce_time", 5)  # seconds

MONITORED_EXTENSIONS = config.get("monitored_extensions", [".rrd", ".txt"])

RSYNC_OPTIONS = config.get("rsync_options", {})
RSYNC_ARCHIVE = RSYNC_OPTIONS.get("archive", True)
RSYNC_COMPRESS = RSYNC_OPTIONS.get("compress", True)
RSYNC_VERBOSE = RSYNC_OPTIONS.get("verbose", True)
RSYNC_UPDATE = RSYNC_OPTIONS.get("update", True)
RSYNC_CHECKSUM = RSYNC_OPTIONS.get("checksum", True)
RSYNC_PARTIAL = RSYNC_OPTIONS.get("partial", True)
RSYNC_BWLIMIT = RSYNC_OPTIONS.get("bwlimit", 1000)
RSYNC_ITEMIZE = RSYNC_OPTIONS.get("itemize_changes", True)
RSYNC_STATS = RSYNC_OPTIONS.get("stats", True)

CHANGE_TRACKING = config.get("change_tracking", {})
BACKUP_DIR = CHANGE_TRACKING.get("backup_dir", "/opt/opennms/rrd_data_move/backup/")
CHANGE_LOG_FILE = CHANGE_TRACKING.get("change_log_file", "/opt/opennms/rrd_data_move/change_log.txt")
DIFF_TOOL = CHANGE_TRACKING.get("diff_tool", "difflib")  # Options: "difflib" or "diff"

# ---------------------------
# Setup Logging
# ---------------------------

LOG_LEVEL_MAPPING = {
    "DEBUG": logging.DEBUG,
    "INFO": logging.INFO,
    "WARNING": logging.WARNING,
    "ERROR": logging.ERROR,
    "CRITICAL": logging.CRITICAL
}

logging.basicConfig(
    filename=LOG_FILE,
    level=LOG_LEVEL_MAPPING.get(LOG_LEVEL, logging.INFO),
    format='%(asctime)s - %(levelname)s - %(message)s'
)

# ---------------------------
# Utility Functions
# ---------------------------

def load_state():
    """Load the synchronization state from a JSON file."""
    if os.path.exists(STATE_FILE):
        with open(STATE_FILE, 'r') as f:
            try:
                return json.load(f)
            except json.JSONDecodeError:
                logging.error("State file is corrupted. Starting fresh.")
                return {}
    return {}

def save_state(state):
    """Save the synchronization state to a JSON file."""
    with open(STATE_FILE, 'w') as f:
        json.dump(state, f, indent=4)

def ensure_directories():
    """Ensure that backup and change log directories exist."""
    os.makedirs(BACKUP_DIR, exist_ok=True)
    change_log_dir = os.path.dirname(CHANGE_LOG_FILE)
    os.makedirs(change_log_dir, exist_ok=True)
    # Ensure change_log_file exists
    Path(CHANGE_LOG_FILE).touch(exist_ok=True)

ensure_directories()

def log_change(file_path, change_details):
    """Log the changes made to a file."""
    timestamp = time.strftime("%Y-%m-%d %H:%M:%S")
    with open(CHANGE_LOG_FILE, 'a') as log_file:
        log_file.write(f"Timestamp: {timestamp}\n")
        log_file.write(f"File: {file_path}\n")
        log_file.write(f"Changes:\n{change_details}\n")
        log_file.write("-" * 80 + "\n")

def generate_diff(old_file, new_file):
    """Generate a diff between two text files using difflib."""
    with open(old_file, 'r') as f1, open(new_file, 'r') as f2:
        old_lines = f1.readlines()
        new_lines = f2.readlines()
    diff = difflib.unified_diff(old_lines, new_lines, fromfile='before', tofile='after', lineterm='')
    return '\n'.join(list(diff))

# ---------------------------
# File System Event Handler
# ---------------------------

class RRDHandler(FileSystemEventHandler):
    """Handles filesystem events for synchronization."""
    def __init__(self, state):
        super().__init__()
        self.state = state
        self.monitored_extensions = MONITORED_EXTENSIONS
        self.lock = Lock()
        self.debounce_timers = {}

    def on_created(self, event):
        """Handle file creation events."""
        if not event.is_directory and self._is_monitored(event.src_path):
            self._debounced_sync(event.src_path, event_type="created")

    def on_modified(self, event):
        """Handle file modification events."""
        if not event.is_directory and self._is_monitored(event.src_path):
            self._debounced_sync(event.src_path, event_type="modified")

    def _is_monitored(self, path):
        """Check if the file has a monitored extension."""
        return any(path.endswith(ext) for ext in self.monitored_extensions)

    def _debounced_sync(self, src_path, event_type):
        """Debounce synchronization to prevent multiple triggers."""
        with self.lock:
            if src_path in self.debounce_timers:
                self.debounce_timers[src_path].cancel()
                logging.debug(f"Debounce timer canceled for {src_path}")
            timer = Timer(DEBOUNCE_TIME, self.sync_file, [src_path, event_type])
            self.debounce_timers[src_path] = timer
            timer.start()
            logging.debug(f"Debounce timer set for {src_path} with {DEBOUNCE_TIME} seconds")

    def sync_file(self, src_path, event_type):
        """Synchronize a single file to the remote server and track changes."""
        relative_path = os.path.relpath(src_path, SOURCE_DIR)
        backup_path = os.path.join(BACKUP_DIR, relative_path)
        backup_dir = os.path.dirname(backup_path)
        os.makedirs(backup_dir, exist_ok=True)

        try:
            src_stat = os.stat(src_path)
            src_mtime = src_stat.st_mtime

            # Check if the file needs to be copied
            if (relative_path not in self.state) or (self.state[relative_path] < src_mtime):
                logging.info(f"Syncing {relative_path} due to {event_type} event.")

                # If file exists in backup, generate diff
                if os.path.exists(backup_path) and src_path.endswith(".txt"):
                    diff = generate_diff(backup_path, src_path)
                    if diff:
                        log_change(relative_path, diff)
                        logging.info(f"Recorded changes for {relative_path}.")
                elif src_path.endswith(".rrd"):
                    # For binary files, log that they were modified
                    log_change(relative_path, "Binary file modified.")
                    logging.info(f"Recorded modification for {relative_path}.")

                # Build rsync command dynamically based on configuration
                rsync_command = ["rsync"]

                if RSYNC_ARCHIVE:
                    rsync_command.append("-a")
                if RSYNC_COMPRESS:
                    rsync_command.append("-z")
                if RSYNC_VERBOSE:
                    rsync_command.append("-v")
                if RSYNC_UPDATE:
                    rsync_command.append("--update")
                if RSYNC_CHECKSUM:
                    rsync_command.append("--checksum")
                if RSYNC_PARTIAL:
                    rsync_command.append("--partial")
                if RSYNC_BWLIMIT:
                    rsync_command.extend([f"--bwlimit={RSYNC_BWLIMIT}"])
                if RSYNC_ITEMIZE:
                    rsync_command.append("--itemize-changes")
                if RSYNC_STATS:
                    rsync_command.append("--stats")

                rsync_command.extend([
                    "-e", f"ssh -i {SSH_KEY_PATH}",
                    src_path,
                    f"{REMOTE_USER}@{REMOTE_HOST}:{REMOTE_DEST_DIR}/"
                ])

                # Execute rsync command
                result = subprocess.run(
                    rsync_command,
                    stdout=subprocess.PIPE,
                    stderr=subprocess.PIPE,
                    text=True
                )

                # Log rsync outputs
                if result.stdout:
                    logging.info(f"rsync output for {relative_path}:\n{result.stdout.strip()}")
                if result.stderr:
                    logging.warning(f"rsync warning/error for {relative_path}:\n{result.stderr.strip()}")

                if result.returncode == 0:
                    logging.info(f"Synced {relative_path} successfully.")
                    # Update state
                    self.state[relative_path] = src_mtime
                    save_state(self.state)
                    # Update backup
                    if src_path.endswith(".txt"):
                        subprocess.run(["cp", src_path, backup_path])
                        logging.debug(f"Updated backup for {relative_path}.")
                else:
                    logging.error(f"Failed to sync {relative_path}. Error: {result.stderr.strip()}")
            else:
                logging.info(f"No changes detected for {relative_path}. Skipping.")
        except FileNotFoundError:
            logging.error(f"File {relative_path} not found for syncing.")
        except Exception as e:
            logging.error(f"Error syncing {relative_path}: {e}")

# ---------------------------
# Initial Synchronization
# ---------------------------

def initial_sync(handler):
    """Perform initial synchronization of existing files and create backups."""
    logging.info("Starting initial synchronization of existing files...")
    for root, dirs, files in os.walk(SOURCE_DIR):
        for file in files:
            if any(file.endswith(ext) for ext in handler.monitored_extensions):
                src_path = os.path.join(root, file)
                handler.sync_file(src_path, event_type="initial sync")
    logging.info("Initial synchronization completed.")

# ---------------------------
# Main Execution
# ---------------------------

def main():
    logging.info("Starting RRD synchronization script.")

    # Load synchronization state
    state = load_state()

    # Create event handler
    event_handler = RRDHandler(state)

    # Perform initial synchronization
    initial_sync(event_handler)

    # Set up observer
    observer = Observer()
    observer.schedule(event_handler, SOURCE_DIR, recursive=True)
    observer.start()
    logging.info("Started monitoring for file changes.")

    try:
        while True:
            time.sleep(1)
    except KeyboardInterrupt:
        observer.stop()
        logging.info("Stopping RRD synchronization script.")
    observer.join()

if __name__ == "__main__":
    main()
```

### **3.3. Detailed Code Explanation**

Let’s break down the new components added for change tracking:

#### **a. Additional Configuration Parameters**

- **`change_tracking`:** Defines settings related to change tracking.
  - **`backup_dir`:** Directory where backup copies of synchronized files are stored.
  - **`change_log_file`:** File where detailed change logs (diffs) are recorded.
  - **`diff_tool`:** Specifies the tool used for generating diffs. Options can be `"difflib"` or `"diff"` (Unix command).

#### **b. Utility Functions for Change Tracking**

1. **`ensure_directories()`:**
   - **Purpose:** Ensures that the backup directory and change log file exist.
   - **Implementation:** Creates directories if they don’t exist and touches the change log file to ensure it exists.

2. **`log_change(file_path, change_details)`:**
   - **Purpose:** Logs the specific changes made to a file.
   - **Implementation:** Appends the timestamp, file path, and change details to the `change_log_file`.

3. **`generate_diff(old_file, new_file)`:**
   - **Purpose:** Generates a unified diff between two text files.
   - **Implementation:** Uses Python's `difflib.unified_diff` to create a diff string that highlights additions, deletions, and modifications.

#### **c. Modifications in `RRDHandler` Class**

1. **Backup Handling:**
   - **Backup Path:** Determines the backup path corresponding to the source file.
   - **Backup Directory Creation:** Ensures the backup directory exists.
   - **Updating Backups:** After successful synchronization, updates the backup copy with the latest version of the file.

2. **Change Detection and Logging:**
   - **For `.txt` Files:**
     - **Existence of Backup:** If a backup exists, generate a diff between the backup and the current file.
     - **Logging Diffs:** If differences are found, log them to the `change_log_file`.
   - **For `.rrd` Files:**
     - **Logging Modification:** Since `.rrd` files are binary, log that the file was modified without detailing content changes.

3. **Updating Backups:**
   - **Implementation:** After successful synchronization of a `.txt` file, copies the current file to the backup directory to serve as the new "previous version."

4. **Exception Handling Enhancements:**
   - **Purpose:** Ensures that any errors during synchronization or backup updating are logged, preventing the script from crashing.

---

## **4. Handling Different File Types**

Given that `.txt` and `.rrd` files have different characteristics, the script handles them accordingly:

1. **`.txt` Files (Text Files):**
   - **Change Detection:** Generates detailed diffs to capture specific changes within the file.
   - **Logging:** Records these diffs in the `change_log_file` for auditing and review.
   - **Backup Updates:** Keeps an updated backup copy to serve as the reference for future diffs.

2. **`.rrd` Files (Binary Files):**
   - **Change Detection:** Does not generate diffs since binary files aren’t human-readable and diffs aren’t meaningful.
   - **Logging:** Logs that the file was modified without detailing content changes.
   - **Backup Updates:** Can be implemented similarly if needed, but it's optional since diffs aren’t generated.

---

## **5. Enhancing Logs for Better Readability**

Your logging strategy is critical for monitoring and auditing. Here’s how the enhanced logging works:

1. **Synchronization Logs (`rrd_sync_new.log`):**
   - **Purpose:** Capture synchronization activities, `rsync` outputs, and errors.
   - **Content Examples:**
     ```
     2024-12-12 15:05:40,908 - INFO - Syncing large_test_file.txt due to created event.
     2024-12-12 15:05:40,908 - INFO - rsync output for large_test_file.txt:
     >f+++++++++ large_test_file.txt
     sent 244,867 bytes  received 35 bytes  54,422.67 bytes/sec
     total size is 251,658,240  speedup is 1,027.59
     2024-12-12 15:05:40,908 - INFO - Synced large_test_file.txt successfully.
     ```

2. **Change Logs (`change_log.txt`):**
   - **Purpose:** Store detailed changes made to `.txt` files.
   - **Content Examples:**
     ```
     Timestamp: 2024-12-12 15:06:38
     File: large_test_file.txt
     Changes:
     --- before
     +++ after
     @@ -1,3 +1,4 @@
      Line one
      Line two
      Line three
     +Line four added
     --------------------------------------------------------------------------------
     ```

   - **For `.rrd` Files:**
     ```
     Timestamp: 2024-12-12 15:07:50
     File: data_file.rrd
     Changes:
     Binary file modified.
     --------------------------------------------------------------------------------
     ```

3. **Debug Logs:**
   - **Purpose:** Track internal script operations, such as debounce timer settings and cancellations.
   - **Content Examples:**
     ```
     2024-12-12 15:05:40,908 - DEBUG - Debounce timer set for /opt/opennms/rrd_data_move/snmp/1/opennms-jvm/large_test_file.txt with 5 seconds
     2024-12-12 15:05:45,908 - DEBUG - Debounce timer canceled for /opt/opennms/rrd_data_move/snmp/1/opennms-jvm/large_test_file.txt
     ```

---

## **6. Summary and Best Practices**

### **6.1. Summary of Enhancements**

- **Backup Mechanism:**
  - Maintains copies of synchronized files to enable change detection.
  
- **Change Detection:**
  - Generates diffs for `.txt` files to capture specific content changes.
  - Logs modifications for `.rrd` binary files without detailing content changes.
  
- **Comprehensive Logging:**
  - Differentiates between synchronization logs and change logs.
  - Provides detailed, timestamped records of changes for auditing and troubleshooting.
  
- **Error Handling:**
  - Continues to handle configuration, state, and synchronization errors gracefully, ensuring continuous operation.

### **6.2. Best Practices**

1. **Secure Backup Directory:**
   - Ensure that the backup directory is secured and has appropriate permissions to prevent unauthorized access.
   
2. **Regularly Review Change Logs:**
   - Periodically audit the `change_log.txt` to monitor changes and detect any unauthorized or unexpected modifications.
   
3. **Implement Log Rotation:**
   - Prevent log files from growing indefinitely by configuring `logrotate` for both synchronization and change logs.
   
   **Example Logrotate Configuration for `change_log.txt`:**
   
   ```ini
   /opt/opennms/rrd_data_move/change_log.txt {
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

4. **Limit Backup Storage:**
   - Monitor the storage space used by the backup directory to prevent disk space exhaustion. Implement periodic cleanup or archival strategies if necessary.

5. **Enhance Security:**
   - If running under a dedicated non-root user, ensure that this user has the minimal required permissions.
   - Secure SSH keys and restrict their usage to prevent unauthorized access.

6. **Automate Testing:**
   - Develop scripts or procedures to regularly test the synchronization and change tracking mechanisms, ensuring they function as expected.

7. **Documentation:**
   - Maintain clear documentation of the synchronization setup, change tracking mechanisms, and recovery procedures to facilitate maintenance and onboarding.

---

