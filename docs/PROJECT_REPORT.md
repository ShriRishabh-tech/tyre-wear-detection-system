# Project Report — Tyre Wear Detection & Straight-Line Correction System

> **Institution**: Indian Institute of Information Technology, Design and Manufacturing, Kancheepuram  
> **Mentor**: Dr. Karthik C, Department of Design  
> **Team**: Vijay Surya, Roshan V, Harish Ram R V, Pranav Parasuram, Tejhaswin S P

---

## Table of Contents

1. [Project Objective](#1-project-objective)
2. [Basic Working Principle](#2-basic-working-principle)
3. [Physical Prototype](#3-physical-prototype)
4. [Hardware Components](#4-hardware-components)
5. [Why a Smaller Tyre Causes Deviation](#5-why-a-smaller-tyre-causes-deviation)
6. [How Deviation Is Detected](#6-how-deviation-is-detected)
7. [How the Vehicle Corrects Its Motion](#7-how-the-vehicle-corrects-its-motion)
8. [Feedback Control Concept](#8-feedback-control-concept)
9. [Role of the Tachometers](#9-role-of-the-tachometers)
10. [Determining Vehicle Velocity](#10-determining-vehicle-velocity)
11. [Estimating Effective Wheel Radius](#11-estimating-effective-wheel-radius)
12. [Tyre Wear Calculation](#12-tyre-wear-calculation)
13. [Control Algorithm](#13-control-algorithm)
14. [Combining Wear Detection and Deviation Correction](#14-combining-wear-detection-and-deviation-correction)
15. [Key Concepts and Distinctions](#15-key-concepts-and-distinctions)
16. [What the Project Demonstrates](#16-what-the-project-demonstrates)
17. [Future Development](#17-future-development)
18. [Summary](#18-summary)

---

## 1. Project Objective

The main objective of this project is to develop a system that can **detect the effect of tyre wear** on a vehicle and simultaneously **correct the deviation** in the vehicle's motion caused by unequal tyre sizes.

When a tyre wears over time, its effective rolling radius decreases. If the tyres on a vehicle do not have the same effective radius, the wheels rotate at different speeds even when the vehicle is expected to travel in a straight line. This difference causes the vehicle to gradually deviate from its intended straight path.

The project combines two important functions:

1. **Detecting and correcting** the deviation of the vehicle from straight-line motion.
2. **Estimating the amount of wear** in each tyre by determining its current effective radius and comparing it with the original tyre radius.

For the prototype, instead of waiting for actual tyre wear to occur naturally, we intentionally use **one tyre with a smaller radius** than the other three tyres. This creates a controlled and measurable condition that represents the effect of tyre wear.

---

## 2. Basic Working Principle

Consider a four-wheel vehicle in which three tyres have the same radius while one tyre has a smaller radius.

For a given angular velocity, the linear distance travelled by a wheel is:

$$v = \omega \cdot r$$

Where:
- $v$ = linear velocity of the wheel
- $\omega$ = angular velocity of the wheel
- $r$ = effective wheel radius

When one wheel has a smaller radius, it covers a smaller distance per revolution compared with the other wheels. If all wheels are driven at approximately the same rotational speed, the difference in distance travelled produces an imbalance, causing the vehicle to curve toward one side instead of travelling straight.

This intentional radius difference is used in the prototype to reproduce the effect of unequal tyre wear.

---

## 3. Physical Prototype

The prototype consists of a **four-wheel motorised vehicle** with one critical feature: **one tyre has a smaller radius** than the remaining three.

```
┌─────────────────────┐
│     Vehicle Top      │
│                     │
│  FL ○──────○ FR     │  ← Three wheels: standard radius
│  │          │       │
│  │  [ESP32] │       │
│  │  [MPU ]  │       │
│  │          │       │
│  RL ○──────● RR     │  ← One wheel (●): smaller radius
│                     │
│     ← FRONT →       │
└─────────────────────┘
```

This creates an artificial tyre-wear condition that is controllable and repeatable during testing.

---

## 4. Hardware Components

| Component | Specification | Qty | Purpose |
|---|---|:---:|---|
| ESP32 WROOM | Dual-core, 240 MHz, Bluetooth | 1 | Main controller |
| MPU6050 | 6-axis IMU (gyro + accel) | 1 | Yaw deviation detection |
| TB6612FNG | Dual H-bridge motor driver | 2 | Motor power control |
| IR Tachometer | Slot-type optical sensor | 4 | Wheel RPM measurement |
| Encoder Disc | 20-hole slotted disc | 4 | Pulse generation |
| DC Geared Motor | 3V–6V | 4 | Wheel drive |

> For detailed specifications, pin assignments, and wiring diagrams, see [HARDWARE.md](HARDWARE.md).

---

## 5. Why a Smaller Tyre Causes Deviation

Suppose two wheels rotate at the same angular velocity $\omega$:

- Larger wheel: $v_{large} = \omega \times r_{large}$
- Smaller wheel: $v_{small} = \omega \times r_{small}$

Since $r_{small} < r_{large}$, it follows that $v_{small} < v_{large}$.

Even though the wheels rotate at the same RPM, they do not travel the same linear distance per revolution. Within a four-wheel vehicle, this difference produces a **turning tendency** toward the side with the smaller wheel.

---

## 6. How Deviation Is Detected

The vehicle is commanded to move forward. Because one wheel has a smaller radius, the vehicle begins to deviate from its straight path.

The **MPU6050 IMU** continuously measures the rotational (yaw) change of the vehicle:

| Condition | Expected Yaw |
|---|---|
| Vehicle moves straight | Yaw ≈ 0° |
| Vehicle starts turning | Yaw changes from initial value |

The ESP32 reads the gyroscope Z-axis data, converts it to degrees per second (dividing by 131 for the ±250°/s range), and integrates over time to obtain the cumulative yaw angle:

$$\text{yaw} = \sum (\text{gyroZ} \times \Delta t)$$

The magnitude and sign of the yaw angle indicate the direction and extent of the deviation.

---

## 7. How the Vehicle Corrects Its Motion

Once deviation is detected, the ESP32 adjusts the speed of the **front wheels** differentially:

$$\text{Left front PWM} = \text{Base Speed} + \text{offset}$$

$$\text{Right front PWM} = \text{Base Speed} - \text{offset}$$

The offset is calculated as:

$$\text{offset} = \text{CORRECTION\_GAIN} \times \text{yaw}$$

With `CORRECTION_GAIN = 1.5`, a yaw deviation of 2° produces a PWM correction of ±3.

**Key design choice**: Only the front wheels receive correction. The rear wheels maintain a constant base speed, providing stability and isolating the correction effect to the steering axis.

The corrections **accumulate across runs**, so each successive `GO` command applies the sum of all previous corrections plus the new one.

---

## 8. Feedback Control Concept

The project implements a **closed-loop feedback control system**:

```
                 ┌────────────────────────────────────────────┐
                 │                                            │
                 ▼                                            │
          Vehicle Moves                                       │
                 │                                            │
                 ▼                                            │
     ┌───────────────────────┐                                │
     │  Sensors Measure:     │                                │
     │  • Wheel RPM (×4)     │                                │
     │  • Yaw angle (IMU)    │                                │
     └───────────┬───────────┘                                │
                 │                                            │
                 ▼                                            │
     ┌───────────────────────┐                                │
     │   ESP32 Processes     │                                │
     │   Deviation Data      │                                │
     └───────────┬───────────┘                                │
                 │                                            │
                 ▼                                            │
     ┌───────────────────────┐                                │
     │  Calculate Correction │                                │
     │  Adjust Wheel Speeds  │                                │
     └───────────┬───────────┘                                │
                 │                                            │
                 └────────────────────────────────────────────┘
```

The vehicle's actual motion is measured and fed back into the controller, making this a true **closed-loop system** that continuously responds to real behaviour.

---

## 9. Role of the Tachometers

The four tachometer sensors serve two critical functions:

### Function 1: Individual Wheel RPM Measurement

Each tachometer counts pulses from a 20-hole encoder disc. The RPM is calculated as:

$$\text{RPM} = \frac{\text{pulse\_count}}{\text{HOLES}} \times \frac{60}{\text{time (s)}}$$

### Function 2: Effective Radius Estimation

By comparing RPM values across wheels, the system estimates the effective radius of each wheel relative to a reference wheel. If a wheel rotates faster than expected for the same vehicle speed, its radius must be smaller:

$$r_{wheel} = \frac{RPM_{reference}}{RPM_{wheel}} \times r_{reference}$$

In our prototype, the reference wheel radius is **3.25 cm**.

---

## 10. Determining Vehicle Velocity

The system estimates linear velocity using two approaches:

### IMU-Based Estimation

Acceleration from the MPU6050 X-axis is integrated over time:

$$v = \int a \, dt$$

A dead zone (|a| < 0.05 m/s²) filters sensor noise, and a gentle decay factor (×0.995 per step) combats integration drift.

### Tachometer-Based Calibration

On the **first run**, the IMU-derived distance is compared with the tachometer-derived distance (from the reference wheel):

$$\text{SCALE} = \frac{\text{distance}_{tachometer}}{\text{distance}_{IMU}}$$

This scale factor calibrates the IMU estimate for all subsequent runs.

---

## 11. Estimating Effective Wheel Radius

Once angular velocity and vehicle velocity are known:

$$r = \frac{v}{\omega}$$

The system calculates the effective rolling radius of each wheel:

| Wheel | Estimated Radius |
|---|---|
| Wheel 1 (FL) | $r_1$ |
| Wheel 2 (FR) | $r_2$ |
| Wheel 3 (RL) | $r_3$ (reference) |
| Wheel 4 (RR) | $r_4$ |

For unworn, equally sized tyres: $r_1 \approx r_2 \approx r_3 \approx r_4$.

A significant difference in estimated radius indicates tyre wear or an abnormal wheel condition.

---

## 12. Tyre Wear Calculation

Let $R_{original}$ be the original tyre radius and $R_{current}$ be the estimated effective radius.

**Absolute wear**:

$$\text{Wear} = R_{original} - R_{current}$$

**Percentage wear**:

$$\text{Wear \%} = \frac{R_{original} - R_{current}}{R_{original}} \times 100$$

### Example

| Parameter | Value |
|---|---|
| Original radius | 3.25 cm |
| Estimated radius | 3.06 cm |
| Absolute wear | 0.19 cm |
| Wear percentage | 5.85% |

---

## 13. Control Algorithm

The step-by-step control logic implemented in the firmware:

1. **Initialise** ESP32, MPU6050, tachometers, and Bluetooth.
2. Wait for `GO` command via Bluetooth.
3. **Calibrate** gyroscope (200 samples) and accelerometer (500 samples) while stationary.
4. **Reset** yaw, velocity, distance, and pulse counters.
5. **Drive forward** at base PWM speed with current correction offsets.
6. **For 5 seconds**, continuously:
   - Integrate gyroscope Z-axis → yaw angle
   - Integrate accelerometer X-axis → velocity and distance
   - Count tachometer pulses via interrupts
7. **Stop** all motors.
8. **Calculate** RPM for each wheel from pulse counts.
9. **Calculate** correction offset from yaw × gain.
10. **Accumulate** correction into left/right offsets.
11. **Estimate** tyre radius from RPM ratios.
12. **Report** all results over Bluetooth.
13. **Wait** for next `GO` command.

---

## 14. Combining Wear Detection and Deviation Correction

The most important aspect of this project is that **both functions operate together** within the same system and the same measurement cycle:

```
      Tyre Wear / Unequal Radius
                 │
                 ▼
      Difference in Wheel Behaviour
                 │
                 ▼
        Vehicle Deviates
                 │
          ┌──────┴──────┐
          ▼              ▼
     IMU Detects      Tachometers
     Deviation        Measure RPM
          ▼              ▼
          └──────┬──────┘
                 ▼
             ESP32
          ┌──────┴──────┐
          ▼              ▼
    Correct Speed    Estimate Radius
    Differentially   per Wheel
          ▼              ▼
    Vehicle Goes     Compare with
    Straighter       Original Radius
                         ▼
                    Estimate Wear
```

---

## 15. Key Concepts and Distinctions

### RPM Alone Does Not Indicate Wear

A wheel can rotate at any RPM regardless of its radius. What matters is the relationship between angular and linear velocity.

For the same vehicle speed, a smaller-radius wheel must rotate **faster**:

$$\omega = \frac{v}{r}$$

Therefore, if the vehicle velocity remains comparable and a wheel's RPM increases relative to the original condition, this indicates a **decrease in effective radius**.

### Correction Strategy — Front Wheels Only

The system corrects only the **front wheel** speeds while keeping the rear wheels at a constant base speed. This approach:
- Provides a stable reference from the rear axle
- Applies correction where steering effect is most pronounced
- Prevents oscillation from over-correction on all four wheels

---

## 16. What the Project Demonstrates

The prototype demonstrates the following engineering concepts in one integrated system:

1. Unequal wheel radius causes measurable vehicle deviation
2. Individual wheel rotational speed can be measured independently
3. Vehicle deviation can be detected using an IMU (gyroscope integration)
4. Individual wheel speeds can be controlled via differential PWM
5. Wheel angular velocity and vehicle velocity can estimate wheel radius
6. Estimated radius compared with original radius quantifies tyre wear
7. Closed-loop feedback progressively reduces trajectory error

**Disciplines combined**:

| Domain | Application |
|---|---|
| Vehicle Dynamics | Tyre-road interaction, turning tendency |
| Sensor Technology | IMU, optical encoders, tachometers |
| Embedded Systems | ESP32 firmware, ISR, PWM generation |
| Feedback Control | Closed-loop correction with cumulative offsets |
| Mathematical Modelling | Kinematic equations, radius estimation |
| Wireless Communication | Bluetooth telemetry |

---

## 17. Future Development

| Enhancement | Description |
|---|---|
| High-resolution encoders | More accurate RPM measurement for better radius estimation |
| PID controller | Smoother, faster convergence with proportional-integral-derivative control |
| Kalman filter | Robust sensor fusion for IMU and tachometer data |
| ToF tread sensor | Direct tread-depth measurement using Time-of-Flight sensor |
| SD card logging | On-board data recording for post-analysis |
| Mobile app | Live dashboard with graphical monitoring |
| Multi-surface testing | Validate on asphalt, concrete, and uneven terrain |
| Real vehicle extension | Apply the methodology to actual passenger vehicles |
| Automatic wear alerts | Threshold-based warnings when wear exceeds a safe limit |

---

## 18. Summary

The project intentionally creates an unequal tyre radius condition by using one smaller-radius tyre on a four-wheel vehicle. This causes the vehicle to deviate from a straight path.

The **ESP32** receives data from **four tachometers** and an **MPU6050 IMU**. The tachometers measure individual wheel RPM, while the IMU detects yaw deviation.

When the vehicle deviates, the ESP32 changes the relative speed of the front wheels through differential PWM control so that the vehicle corrects toward a straight path. Corrections accumulate across runs, progressively improving the trajectory.

Simultaneously, the measured wheel RPM and vehicle velocity are used to estimate the effective radius of each wheel. Since the original radius is known, the current estimated radius is compared against it, and the reduction is used to estimate the amount of tyre wear.

The project is a combination of:

1. Tyre wear estimation
2. Unequal wheel radius detection
3. Straight-line trajectory correction
4. Individual wheel speed measurement
5. IMU-based deviation detection
6. Closed-loop feedback control
7. ESP32-based embedded implementation

The final goal is to develop a system that can both **identify** the effect of tyre wear and **compensate** for its effect on vehicle motion.
