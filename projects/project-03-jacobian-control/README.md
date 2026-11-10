# Project 3: The Manipulator Jacobian, Velocity Control, and Numerical Inverse Kinematics

**Individual project.** The work extends the WidowX AI library the team developed in Project 2.
Most of the project runs in visualization; one supervised session runs on hardware.

Check Canvas for deliverables, deadlines, and the grading rubric.

## 1. Objectives

Upon completion of this project, students will be able to:

- Build the geometric Jacobian of a serial manipulator from its forward kinematics.
- Check an analytically derived Jacobian against a numerical approximation.
- Use the Jacobian to command motion in task space rather than joint space.
- Identify kinematic singularities, describe their effect on the velocity solution, and apply
  damped least squares to work around them.
- Set up inverse kinematics as an iterative problem and assess the resulting solver against an
  independent closed-form solution.
- Account for the difference between a kinematic model and the behavior of real hardware.

## 2. Starting point

Import the team's Project 2 `kinematics/` package. This project extends that work rather than
replacing it. The arm is the same: the WidowX AI, six revolute joints, commanded over Ethernet
through the Python driver.

Extend the package with the following modules:

```
kinematics/
  jacobian.py     Jacobian construction and the velocity mappings
  control.py      resolved-rate loop, pseudoinverse and damped least squares
  ik_numeric.py   iterative inverse kinematics
robot/
  viz.py          publication of a joint trajectory to RViz
```

The closed-form solver from Project 2 has to stay importable, since Part F compares the two
solutions directly.

### 2.1 Visualization environment

Most of this project runs in RViz rather than on hardware. Resolved-rate control is purely
kinematic: joint velocities are integrated to get joint positions, and no contact, gravitational, or
inertial effects enter the formulation. A dynamic simulator is therefore unnecessary. All that is
needed is a way to watch the arm follow a trajectory.

Publishing the integrated joint angles as `sensor_msgs/JointState` makes the robot state publisher
place the arm in RViz, exactly as the joint state publisher interface did in Project 1. A full
trajectory can be replayed by publishing successive configurations on a timer.

Two things motivate this arrangement. Hardware access is shared among all teams, and this project
calls for repeated runs at different damping factors. More importantly, Part D requires driving the
arm deliberately into a singular configuration, which is not permitted on hardware under any
circumstances.

## 3. Part A: Building the Jacobian

Build the 6 × n **geometric Jacobian** one column at a time, taking every quantity from the
forward kinematics developed in Project 2.

For a revolute joint, the i-th column is

```math
J_i = \begin{bmatrix} z_{i-1} \times (o_n - o_{i-1}) \\ z_{i-1} \end{bmatrix}
```

and for a prismatic joint it is `z_{i-1}` over a zero vector.

Every quantity in this expression is already available. The vector `z_{i-1}` is the third column and
`o_{i-1}` the fourth column of the homogeneous transformation from the base frame to frame i-1,
which is a partial product of the forward kinematics. The construction does not depend on how that
forward kinematics was obtained: whether the model came from a Denavit-Hartenberg table or from the
product of exponentials, the Jacobian needs only the direction of each joint axis and the position
of each frame origin.

The resulting matrix maps joint rates to the linear velocity of the end effector over its angular
velocity, both expressed in the **base frame**. Record this convention explicitly in the code, since
it governs how the error vector is formed in Parts C and E.

The WidowX AI has only revolute joints, so only the first form gets exercised. Implement the
prismatic case anyway: the general construction is no harder, and prismatic joints are common in
practice.

> Texts differ in the order of the two blocks. This project puts linear velocity above angular
> velocity. Before comparing a derivation against a Jacobian from a software library, check which
> order that library uses.

## 4. Part B: Numerical verification

Check an analytically derived Jacobian against a finite-difference approximation before using it.

For each joint i, perturb that joint variable by a small increment, recompute the end-effector
pose from the forward kinematics, and form the difference quotient. The result approximates the
i-th column of the Jacobian and can be compared against the analytical expression.

The position rows follow directly from a difference of position vectors. The orientation rows do
not, because the difference of two rotation matrices is not itself a rotation and carries no
kinematic meaning. Form the relative rotation `R_pert Rᵀ` instead, pull out its
axis and angle with the matrix logarithm on SO(3) from §3.5 of the product of exponentials notes,
and divide by the perturbation. That result approximates the angular velocity column.

The verification should meet the following requirements:

- At least fifty configurations spread across the workspace, rather than a single configuration
  picked by hand.
- The largest discrepancy seen across all configurations and all columns.
- A study of the perturbation size. An increment that is too large gives a poor approximation, and
  one that is too small is swamped by floating-point error. Plot the discrepancy against increment
  size and choose the increment from that data.

This procedure catches most derivation errors, including sign errors and off-by-one errors in frame
indexing, which are the two most common.

## 5. Part C: Resolved-rate control

Given a desired end-effector velocity, find the joint velocities that produce it and integrate the
result into a trajectory.

**Task.** Move the gripper in a straight line in task space between two reachable points, holding
orientation constant.

1. Implement the velocity mapping. The WidowX AI has six joints and the task has six components, so
   the Jacobian is square. Away from singular configurations it can be inverted directly; prefer
   `numpy.linalg.solve` over forming the inverse explicitly.
2. Generate the trajectory by integrating joint velocities over small time steps.
3. Recover the realized path by running the resulting joint angles through the forward kinematics.
4. Replay the trajectory in RViz. A path that looks right in a plot but wrong on screen usually means
   the frames are inconsistent, and the visualization finds that far faster than numerical
   inspection does.
5. Plot the realized path against the commanded straight line and report the largest deviation.

The deviation will not be zero, since the procedure integrates a linearization of a nonlinear
mapping. Report how it varies with the integration step size.

## 6. Part D: Singular configurations

A six-axis arm is singular wherever J loses rank. Where the last three axes intersect at a point,
det J factors into an arm term and a wrist term and the two families separate cleanly. The WidowX AI
does not have that geometry: `joint_3` and `joint_4` are skew, offset by 67.75 mm, so det J does not
factor, and it varies with the fourth joint variable in a way the factored form forbids. Confirm
that for yourself in item 1; it is the reason the two families have to be found numerically here.

1. **Find two distinct singular configurations.** One arises from the first three joints, as when
   the elbow is fully extended, and that one can be derived in closed form. The other involves the
   wrist. Locate them by searching numerically for configurations where det J reaches zero, or where
   the smallest singular value does. Report both configurations and say which joints are responsible
   for each. Before searching, sweep each joint in turn and record whether det J depends on it: an
   arm whose wrist axes intersect gives a determinant independent of the first and last joint
   variables, and the departure from that is the offset above.
2. **Measure proximity to singularity.** Use a manipulability measure or the smallest singular value
   of the Jacobian, and plot it along a straight-line path that passes near the arm singularity.
3. **Show the failure.** Run resolved-rate control along that path using the direct inverse, and plot
   the commanded joint velocities. Expect them to grow without bound as the Jacobian becomes
   ill-conditioned. Replay this trajectory in RViz as well: the plot shows that the velocities
   diverged, while the visualization shows what that means physically. Do not run this trajectory on
   hardware.
4. **Apply damped least squares.** Repeat the trajectory using

   ```math
   \dot{\theta} = J^{\mathrm{T}}\left(JJ^{\mathrm{T}} + \lambda^{2} I\right)^{-1} v
   ```

   and plot the commanded joint velocities for comparison.
5. **Characterize the trade-off.** Damping bounds the joint velocities at the cost of tracking
   accuracy, so the end effector no longer follows the commanded path exactly. Quantify both effects
   from the collected data and describe how the damping factor would be chosen for a given
   application.
6. **Compare the two classes.** With both an arm and a wrist singularity in hand, determine which is
   easier to avoid during task planning, and explain why.

Item 5 is the central result of this project.

## 7. Part E: Numerical inverse kinematics

The damped least squares formulation from Part D solves the inverse kinematic problem when it is
given a pose **error** in place of a desired velocity. The procedure is iterative: compute the
displacement between the current and desired poses, map it through the damped inverse to a joint
correction, apply it, and repeat until the error falls below tolerance.

### 7.1 The error vector

The position error is a vector difference. The orientation error is not.

```math
e_p = p_d - p(\theta)
```

```math
R_e = R_d\,R(\theta)^{\mathrm{T}}
\qquad\Longrightarrow\qquad
(\hat{\omega}, \theta_e) = \log R_e
\qquad\Longrightarrow\qquad
e_\omega = \hat{\omega}\,\theta_e
```

The logarithm is the same SO(3) operation used in Part B. The two components stack in the order
set by the Jacobian of Part A, linear above angular:

```math
e = \begin{bmatrix} e_p \\ e_\omega \end{bmatrix},
\qquad
\Delta\theta = J^{\mathrm{T}}\left(JJ^{\mathrm{T}} + \lambda^{2} I\right)^{-1} e
```

Two points matter here.

**Orientation error is not a difference of Euler angles.** Subtracting two sets of Euler angles
componentwise does not give a rotation. The result jumps where the angles wrap, and near a
representation singularity it can be large even when the two orientations are close together. The
logarithmic map does neither.

**The error and the Jacobian have to be in the same frame.** The product `R_d Rᵀ` gives
the orientation error in the base frame, which is the frame the Jacobian of Part A is written in.
The product `Rᵀ R_d` gives the same error in the end-effector frame and needs a matching
Jacobian. Mixing one with the other gives a solver that converges slowly, converges to the wrong
answer, or does not converge at all.

### 7.2 Implementation requirements

1. Two convergence tolerances, one on position in meters and one on orientation in radians. A single
   combined threshold would add quantities with different units.
2. An iteration limit and a status flag. Convergence is not guaranteed from an arbitrary starting
   estimate, and the routine has to report whether it converged rather than returning the closest
   configuration it happened to reach.
3. A per-iteration record of the joint variables and both error norms.

### 7.3 Analysis

Report the following:

- Convergence behavior, error against iteration count, for several starting configurations.
- Sensitivity to the starting estimate, comparing a good estimate against a deliberately poor one.
- The solution branch reached in each case, and how far the branch can be controlled through the
  starting estimate.
- Behavior in failure cases, specifically an unreachable target and a target at a singular
  configuration.

## 8. Part F: Comparison of the two solvers

Two independent solutions to the same inverse kinematic problem are now available, obtained by
different methods in different projects. Each one can be used to check the other.

1. Generate at least one hundred reachable poses by sampling joint configurations and running them
   through the forward kinematics, so that a solution is guaranteed to exist by construction.
2. Solve each pose both ways: the closed-form solution from Project 2 and the iterative
   solution from Part E.
3. Compare the results. Agreement is evidence that both are correct. Where the two disagree,
   determine which one is wrong and find the cause.
4. Measure the run time of each and report the ratio.

Expect disagreement near singularities, at joint limits, and wherever multiple solutions exist and
the two methods pick different branches. These cases are the substance of the comparison rather than
flaws in it.

Close with a recommendation. For an application that has to solve many poses per second, which
method is better, and on what grounds? For an application whose arm geometry keeps changing, which
method is better?

## 9. Part G: Hardware session

One supervised session, in teams of three.

> **Safety.** Put joint velocity limits in software before connecting to the arm, holding commanded
> velocities well below the servo maxima. Run the full trajectory in visualization first and stay
> within reach of the power disconnect. A resolved-rate loop with a sign error will drive the arm
> into the work surface quickly.

Run the straight-line trajectory from Part C on the physical arm, using the same joint angles
replayed in RViz. Do not run the near-singular trajectory of Part D.

Compare the two outcomes. The same joint angles were commanded in both cases, so any difference in
the resulting gripper position comes from the gap between the kinematic model and a real machine
with servo error, gravitational deflection, and finite structural stiffness. Measure the endpoint by
the procedure from Project 1 and report the difference. This gap between model and machine is
usually called the reality gap, and it is worth seeing with a trajectory simple enough that no other
explanation is available.

## 10. Deliverables

- The extended `kinematics/` package with `jacobian.py`, `control.py`, `ik_numeric.py`, and
  `robot/viz.py`.
- The finite-difference check: largest discrepancy across the test configurations, and the study of
  increment size.
- The straight-line path plot, realized against commanded, with the largest deviation.
- Confirmation that both trajectories were replayed in visualization, with a description of the
  near-singular replay.
- The manipulability plot along a path passing near a singularity, together with both singular
  configurations found in Part D.
- Joint velocity plots for the direct inverse and for damped least squares along that path.
- A discussion of the damping trade-off, supported by the collected data.
- The comparison of the two singular configurations: which is easier to avoid in task planning,
  and why.
- Convergence plots for the iterative solver and its behavior in the failure cases listed above.
- The Part F comparison over one hundred or more poses, including timing, with the closing
  recommendation.
- A record of the hardware session.

## 11. Remarks

A Jacobian that passes the finite-difference check but behaves badly inside the control loop usually
means inconsistent frames, with velocities in the base frame combined with velocities in the
end-effector frame. Fix the convention at the start and note it in the code.

A straight-line path that drifts steadily in one direction, rather than wandering around the
commanded path, usually means an error in the orientation part of the velocity command rather than
the position part.

The error vector in §7.1 is a position difference together with a rotation logarithm. This is the
standard formulation in practice and it converges reliably. Note, though, that it is not the same as
the logarithm of the full pose error on SE(3), which couples rotation and translation into a single
screw displacement and takes a different path to the same answer. The two formulations reach the same
configuration; only the intermediate iterates differ.
