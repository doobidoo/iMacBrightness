## iMac Idle Script for Ubuntu

### **Script: `imac_idle.py`**
```python
import os
import time
from pynput import mouse, keyboard
import subprocess

# Adjustable idle timeout (in seconds)
IDLE_TIMEOUT = 60

last_activity_time = time.time()

def reset_timer(*args):
    """Reset the idle timer on any activity."""
    global last_activity_time
    last_activity_time = time.time()

def check_idle_time():
    """Monitor idle time and adjust brightness."""
    global last_activity_time
    while True:
        idle_time = time.time() - last_activity_time
        if idle_time > IDLE_TIMEOUT:
            # Turn off the display by setting brightness to 0
            subprocess.run(["/usr/local/bin/imacdisplay.py", "-s", "0"])
            while idle_time > IDLE_TIMEOUT:
                idle_time = time.time() - last_activity_time
                time.sleep(1)  # Wait until user activity is detected
            # Restore brightness to a default level, e.g., 50
            subprocess.run(["/usr/local/bin/imacdisplay.py", "-s", "50"])
        time.sleep(1)

# Start mouse and keyboard listeners
mouse_listener = mouse.Listener(
    on_move=reset_timer, on_click=reset_timer, on_scroll=reset_timer
)
keyboard_listener = keyboard.Listener(on_press=reset_timer)

mouse_listener.start()
keyboard_listener.start()

# Run idle time checker
check_idle_time()
```

### **Installation Requirements**

#### **1. Install Python and `pynput`**
Ensure Python is installed:
```bash
sudo apt update
sudo apt install python3 python3-pip
```

Install the required `pynput` library:
```bash
pip3 install pynput --user
```

#### **2. Adjust `imacdisplay.py` Minimum Brightness**
Edit your existing `/usr/local/bin/imacdisplay.py` script to allow a minimum brightness of `0`. Locate the line in the script that enforces the minimum brightness (e.g., `min_brightness = 5`) and change it to:
```python
min_brightness = 0
```
Save the changes.

#### **3. Save the Script**
Save the above `imac_idle.py` script to a suitable location, e.g., `/usr/local/bin/imac_idle.py`. Ensure it is executable:
```bash
sudo chmod +x /usr/local/bin/imac_idle.py
```

### **Automating Script at Startup**

#### **Option 1: Systemd Service (Recommended)**

1. **Create a Systemd Service File**
   ```bash
   sudo nano /etc/systemd/system/imac_idle.service
   ```

   Add the following content:
   ```ini
   [Unit]
   Description=iMac Idle Backlight Script
   After=multi-user.target

   [Service]
   ExecStart=/usr/bin/python3 /usr/local/bin/imac_idle.py
   Restart=always
   User=your-username

   [Install]
   WantedBy=multi-user.target
   ```
   Replace `your-username` with your Ubuntu username.

2. **Enable and Start the Service**
   ```bash
   sudo systemctl daemon-reload
   sudo systemctl enable imac_idle.service
   sudo systemctl start imac_idle.service
   ```

3. **Check Service Status**
   Verify that the service is running:
   ```bash
   sudo systemctl status imac_idle.service
   ```

#### **Option 2: Add to Startup Applications (GUI)**

1. Open the **Startup Applications** tool:
   ```bash
   gnome-session-properties
   ```
   If unavailable, install it:
   ```bash
   sudo apt install gnome-startup-applications
   ```

2. Click **Add** and configure:
   - **Name**: `iMac Idle Script`
   - **Command**: `python3 /usr/local/bin/imac_idle.py`
   - **Comment**: (Optional) A brief description.

3. Save and close the tool.

### **Testing the Script**
Run the script manually to ensure it works as expected:
```bash
python3 /usr/local/bin/imac_idle.py
```

### **Notes**
- Make sure the `imacdisplay.py` script works as intended with the adjusted brightness levels.
- Test the idle timeout and brightness restoration functionality to verify proper operation.

