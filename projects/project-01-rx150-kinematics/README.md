# Project 1: Kinematics of the RX150

**Individual work.** You write your own code and your own derivation. Hardware measurement
sessions are done in your team of three, since arms are time-shared and measuring is a job for
more than one person.

Check Canvas for deliverables, deadlines, and grading rubric.

## 1. Objectives

- Assign coordinate frames to a real manipulator and derive its DH parameter table.
- Implement forward kinematics from your own table and verify it against three independent
  sources.
- Implement inverse kinematics for an arm with fewer than six degrees of freedom, handling
  multiple solutions, joint limits, and unreachable targets.
- Command the physical arm, measure where the gripper actually goes, and account for the
  difference between your model and the machine.

## 2. What you are building

You are writing a small kinematics library, not a script. The structure matters and is graded:

```
kinematics/     pure math. Imports numpy and nothing else.
  transforms.py rotation matrices, homogeneous transforms, helpers
  dh.py         DH table to A matrix, forward kinematics
  ik.py         inverse kinematics
robot/
  rx150.py      the ONLY file permitted to import a robot SDK or talk to hardware
tests/
  test_roundtrip.py
results/
  measurements.md
```

**Nothing in `kinematics/` may import a robot driver, ROS, or anything hardware specific.** It
takes numbers in and returns numbers out. The arm-specific code lives in `robot/`, and its job is
to accept a vector of joint angles and send it.

This is not stylistic. In Project 3 you will move this library onto a different arm with an extra
joint, driven through a completely different software stack. Both arms come from the same
manufacturer and they still share almost nothing on the software side: the RX150 goes through the
Interbotix ROS packages, and the WidowX AI goes through a plain Python driver over Ethernet. If
the math is tangled up with the driver, that port is a rewrite. If it is clean, it is one new
file.

## 3. Setup

1. Accept the assignment from the link on Canvas. It creates a private repository for you with
   this structure already in place.
2. Create a Python virtual environment in your home directory and install the requirements. Do
   not use `sudo pip`.
3. Confirm you can launch the RX150 description and see the arm in RViz before you write any
   kinematics.

> **Never commit credentials.** No tokens, no `.env` files, no API keys. The `.gitignore` already
> excludes the common cases, but check `git status` before you commit.

## 4. Part A: Frames and the DH table

**Start and end where everyone else does.** A DH table only means anything relative to where the
chain begins and ends, so these two are given to you rather than chosen:

- **Base frame: `base_link`.** Origin at the mounting surface, axes in the ROS convention, **x
  forward, y left, z up**, with z along the waist rotation axis.
- **Tool frame: `ee_gripper_link`.** This is not the last joint. Four **fixed** joints sit between
  the last revolute joint and the tool point, and your chain must include all of them.

Then:

1. Using the frame assignment rules from lecture, attach a frame to every link of the RX150.
   Draw this by hand. Photograph or scan the drawing; it is part of your submission.
2. Derive the DH parameter table. Mark the joint variable in each row.
3. **Cross-check the dimensions against the URDF**, as described below.

### Reading the URDF

> **Read this first.** Lynch and Park, *Modern Robotics*, **§4.2, The Universal Robot Description
> Format**, pages 152 to 158. Seven pages, and they cover exactly what you are about to do: what
> `<joint>`, `<parent>`, `<child>`, `<origin>` and `<axis>` each mean, why a URDF describes a
> robot as a tree rather than a chain, why most links carry two frames rather than one, and a
> fully annotated URDF for the UR5 printed beside a diagram of its frames. The preprint is free
> at [modernrobotics.org](http://modernrobotics.org).
>
> Two things in it are worth carrying into this project directly. An `<origin>` is the pose of
> the **child** link's frame in the **parent's** frame when the joint variable is zero, which is
> the point laboured below. And an `<axis>` is a unit vector in the **child** link's frame, not
> the base frame, which is why you cannot read the axes off and use them as they stand.

The description lives in the `interbotix_xsarm_descriptions` package as `rx150.urdf.xacro`. It is
a **xacro** file, not plain URDF, so it is full of unresolved `$(arg ...)` substitutions and link
names will read as `$(arg robot_name)/base_link`. Either substitute mentally as you read, or run
it through `xacro` first to produce a plain URDF, which is easier.

What to read and what to ignore:

- **Read** the `<joint>` blocks. Within each one, `<parent>`, `<child>`, `<origin>`, and `<axis>`
  are the kinematics, and nothing else in the file is.
- **Ignore** every `<visual>`, `<collision>`, and `<inertial>` block. They describe appearance and
  mass, not geometry of the chain.

Three traps, all of which have caught people before:

- `base_link` has a **visual** origin with a nonzero `rpy`. That rotates the mesh so the model
  looks right on screen. It is not a kinematic transform and it must not appear in your table.
- Inside a joint block in this file, `<axis>` is written **before** `<origin>`. If you grab the
  first `xyz` you see, you have grabbed the rotation axis and called it a translation.
- One joint in the gripper is a **branch**, not part of the chain to the tool point. Follow
  parent and child links to the tool frame and ignore anything that leaves that path.

### The chain, transcribed

Here is the kinematic chain pulled out of that file, so that twenty-one people do not each
misread it in a different way. **Verify it against the file yourself.** It is reference data, not
an answer: converting these offsets into DH parameters is still entirely your work.

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
pure translation, and the joint axes do all of the orienting.

Read the last four rows carefully. They are all fixed, they carry no axis, and together they add
**0.093575 m** between the last revolute joint and the tool frame. Drop any one of them and your
model is short by that amount at every pose you ever compute.

Remember what a joint origin means: it is the transform **from the parent link's frame to the
child link's frame**, not a position in the base frame. The waist entry of `0, 0, 0.06566` places
`shoulder_link` 65.66 mm above the mounting surface. It does not move the waist rotation axis,
which stays on the z line through the base origin. Add the offsets down the chain to get anything
in base coordinates: the shoulder axis, for instance, ends up 104.57 mm up.

### Now compare

The URDF does not use DH parameters, so the rows will not correspond to yours. What must agree is
the **physical geometry**: link lengths, offsets, and the total distance from base to tool. If
those disagree with your table, your table is wrong, and everything downstream will be wrong with
it.

> Nothing in the URDF states that x is forward. The root frame is the reference, and the joint
> that attaches it to `world` is an identity transform. The convention comes from ROS REP 103,
> and it is made concrete by the link offsets: the arm stacks along `+z` and reaches along `+x`.
> This is worth understanding rather than memorizing, because you will meet a robot whose author
> did not follow the convention.

## 5. Part B: Forward kinematics

Implement `fk(thetas) -> H`, taking five joint angles and returning the 4x4 pose of the gripper
frame in the base frame.

Then verify it three ways. All three must agree before you move on.

1. **Against RViz.** Publish a set of joint angles, let the robot state publisher place the arm,
   and compare the gripper pose it reports against your computed `H`. This checks your table
   against an independent implementation of the same robot.
2. **Against the URDF chain.** Multiply the joint origin transforms from the URDF directly and
   confirm you land in the same place. This checks your table against the manufacturer's own
   description.
3. **Against product of exponentials.** Write the screw axis for each joint in the base frame and
   compute the same pose with the PoE formula. This checks your table against a method that uses
   no frame assignment at all, so an error in your frame assignment cannot hide in both.

Report all three comparisons. If any disagree, the disagreement is the finding, and you should
chase it before continuing.

## 6. Part C: Inverse kinematics

**Read this section before you write code.** The RX150 has five joints. A general pose in space
requires six numbers, three for position and three for orientation. Five joints cannot satisfy six
constraints, so **for almost any 4x4 matrix you invent, there is no solution at all.** An IK
function that accepts an arbitrary `H` is asking a question the arm cannot answer.

Look at what each joint actually buys you:

- The **waist** rotates the whole arm about the vertical. It sets which vertical plane the rest of
  the arm works in, and that plane is fixed the moment you choose `x` and `y`.
- The **shoulder, elbow, and wrist angle** have parallel axes, so they form a planar three-link
  chain inside that plane. They give you radial distance, height, and the pitch of the approach.
- The **wrist rotate** spins the gripper about its own approach axis. That is roll.

So the arm controls position, pitch, and roll, and **yaw is not yours to choose.** The approach
direction must lie in the vertical plane through the waist, and that plane is already determined
by where you asked the gripper to go.

Your solver therefore solves **four joints for four constraints**: `x`, `y`, `z`, and pitch. Roll
is not solved at all. It is handed to you and passed straight through to the wrist rotate joint,
because nothing else in the chain depends on it.

Implement `ik(x, y, z, pitch, roll=0.0) -> list of solutions`, and make it behave properly at the
edges:

- **Return every solution, not one.** Elbow up and elbow down are both valid, and the waist has a
  second branch. A solver that silently picks one is hiding information from its caller.
- **Flag joint limit violations.** A mathematically valid solution the arm cannot physically adopt
  is not a usable answer. Return it, but mark it.
- **Detect unreachable targets and say so.** Return an empty list and a reason. Never return a
  nearby pose as though it were the answer, and never return `NaN`.

Then write the round-trip test in `tests/test_roundtrip.py`: for a set of reachable targets, run
IK, feed each solution back through FK, and confirm you recover the target within tolerance. This
is how you find out your solver works without asking anyone, and it covers you through Project 2,
which stays on this arm.

The test itself retires in Project 3, where the arm changes and the solver changes with it. The
technique does not. You will check two independent solvers against each other there for the same
reason you are checking one against itself here, so build the habit now.

> A passing round trip proves a solution is *valid*. It does not prove you found *all* of them,
> and it does not prove the arm can adopt it. That is why the two requirements above exist.

## 7. Part D: Hardware verification

This part is done in your team of three, sharing an arm. All three submit the same measurement
data; **each of you writes your own analysis against your own forward kinematics.** Your
predictions will differ if your DH tables differ, and comparing three independent predictions
against one shared measurement is itself informative.

Split the roles and rotate them so everyone does each job at least once: one person commands the
arm, one measures, one records and calls out the predicted value afterward rather than before.

> **Before any hardware motion:** run the configuration in RViz first, keep joint velocities low,
> and keep a hand on the power switch. If the arm moves toward the table, cut power. Do not try to
> catch it.

1. Choose five reachable configurations that spread across the workspace, not five variations of
   the same pose.
2. For each, compute the predicted gripper position with your FK.
3. Command the arm to that configuration and let it settle.
4. Measure the actual gripper tip position relative to the base frame origin with a ruler or
   calipers. Decide on a measurement procedure first and use the same one every time.
5. Record predicted and measured, and the error per axis in millimetres.

Then account for the difference. Your error will not be zero. Candidates include servo position
error under gravity load, link dimensions that differ from the datasheet, your measurement
technique, a wrong DH parameter, and the fact that the gripper tip is not exactly where you
decided the frame was. Argue for which ones dominate in your data and say how you would test that
claim.

## 8. Deliverables

Everything goes in your repository except the live demo.

- The hand-drawn frame assignment and DH table, photographed or scanned.
- The `kinematics/` library, with `robot/rx150.py` as the only hardware-aware file.
- Evidence of all three forward kinematics verifications.
- A passing round-trip test.
- `results/measurements.md` with the predicted versus measured table and your error analysis.
- A short README in your repo saying how to run your code.
- **Live demo.** You will be given a target position and approach angle, and asked to command the
  arm to it and explain why it went where it went. **Run it from your station**, working from the
  code in your repository. Personal laptops are not used for demos.

## 9. Notes

The gripper frame is yours to define, but define it once and use the same definition in your
derivation, your code, and your measurements. Most large discrepancies in this project trace back
to measuring to a different point than the one the model computes.

Joint angle sign conventions and zero positions come from the manufacturer, not from your
drawing. If your FK is off by a sign on one joint, this is the first place to look.
