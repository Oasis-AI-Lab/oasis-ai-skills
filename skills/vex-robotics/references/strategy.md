# VEX Robotics Competition — Strategy & Robot Design Reference

## Robot Design Process (Engineering Design Process)

### Phase 1: Game Analysis
1. Read the full Game Manual carefully (published on vexrobotics.com)
2. Identify all scoring mechanisms and point values
3. Calculate maximum theoretical score
4. Identify the highest-value actions per unit time
5. Analyze field layout and traffic patterns

### Phase 2: Strategy Brainstorm
- What is the minimum viable robot that can score?
- What scoring actions are most efficient (points/second)?
- What can one robot do vs. needing alliance partner?
- Is defense viable or does it hurt both alliances?
- What does the autonomous period allow?

### Phase 3: Design Selection
Use a **Decision Matrix**:

| Design Option | Points Potential | Reliability | Build Complexity | Time to Build | Score |
|---------------|-----------------|-------------|------------------|---------------|-------|
| Option A      | 5               | 4           | 3                | 4             | 16    |
| Option B      | 4               | 5           | 4                | 5             | 18    |
| Option C      | 3               | 3           | 5                | 3             | 14    |

Weights can be applied to columns based on team priorities.

---

## Common Robot Subsystems

### Drive Train Types

| Type | Pros | Cons | Best For |
|------|------|------|----------|
| Tank (2-motor) | Simple, reliable | No strafing | Most situations |
| X-Drive (4-wheel omni) | Strafes freely | Less pushing power | Agility-focused games |
| Mecanum (4-wheel) | Full strafing | Complex, slips | Precision placement |
| H-Drive (3-axis) | Strafes with center wheel | Maintenance | Moderate agility |
| 6-motor tank | Maximum pushing force | Uses many motor ports | Defense-heavy games |

### Gear Ratios for Drive (VEX V5 Smart Motor)

Speed formula: `RPM × π × wheel_diameter_inches × 2.54 cm/in ÷ 60 = m/s`
Base: 4" (102 mm) wheel → circumference ≈ 31.9 cm

| Cartridge | Motor RPM | 4" Wheel Speed | Best For |
|-----------|-----------|---------------|----------|
| **Green** | 200 RPM | **~1.06 m/s** | Heavy defense robots, high-push-force drives |
| **Blue** | 600 RPM | **~3.19 m/s** | Scoring-focused robots, fast manipulators |
| **Red** | 100 RPM | **~0.53 m/s** | High-torque lifts, arms, mechanisms (not drives) |

> The original table listed ~1.7 m/s for 200RPM — that was incorrect.
> Actual: 200 × 31.9 ÷ 60 ≈ 1.06 m/s (based on direct-drive 4" wheels).
> The ~333 RPM row used a different (non-standard) internal gear ratio assumption.

Rule of thumb for VRC drives:
- **200 RPM Green** = reliable pushing power, slower cycle time
- **600 RPM Blue** = fast cycles, but noticeably less push force — avoid on defense-heavy bots

### Intake Mechanisms
- **Roller intake**: High volume, continuous — ideal for ball/ring games
- **Claw/grip**: Precise single-object pickup — ideal for large objects
- **Conveyor**: Moves objects vertically — combines with roller intake
- **Pneumatic claw**: Fast, reliable grip — requires pneumatic tank management

### Scoring Mechanisms
- **Catapult**: High-arc scoring, repeatable — needs good tensioning
- **Flywheel**: Fast fire rate, adjustable power — complex PID tuning
- **Lift (4-bar, 6-bar, DR4B)**: Precise height control
- **Chain bar**: Linear reach extension

### DR4B (Double Reverse 4-Bar) Lift
A common high-reach mechanism:
- Maintains horizontal platform orientation through full range
- Requires careful counterbalancing
- Use rubber bands to offset weight at each stage
- Gear down significantly (torque > speed for lifts)

---

## Autonomous Strategy

### Autonomous Win Point (AWP) — Priority Decision
Analyze the game manual to determine AWP requirements. Common patterns:
- Touch a specific object/zone at match end
- Score a certain number of objects
- Complete a specific task AND win auton

Teams should choose between:
1. **Safe AWP routine**: Low risk, guaranteed AWP (worth it if reliably achievable)
2. **Aggressive scoring**: Maximize auton points even if AWP is missed

### Autonomous Path Planning
- **Odometry**: Track (x, y, heading) using wheel encoders + inertial sensor
- **Pure Pursuit**: Smooth path following along pre-defined waypoints
- **Motion Profile**: Trapezoidal velocity curves for precise distance moves

### Autonomous Routine Length Categories
- **Simple (< 30 sec dev time)**: Drive into zone, touch object — good for new teams
- **Moderate**: Score 2–3 objects, consistent route
- **Advanced**: Full field autonomous, 5+ scored objects, uses odometry

### Selecting Autonomous at Tournament
Many teams program multiple autonomous routines and select via controller at match start:
```cpp
// In autonomous()
// IMPORTANT: The driver must be HOLDING the button at the moment autonomous
// begins (during the pre-match countdown). Pressing after auton starts is too late.
if (Controller1.ButtonA.pressing()) { routine_red_side(); }
else if (Controller1.ButtonX.pressing()) { routine_blue_side(); }
else { default_routine(); }
```

---

## Alliance Selection Strategy

### As a High Seed (Top 5 Picks)
- Prioritize partners who complement your weaknesses
- Check: consistent autonomous, can they play defense if needed?
- Avoid picking teams who will "flip" to higher seed alliance
- Look at skills scores for overall capability signal

### As a Lower-Ranked Team (Getting Picked)
- Be approachable and visit pits of top teams before selection
- Highlight your specific strengths (fast driver, reliable auton, skills score)
- Demonstrate your robot performs consistently
- Offer to run skills or show autonomous to interested teams

### Red Flags When Evaluating Partners
- High variance match scores (inconsistent)
- Frequent autonomous failures
- Pinning/entanglement violations in matches
- Uncooperative attitude

---

## Competition Day Checklist

### Morning Setup
- [ ] Charge all batteries (3–4 match batteries + 1 controller battery)
- [ ] Robot passes inspection (size, weight, legal components)
- [ ] Test autonomous routine on actual field tiles (friction matters!)
- [ ] Confirm driver practice / test drive

### Pre-Match Routine
- [ ] Charge battery to full (use fresh battery each match if possible)
- [ ] Confirm autonomous selection method works
- [ ] Communicate strategy with alliance partner
- [ ] Scout upcoming opponents during previous matches

### Post-Match Analysis
- [ ] Note what scored and what failed
- [ ] Log match score and time of scoring events
- [ ] Identify mechanical issues for pit repair
- [ ] Update scouting data

---

## Scouting System

### Data to Track Per Team
- Match scores (average, max, min, variance)
- Autonomous reliability (0–1 per match)
- Scoring objects per match
- Defense attempts / violations
- Skills scores (Autonomous + Driver)

### Scouting Sheet Template (Simplified)
```
Team: ___    Match #: ___    Alliance: Red / Blue
Auton: Success / Fail / Partial    Auton Pts: ___
Drive Pts: ___    Notes: _______________
Fouls/DQ: Y / N
```

### Analysis Methods
- Average match score → Overall offensive output
- Score variance → Consistency
- Max score → Ceiling capability
- Auton success rate → Reliability of autonomous win point

---

## Pit Strategy

### Common Quick Repairs
- **Motor replacement**: Keep spare motor + gear cartridge
- **Pneumatic leak**: Carry extra fittings and tubing; check before each match
- **Battery connector**: Have spare battery connectors (XT30 or equivalent)
- **Screw/standoff**: Carry a complete fastener kit

### Between-Match Priority Order
1. Fix anything that broke during the match
2. Recharge or swap battery
3. Verify autonomous still works after repairs
4. Driver practice if time permits

---

## Remote Invitational / Online Challenges (VEX)

### STEM Research Project
- Topic changes each season
- Paper format: ~1500–2000 words
- Judged on: research quality, citations, real-world application

### Online Challenge Categories
- Robot Showcase (video of robot in action)
- Team Spotlight (highlight video)
- Coding / Programming challenge
- VEX U engineering challenge

Submit via: robotevents.com or the official online challenge portal
