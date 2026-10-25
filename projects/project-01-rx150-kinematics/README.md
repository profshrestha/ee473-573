# Project 1: Kinematics of the RX150

**Individual project, conducted within a shared repository.** Each student produces their own
derivation and their own implementation, in their own directory at `students/<your-username>/`.
The hardware measurement session of Part D is conducted in teams of three, as the manipulators
are time-shared and measurement requires more than one person; the resulting measurements are
recorded in a single shared file and analyzed independently by each team member.

Consult Canvas for deliverables, deadlines, and the grading rubric.

## 1. Objectives

Upon completion of this project, students will be able to:

- Assign coordinate frames to a physical manipulator and derive its Denavit-Hartenberg parameter
  table.
- Implement forward kinematics from that table and verify the implementation against three
  independent sources.
- Implement inverse kinematics for a manipulator having fewer than six degrees of freedom,
  accounting for multiple solutions, joint limits, and unreachable targets.
- Command a physical manipulator, measure the resulting end-effector position, and account for
  the discrepancy between the kinematic model and the machine.

## 2. Scope of the implementation

The deliverable is a kinematics library rather than a script. Its organization is prescribed
below and forms part of the assessment:

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

This structure is already present in the repository, with each function stubbed. Complete the
stubs; do not reorganize the package.

**One repository per station, one directory per student.** The implementation is individual: each
student produces their own frame assignment, parameter table, kinematic routines, and report. The
repository is shared so that the Part D measurement session has a single home, and so that three
independently derived models may be compared against one set of measurements. Work within your own
directory and do not copy between them.

**No module within `kinematics/` may import a robot driver, ROS, or any hardware-specific
package.** The kinematic routines accept numerical arguments and return numerical results. Code
specific to the manipulator belongs in `robot/`, whose responsibility is to accept a vector of
joint angles and transmit it.

This separation is not a matter of style. In Project 2 the same library is extended to a different
manipulator possessing an additional joint and addressed through an entirely different software
stack. Both arms originate with the same manufacturer and nevertheless share almost nothing at the
software level: the RX150 is commanded through the Interbotix ROS packages, while the WidowX AI is
addressed by a Python driver over Ethernet. Where the kinematics are entangled with the driver,
that extension constitutes a rewrite; where they are separated, it requires one additional file.

## 3. Setup

1. **Accept the GitHub invitation** sent to your `@sonoma.edu` address. It confers membership in
   the `SSU-ISL` organization, which contains one private repository named for your station and
   shared with your station team. The structure described above is already in place.

   The invitation can be accepted only by a GitHub account on which your `@sonoma.edu` address is
   verified. If an account already exists under a personal address, add the SSU address first
   under Settings, Emails. Invitations expire after seven days.

2. Clone the repository to your station and change into your own directory.

   ```bash
   git clone https://github.com/SSU-ISL/ee473-2026f-p1-station-NN.git
   cd ee473-2026f-p1-station-NN/students/YOUR-USERNAME
   ```

   Your username is the portion of your SSU address preceding the `@`.

3. **Run the test suite.** The station provides everything required, active in any new terminal:
   ROS 2 Jazzy, the Interbotix packages, and a Python environment containing NumPy,
   `modern_robotics`, PyTorch, and the remaining dependencies.

   ```bash
   pytest -v        # 14 failures, all NotImplementedError
   ```

   Invoke it from your own directory rather than from the repository root. Three copies of the
   same package reside alongside one another, and collecting them simultaneously confuses the
   import machinery.

   Should `pytest` not be found, open a new terminal or run `source /opt/isl-venv/bin/activate`.

   The `requirements.txt` in the repository documents what the station provides. Its PyTorch is
   pinned to the CUDA 12.8 build required by the RTX 5080.

4. Confirm that the RX150 description launches and that the manipulator is visible in RViz before
   writing any kinematic code.

5. **Codex CLI is available** for students who wish to use an assistant at the terminal. Run
   `codex login` once, then invoke `codex` from within the repository. Run `codex logout` upon
   leaving the station, which is shared.

> **Credentials are never committed.** No tokens, no `.env` files, no API keys. The `.gitignore`
> excludes the common cases; verify with `git status` before committing.

## 4. Working with the manipulator

This section should be read before handling an arm. It applies from Part A onward.

The ROS 2 mechanics, launching the description and the control stack, the interpretation of what
RViz displays, torque control, the RViz panel, and the Python API, are documented in
**[ROS 2, RViz and the RX150](../../lecture-resources/ros2-rx150/README.md)**. That material
should be worked through once. What follows is specific to this project.

### 4.1 Hardware checkout

The RX150 manipulators are located in **Salazar 2008**. Each session requires four items:

| | |
|---|---|
| **Arm** | An RX150 |
| **Power adapter** | The arm's own 12 V supply |
| **USB cable** | USB to micro USB. Three dedicated cables remain; any USB to micro USB cable is suitable if these are unavailable |
| **Clamp** | For securing the arm to the table |

Set up at the station in the following order:

1. **Secure the arm to the table with the clamp.** The base must be fixed before power is
   applied. An unclamped manipulator will translate across the work surface under its own
   reaction torques.
2. **Clear the workspace.** The RX150 reaches approximately 450 mm in every direction; that
   radius must be free of laptops, bottles, keyboards, and hands.
3. Connect the power adapter, then the USB cable to the station.
4. Confirm that the arm is detected:

   ```bash
   ls /dev | grep ttyDXL
   ```

   An empty result indicates that either the USB or the power connection has not been made.
   Everything subsequent depends upon this result.

**Upon finishing, return the arm, adapter, cable, and clamp to their storage location.** The
three cables in particular are shared among eight teams.

**Log out, and leave the station running.** Logging out terminates the session and protects your
files while leaving the machine itself powered. The stations are maintained and updated remotely,
and one that has been powered off is unreachable: it misses that maintenance and is found
inoperable by the next user. Leave the station at the login screen.

### 4.2 Release of the servos

**The manipulator falls when its motors are released.** This occurs in two circumstances, both of
which arise in this project:

- Disabling the motor torque through a service call
- Terminating the control launch with Ctrl + C

In either case, **command the arm to its Sleep Pose beforehand, or support it by hand.** Sleep is
the folded configuration in which the arm rests upon itself and has nowhere to fall. It should be
the final action of every session.

Should the manipulator begin to fall, disconnect power rather than attempting to catch it. A
falling arm carries more momentum than anticipated, and a hand placed in its path becomes a pinch
point.

### 4.3 Launch configuration by part

| Part | Launch | Purpose |
|---|---|---|
| A, B | `xsarm_description.launch.py` | The model and the slider interface, without hardware. The sliders are calibrated in radians and their end stops correspond to the physical joint limits, which makes this the least costly means of determining whether a derived configuration is reachable |
| C | none | Inverse kinematics is purely computational; verification is by the round-trip test |
| D | `xsarm_control.launch.py` | The driver and the physical manipulator |

```bash
ros2 launch interbotix_xsarm_descriptions xsarm_description.launch.py \
  robot_model:=rx150 use_joint_pub_gui:=true

ros2 launch interbotix_xsarm_control xsarm_control.launch.py robot_model:=rx150
```

Backdriving the manipulator by hand through the five Part D configurations, with the motors
disabled, establishes quickly which are awkward or unreachable. This is worth doing before
commanding them.

### 4.4 Placement of hardware code

All hardware code belongs in the `Arm` class within `robot/rx150.py`. That file is the only one
in the project permitted to import a robot SDK, and the boundary forms part of the assessment.

Implement `__enter__` and `__exit__` using `InterbotixManipulatorXS`, `robot_startup()`, and
`robot_shutdown()`, followed by `send()`, `read()`, and `driver_ee_pose()`. The Interbotix imports
should remain within the method bodies: the test suite must run on a machine with no manipulator
attached, and a module-level import causes `pytest` to fail during collection for reasons
unrelated to the code under test.

Employed as a context manager, the class restores the arm even when a script raises partway
through:

```python
from robot.rx150 import Arm

with Arm() as arm:
    arm.send(my_solution)
    measured = arm.read()
    print(arm.driver_ee_pose())
```

Three practical matters are worth noting.

**Angles are expressed in radians**, ordered according to `JOINT_NAMES`. Use `math.radians()` if
degrees are preferred.

**`driver_ee_pose()` returns the manufacturer's forward kinematics, not your own.** It therefore
provides a check on `fk()` requiring no measurement, and it separates model error from measurement
error. It does not substitute for the physical measurement required by Part D, since the driver's
model and the physical arm are themselves distinct.

**Begin with small motions near the home configuration.** Increase `moving_time` while becoming
familiar with the system. A single large command follows the shortest path in joint space, which
is not generally the path anticipated, and the manipulator may pass through the work surface in
transit.

## 5. Part A: Frames and the Denavit-Hartenberg table

A Denavit-Hartenberg parameterization is meaningful only relative to a specified base frame and
tool frame. Both are prescribed here rather than selected by the student:

- **Base frame: `base_link`.** Origin at the mounting surface, axes following the ROS convention,
  **x forward, y left, z up**, with z coincident with the waist rotation axis.
- **Tool frame: `ee_gripper_link`.** This frame is not coincident with the final joint. Four
  **fixed** joints lie between the last revolute joint and the tool point, and the chain must
  include all of them.

Proceed as follows:

1. Applying the frame assignment procedure from lecture, attach a frame to each link of the
   RX150. Produce this assignment as a hand drawing, and photograph or scan it; the drawing forms
   part of the submission.
2. Derive the Denavit-Hartenberg parameter table, identifying the joint variable in each row.
3. **Cross-check the dimensions against the URDF**, as described below.

### 5.1 Reading the URDF

> **Preliminary reading:** [The Universal Robot Description Format
> (PDF, 7 pages)](references/lynch-park-4.2-urdf.pdf)
>
> It treats precisely the task at hand: the meaning of `<joint>`, `<parent>`, `<child>`,
> `<origin>` and `<axis>`, why a URDF describes a robot as a tree rather than a chain, why most
> links carry two frames rather than one, and a fully annotated URDF for the UR5 presented
> alongside a diagram of its frames.
>
> From Lynch and Park, *Modern Robotics*, §4.2, pages 152 to 158. Cambridge University Press,
> 2017. The complete text is freely available at
> [modernrobotics.org](http://modernrobotics.org) and is worth acquiring: §4.1 treats the product
> of exponentials, Appendix C the Denavit-Hartenberg convention, and Chapter 5 velocity kinematics
> and the Jacobian, which Project 3 takes up.
>
> Two points from that reading carry directly into this project. An `<origin>` specifies the pose
> of the **child** link's frame within the **parent's** frame when the joint variable is zero,
> which is the point developed below. And an `<axis>` is a unit vector in the **child** link's
> frame rather than the base frame, which is why the axes cannot be read off and used as they
> stand.

The description resides in the `interbotix_xsarm_descriptions` package as `rx150.urdf.xacro`. It
is a **xacro** file rather than plain URDF, so it contains unresolved `$(arg ...)` substitutions,
and link names appear as `$(arg robot_name)/base_link`. Either substitute mentally while reading
or process the file with `xacro` to obtain plain URDF, which is the more convenient approach.

What to read, and what to disregard:

- **Read** the `<joint>` blocks. Within each, `<parent>`, `<child>`, `<origin>`, and `<axis>`
  constitute the kinematics; nothing else in the file does.
- **Disregard** every `<visual>`, `<collision>`, and `<inertial>` block. These describe appearance
  and mass distribution rather than the geometry of the chain.

Three features of this file warrant particular attention:

- `base_link` carries a **visual** origin with a nonzero `rpy`. This rotates the mesh so that the
  model renders correctly on screen. It is not a kinematic transformation and must not appear in
  the parameter table.
- Within a joint block in this file, `<axis>` precedes `<origin>`. Taking the first `xyz`
  encountered yields the rotation axis in place of a translation.
- One joint within the gripper constitutes a **branch** rather than part of the chain to the tool
  point. Follow the parent and child links to the tool frame and disregard anything departing
  from that path.

### 5.2 The kinematic chain

The chain is transcribed below from that file, so that twenty-one readers do not each misread it
differently. **Verify the transcription against the file independently.** It is reference data
rather than a solution: converting these offsets into Denavit-Hartenberg parameters remains
entirely the student's work.

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

**Every `rpy` in the chain is `0 0 0`**, which is why no column for it appears. The description
consists entirely of translations, and the joint axes account for all of the orientation.

The final four rows merit careful reading. All are fixed, none carries an axis, and together they
contribute **0.093575 m** between the last revolute joint and the tool frame. Omitting any one of
them leaves the model short by that amount at every configuration.

Recall the meaning of a joint origin: it is the transformation **from the parent link's frame to
the child link's frame**, not a position expressed in the base frame. The waist entry of
`0, 0, 0.06566` places `shoulder_link` 65.66 mm above the mounting surface; it does not displace
the waist rotation axis, which remains on the z line through the base origin. Quantities in base
coordinates are obtained by accumulating the offsets along the chain: the shoulder axis, for
example, lies 104.57 mm above the mounting surface.

### 5.3 Comparison

The URDF does not employ Denavit-Hartenberg parameters, so its rows will not correspond to those
of the derived table. What must agree is the **physical geometry**: link lengths, offsets, and the
total distance from base to tool. Disagreement there indicates an error in the derived table, and
every subsequent result will inherit it.

> Nothing within the URDF states that x is forward. The root frame serves as the reference, and
> the joint attaching it to `world` is an identity transformation. The convention originates in
> ROS REP 103 and is made concrete by the link offsets: the arm stacks along `+z` and extends
> along `+x`. This is worth understanding rather than memorizing, as manipulators exist whose
> authors did not adopt the convention.

## 6. Part B: Forward kinematics

Implement `fk(thetas) -> H`, accepting five joint angles and returning the $4 \times 4$ pose of
the gripper frame expressed in the base frame.

Verify the implementation by three independent means. All three must agree before proceeding.

1. **Against RViz.** Publish a set of joint angles, allow the robot state publisher to position
   the manipulator, and compare the reported gripper pose against the computed `H`. This tests
   the derived table against an independent implementation of the same manipulator.
2. **Against the URDF chain.** Multiply the joint origin transformations taken from the URDF
   directly and confirm that the result agrees. This tests the derived table against the
   manufacturer's own description.
3. **Against the product of exponentials.** Express the screw axis of each joint in the base
   frame and compute the same pose by the PoE formula. This tests the derived table against a
   method requiring no frame assignment, so that an error in frame assignment cannot be present
   in both.

Report all three comparisons. Where any disagree, that disagreement constitutes the result and
should be resolved before proceeding.

## 7. Part C: Inverse kinematics

This section should be read in full before implementation begins.

The RX150 possesses five joints. A general pose in space requires six parameters, three
specifying position and three specifying orientation. Five joints cannot satisfy six constraints,
and consequently **for an arbitrarily specified $4 \times 4$ matrix no solution exists**. An
inverse kinematic routine accepting an arbitrary `H` therefore poses a question the manipulator
cannot answer.

Consider what each joint contributes:

- The **waist** rotates the entire manipulator about the vertical axis. It determines the vertical
  plane within which the remaining joints operate, and that plane is fixed once `x` and `y` are
  specified.
- The **shoulder, elbow, and wrist angle** have parallel axes and therefore constitute a planar
  three-link chain within that plane. They supply radial distance, height, and the pitch of the
  approach.
- The **wrist rotate** revolves the gripper about its own approach axis, supplying roll.

The manipulator therefore commands position, pitch, and roll, and **yaw is not available as a
free parameter**. The approach direction must lie within the vertical plane through the waist, and
that plane is determined by the commanded position.

The solver accordingly determines **four joint variables from four constraints**: `x`, `y`, `z`,
and pitch. Roll is not solved for. It is supplied by the caller and passed directly to the wrist
rotate joint, as no other element of the chain depends upon it.

Implement `ik(x, y, z, pitch, roll=0.0) -> list of solutions`, with the following behavior at the
boundaries of the workspace:

- **Return the complete solution set.** Elbow-up and elbow-down configurations are both valid, and
  the waist admits a second branch. A solver that silently selects one withholds information from
  its caller.
- **Identify joint limit violations.** A mathematically valid solution that the manipulator cannot
  physically adopt is not a usable result. Return it, but marked as such.
- **Detect unreachable targets and report them.** Return an empty set together with a reason.
  Under no circumstances return a nearby configuration as though it were a solution, and never
  return `NaN`.

Implement the round-trip test in `tests/test_roundtrip.py`: for a set of reachable targets, run
the inverse kinematics, propagate each solution back through the forward kinematics, and confirm
that the target is recovered within tolerance. This establishes the correctness of the solver
without external reference.

The test itself is retired in Project 2, where the manipulator changes and the solver changes with
it. The technique is not. Two independent solvers are compared against one another in Project 3,
for the same reason that one is compared against itself here.

> A successful round trip establishes that a solution is *valid*. It does not establish that the
> complete set has been found, nor that the manipulator can adopt the solution. The two
> requirements above address those questions.

## 8. Part D: Experimental verification

This part is conducted in teams of three sharing one manipulator. All three members submit the
same measurement data; **each member produces an independent analysis against their own forward
kinematics.** Predictions will differ where the derived tables differ, and the comparison of three
independent predictions against a single set of measurements is itself informative.

Divide the roles and rotate them so that each member performs each task at least once: one member
commands the manipulator, one measures, and one records and states the predicted value after the
measurement is taken rather than before.

> **Before commanding any hardware motion:** verify the configuration in RViz, command low joint
> velocities, and remain within reach of the power disconnect. Should the manipulator move toward
> the work surface, disconnect power rather than attempting to intervene physically.

Implement the `Arm` class in `robot/rx150.py` first. It is a context manager, so that the
manipulator is returned to the sleep pose even where a script raises partway through, and the
Interbotix imports belong within its methods so that the test suite remains runnable with no
manipulator attached.

1. Select five reachable configurations distributed across the workspace, rather than five
   variations upon a single configuration.
2. Compute the predicted gripper position for each using your forward kinematics.
3. Command the manipulator to each configuration and allow it to settle.
4. Measure the actual gripper tip position relative to the base frame origin using a rule or
   calipers. Establish the measurement procedure in advance and apply it consistently.
5. Record the joint angles, the measurements, and any observations in
   `data/partD-measurements.csv` at the repository root. That file is shared: one session, one
   dataset, three independent analyses.
6. In your own report, tabulate predicted against measured values and the error along each axis
   in millimeters.

Record the configuration the servos actually attained using `arm.read()`, not solely the commanded
configuration. The two differ under gravitational load, and that difference is among the
explanations to be evaluated.

Account for the discrepancy. The error will not be zero. Candidate explanations include servo
position error under gravitational load, link dimensions differing from the published values, the
measurement technique, an incorrect Denavit-Hartenberg parameter, and the gripper tip not lying
precisely where the tool frame was defined. Argue from the collected data for the explanations
that dominate, and describe how that claim would be tested.

## 9. Deliverables

All deliverables reside in the repository with the exception of the oral examination.

- **`REPORT.md`**, the document through which this project is assessed. The template in the
  repository defines the sections. It presents the photographed frame assignment, the
  Denavit-Hartenberg table, the three forward kinematic verifications together with the output
  substantiating them, the inverse kinematic formulation, the table of predicted against measured
  values, and the error analysis.

  The report should be composed as a technical document rather than as a form with its blanks
  completed. For an indication of the form to aim at, see [Forward Kinematics of the UR5e: DH and
  PoE](https://www.roboticsunveiled.com/forward-kinematics-ur5e-dh-poe/). That treatment states
  the frame assignment, presents the table, gives the equations, shows the implementing code, and
  then presents the **actual output** establishing that the two methods agree. Every claim is
  supported by evidence the reader can verify. The report need not be as long. It should be as
  rigorous.

  The report is read on GitHub, so its equations must render there. The template concludes with a
  note on the limitations of that renderer, which should be consulted before the mathematics is
  written rather than afterward.
- The `kinematics/` library, with `robot/rx150.py` as the sole hardware-aware module.
- A passing round-trip test.
- A brief note in `README.md` describing how the code is run.
- **Oral examination.** Each student will be assigned a target position and approach angle, asked
  to command the manipulator to it, and asked to account for the resulting configuration. **The
  demonstration is conducted at your station**, from the code in your repository. Personal
  machines are not used.

## 10. Remarks

The gripper frame is the student's to define, but it must be defined once and employed
consistently in the derivation, the implementation, and the measurements. The majority of large
discrepancies in this project are traceable to measuring to a point other than the one the model
computes.

Joint angle sign conventions and zero configurations are established by the manufacturer rather
than by the student's drawing. Where the forward kinematics differs by a sign at a single joint,
this is the first place to examine.
