# Project 3: The Manipulator Jacobian and Velocity Control

**Team project.** The work builds on the WidowX AI library your team developed in Project 2, in
the same teams, with one submission and one report per team. Most of the project runs in
visualization; one supervised session runs on hardware.

Check Canvas for deliverables, deadlines, and the grading rubric.

## 1. Objectives

Upon completion of this project, students will be able to:

- Build the geometric Jacobian of a serial manipulator from its forward kinematics, and verify it
  against a numerical approximation.
- Use the Jacobian to command motion in task space rather than joint space.
- Identify kinematic singularities, describe their effect on the velocity solution, and apply
  damped least squares to work around them.

## 2. Starting point

Copy your team's Project 2 `kinematics/` package into the Project 3 repository. The forward
kinematics, the parameter table in `robot/widowx.py`, and the frame conventions all carry over
unchanged. The arm is the same: the WidowX AI, six revolute joints,
commanded over Ethernet through the Python driver.

Two modules are added. Stubs for them are in the repository:

```
kinematics/
  jacobian.py     Jacobian construction and the velocity mapping
  control.py      resolved-rate loop, direct inverse and damped least squares
```

### 2.1 What your forward kinematics has to satisfy

Any forward kinematics that reproduces the tool pose is acceptable here. A classical
Denavit-Hartenberg table is one route, multiplying the joint origin transformations out of
`wxai_follower.urdf` is another, and the two give the same answer. Nothing this project grades can
tell them apart: the i-th Jacobian column needs the direction of joint axis i and any one point on
that axis, and every valid frame assignment supplies the same two. Teams whose intermediate frames
differ row by row will produce Jacobians agreeing to machine precision.

The same goes for the tool point. Displacing it by a fixed vector r in the end-effector frame
pre-multiplies the Jacobian by a block matrix whose determinant is 1, so the determinant of J, the
manipulability, and the location of every singular configuration are all unchanged. Keep whatever
tool point you used in Project 2.

What does matter is that the pose itself is right, and Part A will not catch it if it is wrong:
the finite-difference check differentiates your own `fk()`, so it confirms that `jacobian()` and
`fk()` agree with each other and nothing more. A forward kinematics with a sign error passes it
cleanly and then corrupts every number in Parts B and C. Check against the eight poses below
before building anything on top of your `fk()`.

Angles are listed in degrees in joint order, positions are the tool point in millimetres relative
to `base_link`, and the orientation is given as fixed-axis XYZ roll, pitch and yaw in radians. The
values below are rounded to 0.001 mm and 0.0001 rad, so match them to 0.01 mm and 0.0002 rad rather
than exactly. That is still a sharp test: a sign error or a dropped offset misses by millimetres.
These poses were computed from the Denavit-Hartenberg table and independently from the raw URDF
chain, the two agreeing to 1.1e-15 m over 4000 random configurations inside the joint limits.

| # | Configuration | q (deg) | x, y, z (mm) | roll, pitch, yaw (rad) |
|---|---|---|---|---|
| 1 | home | 0, 0, 0, 0, 0, 0 | 253.762, 0.000, 163.500 | 0.0000, 0.0000, 0.0000 |
| 2 | rest | 0, 90, 90, 0, 0, 0 | 517.762, 0.000, 427.500 | 0.0000, 0.0000, 0.0000 |
| 3 | waist only | 45, 0, 0, 0, 0, 0 | 179.437, 179.437, 163.500 | 0.0000, 0.0000, 0.7854 |
| 4 | shoulder and elbow | 0, 60, 45, 0, 0, 0 | 384.330, 0.000, 261.256 | 0.0000, 0.2618, 0.0000 |
| 5 | wrist exercised | 0, 45, 90, 30, -40, 60 | 18.355, 118.923, 708.183 | -0.1286, -0.8330, 1.2716 |
| 6 | general A | 30, 75, 60, -45, 35, -90 | 324.562, 64.851, 163.127 | -2.3529, 0.7887, -0.4271 |
| 7 | general B | -120, 20, 120, 80, -80, 150 | 372.585, 280.934, 424.652 | -0.5236, 0.0000, -0.3491 |
| 8 | near limits | -175, 0, 135, -90, 90, 180 | 394.069, 220.195, 282.221 | -2.3562, 0.0000, 1.6581 |

`robot/widowx_reference.py` in the repository will run the comparison for you:

```
python3 -c "from robot.widowx_reference import check; from kinematics.fk import fk; check(fk)"
```

If your Project 2 forward kinematics does not reproduce these poses, say so in the report and use
that module, which does. Doing so costs nothing in Project 3, which grades the Jacobian and the
control loops rather than the kinematics underneath them. Carrying a broken `fk()` forward
silently, on the other hand, costs everything downstream. Whichever you use, state which in the
report.

### 2.2 Visualization

Most of this project runs in RViz rather than on hardware. Resolved-rate control is purely
kinematic: joint velocities are integrated to get joint positions, and no contact, gravitational,
or inertial effects enter the formulation. A dynamic simulator is unnecessary. All that is needed
is a way to watch the arm follow a trajectory.

Publishing the integrated joint angles as `sensor_msgs/JointState` makes the robot state publisher
place the arm in RViz, exactly as the joint state publisher interface did in Project 1. The
description package `trossen_arm_description` is installed on every station. A full trajectory is
replayed by publishing successive configurations on a timer.

Two things motivate this. Hardware access is shared among all teams, and Part C requires driving
the arm deliberately into a singular configuration, which is not permitted on hardware under any
circumstances.

## 3. Part A: Build the Jacobian, and verify it

Build the 6 × n **geometric Jacobian** one column at a time, taking every quantity from the forward
kinematics you already have. For a revolute joint, the i-th column is

```math
J_i = \begin{bmatrix} z_{i-1} \times (o_n - o_{i-1}) \\ z_{i-1} \end{bmatrix}
```

Every quantity in this expression is already available. The vector `z_{i-1}` is the third column and
`o_{i-1}` the fourth column of the transformation from the base frame to frame i-1, which is a
partial product of the forward kinematics.

The resulting matrix maps joint rates to the linear velocity of the end effector over its angular
velocity, both in the **base frame**. Call that stacked six-vector ζ, the end-effector twist, so
that ζ = J θ̇. Record this convention in the code, since it governs how the velocity command is
formed in Part B.

**Verify it before building anything on it.** Compare each column against a central-difference
approximation,

```math
J_{\mathrm{fd}}[:,i] = \frac{f(\theta + h\,e_i) - f(\theta - h\,e_i)}{2h}
```

where `e_i` is the i-th unit vector in joint space and h the perturbation in radians. The position
rows follow from a difference of position vectors. The orientation rows do not, because the
difference of two rotation matrices is not a rotation: form `R_pert Rᵀ` instead, take its axis and
angle by the matrix logarithm on SO(3) from
[§3.5 of the product of exponentials notes](../../lecture-resources/product-of-exponentials/README.md#35-the-inverse-matrix-back-to-axis-and-angle),
and divide by h.

Report the largest disagreement over at least twenty random configurations, and the value of h you
chose. The error does not fall monotonically as h shrinks, and your choice should reflect that.

> Texts differ in the order of the two blocks. This project puts linear velocity above angular
> velocity. Before comparing against a Jacobian from a software library, check which order it uses.

## 4. Part B: Resolved-rate control

Given a desired end-effector velocity, find the joint velocities that produce it and integrate the
result into a trajectory.

**Task.** Move the gripper in a straight line in task space between two reachable points, holding
orientation constant.

1. Implement the velocity mapping. The arm has six joints and the task has six components, so J is
   square. Away from singular configurations it can be inverted directly; prefer
   `numpy.linalg.solve` over forming the inverse.
2. Generate the trajectory by integrating joint velocities over small time steps.
3. Recover the realized path by running the resulting joint angles through the forward kinematics,
   and replay it in RViz. A path that looks right in a plot but wrong on screen usually means the
   frames are inconsistent, and the visualization finds that faster than numerical inspection does.
4. Plot the realized path against the commanded straight line and report the largest deviation.

The deviation will not be zero, since the procedure integrates a linearization of a nonlinear
mapping.

## 5. Part C: Singular configurations and damping

A six-axis arm is singular wherever J loses rank. Where the last three axes intersect at a point,
det J factors into an arm term and a wrist term and the two families separate cleanly. The WidowX AI
does not have that geometry: `joint_3` and `joint_4` are skew, offset by 67.75 mm, so det J does not
factor, and it varies with the fourth joint variable in a way the factored form forbids.

1. **Confirm that, then find a singular configuration.** Sweep each joint in turn and record
   whether det J depends on it. An arm whose wrist axes intersect gives a determinant independent
   of the first and last joint variables; the departure from that is the offset above. Then locate
   one singular configuration by searching numerically for det J near zero, and say which joints
   are responsible.
2. **Measure proximity.** Plot a manipulability measure, or the smallest singular value of J, along
   a straight-line path that passes near that configuration.

   > Choose the line so that the arm can travel it. At a singular configuration the arm loses one
   > task-space direction, given by the last left singular vector of J, and that is the one
   > direction it cannot produce there. A line aimed along it stalls the damped run of item 3
   > rather than demonstrating anything. Approach along a direction the Jacobian still covers
   > well, and let the configuration degrade as the arm travels.
3. **Show the failure, then fix it.** Run resolved-rate control along that path with the direct
   inverse and plot the commanded joint velocities; expect them to grow without bound. Repeat with
   damped least squares,

   ```math
   \dot{\theta} = J^{\mathrm{T}}\left(JJ^{\mathrm{T}} + \lambda^{2} I\right)^{-1} \zeta
   ```

   and plot both for comparison. Replay the damped run in RViz. Do not run either on hardware.
4. **Characterize the trade-off.** Damping bounds the joint velocities at the cost of tracking
   accuracy. Quantify both effects from your data and describe how λ would be chosen for a given
   application.

   Report which of the two runs tracked the commanded line more closely, and account for what you
   find. The direct inverse is the exact solution of the velocity mapping, so the expectation is
   that it tracks better. Compare the joint displacement it asks for in a single time step against
   the range over which the Jacobian describes the mapping, and say whether that expectation
   survives near a singular configuration.

Item 4 is the central result of this project.

## 6. Part D: Hardware session

One supervised session per team.

> **Safety.** Put joint velocity limits in software before connecting to the arm, holding commanded
> velocities well below the servo maxima. Run the trajectory in visualization first, send the arm to
> the rest pose `[0, pi/2, pi/2, 0, 0, 0]` at the start and end, raise `goal_time` on the first
> motion, and stay within reach of the power disconnect. A resolved-rate loop with a sign error will
> drive the arm into the work surface quickly.

Run the straight-line trajectory from Part B on the physical arm, using the same joint angles you
replayed in RViz. Do not run the near-singular trajectory of Part C.

The same joint angles were commanded in both cases, so any difference in the resulting gripper
position comes from the gap between the kinematic model and a real machine with servo error,
gravitational deflection, and finite structural stiffness. Measure the endpoint by the procedure
from Project 1 and report the difference. This gap is usually called the reality gap, and it is
worth seeing with a trajectory simple enough that no other explanation is available.

## 7. Deliverables

- The extended `kinematics/` package with `jacobian.py` and `control.py`.
- The Jacobian verification: largest disagreement against finite differences, and the h you chose.
- The straight-line path plot, realized against commanded, with the largest deviation.
- The determinant sweep from Part C item 1, the singular configuration found, and the
  manipulability plot.
- Joint velocity plots for the direct inverse and for damped least squares along that path, and the
  discussion of the damping trade-off.
- A record of the hardware session.
- A statement of how the work was divided among team members.

## 8. Remarks

A Jacobian that passes the finite-difference check but behaves badly inside the control loop usually
means inconsistent frames, with velocities in the base frame combined with velocities in the
end-effector frame. Fix the convention at the start and note it in the code.

A straight-line path that drifts steadily in one direction, rather than wandering around the
commanded path, usually means an error in the orientation part of the velocity command rather than
the position part.
