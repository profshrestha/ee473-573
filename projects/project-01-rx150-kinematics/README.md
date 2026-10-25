# Project 1: Kinematics of the RX150

**Individual project, in a shared repository.** Each student produces their own derivation and
their own implementation, in their own directory at `students/<your-username>/`. The hardware
measurement session of Part D is done in teams of three, since the arms are time-shared and
measuring takes more than one person; the measurements are recorded in a single shared file and
analyzed separately by each team member.

Check Canvas for deliverables, deadlines, and the grading rubric.

## 1. Objectives

Upon completion of this project, students will be able to:

- Assign coordinate frames to a physical manipulator and derive its Denavit-Hartenberg parameter
  table.
- Implement forward kinematics from that table and verify the implementation against three
  independent sources.
- Implement inverse kinematics for a manipulator with fewer than six degrees of freedom,
  accounting for multiple solutions, joint limits, and unreachable targets.
- Command a physical manipulator, measure the resulting end-effector position, and account for the
  difference between the kinematic model and the machine.

## 2. Scope of the implementation

The deliverable is a kinematics library, not a script. Its organization is prescribed below and
forms part of the grade:

```
students/<your-username>/      individual work; teammates have their own directories alongside
  kinematics/     pure mathematics; imports numpy and nothing further
    transforms.py rotation matrices, homogeneous transforms, helpers
    fk.py         DH table to A matrix, forward kinematics, and the PoE check
    ik.py         inverse kinematics
  robot/
    rx150.py      the ONLY file permitted to import a robot SDK or address hardware
  tests/
    test_roundtrip.py
  scripts/
    verify_fk.py  harness for the three forward kinematics checks of Part B
  REPORT.md       the report; this is the deliverable the reader engages with
  figures/        photographs and plots referenced by the report

data/
  partD-measurements.csv   shared within the team: one session, three analyses
```

This structure is already in the repository, with each function stubbed. Complete the stubs; do not
reorganize the package.

**One repository per station, one directory per student.** The implementation is individual: each
student produces their own frame assignment, parameter table, kinematic routines, and report. The
repository is shared so that the Part D measurement session has a single home, and so that three
independently derived models can be compared against one set of measurements. Work in your own
directory and do not copy between them.

**No module in `kinematics/` may import a robot driver, ROS, or any hardware-specific package.**
The kinematic routines take numbers and return numbers. Code specific to the arm belongs in
`robot/`, whose job is to take a vector of joint angles and send it.

This separation is not a matter of style. In Project 2 the same library is extended to a different
manipulator with an extra joint and a completely different software stack. Both arms come from the
same manufacturer and still share almost nothing at the software level: the RX150 is commanded
through the Interbotix ROS packages, while the WidowX AI is addressed by a Python driver over
Ethernet. Where the kinematics are tangled up with the driver, that extension is a rewrite; where
they are separated, it is one more file.

## 3. Setup

1. **Accept the GitHub invitation** sent to your `@sonoma.edu` address. It makes you a member of
   the `SSU-ISL` organization, which holds one private repository named for your station and shared
   with your station team. The structure described above is already in place.

   The invitation can only be accepted by a GitHub account with your `@sonoma.edu` address verified
   on it. If you already have an account under a personal address, add the SSU address first under
   Settings, Emails. Invitations expire after seven days.

2. Clone the repository to your station and change into your own directory.

   ```bash
   git clone https://github.com/SSU-ISL/ee473-2026f-p1-station-NN.git
   cd ee473-2026f-p1-station-NN/students/YOUR-USERNAME
   ```

   Your username is the part of your SSU address before the `@`.

3. **Run the test suite.** The station has everything you need, active in any new terminal: ROS 2
   Jazzy, the Interbotix packages, and a Python environment with NumPy, `modern_robotics`, PyTorch,
   and the rest of the dependencies.

   ```bash
   pytest -v        # 14 failures, all NotImplementedError
   ```

   Run it from your own directory rather than from the repository root. Three copies of the same
   package sit alongside one another, and collecting them all at once confuses the import
   machinery.

   If `pytest` is not found, open a new terminal or run `source /opt/isl-venv/bin/activate`.

   The `requirements.txt` in the repository documents what the station provides. Its PyTorch is
   pinned to the CUDA 12.8 build the RTX 5080 needs.

4. Confirm that the RX150 description launches and that the arm shows up in RViz before writing any
   kinematic code.

5. **Codex CLI is available** for students who want an assistant at the terminal. Run `codex login`
   once, then run `codex` from inside the repository. Run `codex logout` when you leave the
   station, which is shared.

> **Credentials are never committed.** No tokens, no `.env` files, no API keys. The `.gitignore`
> covers the common cases; check with `git status` before committing.

## 4. Working with the manipulator

Read this section before handling an arm. It applies from Part A onward.

The ROS 2 mechanics, launching the description and the control stack, reading what RViz shows,
torque control, the RViz panel, and the Python API, are documented in
**[ROS 2, RViz and the RX150](../../lecture-resources/ros2-rx150/README.md)**. Work through that
material once. What follows is specific to this project.

### 4.1 Hardware checkout

The RX150 arms are located in **Salazar 2008**. Each session needs four items:

| | |
|---|---|
| **Arm** | An RX150 |
| **Power adapter** | The arm's own 12 V supply |
| **USB cable** | USB to micro USB. Three dedicated cables remain; any USB to micro USB cable works if these are unavailable |
| **Clamp** | For securing the arm to the table |

Set up at the station in this order:

1. **Clamp the arm to the table.** The base has to be fixed before power is applied. An unclamped
   arm will walk across the work surface under its own reaction torques.
2. **Clear the workspace.** The RX150 reaches about 450 mm in every direction; that radius has to
   be free of laptops, bottles, keyboards, and hands.
3. Connect the power adapter, then the USB cable to the station.
4. Confirm that the arm is detected:

   ```bash
   ls /dev | grep ttyDXL
   ```

   An empty result means that either the USB or the power connection has not been made. Everything
   after this depends on it.

**When finished, return the arm, adapter, cable, and clamp to storage.** The three cables in
particular are shared among eight teams.

**Log out, and leave the station running.** Logging out ends your session and protects your files
while leaving the machine itself powered. The stations are maintained and updated remotely, and one
that has been powered off is unreachable: it misses that maintenance, and the next user finds it
broken. Leave the station at the login screen.

### 4.2 Releasing the servos

**The arm falls when its motors are released.** This happens in two situations, both of which come
up in this project:

- Disabling the motor torque through a service call
- Stopping the control launch with Ctrl + C

In either case, **command the arm to its Sleep Pose first, or hold it by hand.** Sleep is the
folded configuration where the arm rests on itself and has nowhere to fall. Make it the last action
of every session.

If the arm starts to fall, disconnect power rather than trying to catch it. A falling arm carries
more momentum than you expect, and a hand in its path becomes a pinch point.

### 4.3 Launch configuration by part

| Part | Launch | Purpose |
|---|---|---|
| A, B | `xsarm_description.launch.py` | The model and the slider interface, without hardware. The sliders are calibrated in radians and their end stops match the physical joint limits, which makes this the cheapest way to find out whether a derived configuration is reachable |
| C | none | Inverse kinematics is purely computational; verification is by the round-trip test |
| D | `xsarm_control.launch.py` | The driver and the physical manipulator |

```bash
ros2 launch interbotix_xsarm_descriptions xsarm_description.launch.py \
  robot_model:=rx150 use_joint_pub_gui:=true

ros2 launch interbotix_xsarm_control xsarm_control.launch.py robot_model:=rx150
```

Backdriving the arm by hand through the five Part D configurations, with the motors disabled, shows
quickly which ones are awkward or unreachable. It is worth doing before commanding them.

### 4.4 Placement of hardware code

All hardware code belongs in the `Arm` class in `robot/rx150.py`. That file is the only one in the
project allowed to import a robot SDK, and the boundary forms part of the grade.

Implement `__enter__` and `__exit__` using `InterbotixManipulatorXS`, `robot_startup()`, and
`robot_shutdown()`, then `send()`, `read()`, and `driver_ee_pose()`. Keep the Interbotix imports
inside the method bodies: the test suite has to run on a machine with no arm attached, and a
module-level import makes `pytest` fail during collection for reasons unrelated to the code being
tested.

Used as a context manager, the class restores the arm even when a script raises partway through:

```python
from robot.rx150 import Arm

with Arm() as arm:
    arm.send(my_solution)
    measured = arm.read()
    print(arm.driver_ee_pose())
```

Three practical points.

**Angles are in radians**, ordered by `JOINT_NAMES`. Use `math.radians()` if you prefer degrees.

**`driver_ee_pose()` returns the manufacturer's forward kinematics, not yours.** It gives a check on
`fk()` that needs no measurement, and it separates model error from measurement error. It does not
replace the physical measurement Part D requires, since the driver's model and the physical arm are
themselves different.

**Start with small motions near the home configuration.** Increase `moving_time` while getting used
to the system. A single large command follows the shortest path in joint space, which is usually not
the path you pictured, and the arm may pass through the work surface on the way.

## 5. Part A: Frames and the Denavit-Hartenberg table

A Denavit-Hartenberg parameter table means nothing without a stated base frame and tool frame. Both
are prescribed here rather than chosen by the student:

- **Base frame: `base_link`.** Origin at the mounting surface, axes following the ROS convention,
  **x forward, y left, z up**, with z along the waist rotation axis.
- **Tool frame: `ee_gripper_link`.** This frame is not at the final joint. Four **fixed** joints sit
  between the last revolute joint and the tool point, and the chain has to include all of them.

Proceed as follows:

1. Using the frame assignment procedure from lecture, attach a frame to each link of the RX150. Do
   this as a hand drawing and photograph or scan it; the drawing is part of the submission.
2. Derive the Denavit-Hartenberg parameter table, identifying the joint variable in each row.
3. **Cross-check the dimensions against the URDF**, as described below.

### 5.1 Reading the URDF

> **Preliminary reading:** [The Universal Robot Description Format
> (PDF, 7 pages)](references/lynch-park-4.2-urdf.pdf)
>
> It covers exactly the task at hand: the meaning of `<joint>`, `<parent>`, `<child>`, `<origin>`
> and `<axis>`, why a URDF describes a robot as a tree rather than a chain, why most links carry two
> frames rather than one, and a fully annotated URDF for the UR5 shown next to a diagram of its
> frames.
>
> From Lynch and Park, *Modern Robotics*, §4.2, pages 152 to 158. Cambridge University Press, 2017.
> The full text is freely available at [modernrobotics.org](http://modernrobotics.org) and is worth
> having: §4.1 covers the product of exponentials, Appendix C the Denavit-Hartenberg convention, and
> Chapter 5 velocity kinematics and the Jacobian, which Project 3 takes up.
>
> Two points from that reading carry straight into this project. An `<origin>` gives the pose of the
> **child** link's frame in the **parent's** frame when the joint variable is zero, which is the
> point developed below. And an `<axis>` is a unit vector in the **child** link's frame rather than
> the base frame, which is why the axes cannot be read off and used as they stand.

The description lives in the `interbotix_xsarm_descriptions` package as `rx150.urdf.xacro`. It is a
**xacro** file rather than plain URDF, so it has unresolved `$(arg ...)` substitutions, and link
names appear as `$(arg robot_name)/base_link`. Either substitute mentally while reading or run the
file through `xacro` to get plain URDF, which is easier.

What to read, and what to skip:

- **Read** the `<joint>` blocks. Inside each, `<parent>`, `<child>`, `<origin>`, and `<axis>` are
  the kinematics; nothing else in the file is.
- **Skip** every `<visual>`, `<collision>`, and `<inertial>` block. Those describe appearance and
  mass distribution rather than the geometry of the chain.

Three features of this file deserve attention:

- `base_link` carries a **visual** origin with a nonzero `rpy`. That rotates the mesh so the model
  renders correctly on screen. It is not a kinematic transformation and must not appear in the
  parameter table.
- Inside a joint block in this file, `<axis>` comes before `<origin>`. Taking the first `xyz` you
  see gives the rotation axis instead of a translation.
- One joint in the gripper is a **branch** rather than part of the chain to the tool point. Follow
  the parent and child links to the tool frame and ignore anything that leaves that path.

### 5.2 The kinematic chain

The chain is transcribed below from that file, so that twenty-one readers do not each misread it a
different way. **Verify the transcription against the file yourself.** It is reference data rather
than a solution: turning these offsets into Denavit-Hartenberg parameters is still entirely your
work.

| Joint | Type | Axis | Origin xyz (m) | Parent link to child link |
|---|---|---|---|---|
| waist | revolute | `0 0 1` | 0, 0, 0.06566 | base_link to shoulder_link |
| shoulder | revolute | `0 1 0` | 0, 0, 0.03891 | shoulder_link to upper_arm_link |
| elbow | revolute | `0 1 0` | 0.05, 0, 0.15 | upper_arm_link to forearm_link |
| wrist_angle | revolute | `0 1 0` | 0.15, 0, 0 | forearm_link to wrist_link |
| wrist_rotate | revolute | `1 0 0` | 0.065, 0, 0 | wrist_link to gripper_link |
| ee_arm | fixed | | 0.043, 0, 0 | gripper_link to ee_arm_link |
| gripper_bar | fixed | | 0, 0, 0 | ee_arm_link to gripper_bar_link |
| ee_bar | fixed | | 0.023, 0, 0 | gripper_bar_link to fingers_link |
| ee_gripper | fixed | | 0.027575, 0, 0 | fingers_link to ee_gripper_link |

**Every `rpy` in the chain is `0 0 0`**, which is why there is no column for it. The description is
all translations, and the joint axes account for all of the orientation.

The last four rows need careful reading. All are fixed, none carries an axis, and together they add
**0.093575 m** between the last revolute joint and the tool frame. Leaving any one of them out makes
the model short by that amount at every configuration.

Recall what a joint origin means: it is the transformation **from the parent link's frame to the
child link's frame**, not a position expressed in the base frame. The waist entry of
`0, 0, 0.06566` puts `shoulder_link` 65.66 mm above the mounting surface; it does not move the waist
rotation axis, which stays on the z line through the base origin. To get quantities in base
coordinates, accumulate the offsets along the chain: the shoulder axis, for example, sits 104.57 mm
above the mounting surface.

### 5.3 Comparison

The URDF does not use Denavit-Hartenberg parameters, so its rows will not line up with those of the
derived table. What has to agree is the **physical geometry**: link lengths, offsets, and the total
distance from base to tool. A disagreement there means an error in the derived table, and every
later result will inherit it.

> Nothing in the URDF says that x is forward. The root frame is the reference, and the joint
> attaching it to `world` is an identity transformation. The convention comes from ROS REP 103 and
> is made concrete by the link offsets: the arm stacks along `+z` and extends along `+x`. This is
> worth understanding rather than memorizing, since there are manipulators whose authors did not
> follow the convention.

## 6. Part B: Forward kinematics

Implement `fk(thetas) -> H`, taking five joint angles and returning the $4 \times 4$ pose of the
gripper frame expressed in the base frame.

Verify the implementation three independent ways. All three must agree before going on.

1. **Against RViz.** Publish a set of joint angles, let the robot state publisher position the arm,
   and compare the reported gripper pose against the computed `H`. This tests the derived table
   against an independent implementation of the same arm.
2. **Against the URDF chain.** Multiply the joint origin transformations from the URDF directly and
   confirm that the result agrees. This tests the derived table against the manufacturer's own
   description.
3. **Against the product of exponentials.** Express the screw axis of each joint in the base frame
   and compute the same pose with the PoE formula. This tests the derived table against a method
   that needs no frame assignment, so an error in frame assignment cannot be hiding in both.

Report all three comparisons. If any of them disagree, that disagreement is the result, and it
should be resolved before going on.

## 7. Part C: Inverse kinematics

Read this section in full before implementing anything.

The RX150 has five joints. A general pose in space needs six parameters, three for position and
three for orientation. Five joints cannot satisfy six constraints, so **for an arbitrarily specified
$4 \times 4$ matrix no solution exists**. An inverse kinematic routine that accepts an arbitrary `H`
therefore asks a question the arm cannot answer.

Consider what each joint contributes:

- The **waist** rotates the whole arm about the vertical axis. It sets the vertical plane the
  remaining joints work in, and that plane is fixed once `x` and `y` are given.
- The **shoulder, elbow, and wrist angle** have parallel axes and so form a planar three-link chain
  inside that plane. They supply radial distance, height, and the pitch of the approach.
- The **wrist rotate** spins the gripper about its own approach axis, supplying roll.

The arm therefore commands position, pitch, and roll, and **yaw is not available as a free
parameter**. The approach direction has to lie in the vertical plane through the waist, and that
plane is set by the commanded position.

The solver accordingly finds **four joint variables from four constraints**: `x`, `y`, `z`, and
pitch. Roll is not solved for. The caller supplies it and it passes straight to the wrist rotate
joint, since nothing else in the chain depends on it.

Implement `ik(x, y, z, pitch, roll=0.0) -> list of solutions`, with this behavior at the edges of
the workspace:

- **Return the complete solution set.** Elbow-up and elbow-down are both valid, and the waist has a
  second branch. A solver that quietly picks one withholds information from its caller.
- **Identify joint limit violations.** A mathematically valid solution the arm cannot physically
  reach is not a usable result. Return it, but marked as such.
- **Detect unreachable targets and report them.** Return an empty set together with a reason. Never
  return a nearby configuration as though it were a solution, and never return `NaN`.

Implement the round-trip test in `tests/test_roundtrip.py`: for a set of reachable targets, run the
inverse kinematics, push each solution back through the forward kinematics, and confirm that the
target comes back within tolerance. This establishes that the solver is correct without any external
reference.

The test itself is retired in Project 2, where the arm changes and the solver changes with it. The
technique is not. Project 3 compares two independent solvers against each other, for the same reason
that one is compared against itself here.

> A successful round trip shows that a solution is *valid*. It does not show that the complete set
> was found, nor that the arm can reach the solution. The two requirements above cover those
> questions.

## 8. Part D: Experimental verification

This part is done in teams of three sharing one arm. All three members submit the same measurement
data; **each member produces an independent analysis against their own forward kinematics.**
Predictions will differ where the derived tables differ, and comparing three independent predictions
against one set of measurements is itself informative.

Divide the roles and rotate them so that each member does each task at least once: one member
commands the arm, one measures, and one records and states the predicted value after the measurement
is taken rather than before.

> **Before commanding any hardware motion:** check the configuration in RViz, command low joint
> velocities, and stay within reach of the power disconnect. If the arm moves toward the work
> surface, disconnect power rather than stepping in physically.

Implement the `Arm` class in `robot/rx150.py` first. It is a context manager, so the arm returns to
the sleep pose even if a script raises partway through, and the Interbotix imports belong inside its
methods so that the test suite still runs with no arm attached.

1. Pick five reachable configurations spread across the workspace, rather than five variations on a
   single configuration.
2. Compute the predicted gripper position for each using your forward kinematics.
3. Command the arm to each configuration and let it settle.
4. Measure the actual gripper tip position relative to the base frame origin with a rule or
   calipers. Settle the measurement procedure in advance and apply it the same way every time.
5. Record the joint angles, the measurements, and any observations in `data/partD-measurements.csv`
   at the repository root. That file is shared: one session, one dataset, three independent
   analyses.
6. In your own report, tabulate predicted against measured values and the error along each axis in
   millimeters.

Record the configuration the servos actually reached using `arm.read()`, not just the commanded
configuration. The two differ under gravitational load, and that difference is one of the
explanations to be weighed.

Account for the difference. The error will not be zero. Candidate explanations include servo
position error under gravitational load, link dimensions that differ from the published values, the
measurement technique, a wrong Denavit-Hartenberg parameter, and the gripper tip not sitting exactly
where the tool frame was defined. Argue from the collected data for the explanations that dominate,
and describe how that claim would be tested.

## 9. Deliverables

All deliverables live in the repository except the oral examination.

- **`REPORT.md`**, the document this project is graded on. The template in the repository defines
  the sections. It presents the photographed frame assignment, the Denavit-Hartenberg table, the
  three forward kinematic verifications together with the output backing them up, the inverse
  kinematic formulation, the table of predicted against measured values, and the error analysis.

  Write the report as a technical document rather than as a form with the blanks filled in. For a
  sense of what to aim at, see [Forward Kinematics of the UR5e: DH and
  PoE](https://www.roboticsunveiled.com/forward-kinematics-ur5e-dh-poe/). That treatment states the
  frame assignment, presents the table, gives the equations, shows the implementing code, and then
  shows the **actual output** establishing that the two methods agree. Every claim is backed by
  evidence the reader can check. The report need not be as long. It should be as rigorous.

  The report is read on GitHub, so its equations have to render there. The template ends with a note
  on the limits of that renderer; read it before writing the mathematics rather than after.
- The `kinematics/` library, with `robot/rx150.py` as the only hardware-aware module.
- A passing round-trip test.
- A brief note in `README.md` describing how the code is run.
- **Oral examination.** Each student will be given a target position and approach angle, asked to
  command the arm to it, and asked to account for the resulting configuration. **The demonstration
  is at your station**, from the code in your repository. Personal machines are not used.

## 10. Remarks

The gripper frame is yours to define, but it has to be defined once and used consistently in the
derivation, the implementation, and the measurements. Most large discrepancies in this project come
from measuring to a point other than the one the model computes.

Joint angle sign conventions and zero configurations are set by the manufacturer rather than by your
drawing. If the forward kinematics is off by a sign at a single joint, look there first.
