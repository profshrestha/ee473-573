# Project 2: Six-Axis Kinematics of the WidowX AI

**Team project.** The work is done in your team's repository, `ee473-2026f-p2-station-NN`, with
one submission and one report per team.

Check Canvas for deliverables, deadlines, and the grading rubric.

## 1. Objectives

Upon completion of this project, students will be able to:

- Build a kinematics library for a six-axis manipulator, keeping the mathematics separate from the
  hardware interface.
- Derive a Denavit-Hartenberg parameter table for a six-axis arm and compare it against the table
  published by the manufacturer.
- Derive and implement a closed-form inverse kinematic solution for a six-axis manipulator, for a
  full end-effector pose.
- Evaluate the accuracy of a kinematic model against measurements taken on hardware.

## 2. Scope and context

The WidowX AI is a six-axis revolute manipulator with a reach of 0.769 m and a rated payload of
1.5 kg. It shares a manufacturer with the RX150 of Project 1. The two arms have little in common at
the software level, however: the RX150 is commanded through the Interbotix ROS packages, while the
WidowX AI is addressed directly over Ethernet by a Python driver.

### 2.1 Scope of the implementation

Derive and implement everything in this project for this manipulator.

The deliverable is a kinematics library rather than a script. Its organization is prescribed below
and forms part of the assessment:

```
kinematics/     pure mathematics; imports numpy and nothing further
  transforms.py rotation matrices, homogeneous transforms, helpers
  fk.py         DH table to A matrix, forward kinematics, and the PoE check
  ik.py         inverse kinematics by wrist decoupling
robot/
  widowx.py     the ONLY file permitted to import a robot SDK or address hardware
tests/
  test_roundtrip.py
scripts/
  verify_fk.py  harness for the forward kinematics checks of Part A
REPORT.md       the report; this is the deliverable the reader engages with
data/           the Part C measurement session
figures/        photographs and plots referenced by the report
```

The layout is similar to the one used in Project 1, so it should already be familiar. It is
already present in the repository, with each function stubbed. Complete the stubs; do not
reorganize the package.

**No module within `kinematics/` may import a robot driver, ROS, or any hardware-specific
package.** The kinematic routines accept numerical arguments and return numerical results.
Everything particular to this arm, the joint names, the Denavit-Hartenberg table, the joint
limits, and the tool offset, belongs in `robot/widowx.py`. Project 3 extends this library to
velocity control and numerical inverse kinematics, and that extension is far easier where the
boundary has held.

Every member commits under their own account. For a team submission the commit history is the
only record of who did what.

## 3. Part A: Two kinematic models

### 3.1 Assigning frames

A Denavit-Hartenberg parameter table is only meaningful relative to a stated base frame and tool
frame. If the frames used here differ from the manufacturer's, the two tables will disagree for
reasons unrelated to the correctness of either, and the comparison in §3.4 becomes uninformative.
The frames are therefore prescribed here.

**Base frame.** Use Trossen's `base_link`. Its origin lies at the mounting surface and its axes
follow the ROS convention of REP 103, **x forward, y left, z up**, with z along the rotation axis
of the first joint. The manufacturer's specification page publishes a parameter table but never
draws the frame, which is why it is given here.

**Intermediate frames.** Align the z axis of frame i-1 with the axis of joint i, in the direction
the driver treats as positive rotation. These directions do not all coincide with the base axes,
and the signs carry through into the parameter table. Read them from `wxai_follower.urdf` in the
`trossen_arm_description` package, or determine them on the hardware by commanding a small positive
move at one joint at a time and observing which way the link turns. The remaining freedom in each
frame, the position of its origin along the axis and the direction of its x axis, is left open and
follows the usual Denavit-Hartenberg rules.

**Tool frame.** Use Trossen's `Tool` frame, which is not at the final joint. The manufacturer's
chain runs from the base through the six joints to a `Flange` frame and then to `Tool`. That last
segment is two fixed transformations: a rotation carrying the final joint frame into the flange, and
a translation of 156.06 mm from the flange to the tool point. The full chain therefore holds eight
transformations for six joints. A table that stops at the sixth joint drops both, and the result
shows up as a large constant position offset.

**Joint zero configuration.** Use the zero configuration the driver reports. Find out what that
configuration looks like physically.

**Convention.** Use the classical Denavit-Hartenberg convention from lecture, in which each link
transformation is `Rot(z, θ) Trans(z, d) Trans(x, a) Rot(x, α)`. Trossen publishes parameters in
both the classical and the modified
convention. These are different parameterizations whose rows do not line up.

### 3.2 Derivation

With the frames fixed, finish the assignment for the six joints and derive the Denavit-Hartenberg
parameter table, following the procedure used in Project 1.

Finish the derivation before looking at the manufacturer's table. The derivation is the substance of
the exercise, and the published table is an independent check on it.

### 3.3 Implementation and verification

Implement `fk(thetas)`, taking six joint angles and returning the 4 × 4 pose of the `Tool` frame
in the base frame, together with `fk_all_frames()` for the intermediate frames. A single terminal
pose cannot indicate where a chain diverges; a frame-by-frame comparison can.

Verify the result by two independent means before anything is built upon it.
`scripts/verify_fk.py` is the harness. Complete the comparisons, run it, and include the output in
the report.

1. **Against the URDF chain.** Multiply the joint origin transformations from
   `wxai_follower.urdf` directly, with no parameter table involved. Every origin in that file is a
   pure translation and each joint contributes a rotation about its own axis, so this tests the
   derived table against the manufacturer's own description. Note that the axis signs are not
   uniform.
2. **Against the product of exponentials.** Express the screw axis of each joint in the base frame
   and compute the same pose by the PoE formula. This tests the derived table against a method
   requiring no frame assignment, so an error in frame assignment cannot be present in both.

Report both comparisons. Where either disagrees, that disagreement is the result, and it should
be resolved before Part B. An inverse kinematic solver built upon a faulty forward model will
appear to function and will be incorrect at every configuration.

### 3.4 Comparison against the manufacturer's table

Obtain the manufacturer's parameter table and compare it against your own. For each disagreement,
find the cause.

Compare the two tables numerically as well as row by row. `scripts/verify_fk.py` computes the
tool pose from Trossen's parameters alongside your own, and two tables may differ in every row
while agreeing at every pose. That distinction is exactly the one between a different frame
assignment and an error.

Expect constant offsets in the θ column. These arise from the same mechanism as the +90° offset
seen in lecture. The manufacturer's table contains several, at least one of which is not a multiple
of 90°. A derived table with no such offsets almost certainly reflects a frame assignment different
from the manufacturer's. In that case the useful question is not which table is correct, since both
may be, but which is more convenient to work with, and why.

Comparing an independently derived model against a vendor's is routine engineering work, and it is
rarely free of discrepancies. Any disagreement left unexplained counts as incomplete work.

## 4. Part B: Closed-form inverse kinematics by decoupling

The last three joints of the WidowX AI form a **spherical wrist**: their axes intersect at a common
point. This is a deliberate design choice rather than an accident of the mechanism, and it is the
condition that makes a closed-form inverse solution possible. A mechanical decision has been made
here to serve a mathematical one, which is a pattern worth watching for throughout manipulator
design.

Because the wrist axes intersect, the problem **decouples** into two subproblems of three unknowns
each:

1. The position of the wrist center depends only on the first three joint variables, so those three
   follow from the desired position alone.
2. With the first three known, the orientation the wrist has to supply is fully determined, and the
   last three joint variables follow from it.

Implement this decoupled solution. Unlike the five-axis RX150 of Project 1, this arm has six joints
for six task-space constraints, so a full pose, position and orientation together, can be
specified.

The solver should return the complete solution set, flag the solutions that violate joint limits,
and detect targets outside the reachable workspace, matching the requirements of Project 1.

`tests/test_roundtrip.py` is supplied. Each target is generated by propagating a joint
configuration through `fk()`, so a solution is guaranteed to exist by construction, and a failure
therefore indicates the solver rather than an impossible request. The
suite confirms that every solution returned reproduces the requested pose, that the configuration
from which the target was generated appears among them, and that more than one branch is returned.
Run it with `pytest -v` and include the output in the report.

A passing run establishes that every solution returned is valid. It does not establish that the
complete set was found, nor that the manipulator can physically adopt them, which is why the two
requirements above are stated separately.

## 5. Part C: Experimental verification

> **Safety.** The precautions from Project 1 apply, and apply with more force here: this arm is
> heavier, more powerful, and more expensive than the RX150. Check each configuration in
> visualization before commanding it, command motions at reduced speed, and stay within reach of
> the power disconnect throughout.

A pose in this context is a 4 × 4 homogeneous transformation, which is what `ik()` accepts.
Construct it from a position and an approach orientation using `pose_from_xyz_rpy()` in
`transforms.py` rather than writing the entries out directly. Report which convention that helper
implements, including the order in which the three rotations are applied, and apply the same
convention to every prediction and measurement that follows. A target constructed under one
convention and evaluated under another will disagree for reasons unrelated to the manipulator.

The procedure is the same for each pose: construct the target, solve it with `ik()`, select one
branch from the returned set and be prepared to justify the selection, then command the joint
angles that branch contains. Command the manipulator in joint space throughout, from the angles
your own solver returns, so that it is your solution being exercised.

1. Command at least five full poses, specifying both position and orientation, using the
   closed-form solver from Part B.
2. Measure the resulting gripper position relative to the base frame, using the procedure from
   Project 1, and report the error along each axis.
3. Evaluate the orientation against `driver_ee_pose()`, which returns the manufacturer's forward
   kinematics for the configuration the manipulator has actually attained. Orientation cannot be
   measured with a rule, and this comparison separates model error from measurement error. It does
   not substitute for the physical measurement in step 2.
4. Record the commanded angles, the predictions, and the measurements in
   `data/partC-measurements.csv`, together with the configuration the servos actually attained,
   obtained from `arm.read()` rather than assumed from the command. The two differ under
   gravitational load, and that difference is among the explanations to be weighed.
5. Compare the results against Project 1. The method is the same and the arm is different.
   Determine whether the error is larger or smaller, and account for the difference.

## 6. Deliverables

- The team repository, with the `kinematics/` library and the `robot/widowx.py` adapter.
- The derived Denavit-Hartenberg table, together with the frame assignment it came from.
- The forward kinematics and both verifications of §3.3, with the output substantiating them.
- The manufacturer's table and the comparison against your own, with the cause of each
  disagreement.
- The closed-form solver, with the round-trip output and evidence that it finds the complete
  solution set.
- The measurement table from Part C and the comparison against Project 1.
- A statement of how the work was divided among team members.
- **Oral examination.** Each team will be given a full target pose, asked to command the arm to it,
  and asked to account for the solution branch their code selected.

## 7. Remarks

Most of the error in this project comes from a mismatch of convention rather than a mistake in the
mathematics. Classical against modified Denavit-Hartenberg parameters, degrees against radians, a
different joint zero configuration, or a tool point the vendor defines differently than you do will
each give results that are close to correct without being correct. When the computed values come
near the measured ones without matching them, check the conventions before checking the algebra.

Project 3 develops a numerical solver for this same arm and compares the two approaches against
each other. Write the closed-form solver as an importable module rather than burying it in a script.
