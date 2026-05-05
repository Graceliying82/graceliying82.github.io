# Yahboom Raspbot V2 — Architecture, Hardware, and the Road to ROS2

> **Who this is for:** Who have the Raspbot V2 kit, have run a few Jupyter notebook demos, and want to understand exactly *what is happening* under the hood — and eventually take control without the notebook environment.

---

## Table of Contents

1. [System Architecture Overview](#1-system-architecture-overview)
2. [How Jupyter Notebooks and Raspbot_Lib Work Together](#2-how-jupyter-notebooks-and-raspbot_lib-work-together)
3. [Converting Notebooks to Pure Python Scripts](#3-converting-notebooks-to-pure-python-scripts)
4. [Complete Hardware Register Reference](#4-complete-hardware-register-reference)
5. [The Path to ROS2](#5-the-path-to-ros2)
6. [Writing Your Own STM32 Firmware](#6-writing-your-own-stm32-firmware)

---

## 1. System Architecture Overview

Before writing a single line of code, you need to understand the two-computer system inside your Raspbot. Yes — *two* computers.

```
┌─────────────────────────────────────────────────────────┐
│                  Your Python Code                        │
│         (Jupyter Notebook or .py script)                 │
└────────────────────────┬────────────────────────────────┘
                         │  calls
                         ▼
┌─────────────────────────────────────────────────────────┐
│                   Raspbot_Lib.py                         │
│           Python wrapper — pure software                 │
└────────────────────────┬────────────────────────────────┘
                         │  uses
                         ▼
┌─────────────────────────────────────────────────────────┐
│                    smbus library                         │
│       Linux kernel I2C driver (built into the OS)        │
└────────────────────────┬────────────────────────────────┘
                         │  sends bytes over
                         ▼
┌─────────────────────────────────────────────────────────┐
│              I2C Bus 1 (hardware wire)                   │
│          GPIO Pin 2 (SDA) + GPIO Pin 3 (SCL)            │
└────────────────────────┬────────────────────────────────┘
                         │  arrives at
                         ▼
┌─────────────────────────────────────────────────────────┐
│        Expansion Board Microcontroller (MCU)             │
│   I2C Address: 0x2B  (constant named PI5Car_I2CADDR)     │
│   Chip: STM32F103C8T6 or STM32F103RCT6                  │
│   (Pi 5 board revision — confirmed STM32, not STM8.      │
│    Earlier Raspbot used STM8 + TB6612 motor driver chip; │
│    Pi 5 board redesigned to STM32 for 3.3V I2C native    │
│    support and integrated motor/LED/sensor control.)      │
│                                                          │
│  This MCU is responsible for ALL real-time hardware:     │
│  • Generating PWM signals for 4 DC motors                │
│  • Generating PWM signals for 2 servo motors             │
│  • Driving 14 WS2812 RGB LEDs                            │
│  • Reading 4-channel infrared line tracking sensor       │
│  • Controlling the buzzer                                │
│  • Triggering the ultrasonic distance sensor             │
│  • Reading the IR remote receiver                        │
└─────────────────────────────────────────────────────────┘
```

### Why two computers?

The Raspberry Pi runs Linux. Linux is a general-purpose operating system — it is great at running Python, OpenCV, and neural networks, but it is *not* designed for precise real-time timing. Generating a PWM signal that controls a motor requires microsecond-level timing accuracy. If Linux decides to handle a network packet or write to a log file at the wrong moment, your motor would glitch.

The solution: offload all time-critical hardware control to a small dedicated microcontroller (MCU). The Pi sends high-level commands ("motor 0, go forward at speed 150"), and the MCU handles the precise PWM signals. This is a very common design pattern in robotics.

**Key takeaway:** The Raspberry Pi never directly controls a motor pin. It only sends short I2C messages to the MCU, which does the actual work.

---

## 2. How Jupyter Notebooks and Raspbot_Lib Work Together

### 2.1 What a Jupyter Notebook is

A Jupyter Notebook is a file (`.ipynb`) that contains a mix of text, documentation, and Python code broken into **cells**. You run one cell at a time by pressing `Shift+Enter`. This is useful for teaching because you can explain a concept, then immediately run a small piece of code to demonstrate it.

The downside: the code is split across cells that depend on shared state (variables created in earlier cells), and it requires a running Jupyter server to execute. It cannot be run directly with `python3`.

### 2.2 The role of `Raspbot_Lib.py`

`Raspbot_Lib.py` is the only file you need to understand to control the robot. It lives at:

```
Python driver library/py_install/Raspbot_Lib/Raspbot_Lib.py
```

It contains one main class: `Raspbot`. This class does two things:

1. **Opens an I2C connection** to the MCU when you call `Raspbot()`
2. **Provides named methods** that translate human-readable commands into the raw byte sequences the MCU understands

Every single method in `Raspbot` follows the same internal pattern:

```python
def Ctrl_BEEP_Switch(self, state):
    reg = 0x06              # which register (feature) on the MCU to target
    data = [state]          # the payload bytes to send
    self.write_array(reg, data)   # send it over I2C
```

And `write_array` itself is just:

```python
def write_array(self, reg, data):
    self._device.write_i2c_block_data(self._addr, reg, data)
    #                                  ↑            ↑    ↑
    #                              0x2B addr     register  payload
```

That is the entire "magic." There is no wireless connection, no middleware, no ROS. Just bytes over a wire.

---

### 2.3 Step-by-Step Walkthrough: The Buzzer Demo

Let's trace exactly what happens when you run the buzzer notebook, cell by cell.

**The notebook file:** `Source Code/source_code 2/project_demo/03.Basic_car_course/1.Buzzer driver.ipynb`

---

#### Cell 1 — Import

```python
from Raspbot_Lib import Raspbot
```

**What happens:**
- Python finds the `Raspbot_Lib` package (either installed system-wide or on `sys.path`)
- It loads `Raspbot_Lib.py` into memory
- The `Raspbot` class is now available in this notebook session
- No hardware is touched yet

---

#### Cell 2 — Create the robot object

```python
bot = Raspbot()
```

**What happens, line by line inside `__init__`:**

```python
def __init__(self):
    self._device = self.get_i2c_device(PI5Car_I2CADDR, 1)
```

```python
def get_i2c_device(self, address, i2c_bus):
    self._addr = address      # stores 0x2B for later use
    return smbus.SMBus(1)     # opens /dev/i2c-1 (the Linux I2C device file)
```

After this cell, `bot._device` is an open file handle to the I2C bus, and `bot._addr` is `0x2B`. The MCU is reachable. If I2C is not enabled on the Pi, this line throws an error.

---

#### Cell 3 — Turn buzzer ON

```python
bot.Ctrl_BEEP_Switch(1)
```

**Full call stack:**

```
bot.Ctrl_BEEP_Switch(1)
  └─ reg = 0x06
     data = [1]
     self.write_array(0x06, [1])
       └─ self._device.write_i2c_block_data(0x2B, 0x06, [1])
            └─ Linux sends on the wire:
               START | 0x2B (write) | 0x06 | 0x01 | STOP
```

The MCU receives this 3-byte message, sees register `0x06` with value `1`, and activates the buzzer circuit. The buzzer beeps.

**The actual bytes on the I2C wire (hex):**
```
S  57  06  01  P
↑   ↑   ↑   ↑  ↑
│   │   │   │  STOP condition
│   │   │   payload: state=1 (on)
│   │   register: buzzer
│   0x2B << 1 = 0x56, +write bit = 0x57
START condition
```

---

#### Cell 4 — Turn buzzer OFF

```python
bot.Ctrl_BEEP_Switch(0)
```

Identical call path, except the payload byte is `0` instead of `1`. MCU deactivates the buzzer.

---

#### Cell 5 — Release the object

```python
del bot
```

This closes the file handle to `/dev/i2c-1`, releasing the I2C bus so other programs can use it. **This is important.** If you skip this and try to run another notebook, you may get "I2C device busy" errors.

---

#### Summary diagram for the buzzer example

```
Notebook Cell          Raspbot_Lib          smbus          I2C Wire       MCU
─────────────          ───────────          ──────         ────────       ───
bot = Raspbot()   ──► __init__()       ──► SMBus(1)     (no traffic)   (waiting)
                       _addr = 0x2B        open i2c-1

Ctrl_BEEP_Switch(1) ► reg=0x06         ► write_i2c_    ► S|57|06|01|P ► buzzer ON
                       data=[1]             block_data

Ctrl_BEEP_Switch(0) ► reg=0x06         ► write_i2c_    ► S|57|06|00|P ► buzzer OFF
                       data=[0]             block_data

del bot           ──► close file handle    close i2c-1   (no traffic)   (maintains state)
```

---

## 3. Converting Notebooks to Pure Python Scripts

### 3.1 Why convert?

| Jupyter Notebook | Pure Python Script |
|---|---|
| Requires a running Jupyter server | Runs with just `python3 script.py` |
| Hard to run automatically on boot | Easy to add to `systemd` or `cron` |
| Cells run manually one at a time | Runs start to finish automatically |
| Interactive sliders (`ipywidgets`) | Replace with hardcoded values or `argparse` |
| Great for learning | Great for deployment |

### 3.2 The Conversion Rules

| Jupyter pattern | Pure Python equivalent |
|---|---|
| Multiple sequential cells | Merge top-to-bottom into one file |
| `from ipywidgets import interact` | Delete — not needed |
| `interact(func, slider=widgets.IntSlider(...))` | Call `func(value)` directly |
| `del bot` in last cell | Move into a `finally:` block |
| No `try/except` around the main loop | Wrap in `try/except KeyboardInterrupt` |

### 3.3 Setup on the Raspberry Pi

Before running any script, do this once on the Pi:

```bash
# 1. Enable I2C
sudo raspi-config
# → Interface Options → I2C → Yes → Finish → Reboot

# 2. Verify the MCU is visible
sudo i2cdetect -y 1
# You should see "2b" appear in the grid at address 0x2B

# 3. Install the smbus library if not present
pip3 install smbus2

# 4. Install Raspbot_Lib
cd ~/RaspbotV2-Code/Python\ driver\ library/py_install/
pip3 install .
# OR just add it to your path manually (see below)
```

If you don't want to install the library system-wide, add this at the top of every script:

```python
import sys
sys.path.append('/home/pi/RaspbotV2-Code/Python driver library/py_install')
```

---

### 3.4 Example 1: Buzzer (simple, no widgets)

**Original notebook cells:**
```python
# Cell 1
from Raspbot_Lib import Raspbot
# Cell 2
bot = Raspbot()
# Cell 3
bot.Ctrl_BEEP_Switch(1)
# Cell 4
bot.Ctrl_BEEP_Switch(0)
# Cell 5
del bot
```

**Converted pure Python script:**

```python
#!/usr/bin/env python3
"""
buzzer_demo.py — Beep the buzzer for 1 second.
Run on the Raspberry Pi: python3 buzzer_demo.py
"""
import time
from Raspbot_Lib import Raspbot

bot = Raspbot()
try:
    print("Buzzer ON")
    bot.Ctrl_BEEP_Switch(1)
    time.sleep(1)

    print("Buzzer OFF")
    bot.Ctrl_BEEP_Switch(0)
finally:
    del bot
    print("Done.")
```

**Why `try/finally`?** If your script crashes mid-run (e.g., a Python error), the `finally` block still executes, which calls `del bot` and releases the I2C bus. Without this, you'd have to reboot or kill the process to free the bus.

---

### 3.5 Example 2: Motor Control (replacing `ipywidgets`)

**Original notebook** uses interactive sliders. A student drags sliders to set each motor speed in real time.

```python
# Original notebook — ipywidgets version
interact(run_motor,
         M1=widgets.IntSlider(min=-255, max=255, step=1, value=0),
         M2=widgets.IntSlider(min=-255, max=255, step=1, value=0),
         M3=widgets.IntSlider(min=-255, max=255, step=1, value=0),
         M4=widgets.IntSlider(min=-255, max=255, step=1, value=0))
```

**Converted: hardcoded movement sequence**

```python
#!/usr/bin/env python3
"""
motor_demo.py — Drive the robot in a simple sequence.
Motor IDs: 0=Front-Left, 1=Rear-Left, 2=Front-Right, 3=Rear-Right
Speed range: -255 (full reverse) to +255 (full forward)
"""
import time
from Raspbot_Lib import Raspbot

bot = Raspbot()

def set_all_motors(speed):
    """Set all four motors to the same speed. Positive=forward, negative=reverse."""
    bot.Ctrl_Muto(0, speed)
    bot.Ctrl_Muto(1, speed)
    bot.Ctrl_Muto(2, speed)
    bot.Ctrl_Muto(3, speed)

def stop():
    set_all_motors(0)

try:
    print("Forward for 2 seconds...")
    set_all_motors(150)
    time.sleep(2)

    print("Stopping...")
    stop()
    time.sleep(0.5)

    print("Reverse for 1 second...")
    set_all_motors(-100)
    time.sleep(1)

    print("Stopping...")
    stop()

except KeyboardInterrupt:
    print("\nInterrupted by user.")
    stop()
finally:
    del bot
    print("Done.")
```

**What `Ctrl_Muto` does differently from `Ctrl_Car`:**

The notebook uses two motor control methods. Here is the difference:

```python
# Ctrl_Car: direction and speed are separate parameters
bot.Ctrl_Car(motor_id, direction, speed)
# direction: 0 = forward, 1 = reverse
# speed: 0–255
# Example: motor 0, forward, speed 150
bot.Ctrl_Car(0, 0, 150)

# Ctrl_Muto: single signed speed parameter (cleaner to use)
bot.Ctrl_Muto(motor_id, speed)
# speed: -255 to +255 (negative = reverse, positive = forward)
# Example: motor 0, forward at 150
bot.Ctrl_Muto(0, 150)
# Example: motor 0, reverse at 100
bot.Ctrl_Muto(0, -100)
```

Both send to register `0x01`. `Ctrl_Muto` is just a convenience wrapper that handles the sign conversion for you.

---

### 3.6 Example 3: Infrared Line Tracking (sensor reading loop)

This is a more complex example — it reads sensor data and makes decisions in a loop. The notebook was designed to run in Jupyter where you see `print()` output in real time.

```python
#!/usr/bin/env python3
"""
line_track.py — Read the 4-channel infrared line sensor and steer accordingly.
Sensor convention: 0 = black line detected, 1 = no line (white surface)
Layout: L1  L2  R1  R2  (left-outer, left-inner, right-inner, right-outer)
"""
import time
from Raspbot_Lib import Raspbot

bot = Raspbot()
SPEED = 80

def move_forward(speed):
    bot.Ctrl_Muto(0, speed)
    bot.Ctrl_Muto(1, speed)
    bot.Ctrl_Muto(2, speed)
    bot.Ctrl_Muto(3, speed)

def rotate_left(speed):
    bot.Ctrl_Muto(0, -speed)
    bot.Ctrl_Muto(1, -speed)
    bot.Ctrl_Muto(2,  speed)
    bot.Ctrl_Muto(3,  speed)

def rotate_right(speed):
    bot.Ctrl_Muto(0,  speed)
    bot.Ctrl_Muto(1,  speed)
    bot.Ctrl_Muto(2, -speed)
    bot.Ctrl_Muto(3, -speed)

def stop():
    for i in range(4):
        bot.Ctrl_Muto(i, 0)

def read_line_sensors():
    """
    Returns (L1, L2, R1, R2) as a tuple of 0/1 values.
    Register 0x0a returns one byte; the 4 sensor bits are packed into bits 3–0.
    """
    raw = bot.read_data_array(0x0a, 1)
    track = int(raw[0])
    L1 = (track >> 3) & 0x01   # bit 3
    L2 = (track >> 2) & 0x01   # bit 2
    R1 = (track >> 1) & 0x01   # bit 1
    R2 = (track >> 0) & 0x01   # bit 0
    return L1, L2, R1, R2

try:
    print("Line tracking started. Press Ctrl+C to stop.")
    while True:
        L1, L2, R1, R2 = read_line_sensors()

        if L1 == 0 and L2 == 0 and R1 == 0 and R2 == 0:
            # All sensors on black line — go straight fast
            move_forward(SPEED)

        elif L2 == 0 and R1 == 0:
            # Both center sensors on line — go straight
            move_forward(SPEED)

        elif L2 == 0 and R1 == 1:
            # Line drifting left — turn left
            rotate_left(SPEED)

        elif L2 == 1 and R1 == 0:
            # Line drifting right — turn right
            rotate_right(SPEED)

        elif L1 == 0:
            # Far left sensor triggered — sharp left turn
            rotate_left(int(SPEED * 1.5))
            time.sleep(0.15)

        elif R2 == 0:
            # Far right sensor triggered — sharp right turn
            rotate_right(SPEED)
            time.sleep(0.01)

        time.sleep(0.01)

except KeyboardInterrupt:
    print("\nStopping.")
    stop()
finally:
    del bot
    print("Done.")
```

---

### 3.7 Running Your Script on the Pi

```bash
# SSH into the Pi (replace with your Pi's IP address)
ssh pi@192.168.x.x

# Transfer your script from your Mac
scp motor_demo.py pi@192.168.x.x:~/

# On the Pi — run it
python3 motor_demo.py

# To run it automatically on boot, create a systemd service:
sudo nano /etc/systemd/system/raspbot.service
```

```ini
[Unit]
Description=Raspbot Line Tracker
After=multi-user.target

[Service]
ExecStart=/usr/bin/python3 /home/pi/line_track.py
WorkingDirectory=/home/pi
StandardOutput=journal
Restart=on-failure
User=pi

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable raspbot.service
sudo systemctl start raspbot.service
```

---

## 4. Complete Hardware Register Reference

This section is the technical reference for everything the MCU can do. Every command is one I2C write to address `0x2B` with a register number and a small payload.

### I2C Primer

I2C (Inter-Integrated Circuit) is a two-wire serial protocol:
- **SDA** (Serial Data) — the data wire
- **SCL** (Serial Clock) — the clock wire

A transaction looks like:
```
START → [device address + R/W bit] → [register] → [data bytes...] → STOP
```

The device address for the Raspbot MCU is `0x2B` (43 decimal).

In Python, every write is:
```python
smbus_device.write_i2c_block_data(0x2B, register, [byte1, byte2, ...])
```

And every read is:
```python
data = smbus_device.read_i2c_block_data(0x2B, register, num_bytes)
```

---

### Register 0x01 — Motor Control

**Direction:** Write only  
**Payload:** 3 bytes `[motor_id, direction, speed]`

| Parameter | Type | Range | Notes |
|---|---|---|---|
| `motor_id` | int | 0–3 | 0=Front-Left, 1=Rear-Left, 2=Front-Right, 3=Rear-Right |
| `direction` | int | 0 or 1 | 0 = forward, 1 = reverse |
| `speed` | int | 0–255 | 0 = stop, 255 = full speed |

**Python methods:**

```python
# Method 1: separate direction and speed
bot.Ctrl_Car(motor_id, direction, speed)

# Method 2: signed speed (negative = reverse) — easier to use
bot.Ctrl_Muto(motor_id, speed)   # speed: -255 to +255

# Examples
bot.Ctrl_Car(0, 0, 150)    # Front-Left motor, forward, speed 150
bot.Ctrl_Car(0, 1, 150)    # Front-Left motor, reverse, speed 150
bot.Ctrl_Car(0, 0, 0)      # Front-Left motor, stop

bot.Ctrl_Muto(0,  150)     # Front-Left forward at 150
bot.Ctrl_Muto(0, -150)     # Front-Left reverse at 150
bot.Ctrl_Muto(0, 0)        # Front-Left stop
```

**Raw I2C bytes:**
```python
# Forward all motors at speed 150
smbus_device.write_i2c_block_data(0x2B, 0x01, [0, 0, 150])  # Front-Left
smbus_device.write_i2c_block_data(0x2B, 0x01, [1, 0, 150])  # Rear-Left
smbus_device.write_i2c_block_data(0x2B, 0x01, [2, 0, 150])  # Front-Right
smbus_device.write_i2c_block_data(0x2B, 0x01, [3, 0, 150])  # Rear-Right
```

**Mecanum wheel motion patterns:**

The Raspbot uses Mecanum wheels, which allow diagonal and sideways movement. The trick is in which motors spin in which direction:

```
                    FORWARD
          ┌────────────────────────┐
          │  [0] FL ──►  ◄── FR [2]│
          │   wheels angled ╲  ╱   │
          │   wheels angled ╱  ╲   │
          │  [1] RL ──►  ◄── RR [3]│
          └────────────────────────┘

Movement    FL(0)   RL(1)   FR(2)   RR(3)
─────────   ─────   ─────   ─────   ─────
Forward      +       +       +       +
Backward     -       -       -       -
Rotate CW    +       +       -       -
Rotate CCW   -       -       +       +
Strafe Left  -       +       +       -
Strafe Right +       -       -       +
Diag F-Right +       +       +       +  (not all equal, adjust magnitudes)
```

---

### Register 0x02 — Servo Control

**Direction:** Write only  
**Payload:** 2 bytes `[servo_id, angle]`

| Parameter | Type | Range | Notes |
|---|---|---|---|
| `servo_id` | int | 1 or 2 | 1 = pan (horizontal), 2 = tilt (vertical) |
| `angle` | int | 0–180 | Degrees. Servo 2 is clamped to 0–110 by the library |

**Python method:**

```python
bot.Ctrl_Servo(servo_id, angle)

# Examples
bot.Ctrl_Servo(1, 90)    # Pan servo to center (90°)
bot.Ctrl_Servo(1, 0)     # Pan servo full left
bot.Ctrl_Servo(1, 180)   # Pan servo full right
bot.Ctrl_Servo(2, 45)    # Tilt servo down
bot.Ctrl_Servo(2, 90)    # Tilt servo level
# Note: bot.Ctrl_Servo(2, 180) will be clamped to 110 internally
```

**Raw I2C bytes:**
```python
# Pan to 90 degrees
smbus_device.write_i2c_block_data(0x2B, 0x02, [1, 90])
# Tilt to 60 degrees
smbus_device.write_i2c_block_data(0x2B, 0x02, [2, 60])
```

**Scanning pan sweep example:**
```python
import time
for angle in range(0, 181, 5):      # 0 to 180 in steps of 5
    bot.Ctrl_Servo(1, angle)
    time.sleep(0.05)
for angle in range(180, -1, -5):    # 180 back to 0
    bot.Ctrl_Servo(1, angle)
    time.sleep(0.05)
```

---

### Register 0x03 — All RGB LEDs (on/off + color index)

**Direction:** Write only  
**Payload:** 2 bytes `[state, color_index]`

The Raspbot has **14 WS2812 RGB LEDs**. This register controls all of them at once with a predefined color index.

| Parameter | Type | Range | Notes |
|---|---|---|---|
| `state` | int | 0 or 1 | 0 = all LEDs off, 1 = all LEDs on |
| `color_index` | int | 0–6 | See color table below |

**Color index table:**

| Index | Color |
|---|---|
| 0 | Red |
| 1 | Green |
| 2 | Blue |
| 3 | Yellow |
| 4 | Purple |
| 5 | Cyan |
| 6 | White |

**Python method:**

```python
bot.Ctrl_WQ2812_ALL(state, color_index)

# Examples
bot.Ctrl_WQ2812_ALL(1, 0)   # All LEDs on, Red
bot.Ctrl_WQ2812_ALL(1, 2)   # All LEDs on, Blue
bot.Ctrl_WQ2812_ALL(0, 0)   # All LEDs off (color_index ignored when state=0)
```

**Raw I2C bytes:**
```python
smbus_device.write_i2c_block_data(0x2B, 0x03, [1, 0])   # All on, Red
smbus_device.write_i2c_block_data(0x2B, 0x03, [0, 0])   # All off
```

**Cycling through colors example:**
```python
import time
for color in range(7):
    bot.Ctrl_WQ2812_ALL(1, color)
    time.sleep(0.5)
bot.Ctrl_WQ2812_ALL(0, 0)
```

---

### Register 0x04 — Individual RGB LED (on/off + color index)

**Direction:** Write only  
**Payload:** 3 bytes `[led_number, state, color_index]`

Controls a single LED out of the 14.

| Parameter | Type | Range | Notes |
|---|---|---|---|
| `led_number` | int | 0–13 | LED index (0-based) |
| `state` | int | 0 or 1 | 0 = off, 1 = on |
| `color_index` | int | 0–6 | Same color table as register 0x03 |

**Python method:**

```python
bot.Ctrl_WQ2812_Alone(led_number, state, color_index)

# Examples
bot.Ctrl_WQ2812_Alone(0, 1, 0)    # LED 0 on, Red
bot.Ctrl_WQ2812_Alone(5, 1, 2)    # LED 5 on, Blue
bot.Ctrl_WQ2812_Alone(0, 0, 0)    # LED 0 off
```

**Raw I2C bytes:**
```python
smbus_device.write_i2c_block_data(0x2B, 0x04, [0, 1, 0])   # LED 0 on, Red
smbus_device.write_i2c_block_data(0x2B, 0x04, [5, 0, 0])   # LED 5 off
```

---

### Register 0x05 — IR Remote Receiver Switch

**Direction:** Write only  
**Payload:** 1 byte `[state]`

Enables or disables the infrared remote receiver circuit. You must enable it before trying to read key values from register `0x0c`.

| Parameter | Type | Range | Notes |
|---|---|---|---|
| `state` | int | 0 or 1 | 1 = receiver on, 0 = receiver off |

**Python method:**

```python
bot.Ctrl_IR_Switch(state)

bot.Ctrl_IR_Switch(1)   # Turn IR receiver on
bot.Ctrl_IR_Switch(0)   # Turn IR receiver off
```

---

### Register 0x06 — Buzzer Switch

**Direction:** Write only  
**Payload:** 1 byte `[state]`

| Parameter | Type | Range | Notes |
|---|---|---|---|
| `state` | int | 0 or 1 | 1 = buzzer on, 0 = buzzer off |

**Python method:**

```python
bot.Ctrl_BEEP_Switch(state)

bot.Ctrl_BEEP_Switch(1)   # Buzzer on
bot.Ctrl_BEEP_Switch(0)   # Buzzer off
```

**Raw I2C bytes:**
```python
smbus_device.write_i2c_block_data(0x2B, 0x06, [1])   # on
smbus_device.write_i2c_block_data(0x2B, 0x06, [0])   # off
```

**Morse code SOS example:**
```python
import time

def beep(duration):
    bot.Ctrl_BEEP_Switch(1)
    time.sleep(duration)
    bot.Ctrl_BEEP_Switch(0)
    time.sleep(0.1)

dot, dash = 0.1, 0.3

# S O S
for _ in range(3): beep(dot)
for _ in range(3): beep(dash)
for _ in range(3): beep(dot)
```

---

### Register 0x07 — Ultrasonic Sensor Switch

**Direction:** Write only  
**Payload:** 1 byte `[state]`

Enables or disables the HC-SR04 ultrasonic distance sensor. After enabling it, wait at least 1 second before reading, to let the sensor stabilize.

| Parameter | Type | Range | Notes |
|---|---|---|---|
| `state` | int | 0 or 1 | 1 = sensor on, 0 = sensor off |

**Python method:**

```python
bot.Ctrl_Ulatist_Switch(state)

bot.Ctrl_Ulatist_Switch(1)   # Turn ultrasonic on
bot.Ctrl_Ulatist_Switch(0)   # Turn ultrasonic off
```

---

### Register 0x08 — All LEDs Brightness (Raw RGB)

**Direction:** Write only  
**Payload:** 3 bytes `[R, G, B]`

Sets all 14 LEDs to the exact RGB color you specify. This is more powerful than register `0x03` because you mix any color, not just 7 presets.

| Parameter | Type | Range | Notes |
|---|---|---|---|
| `R` | int | 0–255 | Red channel brightness |
| `G` | int | 0–255 | Green channel brightness |
| `B` | int | 0–255 | Blue channel brightness |

**Python method:**

```python
bot.Ctrl_WQ2812_brightness_ALL(R, G, B)

# Examples
bot.Ctrl_WQ2812_brightness_ALL(255, 0, 0)       # Pure red
bot.Ctrl_WQ2812_brightness_ALL(0, 255, 0)       # Pure green
bot.Ctrl_WQ2812_brightness_ALL(128, 0, 128)     # Dim purple
bot.Ctrl_WQ2812_brightness_ALL(0, 0, 0)         # Off
```

**Breathing light effect example:**

```python
import time

# Fade red in and out
for brightness in range(0, 256, 2):
    bot.Ctrl_WQ2812_brightness_ALL(brightness, 0, 0)
    time.sleep(0.01)
for brightness in range(255, -1, -2):
    bot.Ctrl_WQ2812_brightness_ALL(brightness, 0, 0)
    time.sleep(0.01)

bot.Ctrl_WQ2812_brightness_ALL(0, 0, 0)
```

---

### Register 0x09 — Individual LED Brightness (Raw RGB)

**Direction:** Write only  
**Payload:** 4 bytes `[led_number, R, G, B]`

Like register `0x08` but targets a single LED.

| Parameter | Type | Range | Notes |
|---|---|---|---|
| `led_number` | int | 0–13 | LED index |
| `R` | int | 0–255 | Red |
| `G` | int | 0–255 | Green |
| `B` | int | 0–255 | Blue |

**Python method:**

```python
bot.Ctrl_WQ2812_brightness_Alone(led_number, R, G, B)

# Examples
bot.Ctrl_WQ2812_brightness_Alone(0,  255, 0,   0)    # LED 0: red
bot.Ctrl_WQ2812_brightness_Alone(1,  0,   255, 0)    # LED 1: green
bot.Ctrl_WQ2812_brightness_Alone(2,  0,   0,   255)  # LED 2: blue
```

**Rainbow effect example:**
```python
import time, math

for t in range(200):
    for i in range(14):
        phase = (t + i * 10) * 0.05
        r = int((math.sin(phase + 0)         + 1) * 127)
        g = int((math.sin(phase + 2.094)     + 1) * 127)  # 120° offset
        b = int((math.sin(phase + 4.189)     + 1) * 127)  # 240° offset
        bot.Ctrl_WQ2812_brightness_Alone(i, r, g, b)
    time.sleep(0.05)
bot.Ctrl_WQ2812_brightness_ALL(0, 0, 0)
```

---

### Register 0x0a — Line Tracking Sensor (READ)

**Direction:** Read only  
**Read length:** 1 byte

The 4-channel infrared line tracking sensor reads the surface below the robot. Each channel outputs `0` if it detects a dark/black surface, or `1` if it detects a light/white surface. All four channels are packed into one byte.

**Bit layout of the returned byte:**

```
Bit:    7   6   5   4   3   2   1   0
Value:  0   0   0   0  [L1][L2][R1][R2]
                        ↑   ↑   ↑   ↑
                        │   │   │   └── Right-outer sensor
                        │   │   └────── Right-inner sensor
                        │   └────────── Left-inner sensor
                        └────────────── Left-outer sensor
```

**Python usage:**

```python
raw = bot.read_data_array(0x0a, 1)   # returns a list of 1 byte
track = int(raw[0])

L1 = (track >> 3) & 0x01   # Left-outer
L2 = (track >> 2) & 0x01   # Left-inner
R1 = (track >> 1) & 0x01   # Right-inner
R2 = (track >> 0) & 0x01   # Right-outer

print(f"L1={L1} L2={L2} R1={R1} R2={R2}")
# 0 = black line detected under that sensor
# 1 = white surface, no line
```

**Raw I2C read:**
```python
data = smbus_device.read_i2c_block_data(0x2B, 0x0a, 1)
track_byte = data[0]
```

**Decoding all 16 possible sensor states:**

```python
state_names = {
    (0,0,0,0): "All on line — go straight",
    (0,0,0,1): "L1,L2,R1 on — turn right",
    (0,0,1,0): "L1,L2,R2 on — turn right",
    (0,0,1,1): "L1,L2 on — turn right",
    (0,1,0,0): "L1,R1,R2 on — turn left",
    (0,1,0,1): "L1,R1 on — straight",
    (0,1,1,0): "L1,R2 on — slight right",
    (0,1,1,1): "L1 only — sharp left",
    (1,0,0,0): "L2,R1,R2 on — turn left",
    (1,0,0,1): "L2,R1 on — slight left",
    (1,0,1,0): "L2,R2 on — slight right",
    (1,0,1,1): "L2 only — turn left",
    (1,1,0,0): "R1,R2 on — sharp right",
    (1,1,0,1): "R1 only — turn right",
    (1,1,1,0): "R2 only — sharp right",
    (1,1,1,1): "No line detected — stop or search",
}
```

---

### Registers 0x1a / 0x1b — Ultrasonic Distance (READ)

**Direction:** Read only  
**Read length:** 1 byte each  
**Units:** millimeters

The ultrasonic sensor measures distances from ~2 cm to ~400 cm. The result is a 16-bit value in millimeters, split across two registers.

| Register | Description |
|---|---|
| `0x1a` | Low byte of distance |
| `0x1b` | High byte of distance |

**Python usage:**

```python
import time

# Step 1: turn on the sensor
bot.Ctrl_Ulatist_Switch(1)
time.sleep(1)   # wait for sensor to stabilize

# Step 2: read the two bytes
low_byte  = bot.read_data_array(0x1a, 1)[0]
high_byte = bot.read_data_array(0x1b, 1)[0]

# Step 3: combine into a 16-bit distance value
distance_mm = (high_byte << 8) | low_byte

print(f"Distance: {distance_mm} mm  ({distance_mm / 10:.1f} cm)")

# Step 4: turn off when done
bot.Ctrl_Ulatist_Switch(0)
```

**Obstacle avoidance loop example:**

```python
import time

bot.Ctrl_Ulatist_Switch(1)
time.sleep(1)

STOP_DISTANCE_MM = 200   # stop if closer than 20 cm

try:
    while True:
        low  = bot.read_data_array(0x1a, 1)[0]
        high = bot.read_data_array(0x1b, 1)[0]
        dist = (high << 8) | low

        print(f"Distance: {dist} mm")

        if dist < STOP_DISTANCE_MM:
            # Too close — stop and back up
            for i in range(4):
                bot.Ctrl_Muto(i, -80)
            time.sleep(0.5)
            for i in range(4):
                bot.Ctrl_Muto(i, 0)
        else:
            # Clear path — move forward
            for i in range(4):
                bot.Ctrl_Muto(i, 100)

        time.sleep(0.05)

except KeyboardInterrupt:
    for i in range(4):
        bot.Ctrl_Muto(i, 0)
    bot.Ctrl_Ulatist_Switch(0)
```

---

### Register 0x0c — IR Remote Key Value (READ)

**Direction:** Read only  
**Read length:** 1 byte  
**Prerequisite:** Must call `bot.Ctrl_IR_Switch(1)` first

After enabling the IR receiver and pressing a button on the remote, the MCU stores the received key code in register `0x0c`.

**Python usage:**

```python
import time

bot.Ctrl_IR_Switch(1)
time.sleep(0.5)

print("Press a button on the remote...")

prev_val = -1
while True:
    data = bot.read_data_array(0x0c, 1)
    key = int(data[0])
    if key != prev_val and key != 0:
        print(f"Key code received: {key} (hex: {hex(key)})")
        prev_val = key
    time.sleep(0.1)
```

Run this once to discover the key codes for every button on your specific remote control. Write them down — they vary by remote model.

---

### Complete Register Summary Table

| Register | R/W | Method | Payload | Description |
|---|---|---|---|---|
| `0x01` | W | `Ctrl_Car(id, dir, spd)` / `Ctrl_Muto(id, spd)` | 3 bytes | Motor control |
| `0x02` | W | `Ctrl_Servo(id, angle)` | 2 bytes | Servo position |
| `0x03` | W | `Ctrl_WQ2812_ALL(state, color)` | 2 bytes | All LEDs on/off with color index |
| `0x04` | W | `Ctrl_WQ2812_Alone(n, state, color)` | 3 bytes | Single LED on/off |
| `0x05` | W | `Ctrl_IR_Switch(state)` | 1 byte | IR receiver enable |
| `0x06` | W | `Ctrl_BEEP_Switch(state)` | 1 byte | Buzzer on/off |
| `0x07` | W | `Ctrl_Ulatist_Switch(state)` | 1 byte | Ultrasonic sensor enable |
| `0x08` | W | `Ctrl_WQ2812_brightness_ALL(R,G,B)` | 3 bytes | All LEDs raw RGB |
| `0x09` | W | `Ctrl_WQ2812_brightness_Alone(n,R,G,B)` | 4 bytes | Single LED raw RGB |
| `0x0a` | R | `read_data_array(0x0a, 1)` | — | Line tracking sensors (4-bit packed) |
| `0x0c` | R | `read_data_array(0x0c, 1)` | — | IR remote key code |
| `0x1a` | R | `read_data_array(0x1a, 1)` | — | Ultrasonic distance low byte |
| `0x1b` | R | `read_data_array(0x1b, 1)` | — | Ultrasonic distance high byte |

---

## 5. The Path to ROS2

### 5.1 What is ROS2 and why would you want it?

ROS2 (Robot Operating System 2) is not an operating system — it is a middleware framework for building robot software. It provides:

- **Topics:** publish/subscribe messaging between programs running in separate processes
- **Services:** request/response calls
- **Actions:** long-running tasks with feedback (like "navigate to waypoint")
- **Standard message types:** so a camera driver, navigation planner, and joystick controller can all talk to each other without custom code
- **Visualization tools:** RViz2 for seeing sensor data and robot state in 3D
- **Navigation stack (Nav2):** built-in autonomous navigation, path planning, and obstacle avoidance

**Current architecture (no ROS2):**

```
your_script.py  ──►  Raspbot_Lib  ──►  I2C  ──►  MCU
```

Everything lives in one Python process. To add a web controller, a joystick, a camera stream, and obstacle avoidance, you'd have to write it all in one monolithic program.

**ROS2 architecture:**

```
┌──────────────┐    topic:/cmd_vel    ┌──────────────────┐
│  teleop_node │ ──────────────────►  │  raspbot_driver  │
│  (keyboard)  │                      │  (your I2C code) │ ──► MCU
└──────────────┘                      └──────────────────┘
                                               ▲
┌──────────────┐    topic:/cmd_vel             │
│  nav2_stack  │ ──────────────────────────────┘
│  (autopilot) │
└──────────────┘

┌──────────────┐    topic:/scan
│  ultrasonic  │ ──────────────────► nav2_stack
│  _publisher  │
└──────────────┘

┌──────────────┐    topic:/camera/image_raw
│  camera_node │ ──────────────────► object_detector_node
└──────────────┘
```

Each component is an independent process. You can swap, restart, or replace any one of them without touching the others. This is the power of ROS2.

---

### 5.2 Prerequisites Before Starting

Install ROS2 Humble on your Raspberry Pi (Ubuntu 22.04 required):

```bash
# On the Pi — this takes ~30 minutes
sudo apt update && sudo apt install -y locales
sudo locale-gen en_US en_US.UTF-8
sudo update-locale LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8

sudo apt install -y software-properties-common
sudo add-apt-repository universe

sudo apt update && sudo apt install -y curl
sudo curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key \
  -o /usr/share/keyrings/ros-archive-keyring.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] \
  http://packages.ros.org/ros2/ubuntu $(. /etc/os-release && echo $UBUNTU_CODENAME) main" \
  | sudo tee /etc/apt/sources.list.d/ros2.list

sudo apt update
sudo apt install -y ros-humble-ros-base python3-colcon-common-extensions
```

---

### 5.3 Core ROS2 Concepts You Need to Know

Before writing any code, understand these four concepts:

**Node** — a single program that participates in the ROS2 network. Your motor driver will be one node. A joystick reader will be another node.

**Topic** — a named data channel. Any node can publish to it; any node can subscribe. No direct connections between nodes.

**Message type** — the data format on a topic. `geometry_msgs/Twist` is the standard type for robot velocity commands. It has `linear.x` (forward/backward) and `angular.z` (turn left/right).

**`rclpy`** — the Python library for writing ROS2 nodes. Import it exactly like any other Python library.

---

### 5.4 Step 1 — The Motor Driver Node

This is the most important node. It subscribes to the standard `/cmd_vel` topic and translates the velocity commands into motor I2C calls.

```python
#!/usr/bin/env python3
"""
raspbot_driver_node.py

Subscribes to /cmd_vel (geometry_msgs/Twist) and drives
the Raspbot's four Mecanum wheels via I2C.

Run:  ros2 run raspbot_bringup driver_node
"""
import rclpy
from rclpy.node import Node
from geometry_msgs.msg import Twist
from Raspbot_Lib import Raspbot

MAX_SPEED = 200   # maximum motor speed (0–255)

class RaspbotDriverNode(Node):

    def __init__(self):
        super().__init__('raspbot_driver')
        self.bot = Raspbot()
        self.subscription = self.create_subscription(
            Twist,
            '/cmd_vel',
            self.cmd_vel_callback,
            10)   # queue size
        self.get_logger().info('Raspbot driver node started.')

    def cmd_vel_callback(self, msg: Twist):
        """
        Received a Twist message. Convert to Mecanum wheel speeds.

        msg.linear.x   — forward (+) / backward (-),  range -1.0 to 1.0
        msg.linear.y   — strafe right (+) / left (-), range -1.0 to 1.0
        msg.angular.z  — turn left (+) / right (-),   range -1.0 to 1.0
        """
        vx = msg.linear.x    # forward/backward
        vy = msg.linear.y    # strafe
        wz = msg.angular.z   # rotation

        # Mecanum wheel mixing equations
        # Each wheel gets a combination of all three motion components
        fl = vx - vy - wz   # Front-Left  (motor 0)
        rl = vx + vy - wz   # Rear-Left   (motor 1)
        fr = vx + vy + wz   # Front-Right (motor 2)
        rr = vx - vy + wz   # Rear-Right  (motor 3)

        # Scale to motor range and clip
        def scale(v):
            return int(max(-255, min(255, v * MAX_SPEED)))

        self.bot.Ctrl_Muto(0, scale(fl))
        self.bot.Ctrl_Muto(1, scale(rl))
        self.bot.Ctrl_Muto(2, scale(fr))
        self.bot.Ctrl_Muto(3, scale(rr))

    def destroy_node(self):
        # Stop all motors before shutting down
        for i in range(4):
            self.bot.Ctrl_Muto(i, 0)
        del self.bot
        super().destroy_node()


def main():
    rclpy.init()
    node = RaspbotDriverNode()
    try:
        rclpy.spin(node)
    except KeyboardInterrupt:
        pass
    finally:
        node.destroy_node()
        rclpy.shutdown()

if __name__ == '__main__':
    main()
```

---

### 5.5 Step 2 — The Ultrasonic Sensor Publisher Node

This node periodically reads the ultrasonic sensor and publishes the distance on a topic, so any other node (like a navigation planner) can use it.

```python
#!/usr/bin/env python3
"""
ultrasonic_node.py

Publishes ultrasonic distance readings to /ultrasonic/range
as a sensor_msgs/Range message.

Run:  ros2 run raspbot_bringup ultrasonic_node
"""
import rclpy
from rclpy.node import Node
from sensor_msgs.msg import Range
import time
from Raspbot_Lib import Raspbot

class UltrasonicNode(Node):

    def __init__(self):
        super().__init__('ultrasonic_sensor')
        self.bot = Raspbot()
        self.bot.Ctrl_Ulatist_Switch(1)
        time.sleep(1)   # let sensor warm up

        self.publisher = self.create_publisher(Range, '/ultrasonic/range', 10)
        # Read and publish at 10 Hz
        self.timer = self.create_timer(0.1, self.publish_distance)
        self.get_logger().info('Ultrasonic node started.')

    def publish_distance(self):
        low  = self.bot.read_data_array(0x1a, 1)[0]
        high = self.bot.read_data_array(0x1b, 1)[0]
        dist_mm = (high << 8) | low
        dist_m  = dist_mm / 1000.0

        msg = Range()
        msg.header.stamp    = self.get_clock().now().to_msg()
        msg.header.frame_id = 'ultrasonic_sensor'
        msg.radiation_type  = Range.ULTRASOUND
        msg.field_of_view   = 0.26   # ~15 degrees in radians
        msg.min_range       = 0.02   # 2 cm
        msg.max_range       = 4.00   # 400 cm
        msg.range           = dist_m

        self.publisher.publish(msg)

    def destroy_node(self):
        self.bot.Ctrl_Ulatist_Switch(0)
        del self.bot
        super().destroy_node()


def main():
    rclpy.init()
    node = UltrasonicNode()
    try:
        rclpy.spin(node)
    except KeyboardInterrupt:
        pass
    finally:
        node.destroy_node()
        rclpy.shutdown()

if __name__ == '__main__':
    main()
```

---

### 5.6 Step 3 — The Line Tracking Publisher Node

```python
#!/usr/bin/env python3
"""
line_track_node.py

Publishes the 4-channel line sensor state to /line_sensors
as a std_msgs/Int8MultiArray message.

Index:  [0]=L1, [1]=L2, [2]=R1, [3]=R2
Value:  0 = black line detected, 1 = white surface
"""
import rclpy
from rclpy.node import Node
from std_msgs.msg import Int8MultiArray
from Raspbot_Lib import Raspbot

class LineTrackNode(Node):

    def __init__(self):
        super().__init__('line_track_sensor')
        self.bot = Raspbot()
        self.publisher = self.create_publisher(Int8MultiArray, '/line_sensors', 10)
        self.timer = self.create_timer(0.02, self.publish_sensors)  # 50 Hz
        self.get_logger().info('Line tracking node started.')

    def publish_sensors(self):
        raw   = self.bot.read_data_array(0x0a, 1)
        track = int(raw[0])

        msg = Int8MultiArray()
        msg.data = [
            (track >> 3) & 0x01,   # L1
            (track >> 2) & 0x01,   # L2
            (track >> 1) & 0x01,   # R1
            (track >> 0) & 0x01,   # R2
        ]
        self.publisher.publish(msg)

    def destroy_node(self):
        del self.bot
        super().destroy_node()


def main():
    rclpy.init()
    node = LineTrackNode()
    try:
        rclpy.spin(node)
    except KeyboardInterrupt:
        pass
    finally:
        node.destroy_node()
        rclpy.shutdown()

if __name__ == '__main__':
    main()
```

---

### 5.7 Step 4 — Package Structure

ROS2 uses Python packages for organizing nodes. Here is the directory layout:

```
~/ros2_ws/                          ← your ROS2 workspace
└── src/
    └── raspbot_bringup/            ← your package
        ├── package.xml             ← package metadata
        ├── setup.py                ← tells colcon how to install
        ├── setup.cfg
        └── raspbot_bringup/
            ├── __init__.py
            ├── driver_node.py      ← motor driver
            ├── ultrasonic_node.py  ← ultrasonic publisher
            └── line_track_node.py  ← line sensor publisher
```

**`package.xml`** (minimum required):
```xml
<?xml version="1.0"?>
<package format="3">
  <name>raspbot_bringup</name>
  <version>0.0.1</version>
  <description>Raspbot V2 ROS2 driver nodes</description>
  <maintainer email="you@example.com">Your Name</maintainer>
  <license>MIT</license>
  <exec_depend>rclpy</exec_depend>
  <exec_depend>geometry_msgs</exec_depend>
  <exec_depend>sensor_msgs</exec_depend>
  <exec_depend>std_msgs</exec_depend>
</package>
```

**`setup.py`**:
```python
from setuptools import setup

setup(
    name='raspbot_bringup',
    version='0.0.1',
    packages=['raspbot_bringup'],
    install_requires=['setuptools'],
    entry_points={
        'console_scripts': [
            'driver_node      = raspbot_bringup.driver_node:main',
            'ultrasonic_node  = raspbot_bringup.ultrasonic_node:main',
            'line_track_node  = raspbot_bringup.line_track_node:main',
        ],
    },
)
```

**Build and run:**

```bash
# Source ROS2
source /opt/ros/humble/setup.bash

# Build your package
cd ~/ros2_ws
colcon build --packages-select raspbot_bringup

# Source your workspace
source install/setup.bash

# Run the motor driver in one terminal
ros2 run raspbot_bringup driver_node

# Run the ultrasonic sensor in another terminal
ros2 run raspbot_bringup ultrasonic_node

# From your laptop (not the Pi), drive the robot with keyboard
ros2 run teleop_twist_keyboard teleop_twist_keyboard

# See what topics are active
ros2 topic list

# Spy on the ultrasonic readings
ros2 topic echo /ultrasonic/range

# Spy on line sensors
ros2 topic echo /line_sensors
```

---

### 5.8 The Migration Roadmap

Here is a suggested order for migrating your Raspbot to full ROS2:

| Phase | What to build | ROS2 concept learned |
|---|---|---|
| 1 | `driver_node.py` — subscribe to `/cmd_vel`, drive motors | Subscriber, Twist message |
| 2 | `ultrasonic_node.py` — publish distance to `/ultrasonic/range` | Publisher, timer, Range message |
| 3 | `line_track_node.py` — publish sensors to `/line_sensors` | Publisher, custom logic |
| 4 | `buzzer_node.py` — expose buzzer as a ROS2 service | Service server |
| 5 | `servo_node.py` — expose pan/tilt as action server with position feedback | Action server |
| 6 | Add `camera_node` using `image_transport` | Camera streaming |
| 7 | Add `nav2` for autonomous navigation with obstacle avoidance | Full navigation stack |

---

### 5.9 Final Architecture Comparison

**Before ROS2 (what you have now):**
```
one_big_script.py
├── reads sensors via Raspbot_Lib
├── processes camera via OpenCV
├── decides what to do
└── drives motors via Raspbot_Lib
```

**After ROS2:**
```
driver_node        ← reads /cmd_vel, writes I2C
ultrasonic_node    ← reads I2C, publishes /ultrasonic/range
line_track_node    ← reads I2C, publishes /line_sensors
camera_node        ← reads camera, publishes /camera/image_raw
nav2               ← reads sensors, publishes /cmd_vel
teleop_node        ← reads keyboard, publishes /cmd_vel
```

Every component is replaceable. To switch from keyboard control to autonomous navigation, you just start `nav2` instead of `teleop_node` — the `driver_node` doesn't change at all.

---

## Quick Reference Card

```
# Motor IDs
0 = Front-Left   1 = Rear-Left   2 = Front-Right   3 = Rear-Right

# Motor control
bot.Ctrl_Muto(motor_id, speed)       # speed: -255 to +255
bot.Ctrl_Car(motor_id, dir, speed)   # dir: 0=fwd, 1=rev; speed: 0-255

# Servos
bot.Ctrl_Servo(servo_id, angle)      # id: 1=pan, 2=tilt; angle: 0-180

# Buzzer
bot.Ctrl_BEEP_Switch(1)              # on
bot.Ctrl_BEEP_Switch(0)              # off

# RGB LEDs (all, color index 0-6)
bot.Ctrl_WQ2812_ALL(1, color_index)  # on
bot.Ctrl_WQ2812_ALL(0, 0)            # off

# RGB LEDs (all, raw color)
bot.Ctrl_WQ2812_brightness_ALL(R, G, B)

# RGB LEDs (single, color index)
bot.Ctrl_WQ2812_Alone(led_num, 1, color_index)

# RGB LEDs (single, raw color)
bot.Ctrl_WQ2812_brightness_Alone(led_num, R, G, B)

# Ultrasonic (enable → wait → read)
bot.Ctrl_Ulatist_Switch(1)
time.sleep(1)
lo = bot.read_data_array(0x1a, 1)[0]
hi = bot.read_data_array(0x1b, 1)[0]
dist_mm = (hi << 8) | lo

# Line tracking sensors
raw  = bot.read_data_array(0x0a, 1)
byte = int(raw[0])
L1 = (byte >> 3) & 1   # 0=black, 1=white
L2 = (byte >> 2) & 1
R1 = (byte >> 1) & 1
R2 = (byte >> 0) & 1

# IR remote
bot.Ctrl_IR_Switch(1)
key = bot.read_data_array(0x0c, 1)[0]
```

---

## 6. Writing Your Own STM32 Firmware

This section is for when you are ready to go beyond Yahboom's closed firmware and program the STM32 yourself. You will learn real embedded systems development: peripherals, interrupts, I2C slave mode, and PWM — using the actual hardware on your robot as the target.

> **Warning:** Flashing new firmware overwrites Yahboom's firmware permanently. Before you start, check whether Yahboom provides a `.hex` firmware file you can use to restore the original if needed. Look in the tutorial materials or ask their support at support@yahboom.com.

---

### 6.1 What You Need

#### Hardware

**An ST-Link V2 programmer** — the standard tool for programming any STM32. Clones cost around $8–12 on Amazon or AliExpress and work perfectly.

```
ST-Link V2 pin    →    Raspbot expansion board pad
─────────────────────────────────────────────────
SWDIO             →    SWDIO  (look for this label on PCB)
SWDCLK            →    SWDCLK
GND               →    GND
3.3V              →    3.3V   ← CRITICAL: never use 5V on Pi 5 board
```

SWD (Serial Wire Debug) is a 2-wire interface built into every STM32. It gives you:
- Full chip erase and firmware flash
- Live in-circuit debugging (breakpoints, variable inspection, step through code)
- No BOOT0 pin manipulation needed

Finding the SWD pads: they are usually a row of 4 small test points or a 4-pin header somewhere on the expansion board, often near the STM32 chip itself. Look for labels `SWDIO`, `SWDCLK`, `3V3`, `GND` — or sometimes `DIO`, `CLK`, `VCC`, `GND`.

#### Software

Install **STM32CubeIDE** — free from ST Microelectronics (st.com). It includes:
- The full HAL (Hardware Abstraction Layer) driver library
- A graphical pin configuration tool (CubeMX, built in)
- ST-Link GDB server for flashing and debugging
- An Eclipse-based IDE

Alternatively, **PlatformIO** inside VS Code works well and is less heavyweight.

---

### 6.2 STM32 Concepts You Must Understand First

Before writing any code, you need to understand four things that are different from Python programming.

#### Concept 1: There is no operating system

Your C code runs directly on the hardware. There is no Linux, no file system, no `print()`. The program starts at `main()` and runs in an infinite loop forever. If it crashes, the chip freezes until you reset it.

```c
int main(void) {
    HAL_Init();           // initialize the HAL library
    SystemClock_Config(); // set up the CPU clock speed
    MX_GPIO_Init();       // configure GPIO pins
    MX_I2C1_Init();       // configure I2C peripheral

    while (1) {           // infinite loop — this runs forever
        // your robot logic here
    }
}
```

#### Concept 2: Peripherals are memory-mapped registers

Every hardware feature of the STM32 (GPIO, I2C, timers, PWM) is controlled by writing to specific memory addresses called **registers**. The HAL library provides functions so you do not have to write raw addresses — but understanding that `HAL_GPIO_WritePin()` is ultimately just a memory write is important.

#### Concept 3: Interrupts

An interrupt is a hardware signal that temporarily pauses your main loop to run a specific function (the **callback**), then returns. For example:

- When an I2C message arrives → `HAL_I2C_SlaveRxCpltCallback()` is called
- When a timer fires → `HAL_TIM_PeriodElapsedCallback()` is called

You do not poll for these events. The hardware calls your function automatically.

#### Concept 4: HAL naming convention

STM32 HAL functions follow a consistent pattern:

```
HAL_[Peripheral]_[Action]_[Mode]()

HAL_GPIO_WritePin()        → GPIO peripheral, write action
HAL_I2C_Slave_Receive_IT() → I2C peripheral, slave receive, interrupt mode
HAL_TIM_PWM_Start()        → Timer peripheral, PWM action, start
```

`_IT` suffix = interrupt-driven (non-blocking)
`_DMA` suffix = DMA-driven (hardware memory transfer, even faster)
No suffix = blocking (waits until done)

---

### 6.3 Phase 1: Blink an LED (Prove Your Toolchain Works)

Before touching I2C or motors, confirm that your ST-Link is wired correctly and your toolchain can flash code.

**In STM32CubeIDE:**
1. File → New → STM32 Project
2. Search for your chip: `STM32F103C8` (if C8T6) or `STM32F103RC` (if RCT6)
3. In the CubeMX pin view, click any free GPIO pin → set to `GPIO_Output`
4. Project → Generate Code
5. Open `Core/Src/main.c`

**Code to add inside the `while(1)` loop:**

```c
/* USER CODE BEGIN WHILE */
while (1) {
    HAL_GPIO_TogglePin(GPIOC, GPIO_PIN_13);  // toggle the onboard LED
    HAL_Delay(500);                           // wait 500 ms
}
/* USER CODE END WHILE */
```

**Flash it:**
- Click the green Run button (or press F11)
- STM32CubeIDE automatically compiles, connects to ST-Link, erases the chip, and flashes

If an LED blinks every half second, your toolchain is working. If it fails here, fix the ST-Link wiring before going further.

---

### 6.4 Phase 2: Implement the I2C Slave

This is the most important phase. You will recreate the same I2C slave interface that Yahboom's firmware provides, so your existing Python scripts still work without any changes.

#### How STM32 I2C slave mode works

The STM32 I2C peripheral operates in interrupt mode. The sequence for a write from the Pi is:

```
Pi sends: START | 0x2B (write) | register | data bytes | STOP

STM32 callbacks fire in order:
1. HAL_I2C_AddrCallback()       ← Pi addressed us
2. HAL_I2C_SlaveRxCpltCallback() ← Pi finished sending data
```

For a read from the Pi:

```
Pi sends: START | 0x2B (write) | register | REPEATED START | 0x2B (read)

STM32 callbacks fire:
1. HAL_I2C_AddrCallback()       ← Pi addressed us for write (send register)
2. HAL_I2C_SlaveRxCpltCallback() ← register byte received
3. HAL_I2C_AddrCallback()       ← Pi addressed us for read
4. HAL_I2C_SlaveTxCpltCallback() ← we finished sending data back
```

#### CubeMX configuration

In the CubeMX pin view:
- Enable **I2C1** (or I2C2, depending on which pins connect to the Pi)
- Set Mode: `I2C`
- In the I2C parameters: set **Own Address 1** to `0x56` (that is `0x2B << 1` — STM32 uses 8-bit address format internally)
- Enable the I2C global interrupt in NVIC settings

#### The I2C slave implementation

Create a new file `Core/Src/i2c_slave.c`:

```c
#include "main.h"
#include "i2c_slave.h"
#include <string.h>

/* ── register storage ───────────────────────────────────────────── */
#define REG_MOTOR      0x01
#define REG_SERVO      0x02
#define REG_LED_ALL    0x03
#define REG_LED_ALONE  0x04
#define REG_IR_SW      0x05
#define REG_BUZZER     0x06
#define REG_ULTRASONIC 0x07
#define REG_LED_RGB_ALL   0x08
#define REG_LED_RGB_ALONE 0x09
#define REG_LINE_TRACK 0x0A
#define REG_IR_KEY     0x0C
#define REG_DIST_LO    0x1A
#define REG_DIST_HI    0x1B

/* Receive buffer: register byte + up to 4 data bytes */
static uint8_t rx_buf[5];
static uint8_t rx_len = 0;
static uint8_t current_reg = 0;

/* Transmit buffer for reads */
static uint8_t tx_buf[4];

/* Public sensor data (written by other parts of firmware) */
volatile uint8_t g_line_track  = 0xFF;   // 4-bit sensor state
volatile uint16_t g_distance_mm = 0;     // ultrasonic in mm
volatile uint8_t g_ir_key      = 0;      // last IR key code

extern I2C_HandleTypeDef hi2c1;

/* ── start listening ─────────────────────────────────────────────── */
void I2CSlave_Init(void) {
    HAL_I2C_EnableListen_IT(&hi2c1);
}

/* ── called when Pi addresses us ────────────────────────────────── */
void HAL_I2C_AddrCallback(I2C_HandleTypeDef *hi2c,
                           uint8_t TransferDirection,
                           uint16_t AddrMatchCode) {
    if (TransferDirection == I2C_DIRECTION_TRANSMIT) {
        /* Pi is writing to us — receive register + data */
        rx_len = 0;
        HAL_I2C_Slave_Seq_Receive_IT(hi2c, rx_buf, sizeof(rx_buf),
                                      I2C_FIRST_FRAME);
    } else {
        /* Pi wants to read — prepare the response */
        switch (current_reg) {
            case REG_LINE_TRACK:
                tx_buf[0] = g_line_track;
                HAL_I2C_Slave_Seq_Transmit_IT(hi2c, tx_buf, 1,
                                               I2C_LAST_FRAME);
                break;
            case REG_IR_KEY:
                tx_buf[0] = g_ir_key;
                HAL_I2C_Slave_Seq_Transmit_IT(hi2c, tx_buf, 1,
                                               I2C_LAST_FRAME);
                break;
            case REG_DIST_LO:
                tx_buf[0] = (uint8_t)(g_distance_mm & 0xFF);
                HAL_I2C_Slave_Seq_Transmit_IT(hi2c, tx_buf, 1,
                                               I2C_LAST_FRAME);
                break;
            case REG_DIST_HI:
                tx_buf[0] = (uint8_t)(g_distance_mm >> 8);
                HAL_I2C_Slave_Seq_Transmit_IT(hi2c, tx_buf, 1,
                                               I2C_LAST_FRAME);
                break;
            default:
                tx_buf[0] = 0x00;
                HAL_I2C_Slave_Seq_Transmit_IT(hi2c, tx_buf, 1,
                                               I2C_LAST_FRAME);
        }
    }
}

/* ── called when Pi finishes writing ────────────────────────────── */
void HAL_I2C_SlaveRxCpltCallback(I2C_HandleTypeDef *hi2c) {
    current_reg = rx_buf[0];   // first byte is always the register
    uint8_t *d  = &rx_buf[1];  // remaining bytes are the payload

    switch (current_reg) {
        case REG_MOTOR:
            /* [motor_id, direction, speed] */
            Motor_Set(d[0], d[1], d[2]);
            break;
        case REG_SERVO:
            /* [servo_id, angle] */
            Servo_Set(d[0], d[1]);
            break;
        case REG_LED_ALL:
            /* [state, color_index] */
            LED_SetAll(d[0], d[1]);
            break;
        case REG_LED_ALONE:
            /* [led_num, state, color_index] */
            LED_SetOne(d[0], d[1], d[2]);
            break;
        case REG_BUZZER:
            /* [state] */
            Buzzer_Set(d[0]);
            break;
        case REG_LED_RGB_ALL:
            /* [R, G, B] */
            LED_SetBrightnessAll(d[0], d[1], d[2]);
            break;
        case REG_LED_RGB_ALONE:
            /* [led_num, R, G, B] */
            LED_SetBrightnessOne(d[0], d[1], d[2], d[3]);
            break;
        case REG_ULTRASONIC:
            /* [state] — enable/disable sensor */
            Ultrasonic_Enable(d[0]);
            break;
        case REG_IR_SW:
            /* [state] — enable/disable IR receiver */
            IR_Enable(d[0]);
            break;
        default:
            break;
    }

    /* Re-arm the listener for the next transaction */
    HAL_I2C_EnableListen_IT(hi2c);
}

/* ── called when we finish sending data to Pi ───────────────────── */
void HAL_I2C_SlaveTxCpltCallback(I2C_HandleTypeDef *hi2c) {
    HAL_I2C_EnableListen_IT(hi2c);
}

/* ── error recovery ─────────────────────────────────────────────── */
void HAL_I2C_ErrorCallback(I2C_HandleTypeDef *hi2c) {
    HAL_I2C_EnableListen_IT(hi2c);   // always re-arm, never get stuck
}
```

---

### 6.5 Phase 3: PWM for DC Motors

The four DC motors are driven by PWM signals. On STM32, PWM is generated by a **Timer** peripheral configured in PWM mode — no CPU time is used once started, the hardware generates the signal automatically.

#### CubeMX configuration

- Enable **TIM2** (or another available timer)
- Mode: **PWM Generation CH1** through **CH4** (one channel per motor)
- Set the **Prescaler** and **Period** to get your target PWM frequency
  - DC motors typically use 1–20 kHz
  - For 72 MHz clock, 1 kHz PWM: Prescaler = 71, Period = 999
- Set each channel's **Pulse** to 0 (motors stopped at startup)

#### Motor control code

Create `Core/Src/motor.c`:

```c
#include "main.h"
#include "motor.h"

/* Motor direction pins — adjust to match your actual PCB wiring */
/* Each motor needs 2 GPIO pins for direction (IN1/IN2 pattern)  */
typedef struct {
    GPIO_TypeDef *port_in1;
    uint16_t      pin_in1;
    GPIO_TypeDef *port_in2;
    uint16_t      pin_in2;
    TIM_HandleTypeDef *htim;
    uint32_t      channel;
} Motor_t;

extern TIM_HandleTypeDef htim2;

static Motor_t motors[4] = {
    /* Motor 0: Front-Left  */
    { GPIOA, GPIO_PIN_0, GPIOA, GPIO_PIN_1, &htim2, TIM_CHANNEL_1 },
    /* Motor 1: Rear-Left   */
    { GPIOA, GPIO_PIN_2, GPIOA, GPIO_PIN_3, &htim2, TIM_CHANNEL_2 },
    /* Motor 2: Front-Right */
    { GPIOB, GPIO_PIN_0, GPIOB, GPIO_PIN_1, &htim2, TIM_CHANNEL_3 },
    /* Motor 3: Rear-Right  */
    { GPIOB, GPIO_PIN_6, GPIOB, GPIO_PIN_7, &htim2, TIM_CHANNEL_4 },
};

void Motor_Init(void) {
    for (int i = 0; i < 4; i++) {
        HAL_TIM_PWM_Start(motors[i].htim, motors[i].channel);
    }
}

/*
 * motor_id : 0–3
 * direction: 0 = forward, 1 = reverse
 * speed    : 0–255
 */
void Motor_Set(uint8_t motor_id, uint8_t direction, uint8_t speed) {
    if (motor_id > 3) return;
    Motor_t *m = &motors[motor_id];

    if (direction == 0) {
        HAL_GPIO_WritePin(m->port_in1, m->pin_in1, GPIO_PIN_SET);
        HAL_GPIO_WritePin(m->port_in2, m->pin_in2, GPIO_PIN_RESET);
    } else {
        HAL_GPIO_WritePin(m->port_in1, m->pin_in1, GPIO_PIN_RESET);
        HAL_GPIO_WritePin(m->port_in2, m->pin_in2, GPIO_PIN_SET);
    }

    /* Scale 0–255 to 0–999 (timer period) */
    uint32_t duty = (uint32_t)speed * 999 / 255;
    __HAL_TIM_SET_COMPARE(m->htim, m->channel, duty);
}

void Motor_Stop(uint8_t motor_id) {
    Motor_Set(motor_id, 0, 0);
    Motor_t *m = &motors[motor_id];
    HAL_GPIO_WritePin(m->port_in1, m->pin_in1, GPIO_PIN_RESET);
    HAL_GPIO_WritePin(m->port_in2, m->pin_in2, GPIO_PIN_RESET);
}

void Motor_StopAll(void) {
    for (int i = 0; i < 4; i++) Motor_Stop(i);
}
```

> **Note on pin numbers:** The GPIO pin and timer channel assignments above are examples. You need to trace your actual PCB to find which STM32 pins connect to the motor driver circuit. Use a multimeter in continuity mode or study the board schematic if Yahboom provides one.

---

### 6.6 Phase 4: Servo Control

Servos expect a 50 Hz PWM signal where the pulse width encodes the angle:
- 1.0 ms pulse → 0°
- 1.5 ms pulse → 90°
- 2.0 ms pulse → 180°

At 50 Hz the period is 20 ms, so pulse width as a fraction of period:
- 0°   → 5% duty cycle  (1.0 ms / 20 ms)
- 90°  → 7.5% duty cycle (1.5 ms / 20 ms)
- 180° → 10% duty cycle (2.0 ms / 20 ms)

```c
/* Core/Src/servo.c */
#include "main.h"
#include "servo.h"

extern TIM_HandleTypeDef htim3;   /* dedicate TIM3 to servos at 50 Hz */

/* CubeMX setup for 50 Hz:
   Clock = 72 MHz, Prescaler = 1439, Period = 999
   → 72,000,000 / 1440 / 1000 = 50 Hz                    */

#define SERVO_MIN_PULSE  50    /* timer counts for 1.0 ms */
#define SERVO_MAX_PULSE  100   /* timer counts for 2.0 ms */

void Servo_Init(void) {
    HAL_TIM_PWM_Start(&htim3, TIM_CHANNEL_1);   /* servo 1 */
    HAL_TIM_PWM_Start(&htim3, TIM_CHANNEL_2);   /* servo 2 */
    Servo_Set(1, 90);   /* center both servos on startup */
    Servo_Set(2, 90);
}

/*
 * servo_id: 1 or 2
 * angle   : 0–180 degrees
 */
void Servo_Set(uint8_t servo_id, uint8_t angle) {
    if (angle > 180) angle = 180;
    if (servo_id == 2 && angle > 110) angle = 110;   /* tilt limit */

    uint32_t pulse = SERVO_MIN_PULSE +
                     (uint32_t)angle * (SERVO_MAX_PULSE - SERVO_MIN_PULSE) / 180;

    uint32_t channel = (servo_id == 1) ? TIM_CHANNEL_1 : TIM_CHANNEL_2;
    __HAL_TIM_SET_COMPARE(&htim3, channel, pulse);
}
```

---

### 6.7 Phase 5: Reading the Line Tracking Sensor

The 4-channel IR sensor returns 4 digital signals — one per channel. Each GPIO pin reads high (1) for white surface or low (0) for black line.

```c
/* Core/Src/line_track.c */
#include "main.h"
#include "line_track.h"

/* Connect each sensor output to a GPIO input pin.
   Adjust port/pin to match your actual PCB wiring.   */
#define L1_PORT  GPIOC
#define L1_PIN   GPIO_PIN_0
#define L2_PORT  GPIOC
#define L2_PIN   GPIO_PIN_1
#define R1_PORT  GPIOC
#define R1_PIN   GPIO_PIN_2
#define R2_PORT  GPIOC
#define R2_PIN   GPIO_PIN_3

/*
 * Returns a packed byte matching the Yahboom register format:
 * bit3=L1, bit2=L2, bit1=R1, bit0=R2
 * 0 = black line detected, 1 = white surface
 */
uint8_t LineTrack_Read(void) {
    uint8_t L1 = HAL_GPIO_ReadPin(L1_PORT, L1_PIN);
    uint8_t L2 = HAL_GPIO_ReadPin(L2_PORT, L2_PIN);
    uint8_t R1 = HAL_GPIO_ReadPin(R1_PORT, R1_PIN);
    uint8_t R2 = HAL_GPIO_ReadPin(R2_PORT, R2_PIN);

    return (L1 << 3) | (L2 << 2) | (R1 << 1) | R2;
}
```

Call this periodically from a timer interrupt and store the result in `g_line_track` so the I2C slave can serve it to the Pi:

```c
/* In HAL_TIM_PeriodElapsedCallback (50 Hz timer) */
void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim) {
    if (htim->Instance == TIM4) {
        g_line_track = LineTrack_Read();
    }
}
```

---

### 6.8 Phase 6: Ultrasonic Distance Sensor

The HC-SR04 sensor works by timing a sound pulse:
1. Send a 10 µs trigger pulse on TRIG pin
2. Measure how long the ECHO pin stays high
3. Distance = (echo duration × speed of sound) / 2

```c
/* Core/Src/ultrasonic.c */
#include "main.h"
#include "ultrasonic.h"

#define TRIG_PORT  GPIOB
#define TRIG_PIN   GPIO_PIN_8
#define ECHO_PORT  GPIOB
#define ECHO_PIN   GPIO_PIN_9

/* Use TIM5 in input capture mode, or a simple polling approach */

/*
 * Simple blocking measurement — returns distance in mm.
 * For non-blocking, use input capture with TIM interrupts.
 * Max measurable: ~4000 mm (400 cm). Returns 9999 on timeout.
 */
uint16_t Ultrasonic_Measure(void) {
    uint32_t timeout;

    /* Send 10 µs trigger pulse */
    HAL_GPIO_WritePin(TRIG_PORT, TRIG_PIN, GPIO_PIN_SET);
    /* 10 µs delay — use a DWT cycle counter for precision */
    DWT_Delay_us(10);
    HAL_GPIO_WritePin(TRIG_PORT, TRIG_PIN, GPIO_PIN_RESET);

    /* Wait for ECHO to go high (timeout 30 ms) */
    timeout = HAL_GetTick() + 30;
    while (HAL_GPIO_ReadPin(ECHO_PORT, ECHO_PIN) == GPIO_PIN_RESET) {
        if (HAL_GetTick() > timeout) return 9999;
    }

    /* Measure how long ECHO stays high */
    uint32_t start = DWT_GetCycle();
    timeout = HAL_GetTick() + 30;
    while (HAL_GPIO_ReadPin(ECHO_PORT, ECHO_PIN) == GPIO_PIN_SET) {
        if (HAL_GetTick() > timeout) return 9999;
    }
    uint32_t end = DWT_GetCycle();

    /* Convert cycles to microseconds then to mm */
    uint32_t cpu_mhz = SystemCoreClock / 1000000;
    uint32_t duration_us = (end - start) / cpu_mhz;

    /* Speed of sound = 343 m/s = 0.343 mm/µs
       Distance = duration_us × 0.343 / 2              */
    return (uint16_t)(duration_us * 343 / 2000);
}

/* Enable/disable the sensor (called from I2C register write) */
static uint8_t ultrasonic_enabled = 0;

void Ultrasonic_Enable(uint8_t state) {
    ultrasonic_enabled = state;
}

/* Call this from a 10 Hz timer callback */
void Ultrasonic_Update(void) {
    if (ultrasonic_enabled) {
        g_distance_mm = Ultrasonic_Measure();
    }
}
```

---

### 6.9 Phase 7: Going Beyond Yahboom

Once you have reproduced the original register map, your robot works exactly as before — but now you own the firmware. Here is what you can add next.

#### Add a PID motor speed controller

Yahboom's firmware is open-loop: you send "speed 150" and it sets the PWM duty cycle to 150/255. It does not check whether the motor is actually turning at that speed. If the robot hits a slope, it slows down and the firmware does not compensate.

A PID controller reads encoder feedback (if your motors have encoders) and adjusts the PWM to maintain the commanded speed:

```c
/* Core/Src/pid.c */
typedef struct {
    float kp, ki, kd;
    float integral;
    float prev_error;
    float output_min, output_max;
} PID_t;

float PID_Compute(PID_t *pid, float setpoint, float measured, float dt) {
    float error    = setpoint - measured;
    pid->integral += error * dt;
    float derivative = (error - pid->prev_error) / dt;
    pid->prev_error  = error;

    float output = pid->kp * error
                 + pid->ki * pid->integral
                 + pid->kd * derivative;

    /* Clamp output */
    if (output > pid->output_max) output = pid->output_max;
    if (output < pid->output_min) output = pid->output_min;
    return output;
}

/* One PID controller per motor */
static PID_t motor_pid[4] = {
    { .kp=2.0f, .ki=0.5f, .kd=0.1f, .output_min=0, .output_max=255 },
    { .kp=2.0f, .ki=0.5f, .kd=0.1f, .output_min=0, .output_max=255 },
    { .kp=2.0f, .ki=0.5f, .kd=0.1f, .output_min=0, .output_max=255 },
    { .kp=2.0f, .ki=0.5f, .kd=0.1f, .output_min=0, .output_max=255 },
};
```

This makes the robot's speed accurate regardless of battery voltage or surface incline — a major improvement over the original firmware.

#### Add new I2C registers

Since you control the firmware, you can add registers Yahboom never implemented. For example, add register `0x20` to report the current actual motor speed (from encoder counts). The Pi can then do odometry: track how far the robot has moved and in which direction.

```c
/* In i2c_slave.c — add to the read switch */
case 0x20:
    tx_buf[0] = (uint8_t)(encoder_speed[0] & 0xFF);
    tx_buf[1] = (uint8_t)(encoder_speed[1] & 0xFF);
    tx_buf[2] = (uint8_t)(encoder_speed[2] & 0xFF);
    tx_buf[3] = (uint8_t)(encoder_speed[3] & 0xFF);
    HAL_I2C_Slave_Seq_Transmit_IT(hi2c, tx_buf, 4, I2C_LAST_FRAME);
    break;
```

On the Pi side, add a method to `Raspbot_Lib.py`:

```python
def Read_Motor_Speeds(self):
    """Read actual motor speeds from encoder feedback (requires custom firmware)."""
    data = self.read_data_array(0x20, 4)
    return data[0], data[1], data[2], data[3]
```

#### Why this matters for ROS2

With encoder feedback and PID control in the STM32 firmware, your ROS2 driver node becomes much simpler and more accurate. The Pi sends a velocity command, the STM32 maintains it precisely, and reports back real odometry. This is exactly how professional robots like TurtleBot3 work — their OpenCR board runs firmware that does exactly this, communicating with the Pi over USB serial instead of I2C.

---

### 6.10 Firmware Project File Structure

```
raspbot_firmware/
├── Core/
│   ├── Inc/
│   │   ├── main.h
│   │   ├── i2c_slave.h
│   │   ├── motor.h
│   │   ├── servo.h
│   │   ├── line_track.h
│   │   ├── ultrasonic.h
│   │   └── pid.h
│   └── Src/
│       ├── main.c           ← entry point, calls Init functions
│       ├── i2c_slave.c      ← I2C register handler (the "brain")
│       ├── motor.c          ← PWM motor control
│       ├── servo.c          ← servo PWM
│       ├── line_track.c     ← GPIO sensor reading
│       ├── ultrasonic.c     ← distance measurement
│       └── pid.c            ← optional: closed-loop speed control
├── Drivers/
│   └── STM32F1xx_HAL_Driver/ ← auto-generated by CubeMX, do not edit
└── raspbot_firmware.ioc      ← CubeMX project file (pin config)
```

---

### 6.11 Debugging Tips

**The STM32 has a live debugger** — this is the biggest advantage over a bare microcontroller. With the ST-Link connected you can:

```
In STM32CubeIDE:
  Run → Debug (F11)
  → Set a breakpoint on any line
  → Inspect any variable in real time
  → Step through code line by line
  → Watch the I2C callback fire when Pi sends a command
```

**SWV (Serial Wire Viewer)** — the STM32 can stream `printf()` output over the debug wire without using a UART:

```c
/* Add to main.c once, then use printf() anywhere */
int _write(int fd, char *ptr, int len) {
    ITM_SendChar(*ptr);   /* sends to SWV console in CubeIDE */
    return len;
}

/* Then anywhere in your code: */
printf("Motor 0 speed: %d\r\n", current_speed[0]);
```

This lets you see debug output from the STM32 in real time while the robot is running, without any extra wiring.

---

### 6.12 Learning Resources

| Resource | What it covers |
|---|---|
| [stm32-base.org](https://stm32-base.org) | STM32 beginner guides, register maps |
| [controllerstech.com](https://controllerstech.com) | HAL tutorials for every peripheral |
| ST's official HAL documentation | `UM1850` (HAL guide for F1 series) — download from st.com |
| *Mastering STM32* by Carmine Noviello | Best book for STM32 HAL development |
| STM32F103 Reference Manual `RM0008` | The definitive register-level reference |
| [TurtleBot3 OpenCR firmware](https://github.com/ROBOTIS-GIT/OpenCR) | Real-world example of exactly this architecture in a commercial ROS robot |
