
Below, you'll find comprehensive guidance on:

1. **Automating the Script Using `systemd`**
2. **Detailed Explanation of the Python Script**
3. **Modifying the Script to Exclude Deletion Events**

---

## **1. Automating the Script Using `systemd`**

Automating your script ensures it starts on system boot, restarts on failures, and runs in the background without manual intervention. We'll use `systemd`, the init system widely adopted in modern Linux distributions, to manage your Python synchronization script.

### **1.1. Create a `systemd` Service File**

1. **Create the Service File:**

   Open a terminal and create a new service file for your script:

   ```bash
   sudo nano /etc/systemd/system/rrd_sync_new.service
   ```

2. **Add the Following Content:**

   ```ini
   [Unit]
   Description=RRD File-Level Synchronization Service
   After=network.target

   [Service]
   Type=simple
   ExecStart=/usr/bin/python3 /opt/opennms/rrd_data_move/rrd_sync_new.py
   Restart=on-failure
   RestartSec=30
   User=rrd_sync_user  # Replace with your dedicated non-root user if applicable
   Environment=PYTHONUNBUFFERED=1

   # Logging
   StandardOutput=append:/opt/opennms/rrd_data_move/rrd_sync_new.log
   StandardError=append:/opt/opennms/rrd_data_move/rrd_sync_new.log

   [Install]
   WantedBy=multi-user.target
   ```

   **Notes:**

   - **`Description`**: A brief description of the service.
   - **`After`**: Ensures the service starts after the network is up.
   - **`ExecStart`**: The command to execute your Python script.
   - **`Restart`**: Automatically restarts the service on failure.
   - **`RestartSec`**: Waits 30 seconds before attempting a restart.
   - **`User`**: The user under which the script runs. It's recommended to use a dedicated non-root user (explained later).
   - **`Environment`**: Ensures Python outputs are unbuffered, making logs more immediate.
   - **`StandardOutput` & `StandardError`**: Directs both standard output and errors to your log file.

3. **Save and Exit:**

   - Press `CTRL + O` to write changes.
   - Press `CTRL + X` to exit the editor.

### **1.2. Reload `systemd` and Enable the Service**

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

4. **Verify the Service Status:**

   ```bash
   sudo systemctl status rrd_sync_new.service
   ```

   **Expected Output:**

   ```
   ● rrd_sync_new.service - RRD File-Level Synchronization Service
        Loaded: loaded (/etc/systemd/system/rrd_sync_new.service; enabled; vendor preset: enabled)
        Active: active (running) since Tue 2024-12-12 15:05:40 UTC; 1min ago
      Main PID: 12345 (python3)
         Tasks: 1 (limit: 4915)
        Memory: 50.0M
           CPU: 0.10s
        CGroup: /system.slice/rrd_sync_new.service
                └─12345 /usr/bin/python3 /opt/opennms/rrd_data_move/rrd_sync_new.py
   ```

### **1.3. Testing the Automated Service**

1. **Check if the Service is Running:**

   ```bash
   sudo systemctl status rrd_sync_new.service
   ```

   Ensure the status is `active (running)`.

2. **View Real-Time Logs:**

   ```bash
   sudo tail -f /opt/opennms/rrd_data_move/rrd_sync_new.log
   ```

   **Alternatively, Use `journalctl`:**

   ```bash
   sudo journalctl -u rrd_sync_new.service -f
   ```

3. **Simulate a System Reboot (Optional):**

   To ensure the service starts on boot, you can perform a system reboot:

   ```bash
   sudo reboot
   ```

   After rebooting, verify the service status:

   ```bash
   sudo systemctl status rrd_sync_new.service
   ```

---

## **2. Detailed Explanation of the Python Script**

Understanding each component of your script is crucial for effective communication and troubleshooting. Below is a comprehensive breakdown of the script's functionality.

### **2.1. Script Structure Overview**

1. **Configuration Loading**
2. **Logging Setup**
3. **State Management**
4. **File System Event Handling**
5. **Synchronization and Deletion Logic**
6. **Initial Synchronization**
7. **Main Execution Loop**

### **2.2. Detailed Section-by-Section Explanation**

#### **a. Importing Necessary Modules**

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
```

- **`os`**: Interacts with the operating system (e.g., file paths, environment variables).
- **`json`**: Handles JSON file reading and writing for configurations and state.
- **`subprocess`**: Executes shell commands (`rsync` in this case).
- **`logging`**: Manages logging of events, errors, and information.
- **`time`**: Provides time-related functions (e.g., sleep).
- **`watchdog` Modules**: Monitors filesystem events.
- **`threading` Modules**: Implements the debouncing mechanism to handle rapid event triggers.
- **`pathlib.Path`**: Simplifies path manipulations.

#### **b. Loading Configuration from `config.json`**

```python
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
```

- **Purpose:** Externalizes configurations, making the script adaptable without modifying the code.
- **Error Handling:** Raises exceptions if the configuration file is missing or malformed, preventing the script from running with incorrect settings.

#### **c. Defining Configuration Variables**

```python
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
```

- **Description:** Extracts specific configuration parameters from the loaded JSON file.
- **Defaults:** Provides default values in case certain configurations are missing, ensuring the script can still run with minimal settings.

#### **d. Setting Up Logging**

```python
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
```

- **Purpose:** Configures the logging system to output messages to a specified log file with timestamps and severity levels.
- **Log Levels:** Controls the verbosity of logs (`DEBUG` being the most verbose and `CRITICAL` the least).

#### **e. State Management Functions**

```python
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
```

- **Purpose:** Keeps track of the last modification times of synchronized files to determine if synchronization is necessary.
- **Error Handling:** Logs an error if the state file is corrupted and initializes an empty state to prevent disruptions.

#### **f. File System Event Handler Class**

```python
class RRDHandler(FileSystemEventHandler):
    """Handles filesystem events for synchronization."""
    def __init__(self, state):
        super().__init__()
        self.state = state
        self.monitored_extensions = MONITORED_EXTENSIONS
        self.lock = Lock()
        self.debounce_timers = {}
```

- **Inheritance:** Extends `FileSystemEventHandler` from `watchdog` to handle file system events.
- **Attributes:**
  - **`self.state`**: Current synchronization state.
  - **`self.monitored_extensions`**: File extensions to monitor (e.g., `.rrd`, `.txt`).
  - **`self.lock`**: Ensures thread-safe operations when handling timers.
  - **`self.debounce_timers`**: Keeps track of active debounce timers for each file.

#### **g. Event Handling Methods**

```python
def on_created(self, event):
    """Handle file creation events."""
    if not event.is_directory and self._is_monitored(event.src_path):
        self._debounced_sync(event.src_path, event_type="created")

def on_modified(self, event):
    """Handle file modification events."""
    if not event.is_directory and self._is_monitored(event.src_path):
        self._debounced_sync(event.src_path, event_type="modified")

def on_deleted(self, event):
    """Handle file deletion events."""
    if not event.is_directory and self._is_monitored(event.src_path):
        self._debounced_delete(event.src_path)
```

- **Purpose:** Responds to file creation, modification, and deletion events.
- **Condition:** Ensures that only monitored files (based on their extensions) trigger synchronization actions.

#### **h. Helper Methods for Monitoring and Debouncing**

```python
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

def _debounced_delete(self, src_path):
    """Debounce deletion to prevent multiple triggers."""
    with self.lock:
        if src_path in self.debounce_timers:
            self.debounce_timers[src_path].cancel()
            logging.debug(f"Debounce timer canceled for {src_path}")
        timer = Timer(DEBOUNCE_TIME, self.delete_file, [src_path])
        self.debounce_timers[src_path] = timer
        timer.start()
        logging.debug(f"Debounce timer set for deletion of {src_path} with {DEBOUNCE_TIME} seconds")
```

- **`_is_monitored`**: Checks if a file's extension is among the monitored types.
- **Debouncing Mechanism**:
  - **Purpose:** Prevents multiple rapid synchronization events for the same file, which is especially useful when large files are being written in chunks.
  - **Implementation**:
    - **Cancel Existing Timer:** If a timer is already running for the file, it's canceled to reset the debounce period.
    - **Start New Timer:** A new timer is set to trigger synchronization after `DEBOUNCE_TIME` seconds.

#### **i. Synchronization Method**

```python
def sync_file(self, src_path, event_type):
    """Synchronize a single file to the remote server."""
    relative_path = os.path.relpath(src_path, SOURCE_DIR)
    try:
        src_stat = os.stat(src_path)
        src_mtime = src_stat.st_mtime

        # Check if the file needs to be copied
        if (relative_path not in self.state) or (self.state[relative_path] < src_mtime):
            logging.info(f"Syncing {relative_path} due to {event_type} event.")

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
            else:
                logging.error(f"Failed to sync {relative_path}. Error: {result.stderr.strip()}")
        else:
            logging.info(f"No changes detected for {relative_path}. Skipping.")
    except FileNotFoundError:
        logging.error(f"File {relative_path} not found for syncing.")
    except Exception as e:
        logging.error(f"Error syncing {relative_path}: {e}")
```

- **Purpose:** Synchronizes a single file to the remote server using `rsync` when changes are detected.
- **Process:**
  1. **Determine File Status:**
     - Checks if the file is new or modified by comparing modification times.
  2. **Build `rsync` Command:**
     - Dynamically constructs the `rsync` command based on configurations.
     - Includes flags like `--itemize-changes` and `--stats` for detailed logging.
  3. **Execute `rsync`:**
     - Runs the `rsync` command and captures output and errors.
  4. **Logging:**
     - Logs the output and any warnings or errors.
     - Confirms successful synchronization.
  5. **Update State:**
     - Updates the state file with the latest modification time upon successful synchronization.

#### **j. Deletion Handling Method**

```python
def delete_file(self, src_path):
    """Delete a file from the remote server."""
    relative_path = os.path.relpath(src_path, SOURCE_DIR)
    try:
        logging.info(f"Deleting {relative_path} from destination...")

        # Build rsync delete command dynamically based on configuration
        rsync_delete_command = ["rsync"]

        if RSYNC_ARCHIVE:
            rsync_delete_command.append("-a")
        if RSYNC_COMPRESS:
            rsync_delete_command.append("-z")
        if RSYNC_VERBOSE:
            rsync_delete_command.append("-v")
        if RSYNC_UPDATE:
            rsync_delete_command.append("--update")
        if RSYNC_CHECKSUM:
            rsync_delete_command.append("--checksum")
        if RSYNC_PARTIAL:
            rsync_delete_command.append("--partial")
        if RSYNC_BWLIMIT:
            rsync_delete_command.extend([f"--bwlimit={RSYNC_BWLIMIT}"])
        if RSYNC_ITEMIZE:
            rsync_delete_command.append("--itemize-changes")
        if RSYNC_STATS:
            rsync_delete_command.append("--stats")
        if "--delete" not in rsync_delete_command:
            rsync_delete_command.append("--delete")

        rsync_delete_command.extend([
            "-e", f"ssh -i {SSH_KEY_PATH}",
            SOURCE_DIR,
            f"{REMOTE_USER}@{REMOTE_HOST}:{REMOTE_DEST_DIR}/"
        ])

        # Execute rsync delete command
        result = subprocess.run(
            rsync_delete_command,
            stdout=subprocess.PIPE,
            stderr=subprocess.PIPE,
            text=True
        )

        # Log rsync outputs
        if result.stdout:
            logging.info(f"rsync output for deletion:\n{result.stdout.strip()}")
        if result.stderr:
            logging.warning(f"rsync warning/error for deletion:\n{result.stderr.strip()}")

        if result.returncode == 0:
            logging.info(f"Deleted {relative_path} from destination successfully.")
            # Remove from state
            if relative_path in self.state:
                del self.state[relative_path]
                save_state(self.state)
        else:
            logging.error(f"Failed to delete {relative_path} from destination. Error: {result.stderr.strip()}")
    except Exception as e:
        logging.error(f"Error deleting {relative_path}: {e}")
```

- **Purpose:** Synchronizes file deletions to the remote server using `rsync` with the `--delete` flag.
- **Process:**
  1. **Build `rsync` Delete Command:**
     - Includes the `--delete` flag to remove files on the destination that no longer exist on the source.
  2. **Execute `rsync`:**
     - Runs the delete command and captures output and errors.
  3. **Logging:**
     - Logs the output and any warnings or errors.
     - Confirms successful deletion synchronization.
  4. **Update State:**
     - Removes the deleted file from the state file upon successful synchronization.

#### **k. Initial Synchronization Function**

```python
def initial_sync(handler):
    """Perform initial synchronization of existing files."""
    logging.info("Starting initial synchronization of existing files...")
    for root, dirs, files in os.walk(SOURCE_DIR):
        for file in files:
            if any(file.endswith(ext) for ext in handler.monitored_extensions):
                src_path = os.path.join(root, file)
                handler.sync_file(src_path, event_type="initial sync")
    logging.info("Initial synchronization completed.")
```

- **Purpose:** Ensures that all existing monitored files are synchronized when the script starts.
- **Process:**
  1. **Traverse Source Directory:**
     - Walks through the source directory and its subdirectories.
  2. **Identify Monitored Files:**
     - Filters files based on monitored extensions.
  3. **Synchronize Each File:**
     - Invokes the `sync_file` method for each monitored file to ensure it's up-to-date on the remote server.

#### **l. Main Execution Loop**

```python
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

- **Purpose:** Orchestrates the overall flow of the script.
- **Process:**
  1. **Logging Start:** Logs the initiation of the synchronization process.
  2. **Load State:** Reads the current synchronization state from the state file.
  3. **Initialize Event Handler:** Creates an instance of `RRDHandler` with the loaded state.
  4. **Initial Synchronization:** Ensures all existing files are synchronized.
  5. **Set Up Observer:**
     - Creates an `Observer` instance from `watchdog` to monitor file system events.
     - Schedules the event handler for the source directory, enabling recursive monitoring.
  6. **Start Observer:** Begins monitoring for file changes.
  7. **Continuous Running:** Enters an infinite loop, keeping the script running.
  8. **Graceful Shutdown:** Handles `KeyboardInterrupt` (e.g., Ctrl+C) to stop the observer and log the shutdown.

---

## **3. Modifying the Script to Exclude Deletion Events**

If you prefer **not to synchronize file deletions**—meaning that deleting a file in the source directory does **not** remove it from the destination—you can simplify the script by removing the deletion handling logic.

### **3.1. Steps to Modify the Script**

1. **Remove Deletion-Related Code:**

   - **Delete Event Method:**
     Remove the `on_deleted` method and any references to deletion events.

   - **Delete Debounce Methods:**
     Remove `_debounced_delete` and `delete_file` methods.

   - **Update `RRDHandler` Class:**
     Ensure the class no longer handles deletion events.

2. **Updated Script Without Deletion Synchronization:**

   Below is the modified script with deletion synchronization removed and enhanced logging for better readability.

### **3.2. Final Script Without Deletion Synchronization**

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
        """Synchronize a single file to the remote server."""
        relative_path = os.path.relpath(src_path, SOURCE_DIR)
        try:
            src_stat = os.stat(src_path)
            src_mtime = src_stat.st_mtime

            # Check if the file needs to be copied
            if (relative_path not in self.state) or (self.state[relative_path] < src_mtime):
                logging.info(f"Syncing {relative_path} due to {event_type} event.")

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
                else:
                    logging.error(f"Failed to sync {relative_path}. Error: {result.stderr.strip()}")
            else:
                logging.info(f"No changes detected for {relative_path}. Skipping.")
        except FileNotFoundError:
            logging.error(f"File {relative_path} not found for syncing.")
        except Exception as e:
            logging.error(f"Error syncing {relative_path}: {e}")
```

### **3.3. Key Changes Explained**

1. **Removal of Deletion Handling:**

   - **`on_deleted` Method:** Completely removed.
   - **Debounce Delete Methods:** Removed `_debounced_delete` and `delete_file`.
   - **Rationale:** Ensures that deletions in the source directory do not affect the destination, leaving files intact on the remote server even if they're deleted locally.

2. **Enhanced Logging:**

   - **Event Type Logging:** Each synchronization event logs whether it was triggered by a `created`, `modified`, or `initial sync` event.
   - **Detailed `rsync` Output:** Includes `--itemize-changes` and `--stats` for clear visibility into what changes are being synchronized.
   - **Debug Logs:** Added debug-level logs for debounce timer activities to aid in monitoring and troubleshooting.

3. **Dynamic `rsync` Command Building:**

   - **Flexibility:** Builds the `rsync` command based on configurations, making it easy to adjust behavior via `config.json`.
   - **Readability:** Clearly separates configuration flags, enhancing maintainability.

4. **State Management:**

   - **Indentation in JSON:** `save_state` now formats the state file with indentation for better readability.
   - **State Update Conditions:** Only updates the state if synchronization is successful, ensuring consistency.

---

## **4. Detailed Script Walkthrough for Presentation**

To effectively explain the script to your manager, it's essential to present its functionality, benefits, and how it aligns with your organization's needs. Below is a structured explanation you can use.

### **4.1. Purpose of the Script**

- **Objective:** Continuously monitor specific directories for changes to `.rrd` and `.txt` files and synchronize these changes to a remote server efficiently.
- **Benefits:**
  - **Automation:** Eliminates the need for manual synchronization, ensuring data consistency.
  - **Efficiency:** Transfers only the changes within files, conserving bandwidth and reducing synchronization time.
  - **Reliability:** Includes mechanisms to prevent multiple rapid syncs and ensures synchronization continues even after failures.
  - **Customizability:** Externalizes configurations, making it adaptable to various environments without code changes.

### **4.2. How the Script Works**

1. **Configuration Loading:**
   - Reads all customizable settings from an external `config.json` file.
   - Parameters include source/destination directories, remote server details, logging preferences, debounce time, monitored file extensions, and `rsync` options.

2. **Logging Setup:**
   - Initializes logging based on configurations.
   - Logs are written to a specified file with timestamps and severity levels.
   - Enhanced logging includes detailed `rsync` outputs for transparency.

3. **State Management:**
   - Maintains a JSON state file tracking the last synchronization times of each file.
   - Prevents unnecessary transfers by checking if a file has been modified since the last sync.

4. **File System Monitoring:**
   - Utilizes the `watchdog` library to monitor the source directory for file creation and modification events.
   - Filters events based on specified file extensions (e.g., `.rrd`, `.txt`).

5. **Debouncing Mechanism:**
   - Implements a debounce period (e.g., 5 seconds) to handle rapid successive events on the same file.
   - Ensures that synchronization occurs only once within the debounce interval, enhancing efficiency.

6. **Synchronization with `rsync`:**
   - Constructs an `rsync` command with configurable options to synchronize files.
   - Transfers only the changes within files, leveraging `rsync`'s checksum and itemization features.
   - Limits bandwidth usage to prevent network saturation.

7. **Automated Service Management:**
   - Managed by `systemd`, ensuring the script runs continuously in the background.
   - Automatically restarts on failures, maintaining synchronization reliability.
   - Starts on system boot, eliminating manual startup needs.

### **4.3. Customization and Flexibility**

- **External Configuration:** All key parameters are defined in `config.json`, allowing easy adjustments without modifying the script.
- **Monitored File Types:** Easily specify which file extensions to monitor.
- **`rsync` Options:** Tailor synchronization behavior (e.g., compression, bandwidth limits) via configuration.
- **Logging Levels:** Adjust verbosity for different operational needs.

### **4.4. Benefits Over Manual Synchronization**

- **Consistency:** Ensures that remote data is always up-to-date with local changes without manual checks.
- **Efficiency:** Optimizes data transfer by sending only incremental changes, saving time and resources.
- **Reliability:** Automated restarts and continuous monitoring prevent data discrepancies and minimize downtime.
- **Scalability:** Can be easily adapted to monitor additional directories or handle more file types as needs grow.

### **4.5. Security Considerations**

- **SSH Key-Based Authentication:** Uses SSH keys for secure, password-less authentication to the remote server.
- **Dedicated Non-Root User (Recommended):** Enhances security by limiting the script's permissions.
- **Restrictive File Permissions:** Ensures that sensitive configuration files and SSH keys are protected.

---

## **5. Security Best Practices (Optional but Recommended)**

For enhanced security, especially in production environments, it's advisable to run the synchronization script under a dedicated non-root user. Below are the steps to set this up.

### **5.1. Create a Dedicated User**

1. **Add New User:**

   ```bash
   sudo adduser rrd_sync_user
   sudo passwd rrd_sync_user  # Set a strong password
   ```

2. **Create SSH Directory for the User:**

   ```bash
   sudo mkdir -p /home/rrd_sync_user/.ssh
   sudo chown -R rrd_sync_user:rrd_sync_user /home/rrd_sync_user/.ssh
   sudo chmod 700 /home/rrd_sync_user/.ssh
   ```

### **5.2. Set Up SSH Key-Based Authentication**

1. **Switch to the Dedicated User:**

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

   **Alternatively, Manually Append the Public Key:**

   ```bash
   cat ~/.ssh/id_rsa_rrd_sync.pub | ssh rrd_sync_user@192.168.29.153 "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
   ```

4. **Test SSH Connection:**

   ```bash
   ssh -i ~/.ssh/id_rsa_rrd_sync rrd_sync_user@192.168.29.153 "echo 'SSH connection successful!'"
   ```

   **Expected Output:**

   ```
   SSH connection successful!
   ```

### **5.3. Update `config.json` for the Dedicated User**

Modify the `config.json` to use the new user and SSH key path:

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
    }
}
```

### **5.4. Adjust Permissions**

1. **Change Ownership of Synchronization Directory:**

   ```bash
   sudo chown -R rrd_sync_user:rrd_sync_user /opt/opennms/rrd_data_move/
   ```

2. **Set Correct Permissions for SSH Keys:**

   ```bash
   sudo chmod 600 /home/rrd_sync_user/.ssh/id_rsa_rrd_sync
   ```

### **5.5. Update `systemd` Service to Use the Dedicated User**

1. **Edit the Service File:**

   ```bash
   sudo nano /etc/systemd/system/rrd_sync_new.service
   ```

2. **Modify the `[Service]` Section:**

   ```ini
   [Service]
   Type=simple
   ExecStart=/usr/bin/python3 /opt/opennms/rrd_data_move/rrd_sync_new.py
   Restart=on-failure
   RestartSec=30
   User=rrd_sync_user  # Change to the dedicated user
   Environment=PYTHONUNBUFFERED=1

   # Logging
   StandardOutput=append:/opt/opennms/rrd_data_move/rrd_sync_new.log
   StandardError=append:/opt/opennms/rrd_data_move/rrd_sync_new.log
   ```

3. **Reload `systemd` and Restart the Service:**

   ```bash
   sudo systemctl daemon-reload
   sudo systemctl restart rrd_sync_new.service
   sudo systemctl status rrd_sync_new.service
   ```

   **Expected Output:**

   ```
   ● rrd_sync_new.service - RRD File-Level Synchronization Service
        Loaded: loaded (/etc/systemd/system/rrd_sync_new.service; enabled; vendor preset: enabled)
        Active: active (running) since Tue 2024-12-12 15:05:40 UTC; 1min ago
      Main PID: 12345 (python3)
         Tasks: 1 (limit: 4915)
        Memory: 50.0M
           CPU: 0.10s
        CGroup: /system.slice/rrd_sync_new.service
                └─12345 /usr/bin/python3 /opt/opennms/rrd_data_move/rrd_sync_new.py
   ```

### **5.6. Update Permissions on Synchronization Directory**

Ensure that the dedicated user owns the synchronization directory:

```bash
sudo chown -R rrd_sync_user:rrd_sync_user /opt/opennms/rrd_data_move/
sudo chmod -R 700 /opt/opennms/rrd_data_move/
```

### **5.7. Verify SSH Permissions on Destination Server**

Ensure that the remote user (`rrd_sync_user`) has the necessary permissions to write to the destination directory:

```bash
sudo ssh -i /home/rrd_sync_user/.ssh/id_rsa_rrd_sync rrd_sync_user@192.168.29.153 "mkdir -p /opt/opennms/rrd_data_move/destination/snmp/2/ && chmod -R 700 /opt/opennms/rrd_data_move/destination/snmp/2/"
```

---

## **6. Conclusion and Next Steps**

Your synchronization setup is now fully automated, secure, and tailored to exclude deletion events. Here's a summary of what you've achieved:

1. **Automation with `systemd`:**
   - Ensures the script runs continuously in the background.
   - Automatically starts on system boot and restarts on failures.

2. **Comprehensive Script Understanding:**
   - Detailed explanation of each script component for effective communication and troubleshooting.

3. **Enhanced Customizability:**
   - Externalized configurations via `config.json` for flexibility and ease of maintenance.

4. **Security Enhancements:**
   - Running the script under a dedicated non-root user.
   - Securing SSH key-based authentication.

5. **Exclusion of Deletion Events:**
   - Modified the script to ignore file deletions, maintaining files on the destination even if deleted locally.

### **Next Steps:**

1. **Thorough Testing:**
   - Create, modify, and delete files in the source directory.
   - Verify synchronization behavior via logs and remote server checks.

2. **Monitor Logs Regularly:**
   - Ensure the script operates as expected and promptly address any issues indicated in the logs.

3. **Implement Log Rotation:**
   - Prevent log files from growing indefinitely by configuring `logrotate` (already covered in earlier sections).

4. **Consider Alerting Mechanisms:**
   - Set up email or other notifications for critical errors or failures in synchronization.

5. **Document the Setup:**
   - Maintain clear documentation of the configuration and setup process for future reference and onboarding.

6. **Regular Updates:**
   - Keep the system and dependencies updated to benefit from security patches and feature improvements.

---
