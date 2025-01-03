import os
import time
from pynput import mouse, keyboard
import subprocess
import #logging

# Configure #logging
#logging.basicConfig(
    #level=#logging.DEBUG,
    level=#logging.OFF,
    format='%(asctime)s - %(levelname)s - %(message)s'
)

# Adjustable idle timeout (in seconds)
IDLE_TIMEOUT = 60


last_activity_time = time.time()

def reset_timer(*args):
    """Reset the idle timer on any activity."""
    global last_activity_time
    last_activity_time = time.time()
    #logging.debug(f"Activity detected - Timer reset at {last_activity_time}")

def check_idle_time():
    """Monitor idle time and adjust brightness."""
    global last_activity_time
    #logging.info("Starting idle time monitoring")
    while True:
        idle_time = time.time() - last_activity_time
        #logging.debug(f"Current idle time: {idle_time:.2f} seconds")
        
        if idle_time > IDLE_TIMEOUT:
            #logging.info(f"Idle timeout ({IDLE_TIMEOUT}s) reached. Turning off display...")
            try:
                result = subprocess.run(
                    ["/usr/local/bin/imacdisplay.py", "-s", "0", "-z"],
                    capture_output=True,
                    text=True
                )
                #logging.debug(f"Display off command result: {result.returncode}")
                if result.stderr:
                    #logging.error(f"Error turning off display: {result.stderr}")
            except Exception as e:
                #logging.error(f"Failed to execute display off command: {str(e)}")

            while idle_time > IDLE_TIMEOUT:
                idle_time = time.time() - last_activity_time
                #logging.debug(f"Waiting for activity... Current idle time: {idle_time:.2f}")
                time.sleep(1)  # Wait until user activity is detected

            #logging.info("Activity detected after idle. Restoring brightness...")
            try:
                result = subprocess.run(
                    ["/usr/local/bin/imacdisplay.py", "-s", "50"],
                    capture_output=True,
                    text=True
                )
                #logging.debug(f"Restore brightness command result: {result.returncode}")
                if result.stderr:
                    #logging.error(f"Error restoring brightness: {result.stderr}")
            except Exception as e:
                logging.error(f"Failed to execute restore brightness command: {str(e)}")
        
        time.sleep(1)

# Start mouse and keyboard listeners
mouse_listener = mouse.Listener(
    #on_move=reset_timer, on_click=reset_timer, on_scroll=reset_timer
)
keyboard_listener = keyboard.Listener(on_press=reset_timer)

mouse_listener.start()
keyboard_listener.start()

# Run idle time checker
check_idle_time()
