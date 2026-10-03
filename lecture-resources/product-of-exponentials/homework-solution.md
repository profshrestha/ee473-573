# Homework Solution: Product of Exponentials

**[Download these solutions (PDF)](homework-solution.pdf)** &nbsp;&middot;&nbsp;
**[Back to the problem set](homework.md)** &nbsp;&middot;&nbsp;
**[Lecture page](README.md)**

Check Canvas for deliverables, deadlines, and grading rubric.

Worked solutions. Every numerical result was checked against an independent implementation. Read
these against your own work rather than instead of it.

---

## Problem 1. SCARA

### (a) Screw axes

All three revolute axes point along the base $z$ axis.

| Joint | $\omega$ | $q$ at home | $v = -\omega \times q$ | $S$ |
|---|---|---|---|---|
| 1, revolute | $(0,0,1)$ | $(0,0,0)$ | $(0,0,0)$ | $(0,0,1,\;0,0,0)$ |
| 2, revolute | $(0,0,1)$ | $(L_1,0,0)$ | $(0,-L_1,0)$ | $(0,0,1,\;0,-L_1,0)$ |
| 3, revolute | $(0,0,1)$ | $(L_1{+}L_2,0,0)$ | $(0,-(L_1{+}L_2),0)$ | $(0,0,1,\;0,-(L_1{+}L_2),0)$ |
| 4, prismatic | $(0,0,0)$ | not used | $(0,0,-1)$ | $(0,0,0,\;0,0,-1)$ |

For joint 2, $(0,0,1) \times (L_1,0,0) = (0, L_1, 0)$, so $v_2 = (0, -L_1, 0)$.

**Why $q_2 = (L_1, 0, 0)$ and not $(L_1, 0, H)$.** The joint 2 axis physically sits at height
$H$, so $(L_1, 0, H)$ looks like the more honest answer.

Both are correct, and they give the same screw axis. The axis is a **line**, not a point: the
vertical line through $x = L_1$, $y = 0$, extending without limit in both $z$ directions. Every
point on that line is a legal choice of $q$, and the point at ground level is on it just as much
as the point at height $H$.

The arithmetic shows why the answer cannot depend on which is chosen:

```math
(0,0,1) \times (L_1, 0, H) = (0 \cdot H - 1 \cdot 0,\;\; 1 \cdot L_1 - 0 \cdot H,\;\; 0) = (0,  L_1,  0)
```

The $H$ cancels. It multiplies the component of $q$ that is **parallel to $\omega$**, and a vector
crossed with anything parallel to itself contributes nothing. So $v_2 = (0,-L_1,0)$ either way,
and the same holds for any third component whatsoever, including a negative one.

This is the invariance stated in §5.1. The model never needs to know where along the axis the
joint physically sits, only which line the axis is.

Either point is a correct answer. What is **not** correct is a point off the axis, such as the
tool position, because that is a different line altogether.

**The prismatic rule is different** because a sliding joint has no axis of rotation to take a
moment about. There is no $q$ at all: $\omega = 0$, and $v$ is simply the unit vector along the
direction of travel. Travel is downward, so $v_4 = (0,0,-1)$. Writing $(0,0,1)$ here builds an
arm whose tool rises when the joint extends.

**On the symbol $\theta_4$.** It carries metres, not radians. The notation is deliberate: §2 of the
lecture page writes every joint variable as $\theta_i$ regardless of type, because the product of
exponentials treats both kinds identically. $e^{[\mathcal{S}]\theta}$ does not care whether
$[\mathcal{S}]$ encodes a rotation or a translation, and that uniformity is one of the things the
method buys. That is why a length is called theta.

### (b) Home configuration

At home the arm is unrotated and lies along $x$, and $\{t\}$ is aligned with $\{0\}$, so the
rotation block is the identity. The tool is forward by $L_1 + L_2$ and sits at height $H - L_3$:

```math
M = \begin{bmatrix} 1 & 0 & 0 & L_1+L_2 \\ 0 & 1 & 0 & 0 \\ 0 & 0 & 1 & H-L_3 \\ 0 & 0 & 0 & 1\end{bmatrix}
```

### (c) The model

```math
H^0_t(\theta) = e^{[\mathcal{S}_1]\theta_1}  e^{[\mathcal{S}_2]\theta_2}  e^{[\mathcal{S}_3]\theta_3}  e^{[\mathcal{S}_4]\theta_4}  M
```

### (d) Numerical answer

```python
import numpy as np
from modern_robotics import FKinSpace

H, L1, L2, L3 = 0.40, 0.35, 0.25, 0.15

# screw axes as COLUMNS of a 6 x 4 matrix: omega on top, v beneath
Slist = np.array([
    [0, 0,   0,        0],
    [0, 0,   0,        0],
    [1, 1,   1,        0],
    [0, 0,   0,        0],
    [0, -L1, -(L1+L2), 0],
    [0, 0,   0,       -1],
], dtype=float)

M = np.array([[1, 0, 0, L1+L2],
              [0, 1, 0, 0],
              [0, 0, 1, H-L3],
              [0, 0, 0, 1]], dtype=float)

print(FKinSpace(M, Slist, [0, 0, 0, 0]))                              # must return M
theta = [np.radians(30), np.radians(-60), np.radians(45), 0.10]
print(FKinSpace(M, Slist, theta))
```

The home check returns $M$ with zero error. At the requested configuration:

```math
H^0_t = \begin{bmatrix}
0.96593 & -0.25882 & 0 & 0.51962 \\
0.25882 & 0.96593 & 0 & 0.05000 \\
0 & 0 & 1 & 0.15000 \\
0 & 0 & 0 & 1
\end{bmatrix}
```

Every entry is checkable by hand, so a disagreement can be traced to a single line:

```math
x = L_1\cos\theta_1 + L_2\cos(\theta_1{+}\theta_2) = 0.35\cos 30° + 0.25\cos(-30°) = 0.51962
```
```math
y = L_1\sin\theta_1 + L_2\sin(\theta_1{+}\theta_2) = 0.175 - 0.125 = 0.05000
```
```math
z = H - L_3 - \theta_4 = 0.40 - 0.15 - 0.10 = 0.15000
```

and the rotation block is $\mathrm{Rot}(z,\; \theta_1{+}\theta_2{+}\theta_3)
= \mathrm{Rot}(z,  15°)$, since the three parallel axes add.

Note that $\theta_3$ moves the tool's **orientation only**. The tool sits on the joint 3 axis, so
rotating about that axis leaves its position fixed. If your $x$ or $y$ changes with $\theta_3$,
$q_3$ has been placed off the axis.

Checked against the closed form over 3000 random configurations: worst disagreement
$9.4 \times 10^{-16}$.

### (e) The two degenerate axes

- $S_1$ has **zero linear part**. The joint 1 axis is the $z_0$ axis itself, so it passes through
  the base origin, $q = 0$, and the moment $-\omega \times q$ vanishes. A line through the origin
  has no moment about the origin.
- $S_4$ has **zero angular part**. A prismatic joint produces no rotation, so $\omega = 0$ by
  definition, and §6 shows the exponential then collapses to a pure translation $v\theta$.

Note that "joint 1 is the first joint" is not the reason. What matters is that its axis happens
to pass through the base origin. Had the base frame been placed elsewhere, $v_1$ would not
vanish.

---

## Problem 2. One rotation, three representations

### (a) Unit check and the skew matrices

$\|\omega\|^2 = \tfrac{4}{9} + \tfrac{1}{9} + \tfrac{4}{9} = 1$. ✓

```math
[\omega] = \frac{1}{3}\begin{bmatrix} 0 & -2 & -1 \\ 2 & 0 & -2 \\ 1 & 2 & 0 \end{bmatrix}
\qquad
[\omega]^2 = \omega\omega^T - I = \frac{1}{9}\begin{bmatrix} -5 & -2 & 4 \\ -2 & -8 & -2 \\ 4 & -2 & -5 \end{bmatrix}
```

Using the identity from §3.1 for $[\omega]^2$ is faster than multiplying the matrix by itself.

### (b) Rodrigues

$\theta = 120°$, so $\sin\theta = \tfrac{\sqrt3}{2}$ and $1 - \cos\theta = \tfrac{3}{2}$:

```math
R = I + \frac{\sqrt3}{2}[\omega] + \frac{3}{2}[\omega]^2
```

```math
R = \frac{1}{6}\begin{bmatrix}
1 & -2-2\sqrt3 & 4-\sqrt3 \\
-2+2\sqrt3 & -2 & -2-2\sqrt3 \\
4+\sqrt3 & -2+2\sqrt3 & 1
\end{bmatrix}
\approx
\begin{bmatrix}
0.1667 & -0.9107 & 0.3780 \\
0.2440 & -0.3333 & -0.9107 \\
0.9553 & 0.2440 & 0.1667
\end{bmatrix}
```

### (c) The inverse, by §3.5

```math
\mathrm{tr} R = \tfrac{1}{6} - \tfrac{1}{3} + \tfrac{1}{6} = 0
\qquad\Longrightarrow\qquad
\theta = \arccos\frac{0-1}{2} = \arccos\left(-\tfrac12\right) = \frac{2\pi}{3} \;\;✓
```

```math
R - R^{T} = \frac{1}{\sqrt3}\begin{bmatrix} 0 & -2 & -1 \\ 2 & 0 & -2 \\ 1 & 2 & 0 \end{bmatrix}
\qquad
[\omega] = \frac{R - R^{T}}{2\sin\theta} = \frac{R - R^{T}}{\sqrt3}
= \frac{1}{3}\begin{bmatrix} 0 & -2 & -1 \\ 2 & 0 & -2 \\ 1 & 2 & 0 \end{bmatrix}
```

Reading off $-\omega_3$, $\omega_2$, $-\omega_1$ gives $\omega = (\tfrac23, -\tfrac13, \tfrac23)$,
which is what we started with. ✓

The trace coming out exactly zero is not a coincidence worth chasing; it is simply what
$\theta = 120°$ gives, since $\mathrm{tr} R = 1 + 2\cos\theta$.

### (d) Quaternion

$\cos\tfrac{\theta}{2} = \cos 60° = \tfrac12$ and $\sin\tfrac{\theta}{2} = \tfrac{\sqrt3}{2}$:

```math
Q = \left(\tfrac12,\;\; \tfrac{\sqrt3}{3},\;\; -\tfrac{\sqrt3}{6},\;\; \tfrac{\sqrt3}{3}\right)
\approx (0.5000,\; 0.5774,\; -0.2887,\; 0.5774)
```

```math
\|Q\|^2 = \tfrac14 + \tfrac13 + \tfrac{1}{12} + \tfrac13 = 1 \;\;✓
```

---

## Common errors

If your answer disagrees with the above, check these first.

1. $v = +\omega \times q$, with the minus sign of the definition dropped. The symptom is a model
   that is correct at the home configuration and wrong everywhere else.
2. $v_4 = (0,0,1)$ for the prismatic joint, which inverts the $z$ motion of the whole arm.
3. `Slist` built with the screw axes as **rows** instead of columns. `FKinSpace` accepts the array
   without complaint and returns a pose that is simply wrong. This is the reason Problem 1(d) asks
   for the home check first.
4. Decimalizing Problem 2 early, which hides the exact trace of zero and makes part (c) look like a
   numerical accident rather than an identity.
