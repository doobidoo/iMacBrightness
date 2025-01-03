# iMac Display Brightness Control for Ubuntu using ESP32

This guide explains how to control an iMac display's brightness in Ubuntu using an ESP32 microcontroller. This solution is particularly useful for iMacs running Ubuntu where the standard brightness controls don't work.
Derived from https://medium.com/@fixingthings/imac-2009-2010-2011-gpu-upgrade-fixing-led-lcd-pwm-brightness-with-an-esp32-bc32da61a0e7 and adapted for Ubuntu.

## Hardware Requirements

- ESP32 development board
- PCI-E 6-pin extension cable or splitter (to intercept PWM signal)
- USB cable for ESP32
- Basic soldering equipment

## Hardware Setup

1. Open the iMac and locate the LCD driver board
2. Identify the PCI-E 6-pin cable connected to the LCD
3. Cut the PWM wire on your extension cable (typically bottom right pin)
4. Connect the PWM wire to GPIO23 on the ESP32
5. Connect ESP32 to a USB port
6. Install mbpfan (`sudo apt install mbpfan`) to manage fan control

## Software Installation

### 1. ESP32 Code
Using PlatformIO in Visual Studio Code:

1. Create new project with:
   - Board: esp32dev
   - Framework: arduino

2. Add this code to `src/main.cpp`:
```cpp
#include <Arduino.h>

const int pwmPin = 23;        // GPIO23 for PWM output
const int pwmFreq = 10000;    // 10kHz PWM frequency
const int pwmChannel = 0;     // PWM channel (0-15)
const int pwmResolution = 8;  // 8-bit resolution (0-255)
const int ledPin = 2;         // Built-in LED for status

void setup() {
  Serial.begin(115200);
  pinMode(ledPin, OUTPUT);
  digitalWrite(ledPin, HIGH);
  
  ledcSetup(pwmChannel, pwmFreq, pwmResolution);
  ledcAttachPin(pwmPin, pwmChannel);
  
  int initialBrightness = (70 * 255) / 100;
  ledcWrite(pwmChannel, initialBrightness);
  
  Serial.println("ESP32 LCD Brightness Control Ready");
}

void loop() {
  if (Serial.available() > 0) {
    String input = Serial.readStringUntil('\n');
    int percentBrightness = input.toInt();
    percentBrightness = constrain(percentBrightness, 5, 100);
    int pwmValue = map(percentBrightness, 5, 100, 13, 255);
    
    ledcWrite(pwmChannel, pwmValue);
    digitalWrite(ledPin, LOW);
    delay(50);
    digitalWrite(ledPin, HIGH);
    
    Serial.print("Brightness set to: ");
    Serial.print(percentBrightness);
    Serial.println("%");
  }
}
```

### 2. Ubuntu Control Script

1. Install required packages:
```bash
sudo apt install python3-pip python3-serial
```

2. Create the script at `/usr/local/bin/imacdisplay.py`:
```python
#!/usr/bin/env python3
import serial
import argparse
import sys
import time
from pathlib import Path
import json

def get_config_file():
    return Path.home() / '.config' / 'imacdisplay.conf'

def load_config():
    config_file = get_config_file()
    try:
        if config_file.exists():
            with open(config_file) as f:
                return json.load(f)
    except Exception as e:
        print(f"Error loading config: {e}")
    return {'port': '/dev/ttyUSB0', 'last_brightness': 70}

def save_config(brightness):
    config_file = get_config_file()
    config = load_config()
    config['last_brightness'] = brightness
    try:
        config_file.parent.mkdir(parents=True, exist_ok=True)
        with open(config_file, 'w') as f:
            json.dump(config, f)
    except Exception as e:
        print(f"Error saving config: {e}")

def setup_serial(port='/dev/ttyUSB0'):
    try:
        ser = serial.Serial(port, 115200, timeout=2)
        time.sleep(1)  # Wait for connection to stabilize
        return ser
    except serial.SerialException as e:
        print(f"Error: Could not open serial port {port}")
        print(f"Details: {e}")
        return None

# def set_brightness(ser, value):
#     """Set brightness with safety limits"""
#     try:
#         # Ensure brightness is between 5-100%
#         value = max(0, min(100, value))
#         ser.write(f"{value}\n".encode())
#         time.sleep(0.1)  # Wait for response
#         response = ser.readline().decode().strip()
#         print(f"Set brightness to {value}%")
#         save_config(value)  # Save the current brightness
#         return value
#     except Exception as e:
#         print(f"Error setting brightness: {e}")
#         return None
def set_brightness(ser, value, allow_zero=False):
    """Set brightness with safety limits
    
    Args:
        ser: Serial connection
        value: Brightness value to set
        allow_zero: If True, allows 0%, otherwise enforces 5% minimum
    """
    try:
        if allow_zero:
            value = max(0, min(100, value))
        else:
            value = max(5, min(100, value))
            
        ser.write(f"{value}\n".encode())
        time.sleep(0.1)  # Wait for response
        response = ser.readline().decode().strip()
        print(f"Set brightness to {value}%")
        save_config(value)  # Save the current brightness
        return value
    except Exception as e:
        print(f"Error setting brightness: {e}")
        return None

def get_brightness():
    """Get last saved brightness value"""
    config = load_config()
    return config.get('last_brightness', 70)

def main():
    parser = argparse.ArgumentParser(description='iMac Display Brightness Control')
    group = parser.add_mutually_exclusive_group()
    group.add_argument('-s', '--set', type=int, help='Set brightness (5-100, or 0 with -z)')
    group.add_argument('-g', '--get', action='store_true', help='Get current brightness')
    group.add_argument('-i', '--increment', type=int, help='Increase brightness by amount')
    group.add_argument('-d', '--decrement', type=int, help='Decrease brightness by amount')
    parser.add_argument('-p', '--port', default='/dev/ttyUSB0', help='Serial port')
    parser.add_argument('-z', '--allow-zero', action='store_true', help='Allow setting brightness to 0')
    args = parser.parse_args()

    # Debug prints to see what's happening
    print(f"Command arguments: {args}")

    if args.get:
        print(get_brightness())
        return

    # Setup serial connection for all operations that need it
    if args.set is not None or args.increment is not None or args.decrement is not None:
        ser = setup_serial(args.port)
        if not ser:
            print("Failed to setup serial connection")
            return

        try:
            current = get_brightness()
            print(f"Current brightness: {current}")

            if args.set is not None:
                # Explicit check for zero setting
                if args.set == 0 and not args.allow_zero:
                    print("Error: Cannot set brightness to 0 without --allow-zero (-z) flag")
                    return
                print(f"Setting brightness to {args.set} with allow_zero={args.allow_zero}")
                set_brightness(ser, args.set, allow_zero=args.allow_zero)
            
            elif args.increment is not None:
                new_value = min(current + args.increment, 100)
                print(f"Incrementing brightness to {new_value}")
                set_brightness(ser, new_value, allow_zero=False)
            
            elif args.decrement is not None:
                new_value = max(current - args.decrement, 5)
                print(f"Decrementing brightness to {new_value}")
                set_brightness(ser, new_value, allow_zero=False)
        
        finally:
            ser.close()
            print("Serial connection closed")
    else:
        parser.print_help()

if __name__ == '__main__':
    main()
```

3. Make the script executable:
```bash
sudo chmod +x /usr/local/bin/imacdisplay.py
```

## Usage

Basic commands:
```bash
# Get current brightness
imacdisplay.py -g

# Set brightness to 70%
imacdisplay.py -s 70

# Increase brightness by 10%
imacdisplay.py -i 10

# Decrease brightness by 10%
imacdisplay.py -d 10
```

## Setting Up Keyboard Shortcuts

1. Open Settings → Keyboard → Keyboard Shortcuts → Custom
2. Add two new shortcuts:
   - Increase Brightness:
     - Command: `/usr/local/bin/imacdisplay.py -i 10`
     - Shortcut: Ctrl + Shift + Up
   - Decrease Brightness:
     - Command: `/usr/local/bin/imacdisplay.py -d 10`
     - Shortcut: Ctrl + Shift + Down

## Safety Features

- Minimum brightness is limited to 5% to prevent complete blackout
- PWM values are properly mapped to ensure safe operation
- Configuration is saved to restore last brightness setting

## Notes

- The ESP32 must be connected via USB
- Fan control might need mbpfan package
- Initial brightness is set to 70%
- The script maintains a configuration file at `~/.config/imacdisplay.conf`

## Troubleshooting

1. If serial connection fails:
   - Check if ESP32 is properly connected
   - Verify correct port in `/dev/ttyUSB0`
   - Ensure user has proper permissions

2. If brightness doesn't change:
   - Verify PWM wire connection to GPIO23
   - Check if ESP32 LED blinks when receiving commands

3. If script fails:
   - Ensure python3-serial is installed
   - Check script permissions
   - Verify user has access to serial port
