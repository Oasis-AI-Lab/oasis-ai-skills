# VEX Robotics Programming Reference

## Supported Languages & IDEs

| Platform | Language | IDE |
|----------|----------|-----|
| VEX V5 | C++ / Python (MicroPython) | VEXcode V5 / PROS |
| VEX IQ | Blocks / Python | VEXcode IQ |
| VEX EXP | Blocks / Python | VEXcode EXP |

---

## VEXcode V5 — C++ Essentials

### Project Setup (C++ Template)

```cpp
#include "vex.h"
using namespace vex;

brain Brain;
motor LeftMotor(PORT1, ratio18_1, false);
motor RightMotor(PORT2, ratio18_1, true); // reversed
inertial InertialSensor(PORT10);
controller Controller1(primary);

int main() {
  // Initialize
  vexcodeInit();
  InertialSensor.calibrate();
  while (InertialSensor.isCalibrating()) { wait(20, msec); }
  
  // Autonomous or driver control
}
```

### Motor Control

```cpp
// Spin at velocity
LeftMotor.spin(forward, 80, pct);
RightMotor.spin(forward, 80, pct);

// Stop motor
LeftMotor.stop(brake);  // short-circuit coils, quick stop (no active hold)
LeftMotor.stop(coast);  // cut power, free spin
LeftMotor.stop(hold);   // active PID to maintain position (draws current)

// Move specific distance (blocking)
LeftMotor.spinFor(forward, 360, degrees, false); // non-blocking
RightMotor.spinFor(forward, 360, degrees);        // blocking

// Reset encoder
LeftMotor.resetPosition();
double pos = LeftMotor.position(degrees);
```

### Drive Train (Using built-in drivetrain object)

```cpp
drivetrain Drivetrain(LeftMotor, RightMotor, 319.19, 295, 40, mm, 1);
// Parameters: left motors, right motors, wheel circumference, track width, 
//             wheelbase, units, gear ratio

// Drive forward
Drivetrain.driveFor(forward, 300, mm);
Drivetrain.turnFor(right, 90, degrees);
Drivetrain.drive(forward);
Drivetrain.stop();

// Set velocity
Drivetrain.setDriveVelocity(60, pct);
Drivetrain.setTurnVelocity(40, pct);
```

### Sensors

```cpp
// Inertial sensor (heading)
double heading = InertialSensor.heading(degrees);   // 0–360
double rotation = InertialSensor.rotation(degrees); // unbounded

// Distance sensor
distance DistanceSensor(PORT5);
double dist = DistanceSensor.objectDistance(mm);
bool detected = DistanceSensor.isObjectDetected();

// Optical sensor
optical OpticalSensor(PORT6);
OpticalSensor.setLight(ledState::on);
double hue = OpticalSensor.hue();          // 0–360
int brightness = OpticalSensor.brightness(); // 0–100

// Touch / bumper
touchled TouchLed(PORT7);
bool pressed = TouchLed.pressing();

// Limit switch
limit LimitSwitch(Brain.ThreeWirePort.A);
bool triggered = LimitSwitch.pressing();
```

### Controller Input

```cpp
// Axis values (-100 to 100)
int leftY  = Controller1.Axis3.position();
int rightX = Controller1.Axis1.position();

// Button presses
if (Controller1.ButtonA.pressing()) { /* action */ }
if (Controller1.ButtonL1.pressing()) { /* action */ }

// Button events (callbacks)
Controller1.ButtonX.pressed(myFunction);
Controller1.ButtonB.released(myOtherFunction);
```

---

## Autonomous Programming Patterns

### Basic Timed Autonomous

```cpp
void autonomous() {
  // Drive forward for 1 second
  LeftMotor.spin(forward, 80, pct);
  RightMotor.spin(forward, 80, pct);
  wait(1000, msec);
  
  // Stop
  LeftMotor.stop(brake);
  RightMotor.stop(brake);
}
```

### Encoder-Based Driving (More Accurate)

```cpp
// Adjust WHEEL_CIRCUMFERENCE_CM for your actual wheel size
// Common values: 4" wheel = 10.21 cm, 3.25" wheel = 8.25 cm
const double WHEEL_CIRCUMFERENCE_CM = 10.21;

void driveInches(double inches, int speed) {
  double degreesNeeded = (inches * 2.54 / WHEEL_CIRCUMFERENCE_CM) * 360;
  
  LeftMotor.setVelocity(speed, pct);
  RightMotor.setVelocity(speed, pct);
  LeftMotor.resetPosition();
  RightMotor.resetPosition();
  LeftMotor.spinFor(forward, degreesNeeded, degrees, false);
  RightMotor.spinFor(forward, degreesNeeded, degrees);
}

// NOTE: targetDeg is renamed to avoid shadowing the VEX SDK 'degrees' enum
// NOTE: add #include <algorithm> at the top of your file for std::max / std::min
void turnDegrees(double targetDeg, int speed) {
  InertialSensor.resetRotation();
  LeftMotor.spin(forward, speed, pct);
  RightMotor.spin(reverse, speed, pct);

  const int MAX_TURN_ITERATIONS = 300; // ~3 seconds max — prevents infinite loop if gyro fails
  int iterations = 0;
  while (iterations < MAX_TURN_ITERATIONS) {
    if (fabs(InertialSensor.rotation(degrees)) >= fabs(targetDeg)) break;
    iterations++;
    wait(10, msec);
  }
  LeftMotor.stop(brake);
  RightMotor.stop(brake);
}
```

### PID Controller (Competition-Level Accuracy)

```cpp
// Simple PID for straight driving
// Requires WHEEL_CIRCUMFERENCE_CM to be defined (see Encoder-Based Driving above)
void pidDrive(double targetInches) {
  double kP = 0.5, kI = 0.001, kD = 0.2;
  double error, prevError = 0, integral = 0, derivative;
  const double INTEGRAL_LIMIT = 50.0; // anti-windup clamp
  const int    MAX_ITERATIONS  = 500; // timeout guard: 500 × 20 ms = 10 s max
  double targetDeg = (targetInches * 2.54 / WHEEL_CIRCUMFERENCE_CM) * 360;

  LeftMotor.resetPosition();
  RightMotor.resetPosition();
  int iterations = 0;

  while (iterations < MAX_ITERATIONS) {
    // Use average of both motors for feedback — compensates for one-sided drift
    double avgPos = (LeftMotor.position(degrees) + RightMotor.position(degrees)) / 2.0;
    error = targetDeg - avgPos;

    // Anti-windup: clamp integral to prevent runaway accumulation
    integral += error;
    integral = fmax(-INTEGRAL_LIMIT, fmin(INTEGRAL_LIMIT, integral));

    derivative = error - prevError;

    double power = kP * error + kI * integral + kD * derivative;
    power = fmax(-100, fmin(100, power)); // Clamp output to [-100, 100]

    LeftMotor.spin(forward, power, pct);
    RightMotor.spin(forward, power, pct);

    prevError = error;
    if (fabs(error) < 5) break; // Within tolerance — exit early
    iterations++;
    wait(20, msec);
  }
  // Timeout exit: robot is stuck or sensor failed — stop to avoid spinning forever
  LeftMotor.stop(brake);
  RightMotor.stop(brake);
}
```
```

### State Machine Autonomous

```cpp
enum class AutoState { DRIVE_FORWARD, TURN_RIGHT, SCORE, DONE };
AutoState state = AutoState::DRIVE_FORWARD;

void autonomous() {
  while (state != AutoState::DONE) {
    switch (state) {
      case AutoState::DRIVE_FORWARD:
        driveInches(24, 70);
        state = AutoState::TURN_RIGHT;
        break;
      case AutoState::TURN_RIGHT:
        turnDegrees(90, 50);
        state = AutoState::SCORE;
        break;
      case AutoState::SCORE:
        IntakeMotor.spinFor(forward, 500, msec);
        state = AutoState::DONE;
        break;
      default: break;
    }
  }
}
```

---

## Driver Control Patterns

### Tank Drive

```cpp
void usercontrol() {
  while (true) {
    int leftSpeed  = Controller1.Axis3.position();
    int rightSpeed = Controller1.Axis2.position();
    
    LeftMotor.spin(forward, leftSpeed, pct);
    RightMotor.spin(forward, rightSpeed, pct);
    
    wait(20, msec);
  }
}
```

### Arcade Drive

```cpp
void usercontrol() {
  while (true) {
    int forwardBack = Controller1.Axis3.position();
    int leftRight   = Controller1.Axis1.position();
    
    int leftSpeed  = forwardBack + leftRight;
    int rightSpeed = forwardBack - leftRight;
    
    // Clamp values (use integer arithmetic, not fmax/fmin which return double)
    if (leftSpeed  >  100) leftSpeed  =  100;
    if (leftSpeed  < -100) leftSpeed  = -100;
    if (rightSpeed >  100) rightSpeed =  100;
    if (rightSpeed < -100) rightSpeed = -100;
    
    LeftMotor.spin(forward, leftSpeed, pct);
    RightMotor.spin(forward, rightSpeed, pct);
    
    wait(20, msec);
  }
}
```

### Mechanism Control with Toggle

```cpp
bool intakeActive = false;

void toggleIntake() {
  intakeActive = !intakeActive;
  if (intakeActive) {
    IntakeMotor.spin(forward, 100, pct);
  } else {
    IntakeMotor.stop(coast);
  }
}

void usercontrol() {
  Controller1.ButtonR1.pressed(toggleIntake);
  while (true) {
    // drive code ...
    wait(20, msec);
  }
}
```

---

## PROS (Open-Source Alternative to VEXcode)

PROS is a community-maintained C/C++ framework with more advanced features:

### Setup
```bash
# Install PROS CLI
pip install pros-cli

# Create project
pros conductor new-project my-vex-project v5
```

### PROS vs VEXcode Key Differences
| Feature | VEXcode | PROS |
|---------|---------|------|
| Task management | Limited | Full FreeRTOS tasks |
| OkapiLib | No | Yes (built-in) |
| Git-friendly | Poor | Excellent |
| Community support | Official VEX | Community-driven |
| Debugging | Basic | Advanced (printf) |

### OkapiLib (PROS) — Chassis Controller

```cpp
#include "okapi/api.hpp"
using namespace okapi;

auto chassis = ChassisControllerBuilder()
  .withMotors(1, -2)          // left port 1, right port 2 (reversed)
  .withDimensions(AbstractMotor::gearset::green, {{4_in, 11.5_in}, imev5GreenTPR})
  .build();

// Drive straight
chassis->moveDistance(12_in);
chassis->turnAngle(90_deg);

// Drive with velocity
chassis->setMaxVelocity(100);
chassis->moveDistance(24_in);
```

---

## Python (MicroPython) on VEX V5

VEXcode V5 also supports Python for teams preferring Python:

```python
from vex import *

brain = Brain()
left_motor  = Motor(Ports.PORT1, GearSetting.RATIO_18_1, False)
right_motor = Motor(Ports.PORT2, GearSetting.RATIO_18_1, True)
controller  = Controller(PRIMARY)

def autonomous():
    left_motor.spin_for(FORWARD, 360, DEGREES, 80, PERCENT, False)
    right_motor.spin_for(FORWARD, 360, DEGREES, 80, PERCENT)

def user_control():
    while True:
        left  = controller.axis3.position()
        right = controller.axis2.position()
        left_motor.spin(FORWARD, left, PERCENT)
        right_motor.spin(FORWARD, right, PERCENT)
        wait(20, MSEC)

comp = Competition(user_control, autonomous)
```

### Python PID Drive (Micropython)

```python
from vex import *

WHEEL_CIRCUMFERENCE_CM = 10.21  # adjust for your wheel size
INTEGRAL_LIMIT = 50.0
MAX_ITERATIONS = 500

def pid_drive(inches, speed=70):
    """PID-controlled straight drive in Python (Micropython on VEX V5)."""
    target_deg = (inches * 2.54 / WHEEL_CIRCUMFERENCE_CM) * 360
    kP, kI, kD = 0.5, 0.001, 0.2
    integral, prev_error = 0, 0

    left_motor.reset_position()
    right_motor.reset_position()

    for _ in range(MAX_ITERATIONS):
        avg_pos = (left_motor.position(DEGREES) + right_motor.position(DEGREES)) / 2.0
        error = target_deg - avg_pos

        integral = max(-INTEGRAL_LIMIT, min(INTEGRAL_LIMIT, integral + error))
        derivative = error - prev_error

        power = kP * error + kI * integral + kD * derivative
        power = max(-100.0, min(100.0, power))

        left_motor.spin(FORWARD, power, PERCENT)
        right_motor.spin(FORWARD, power, PERCENT)

        prev_error = error
        if abs(error) < 5:
            break
        wait(20, MSEC)

    left_motor.stop(BRAKE)
    right_motor.stop(BRAKE)
```

> **Note:** Python PID runs noticeably slower than C++ due to interpreter overhead.
> For competition-level performance, use the C++ PID above. Python is best for
> learning, prototyping, or simpler timed-based autonomous routines.

---

## Debugging & Tuning Tips

### Brain Screen Logging
```cpp
Brain.Screen.clearScreen();
Brain.Screen.setCursor(1, 1);
Brain.Screen.print("Heading: %.1f", InertialSensor.heading());
Brain.Screen.newLine();
Brain.Screen.print("Left Enc: %.0f", LeftMotor.position(degrees));
```

### Controller Rumble
```cpp
Controller1.rumble(".-"); // Short long pattern (Morse-like)
Controller1.Screen.clearScreen();
Controller1.Screen.print("Autonomous Done!");
```

### Common Tuning Workflow
1. Start with `kP` only, increase until oscillation begins
2. Add `kD` to dampen oscillation (typically `kD = kP * 3`)
3. Add small `kI` only if steady-state error persists
4. Test on actual field tiles (friction varies on carpet vs. tile)
