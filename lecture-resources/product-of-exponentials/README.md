# Forward Kinematics: Product of Exponentials

A second route to forward kinematics, independent of the Denavit-Hartenberg convention
of Chapter 3. Check Canvas for deliverables, deadlines, and grading rubric.

Notations: $H^0_n$ is the pose of frame $n$ expressed in frame 0, $R$ is a rotation, $d$ a
displacement, and $A_i$ the Denavit-Hartenberg link transform.

---

## 1. Why a second method

Below is the DH table for the revolute-plus-prismatic arm example:

| Link | θ | d | a | α |
|---|---|---|---|---|
| 1 | θ₁ + 90 | 0 | l₁₂ | 90 |
| 2 | 0 | l₂ + l₁₁ | 0 | 0 |

and here is an entry from Trossen's published table for the WidowX AI:

```
θ₂ − π/2 − arctan(0.245 / 0.06)
```

Neither offset describes either the mechanical construct of the arm or its operation. No link is
built with a 90 degree bend at that joint, and no part of the arm is tilted by arctan(0.245/0.06)
in any configuration the motors can command. Both numbers exist only because the DH rules force xᵢ
to be perpendicular to z₍ᵢ₋₁₎ and to intersect it, and the axes the mechanism actually provides do
not satisfy that on their own. The offset describes the bookkeeping of the convention, not the
hardware.

Two consequences:

1. Frame assignment is error prone.
2. Frame assignments, and so DH tables, are not unique.

Product of exponentials removes frame assignment entirely. The model needs one base frame, one
measurement of the robot in a home configuration, and one screw axis per joint expressed in that
base frame. There are no intermediate frames, so there is nothing to assign and no offsets to
carry.

---

## 2. Notation

| Symbol | Meaning |
|---|---|
| ω | unit vector along a rotation axis |
| q | any point on that axis, in base coordinates |
| v | the linear part of a screw axis |
| S = (ω, v) | a screw axis, six numbers |
| [ω] | the 3×3 skew-symmetric matrix built from ω |
| [S] | the 4×4 matrix built from S |
| M | the home configuration: the tool pose with every joint at zero |
| θᵢ | joint variable i |

Everything is expressed in the fixed base frame unless stated otherwise.

---

## 3. Rotation as a matrix exponential

### 3.1 Skew-symmetric matrices

For a vector ω = (ω₁, ω₂, ω₃):

```math
[\omega] = \begin{bmatrix} 0 & -\omega_3 & \omega_2 \\ \omega_3 & 0 & -\omega_1 \\ -\omega_2 & \omega_1 & 0 \end{bmatrix}
```

The skew-symmetric matrix $[\omega]$ expresses the cross product as a matrix multiplication:

```math
[\omega] p = \omega \times p
```

The transpose satisfies $[\omega]^T = -[\omega]$ (skew-symmetric definition), and
$[\omega] \omega = 0$, since a vector crossed with itself vanishes.

Three identities hold when ω is a unit vector:

```math
[\omega]^2 = \omega\omega^T - I \qquad\qquad [\omega]^3 = -[\omega] \qquad\qquad [\omega]^4 = -[\omega]^2
```

### 3.2 The matrix exponential

The exponential of a square matrix uses the same series as the scalar exponential:

```math
e^{X} = I + X + \frac{X^2}{2!} + \frac{X^3}{3!} + \cdots
```

Applied to $[\omega]\theta$ with ω a unit vector, the series closes on itself. Because
$[\omega]^3 = -[\omega]$, every higher power falls back onto either $[\omega]$ or $[\omega]^2$, and
the terms group as

```math
e^{[\omega]\theta} = I + \left(\theta - \frac{\theta^3}{3!} + \frac{\theta^5}{5!} - \cdots\right)[\omega] + \left(\frac{\theta^2}{2!} - \frac{\theta^4}{4!} + \frac{\theta^6}{6!} - \cdots\right)[\omega]^2
```

Those two brackets are the Taylor series for sin θ and 1 − cos θ.

### 3.3 Rodrigues' formula

```math
e^{[\omega]\theta} = I + \sin\theta [\omega] + (1 - \cos\theta) [\omega]^2
```

The result is a rotation of θ radians about the axis ω, by the right-hand rule.
$e^{[\omega]\theta}$ is a rotation matrix, and ω and θ are the same axis and angle that would
appear on a drawing of the joint.

### 3.4 Worked check

For ω = (0, 0, 1), a rotation about the base z axis:

```math
[\omega] = \begin{bmatrix} 0 & -1 & 0 \\ 1 & 0 & 0 \\ 0 & 0 & 0\end{bmatrix}
\qquad
[\omega]^2 = \begin{bmatrix} -1 & 0 & 0 \\ 0 & -1 & 0 \\ 0 & 0 & 0\end{bmatrix}
```

Substituting into Rodrigues:

```math
e^{[\omega]\theta} = \begin{bmatrix} \cos\theta & -\sin\theta & 0 \\ \sin\theta & \cos\theta & 0 \\ 0 & 0 & 1\end{bmatrix}
```

which is Rot(z, θ).

### 3.5 The inverse: matrix back to axis and angle

Rodrigues' formula runs from ω and θ to R. The inverse direction is the matrix logarithm on SO(3).
Taking the trace of Rodrigues' formula, and using tr[ω] = 0 and tr[ω]² = −2,

```math
\mathrm{tr} R = 1 + 2\cos\theta \qquad\Longrightarrow\qquad \theta = \arccos\frac{\mathrm{tr} R - 1}{2}
```

and subtracting the transpose cancels the symmetric terms, leaving only the skew part:

```math
R - R^T = 2\sin\theta [\omega] \qquad\Longrightarrow\qquad [\omega] = \frac{R - R^T}{2\sin\theta}
```

Both expressions hold for 0 < θ < π and both degenerate when sin θ = 0. At θ = 0 the matrix is the
identity and there is no axis to recover. At θ = π the difference R − Rᵀ vanishes, and the axis has
to be read from R + I instead, with its sign undetermined, since a half turn about ω and a half
turn about −ω are the same rotation.

---

## 4. Quaternions

Section 3 represents a rotation by an axis and an angle, and Rodrigues' formula turns that pair
into nine matrix entries. A quaternion holds the same axis and angle in four numbers. That is the
form ROS transports and the form most libraries store internally.

### 4.1 Definition

```math
Q = w + x i + y j + z k, \qquad i^2 = j^2 = k^2 = ijk = -1
```

The relation among i, j and k is Hamilton's, and every quaternion product follows from it.

Conventionally this is written as a scalar part and a vector part, Q = (w, v) with v = (x, y, z).
It represents a rotation only when it has unit length, where $\|Q\|$ is the norm, the length of Q
as a vector in four dimensions:

```math
\|Q\|^2 = w^2 + x^2 + y^2 + z^2 = 1
```

### 4.2 From axis and angle

```math
Q = \left(\cos\frac{\theta}{2},\;\; \omega \sin\frac{\theta}{2}\right)
```

The angle appears halved. That is why a full 2π rotation gives Q = (−1, 0) rather than the
identity, and why Q and −Q describe the same rotation. In the other direction, for w ≠ ±1:

```math
\theta = 2\arccos w \qquad\qquad \omega = \frac{\mathbf{v}}{\sin(\theta/2)}
```

At w = ±1 the angle is 0 or 2π, the vector part is zero, and the division becomes 0/0. The axis is
genuinely undefined there, since a rotation by nothing has no axis, and the division is already
ill-conditioned as w approaches either value. This is the same degeneracy as in section 3.5.

The three principal rotations, scalar first:

```math
Q_x = \left(\cos\tfrac{\theta}{2}, \sin\tfrac{\theta}{2},  0,  0\right) \qquad
Q_y = \left(\cos\tfrac{\theta}{2},  0, \sin\tfrac{\theta}{2},  0\right) \qquad
Q_z = \left(\cos\tfrac{\theta}{2},  0,  0, \sin\tfrac{\theta}{2}\right)
```

### 4.3 To a rotation matrix

```math
R = I + 2w [\mathbf{v}] + 2 [\mathbf{v}]^2
```

or written out:

```math
R = \begin{bmatrix}
1-2(y^2+z^2) & 2(xy - wz) & 2(xz + wy) \\
2(xy + wz) & 1-2(x^2+z^2) & 2(yz - wx) \\
2(xz - wy) & 2(yz + wx) & 1-2(x^2+y^2)
\end{bmatrix}
```

### 4.4 Composition, inverse, rotating a vector

The Hamilton product composes rotations:

```math
Q_1 \otimes Q_2 = \big(  w_1w_2 - \mathbf{v}_1\cdot\mathbf{v}_2,\;\; w_1\mathbf{v}_2 + w_2\mathbf{v}_1 + \mathbf{v}_1 \times \mathbf{v}_2 \big)
```

It is not commutative, which it must not be, since rotations do not commute. For a unit quaternion
the inverse is the conjugate, $Q^{-1} = Q^{*} = (w, -\mathbf{v})$, and a vector is rotated by

```math
p' = Q \otimes (0, p) \otimes Q^{*}
```

### 4.5 The double cover

Q and −Q give the identical rotation matrix. Every orientation has exactly two quaternion
representations. A test for equality therefore compares $|Q_1 \cdot Q_2|$ against 1 rather than
components against components; a comparison by components reports a harmless sign flip as a
failure.

### 4.6 Component order

REP 103, one of the ROS Enhancement Proposals, lists the preferred representations in order:
quaternion, rotation matrix, fixed-axis roll-pitch-yaw, and Euler angles last. The four numbers are
not always written in the same order, however:

| Where | Order |
|---|---|
| ROS `geometry_msgs/Quaternion` | x, y, z, w (scalar last) |
| `scipy.spatial.transform.Rotation` | x, y, z, w (scalar last) |
| Eigen `Quaterniond(w,x,y,z)` | scalar first |
| These notes, and most textbooks | scalar first |

A quaternion read in the wrong order is still unit length, so nothing raises an error. The result
is the wrong rotation, produced silently. The order has to be confirmed at every library boundary.

### 4.7 Why not Euler angles: gimbal lock

| | Euler angles | Rotation matrix | Quaternion | Exponential coordinates ωθ |
|---|---|---|---|---|
| Numbers stored | 3 | 9 | 4 | 3 |
| Degrees of freedom | 3 | 3 | 3 | 3 |
| Gimbal lock | yes | no | no | no |
| Composition | convention dependent | matrix product | quaternion product | no closed form |
| Interpolation | poor | awkward | smooth, by spherical linear interpolation (SLERP) | geodesic about a fixed axis |

Consider a physical three-axis gimbal with an outer yaw ring, a middle pitch ring, and an inner
roll ring. When the middle ring sits at exactly 90 degrees, the outer yaw axis and the inner roll
axis are parallel. Either one then produces the same motion as the other, so one degree of freedom
is no longer available, and a mechanism that could reach any orientation is confined to a plane.

The last column is the representation section 3 builds and the one the product of exponentials
carries: each joint contributes the three numbers ωᵢθᵢ. It stores no more than Euler angles and
has no gimbal lock, but it composes badly. The product of two exponentials is not the exponential
of the sum unless the axes are parallel, and no inexpensive formula gives the axis that results.
PoE avoids the difficulty by never composing exponential coordinates with each other. Each ωᵢθᵢ
passes through the exponential map first, and the resulting matrices compose. Two further cautions:
ωθ and ω(θ − 2π) describe the same rotation, so the representation is not unique once θ passes π,
and recovering ω from a rotation matrix is ill-conditioned as θ approaches 0 or π.

---

## 5. Screw axes

A screw axis describes a line in space together with motion along or about that line. It is six
numbers, S = (ω, v).

### 5.1 Revolute joint

Let ω be the unit vector along the joint axis and q any point on that axis, in base coordinates:

```math
\omega = \text{joint axis direction}, \qquad v = - \omega \times q
```

The choice of q is free, and any point on the axis produces the same screw axis. And v is neither a
velocity nor a position. It is the moment of the axis about the origin, and the minus sign belongs
to the definition. When the axis passes through the origin, q = 0 and v = 0.

### 5.2 Prismatic joint

```math
\omega = 0, \qquad v = \text{unit vector along the direction of travel}
```

### 5.3 Component order, again

Two orderings are in circulation:

| Convention | Layout | Used by |
|---|---|---|
| Angular first | S = (ω, v) | Lynch and Park, `modern_robotics`, these notes |
| Linear first | S = (v, ω) | much of the screw theory literature |

A screw axis read in the wrong order is still six plausible numbers. Nothing reports an error and
the arm moves somewhere else.

### 5.4 Matrix form

```math
[\mathcal{S}] = \begin{bmatrix} [\omega] & v \\ 0 & 0 \end{bmatrix} \in \mathbb{R}^{4\times 4}
```

---

## 6. The exponential of a screw axis

For a revolute joint, with ω a unit vector:

```math
e^{[\mathcal{S}]\theta} = \begin{bmatrix} e^{[\omega]\theta} & G(\theta) v \\ 0 & 1 \end{bmatrix}
\qquad
G(\theta) = I\theta + (1-\cos\theta) [\omega] + (\theta - \sin\theta) [\omega]^2
```

The rotation block is Rodrigues' formula from section 3.3. For a prismatic joint ω = 0, and the
expression reduces to a pure translation:

```math
e^{[\mathcal{S}]\theta} = \begin{bmatrix} I & v\theta \\ 0 & 1 \end{bmatrix}
```

---

## 7. Product of exponentials, space form

```math
H^0_n(\theta) = e^{[\mathcal{S}_1]\theta_1}  e^{[\mathcal{S}_2]\theta_2} \cdots e^{[\mathcal{S}_n]\theta_n}  M
```

Three quantities define the model:

1. M, the home configuration, which is the tool pose in the base frame with every joint at zero.
2. Sᵢ, one screw axis per joint, all expressed in the fixed base frame and all evaluated at the
   home configuration.
3. The joint values θᵢ.

The axes may be written at the home configuration even though the joints move, and the structure of
the product is the reason. It is read right to left. M places the tool at home, the rightmost
exponential acts on that pose, and each factor to its left acts on everything already assembled.
Every screw axis is therefore applied to a body that the joints outboard of it have already moved,
so no axis ever needs re-expressing.

---

## 8. Worked example: an articulated RRR arm

Three revolute joints in the articulated arrangement: a waist turning about the base z axis, then a
shoulder and an elbow whose axes are parallel to each other and perpendicular to the waist axis.
This is an RX150 with the wrist removed. L₁ is the height of the shoulder above the base, L₂ the
upper arm, and L₃ the forearm.

The base frame sits on the mounting surface with z up and x forward. At the home configuration
every joint angle is zero and the arm reaches straight out along the base x axis at the height of
the shoulder. The shoulder and elbow axes are taken along negative base y, so that positive θ₂ and
θ₃ raise the arm under the right-hand rule.

At home, the joint positions and the tool position are

```
q₁ = (0, 0, 0)          q₂ = (0, 0, L₁)          q₃ = (L₂, 0, L₁)
tool = (L₂ + L₃, 0, L₁)
```

The waist turns about a different direction from the other two:

```math
\omega_1 = (0, 0, 1) \qquad\qquad \omega_2 = \omega_3 = (0, -1, 0)
```

The linear part of each screw axis follows from v = −ω × q.

**Joint 1, the waist.** The axis passes through the origin, so the cross product vanishes:

```math
v_1 = -(0,0,1) \times (0,0,0) = (0, 0, 0)
\qquad\Rightarrow\qquad
\mathcal{S}_1 = (0, 0, 1,\;\; 0, 0, 0)
```

**Joint 2, the shoulder.** $(0,-1,0) \times (0,0,L_1) = (-L_1, 0, 0)$, so

```math
v_2 = (L_1, 0, 0)
\qquad\Rightarrow\qquad
\mathcal{S}_2 = (0, -1, 0,\;\; L_1, 0, 0)
```

**Joint 3, the elbow.** $(0,-1,0) \times (L_2,0,L_1) = (-L_1, 0, L_2)$, so

```math
v_3 = (L_1, 0, -L_2)
\qquad\Rightarrow\qquad
\mathcal{S}_3 = (0, -1, 0,\;\; L_1, 0, -L_2)
```

The L₁ in v₃ is the moment of the axis ω₃ about the base origin, which the axis sits L₁ above.

**Home configuration.** At zero the arm is unrotated, so the rotation block is the identity, and the
tool sits forward by L₂ + L₃ and up by L₁:

```math
M = \begin{bmatrix} 1 & 0 & 0 & L_2+L_3 \\ 0 & 1 & 0 & 0 \\ 0 & 0 & 1 & L_1 \\ 0 & 0 & 0 & 1\end{bmatrix}
```

The model is complete:

```math
H^0_3(\theta) = e^{[\mathcal{S}_1]\theta_1} e^{[\mathcal{S}_2]\theta_2} e^{[\mathcal{S}_3]\theta_3} M
```

Every number came off the drawing, and no frames were assigned.

### 8.1 Check it against trigonometry

The example is worth carrying out because it can be verified independently. The articulated arm has
a closed-form solution that follows directly from the geometry. The shoulder and elbow move the
tool in a vertical plane, reaching a horizontal distance

```math
r = L_2\cos\theta_2 + L_3\cos(\theta_2{+}\theta_3)
```

from the waist axis at a height of

```math
z = L_1 + L_2\sin\theta_2 + L_3\sin(\theta_2{+}\theta_3)
```

and the waist then swings that plane about the base z axis:

```math
x = r\cos\theta_1 \qquad\qquad y = r\sin\theta_1
```

The orientation is Rot(z, θ₁) Rot(−y, θ₂ + θ₃), since the two parallel axes add.

Evaluated at any joint values, the product of exponentials agrees with those expressions exactly, in
both the translation column and the rotation block.

---

## 9. Compared with the DH table

| | DH | PoE |
|---|---|---|
| Intermediate frames | one per joint, each satisfying two constraints | none |
| Parameters | 4 per joint | 6 per joint, plus M |
| Constant offsets | yes, and they must be tracked | none |
| Unique? | no | not unique either, but only through the choice of q, which does not change S |
| Read off a drawing? | after frame assignment | directly |

PoE is not free. It carries six numbers per joint instead of four, and G(θ) is a heavier expression
than Aᵢ. What it returns is that every number in the model corresponds to something physically
present on the robot.

---

## 10. The URDF connection

A URDF describes each joint with exactly two things:

```xml
<joint name="joint_1" type="revolute">
  <origin rpy="0 0 0" xyz="0 0 0.06566"/>
  <axis xyz="0 0 1"/>
</joint>
```

The `<axis>` is ω. The accumulated `<origin>` translations give a point q on that axis. Those are
precisely the two inputs a screw axis needs.

No URDF contains a DH table, and none of the robots in this course are an exception. The format
that robotics software actually consumes is structurally a product-of-exponentials description, and
DH is the convention that has to be derived from it.

---

## 11. A note on Lie groups and Lie algebras

Three spaces have been in use throughout these notes without being named.

**R³** is three-dimensional real coordinate space, the static environment the robot sits in. A
point is located by three coordinates (x, y, z), and ordinary distances and vector operations
apply. The axis points q and the displacement d live here.

**SO(3)**, the special orthogonal group, is the set of all proper rotations about a fixed origin. A
matrix belongs to it when

```math
R^T R = I \qquad\qquad \det R = +1
```

The first condition preserves lengths and angles; the second excludes reflections, which is what
proper means. SO(3) has three degrees of freedom, and it changes the orientation of a body while
leaving the origin exactly where it is. Every R in these notes is an element of SO(3), including
every $e^{[\omega]\theta}$.

**SE(3)**, the special Euclidean group, is the set of all rigid body motions, rotation and
translation together. It is the semidirect product R³ ⋊ SO(3), semidirect rather than direct
because a translation applied after a rotation is not the same as the same translation applied
before it. SE(3) has six degrees of freedom, three of rotation and three of translation, and it is
exactly the homogeneous transform

```math
H = \begin{bmatrix} R & d \\ 0 & 1\end{bmatrix}, \qquad R \in SO(3), \quad d \in \mathbb{R}^3
```

Every $H^0_n$, every $A_i$, and every $e^{[\mathcal{S}]\theta}$ is an element of SE(3).

A Lie group is a curved, smooth space of configurations, and SO(3) and SE(3) are both Lie groups.
Ordinary linear algebra does not apply to either, because the sum of two rotation matrices is not a
rotation matrix. A Lie algebra, written so(3) or se(3), is the flat vector space tangent to
the group at the identity, and its elements are exactly the [ω] and [S] matrices of sections 3 and
5. The exponential map is the bridge from the flat space to the curved one, which is what
$e^{[\omega]\theta}$ and $e^{[\mathcal{S}]\theta}$ have been doing throughout.

A Lie algebra is more than a vector space. It carries a product of its own, the Lie bracket, which
for matrices is the commutator

```math
[A, B] = AB - BA
```

and the space is closed under it. In so(3) the bracket reproduces the cross product,

```math
\big[[\omega_1], [\omega_2]\big] = [\omega_1 \times \omega_2]
```

so the cross product of section 3.1 is the bracket of so(3) written in vector form. What the
bracket measures is the failure of two motions to commute. It vanishes exactly when the two
exponentials commute, which for rotations means parallel axes, and that is the same condition
under which exponential coordinates may be added rather than composed through the exponential map.

The elements of the algebra are velocities: so(3) holds angular velocities, and se(3) holds
twists, an angular and a linear velocity together. Velocities are what add normally, which is why
the algebra is flat while the group is not.
