# Storm that Castle: Team 10 Autonomous Ball-Launching Robot

## Project Introduction

This project was developed by Team 10 for the ME 588 "Storm the Castle" competition. The goal of the competition was to build a compact autonomous robot capable of identifying buckets controlled by the opposing team and launching balls into those targets. The course rules defined the robot size limit, minimum design requirements, and competition constraints; the team's work focused on turning those requirements into a reliable mechatronic system.

Team 10's robot combines a line-following mobile base, IR beacon sensing, a servo-actuated launcher, ball loading feedback, status indicators, and an ESP32-S3 control system. The overall strategy was to move along the field line without spinning the entire robot, detect crosslines that correspond to hill or bucket positions, read the IR beacon frequency at each hill, and fire only when the hill belonged to the opposing team.

The project report identifies the team members as Christian Scott, David Lu, Gurman Singh, Heng Zheng, Jau-Shiuan Shiao, and Leonardo Franquilino.

## Competition Strategy

The robot was designed to travel back and forth along a line parallel to the target buckets. Instead of rotating to face each target, the chassis and launcher layout allow the robot to strafe along the path while keeping the launcher aligned toward the bucket direction. Crossline sensors are positioned in line with the launcher so the robot can stop or react when it reaches a firing location.

At each crossline, the robot checks the hill allegiance through an IR sensing circuit:

- A 750 Hz beacon corresponds to the Blue team.
- A 1.5 kHz beacon corresponds to the Red team.
- If the beacon belongs to the robot's team, the robot continues to the next target.
- If the beacon belongs to the opposing team, the robot loads and launches a ball.

This strategy keeps the robot behavior simple and repeatable: follow the line, count junctions, check ownership, shoot only when useful, and reverse direction at the end of the corridor.

## System Architecture

The robot is controlled by an Espressif ESP32-S3 microcontroller. The report notes that this controller was chosen for its processing capability, integrated Wi-Fi and Bluetooth, large GPIO count, lower cost compared with many Arduino boards, and flexibility for the number of sensors and actuators used in the robot.

The embedded firmware in `Code/ESP32_S3_arduino/` is organized as a FreeRTOS-based Arduino application. The top-level sketch initializes each subsystem and then relies on tasks, queues, timers, and a hierarchical finite state machine to coordinate behavior.

Major firmware modules include:

- `hfsm.cpp` and `state.cpp`: hierarchical event-driven finite state machine.
- `navigation.cpp`: line following, junction detection, PI/PID-style motion correction, and location updates.
- `mobility.cpp`: DC motor direction and PWM control.
- `ir_beacon_detect.cpp`: ESP32 pulse counter measurement for 750 Hz and 1.5 kHz beacon classification.
- `ball_launcher.cpp`: servo sequencing for loading, pullback, release, and reload timing.
- `team_status.cpp` and `game_status.cpp`: team selection and game active state management.
- `user_interface.cpp`: Wi-Fi web interface, live state updates, event logging, and debug controls.
- `manual_control.cpp`: manual motion support during testing.

## Sensors and Actuators

The robot integrates several sensing systems:

- Ten total reflectance sensors: two arrays of four sensors for line following and two additional sensors for crossline detection.
- A break-beam sensor to confirm whether a ball is loaded.
- A TEKT5400S IR phototransistor circuit for measuring beacon blink frequency.

The main actuators are:

- Two 12 V DC motors for wheel drive.
- Two servo motors, one for feeding balls and one for actuating the launcher.

The robot also includes a game start button, an encoder switch used for team color selection, an emergency stop switch, a green game-mode LED, the ESP32-S3 onboard RGB LED for team color, and a white LED that indicates when the shooting mechanism is active.

## IR Beacon Detection

A central challenge in the project was reliably reading the hill IR beacons from a distance. The raw phototransistor signal is weak and changes with distance, so the team designed a two-stage signal conditioning circuit.

The first stage uses an inverting amplifier with a 2.5 V virtual ground to amplify the received signal. The second stage acts as a comparator, converting the amplified waveform into a clean digital pulse train. The report describes additional high-pass and low-pass filtering that rejects low-frequency lighting noise and higher-frequency interference while preserving the 750 Hz and 1.5 kHz beacon signals.

The conditioned output is compatible with the ESP32-S3's 3.3 V logic and is processed in firmware using the pulse counter peripheral over a fixed sampling window. This lets the robot classify hill allegiance as Blue, Red, or Unknown and pass that result into the finite state machine.

## Launcher and Ball Handling

The launcher design evolved during development. The team initially considered a flywheel launcher, but testing showed that it consumed too much power and produced inconsistent shots because the balls varied slightly in size and surface condition. The final design moved to a rail-gun or trebuchet-style spring-loaded launcher with a ratchet-like pullback mechanism.

In the final mechanism, a servo pulls the cradle back and releases it to shoot. A toothed rack and spring-loaded pawl prevent the mechanism from slipping backward, giving the launcher more repeatable stored energy before release. The pullback and release positions are tuned in code, which makes shot strength adjustable without rebuilding the mechanism.

Ball feeding is handled by a slanted hopper and side-loading path. Balls roll naturally into the loader catch, and a rotating loader plate moves a ball into the launch position. A break-beam sensor reports whether the ball is actually present; if loading fails, the firmware treats the bucket as empty and sends the robot into a reload behavior.

## Chassis and Motion System

The final chassis used a larger fabricated aluminum base plate instead of the initially planned pre-made base. This provided more space for the launcher, sensors, motors, electronics, and power components. Two 12 V motors drive the robot, while caster wheels provide additional stability.

The motor mounts were 3D printed and also helped position the crossline sensors. The L298 H-bridge motor driver was mounted below the aluminum plate to save top-side space. A 3D printed battery and power-system mount consolidated the buck converter, distribution module, and emergency stop switch.

The firmware tracks robot location between junctions and robot heading. When the robot reaches the end of the corridor, it reverses heading and continues navigating back through the same junction sequence. This allows the robot to patrol between the main hill locations without needing a full body rotation.

## Finite State Machine

The control software uses a hierarchical finite state machine that stores the robot's current state, team, beacon state, location, and heading. Sensor readings, timer callbacks, button presses, and interface commands are converted into events and sent through a shared event queue.

The main state hierarchy is:

```text
Startup

GameInactive
|-- Idle
|-- ManualControl
|-- Error

GameActive
|-- Navigation
|   |-- MoveToNextJunction
|
|-- HillInteraction
|   |-- CheckHillLoyalty
|   |-- BallLoading
|   |-- BallLaunching
|
|-- BackHome
|-- WaitBucketReload
```

This structure keeps repeated behavior in parent states while leaving specific actions in leaf states. For example, game-wide timeout handling can live at a higher level, while ball loading, launch completion, and bucket-empty responses remain tied to their specific states.

## Web Interface and Debugging

The ESP32-S3 firmware includes a Wi-Fi web interface using `ESPAsyncWebServer`. The interface reports the current FSM state, team, beacon state, location, heading, and transition log. It also uses server-sent events so the displayed state can update live while the robot runs.

Serial commands and manual control support were included for commissioning. These debug tools make it possible to simulate junction crossings, force selected state transitions, print Wi-Fi information, and test movement or launcher behavior without running a complete competition attempt.

## Submitted Files

This submission package includes:

- `ME_588___Project_Report.pdf`: full project report with strategy, design decisions, figures, results, discussion, and conclusion.
- `CAD/FinalCarAssem.STEP`: final mechanical assembly model.
- `Circuit Diagrams/FinalProject.kicad_sch`: overall electrical schematic.
- `Circuit Diagrams/IR_sensing_Circuit Diagram.kicad_sch`: IR sensing and conditioning schematic.
- `Circuit Diagrams/DisplayCircuit.kicad_sch`: display and indicator circuit schematic.
- `Circuit Diagrams/Launcher.kicad_sch`: launcher circuit schematic.
- `Code/ESP32_S3_arduino/`: main ESP32-S3 robot firmware.

## Results and Lessons Learned

The report states that the robot completed the required design, circuit, and final checkoffs. The launcher was able to shoot consistently, the IR detection system functioned properly, the display requirements were demonstrated, and the robot met the 12 x 12 x 12 inch size constraint.

During the final competition, the launcher and IR detection subsystems worked, but line following and cross-junction sensing were less reliable on the field than expected. The robot was able to drive toward the hill and attempt a shot, but inconsistent reflectance sensor readings affected positioning and alignment. The main improvement identified by the team was additional tuning of the reflectance sensor thresholds and PI control logic for the actual game surface.

Overall, the project demonstrates a complete mechatronic design process: mechanical packaging, custom sensing circuitry, embedded firmware, real-time control, autonomous decision-making, and physical system commissioning. Even with final navigation challenges, the robot showed that the core subsystems could work together and that further tuning would likely produce a fully reliable competition run.

