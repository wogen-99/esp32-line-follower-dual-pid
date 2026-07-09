# esp32-line-follower-dual-pid
ESP32 MicroPython line-following robot with 5-channel ADC sensors, outer line PID, and encoder-based inner wheel-speed PID.
# ESP32 Line Follower Dual PID

An ESP32 MicroPython line-following robot using 5-channel ADC line sensors, differential drive motors, encoder feedback, outer line-position PID control, and inner wheel-speed PID control.

## Features

* 5-channel ADC line sensor input
* Weighted-average line position estimation
* Outer PID/PD controller for line-following correction
* Encoder-based inner wheel-speed PID controller
* Differential motor control with PWM
* Motor direction reversal support
* Encoder direction reversal support
* Lost-line debounce and recovery logic
* Debug output for ADC values, line state, PID correction, target speed, measured speed, and PWM command

## Hardware

Main components:

* ESP32 development board
* 5-channel analog line sensor module
* Two DC motors
* Motor driver module
* Left and right wheel encoders
* Battery power supply
* Robot chassis

## Firmware Requirement

This project requires an ESP32 MicroPython firmware that supports `machine.Encoder`.

If your firmware does not provide `machine.Encoder`, the program will fail at:

```python
from machine import ADC, Pin, PWM, Timer, Encoder

## Pin Configuration

### ADC Line Sensors

| Sensor Position | GPIO |
| --------------- | ---: |
| Leftmost        |   27 |
| Left-middle     |   33 |
| Center          |   32 |
| Right-middle    |   35 |
| Rightmost       |   34 |

The default line-position weights are:

```python
WEIGHTS = [-4, -2, 0, 2, 4]
```

This means the sensor order is assumed to be from left to right.

### Motor PWM Pins

| Motor                 | IN1 | IN2 |
| --------------------- | --: | --: |
| Motor 1 / Left motor  |  13 |  15 |
| Motor 2 / Right motor |  14 |  25 |

The PWM frequency is:

```python
PWM_FREQ = 20000
```

### Encoder Pins

| Encoder       | GPIO A | GPIO B |
| ------------- | -----: | -----: |
| Left encoder  |     17 |     16 |
| Right encoder |     18 |     19 |

## Control Structure

The project uses a cascaded control structure:

```text
5-channel ADC sensor
        ↓
Line error calculation
        ↓
Outer line PID / PD controller
        ↓
Left and right target wheel speeds
        ↓
Encoder speed measurement
        ↓
Inner wheel-speed PID controller
        ↓
PWM motor output
```

## Line Error Calculation

Each ADC value is compared with the line threshold:

```python
LINE_THRESHOLD = 130
```

The signal strength of each sensor is calculated as:

```python
strength = adc_value - LINE_THRESHOLD
```

If the sum of all strengths is zero, the robot treats the line as lost.

The line error is calculated by weighted average:

```text
error = sum(weight_i * strength_i) / sum(strength_i)
```

A negative error means the line is on the left side.
A positive error means the line is on the right side.

## Outer PID Controller

The outer controller uses line error as input and outputs a speed correction:

```python
correction = KP * error + KI * integral + KD * derivative
```

Default parameters:

```python
KP = 22.0
KI = 0.0
KD = 18.0
CORRECTION_LIMIT = 100
```

Since `KI = 0.0`, the current outer controller behaves as a PD controller.

The left and right target speeds are calculated as:

```python
left_target_percent = BASE_SPEED + correction
right_target_percent = BASE_SPEED - correction
```

## Speed Percentage

The `BASE_SPEED` value is not direct PWM duty. It is a target speed percentage based on encoder calibration.

```python
BASE_SPEED = 80
PULSES_PER_SECOND_AT_100 = 91688.5
```

This means:

```text
100% speed = 91688.5 encoder pulses per second
80% speed = 0.8 × 91688.5 encoder pulses per second
```

The target speed is converted into encoder pulses per control period:

```python
target_count = speed_percent * PULSES_PER_SECOND_AT_100 * CONTROL_PERIOD_MS / 100000.0
```

With `CONTROL_PERIOD_MS = 20`, the controller runs at approximately 50 Hz.

## Inner Wheel-Speed PID Controller

The inner controller compares the target encoder count with the measured encoder count in each control period.

Default parameters:

```python
INNER_KP = 0.75
INNER_KI = 0.08
INNER_KD = 0.02
INNER_INTEGRAL_LIMIT = 100.0
```

The output of the inner controller is the actual PWM percentage sent to the motors.

## Lost-Line Handling

If all five sensors fail to detect the line, the robot enters lost-line logic.

The code uses debounce:

```python
LOST_DEBOUNCE_COUNT = 2
```

This means the robot must fail to detect the line for two consecutive control cycles before it is treated as truly lost.

When the line is lost, the robot turns according to the last known line error direction.

## Debug Output

When debug mode is enabled:

```python
DEBUG_PRINT = True
```

The program prints:

* Raw ADC values
* Active line sensor state
* Current line error
* Outer PID correction
* Target encoder counts
* Measured encoder counts
* PWM output percentages
* Lost-line state

Example fields:

```text
raw
line
err
corr_percent
target_count
meas_count
pwm_percent
lost
```

These values are useful for tuning PID parameters and checking whether the sensors, encoders, and motors are working correctly.

## How to Tune

Recommended tuning order:

1. Confirm motor direction.
2. Confirm encoder direction.
3. Confirm ADC sensor order.
4. Confirm whether black line produces higher or lower ADC values.
5. Tune `LINE_THRESHOLD`.
6. Run without high speed first.
7. Tune outer PID parameters.
8. Tune inner wheel-speed PID parameters.
9. Test lost-line behavior.

Common symptoms:

| Symptom                            | Possible Cause                                                                  |
| ---------------------------------- | ------------------------------------------------------------------------------- |
| Robot turns in the wrong direction | ADC order, motor direction, or correction sign is wrong                         |
| Robot shakes on straight line      | Outer `KP` or `KD` is too large                                                 |
| Robot rushes out on curves         | Base speed too high or correction too aggressive                                |
| PWM stays at ±100%                 | Inner PID gain too high, target speed too high, or encoder calibration is wrong |
| Robot frequently enters lost state | Threshold is wrong, sensor height is wrong, or line detection logic is reversed |

## Main File

The main control program is:

```text
main.py
```

Run it on ESP32 using a MicroPython tool such as Thonny, ampy, mpremote, or another ESP32 MicroPython upload method.

## License

This project is for learning and experimental use.
