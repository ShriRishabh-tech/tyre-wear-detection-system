# Hardware Specifications & Wiring Guide

Detailed hardware component descriptions, specifications, and wiring connections for the Tyre Wear Detection & Straight-Line Correction System.

---

## Components List

### 1. ESP32 WROOM — Main Controller

| Specification | Detail |
|---|---|
| **Module** | ESP32-WROOM-32 |
| **Processor** | Dual-core Xtensa LX6, up to 240 MHz |
| **Flash** | 4 MB |
| **GPIO Pins Used** | 20 (motor control, sensors, I2C) |
| **Communication** | Bluetooth Classic (Serial Profile) |
| **PWM** | 8-bit resolution at 5 kHz |
| **Interrupts** | 4 hardware interrupts for tachometers |
| **Operating Voltage** | 3.3V (accepts 5V via USB/VIN) |

**Role**: Central processing unit — reads all sensors, runs the correction algorithm, generates PWM signals, and transmits telemetry over Bluetooth.

---

### 2. MPU6050 (GY-521 Breakout) — IMU Sensor

| Specification | Detail |
|---|---|
| **Sensor** | InvenSense MPU-6050 |
| **Gyroscope Range** | ±250 °/s (default, sensitivity = 131 LSB/°/s) |
| **Accelerometer Range** | ±2g (default, sensitivity = 16384 LSB/g) |
| **Interface** | I2C (address: 0x68) |
| **Operating Voltage** | 3.3V – 5V (onboard regulator) |

**Role**: Measures yaw rotation (gyroscope Z-axis) to detect trajectory deviation, and forward acceleration (X-axis) for IMU-based velocity estimation.

**Calibration**:
- Gyroscope: 200 samples averaged while stationary
- Accelerometer: 500 samples averaged while stationary

---

### 3. TB6612FNG — Motor Driver (×2)

| Specification | Detail |
|---|---|
| **Module** | Toshiba TB6612FNG dual H-bridge |
| **Channels per Module** | 2 (each module drives 2 motors) |
| **Output Current** | 1.2A continuous / 3.2A peak per channel |
| **Motor Voltage** | 2.5V – 13.5V |
| **Logic Voltage** | 2.7V – 5.5V |
| **Control Signals** | IN1, IN2 (direction) + PWM (speed) per channel |

**Role**: Interfaces between the ESP32's 3.3V logic signals and the higher-current DC motors. Each TB6612FNG drives two motors (one left-side pair, one right-side pair).

---

### 4. IR Slot-Type Tachometer Sensor (×4)

| Specification | Detail |
|---|---|
| **Type** | IR transmissive optical sensor (slot type) |
| **Output** | Digital pulse (HIGH/LOW) |
| **Trigger** | CHANGE interrupt (both edges) |
| **Debounce** | Software — 3000 µs minimum pulse interval |

**Role**: One sensor per wheel. An encoder disc with **20 holes** is attached to each wheel axle. As the wheel rotates, the disc's holes pass through the sensor slot, generating digital pulses that the ESP32 counts via hardware interrupts.

**RPM Calculation**:

```
RPM = (pulse_count / HOLES) / time_seconds × 60
```

Where `HOLES = 20` and `time_seconds = 5.0` (run duration).

---

### 5. Encoder Discs (×4)

| Specification | Detail |
|---|---|
| **Type** | Slotted disc |
| **Holes** | 20 equally spaced |
| **Mounting** | Press-fitted or glued to wheel axle |

**Note**: The interrupt triggers on `CHANGE` (both rising and falling edges), which effectively doubles the resolution. However, the code divides by `HOLES = 20`, matching the physical hole count.

---

### 6. DC Geared Motors (×4)

| Specification | Detail |
|---|---|
| **Type** | Small geared DC motor (typically 3V–6V, ~100–200 RPM) |
| **Control** | Speed via PWM (0–255), direction via IN1/IN2 |

---

## Complete Wiring Diagram

### Motor Connections → TB6612FNG → ESP32

```
 ESP32                 TB6612FNG #1              Motors
┌──────┐              ┌──────────┐
│ GP14 │──── IN1_A ───│          │──── OUT_A ──── Front-Left Motor
│ GP12 │──── IN2_A ───│          │
│ GP13 │──── PWM_A ───│          │
│      │              │          │
│ GP4  │──── IN1_B ───│          │──── OUT_B ──── Rear-Left Motor
│ GP2  │──── IN2_B ───│          │
│ GP15 │──── PWM_B ───│          │
└──────┘              └──────────┘

┌──────┐              ┌──────────┐
│ GP26 │──── IN1_A ───│TB6612FNG │──── OUT_A ──── Front-Right Motor
│ GP27 │──── IN2_A ───│   #2     │
│ GP25 │──── PWM_A ───│          │
│      │              │          │
│ GP33 │──── IN1_B ───│          │──── OUT_B ──── Rear-Right Motor
│ GP32 │──── IN2_B ───│          │
│ GP5  │──── PWM_B ───│          │
└──────┘              └──────────┘
```

### Tachometer Connections → ESP32

```
 Tachometer Sensors         ESP32
┌────────────────┐    ┌──────────┐
│ Tacho FL (OUT) │────│ GPIO 16  │  (INPUT_PULLUP, CHANGE interrupt)
│ Tacho FR (OUT) │────│ GPIO 17  │  (INPUT_PULLUP, CHANGE interrupt)
│ Tacho RL (OUT) │────│ GPIO 18  │  (INPUT_PULLUP, CHANGE interrupt)
│ Tacho RR (OUT) │────│ GPIO 19  │  (INPUT_PULLUP, CHANGE interrupt)
│                │    │          │
│ VCC (all)      │────│ 3.3V     │
│ GND (all)      │────│ GND      │
└────────────────┘    └──────────┘
```

### MPU6050 → ESP32 (I2C)

```
 MPU6050 (GY-521)      ESP32
┌──────────────┐    ┌──────────┐
│     SDA      │────│ GPIO 21  │
│     SCL      │────│ GPIO 22  │
│     VCC      │────│ 3.3V     │
│     GND      │────│ GND      │
│     AD0      │────│ GND      │  (sets I2C address to 0x68)
└──────────────┘    └──────────┘
```

---

## Pin Summary Table

| GPIO | Function | Connected To |
|:---:|---|---|
| 2 | RL Motor IN2 | TB6612FNG #1 IN2_B |
| 4 | RL Motor IN1 | TB6612FNG #1 IN1_B |
| 5 | RR Motor PWM | TB6612FNG #2 PWM_B |
| 12 | FL Motor IN2 | TB6612FNG #1 IN2_A |
| 13 | FL Motor PWM | TB6612FNG #1 PWM_A |
| 14 | FL Motor IN1 | TB6612FNG #1 IN1_A |
| 15 | RL Motor PWM | TB6612FNG #1 PWM_B |
| 16 | Tachometer FL | IR Sensor #1 OUT |
| 17 | Tachometer FR | IR Sensor #2 OUT |
| 18 | Tachometer RL | IR Sensor #3 OUT |
| 19 | Tachometer RR | IR Sensor #4 OUT |
| 21 | I2C SDA | MPU6050 SDA |
| 22 | I2C SCL | MPU6050 SCL |
| 25 | FR Motor PWM | TB6612FNG #2 PWM_A |
| 26 | FR Motor IN1 | TB6612FNG #2 IN1_A |
| 27 | FR Motor IN2 | TB6612FNG #2 IN2_A |
| 32 | RR Motor IN2 | TB6612FNG #2 IN2_B |
| 33 | RR Motor IN1 | TB6612FNG #2 IN1_B |

---

## Power Supply Notes

- **ESP32**: Powered via USB (5V) during development, or via VIN pin from a regulated 5V source.
- **Motors**: Powered from the battery pack through the TB6612FNG VM (motor voltage) pin. Ensure the motor voltage matches the motor rating (typically 3V–6V for small chassis motors).
- **TB6612FNG**: VCC (logic) should be connected to 3.3V from ESP32; VM (motor) should be connected to the battery.
- **MPU6050**: Powered from ESP32's 3.3V pin.
- **Tachometers**: Powered from ESP32's 3.3V pin.

> **Important**: Ensure all GND connections (ESP32, motor drivers, sensors, battery) are tied together on a common ground.

---

## Prototype Assembly Notes

1. **Smaller wheel**: One wheel (the test wheel) should have a visibly smaller radius than the other three. This is the intentional "worn tyre" simulation.
2. **Encoder disc alignment**: Ensure each encoder disc is centred on its axle and the slots cleanly pass through the IR sensor gap without physical contact.
3. **MPU6050 placement**: Mount the IMU flat and centred on the chassis, with the X-axis aligned along the vehicle's forward direction.
4. **Wiring**: Keep motor power wires separated from sensor signal wires to reduce electrical noise. Twisting signal wires or using short leads helps reduce interference.
