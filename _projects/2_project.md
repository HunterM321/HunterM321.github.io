---
layout: page
title: CubeSat Control System
description: UBC Orbit Satellite Design Team, Attitude and Orbit Control System subteam
img: assets/img/sat.jpg
importance: 2
category: work
giscus_comments: false
---

#### Overview
[UBC Orbit Satellite Design Team](https://www.ubcorbit.com) is a student-run engineering design team, supported by UBC Faculty of Applied Science. Currently, we are working on our most challenging and most ambitious project called **[ALEASAT](https://www.esa.int/Education/CubeSats_-_Fly_Your_Satellite/Meet_the_team_Aleasat)**. ALEASAT is an Earth-observation **[1U CubeSAT](https://www.nasa.gov/what-are-smallsats-and-cubesats/)** designed for radio amateurs to support disaster relief efforts. It features an on-board camera, enabling users to request and directly receive targeted imagery of specific locations.

We are one of the four teams selected by **European Space Agency**'s [Fly Your Satellite](https://www.esa.int/Education/CubeSats_-_Fly_Your_Satellite) program, also making us the only Canadian team in this program.

<div class="row justify-content-sm-center">
    <div class="col-sm-8 mt-3 mt-md-0">
        {% include figure.liquid path="assets/img/aleasat.png" title="aleasat" class="img-fluid rounded" %}
    </div>
    <div class="col-sm-4 mt-3 mt-md-0">
        {% include figure.liquid path="assets/img/aleasat_structure.png" title="aleasat structure" class="img-fluid rounded" %}
    </div>
</div>
<div class="caption">
    ALEASAT and its breakdown structure
</div>

I have been with the team since 2021 and I currently lead the **Attitude and Orbit Control System** (AOCS) subteam. The AOCS subteam is in charge of determining and stabilizing the orientation of the satellite. The positioning of the satellite is determined using a variety of sensors, such as sun sensors and an Inertial Measurement Unit (IMU). The gathered data is then fed into our custom control system which controls the satellite's actuators, including magnetorquers and reaction wheels.

#### Spacecraft dynamics
To come up with a solution to a control problem, we always need to first start with the dynamics model. This model characterizes how does the state of our system change with time using our existing measurements. Our CubeSat has very complex dynamics as we utilize and fuse multiple sensors to obtain the observed state, yet each sensor has their own noise distributions.

To fully characterize the state of our CubeSat, we _simply_ need to know the orientation, or in a more professional way, the **attitude**, of the CubeSat. We chose to use the industry-standard **[quaternions](https://en.wikipedia.org/wiki/Quaternions_and_spatial_rotation)** to represent the attitude of the CubeSat. Quaternions can be thought as an extension of complex numbers and have the form:

$$
\mathbf{q} = w + x\mathbf{i} + y\mathbf{j} + z\mathbf{k}
$$

Notice that in this definition, the real scalar part comes before the complex vector part; some literature prefer the other way around. Recall in complex analysis, a multiplication of $$i$$ corresponds to a counter-clockwise rotation of 90 degrees. Quaternions generalize this idea into 3D space while delivering several advantages, including:
- Quaternions don't suffer limitations such as **[Gimbal Lock](https://en.wikipedia.org/wiki/Gimbal_lock)** which arises from using Euler angles (row, pitch, yaw) to represent 3D rotations.
- When a representing 3D rotation using Euler angles, we need to establish a $$3 \times 3$$ rotation matrix which consists of 9 entries. A quaternion with the same rotation effect only uses 4 entries, saving half of the space which accumulates in large computations.

Since we are using unit quaternions (i.e., $$\|\mathbf{q}\| = 1$$) to represent to represent attitude, we can reduced the quaternion further to only using the complex vector part, called the **reduced quaternion**. From now on, I will be using $$\mathbf{q}$$ to exclusively indicate reduced quaternion. Equipped with the knowledge of quaternions, we can proceed to writing down the dynamics of our CubeSat:

$$
\dot{\omega} = -\mathbf{J}^{-1} \omega \times (\mathbf{J} \omega) + \mathbf{J}^{-1} \mathbf{u} + \phi_1
$$

$$
\dot{\mathbf{q}} = \frac{1}{2} \mathbf{\Omega}(\omega + \phi_2)
$$

$$
\dot{\beta} = \phi_3
$$

$$
\omega_y = \omega + \beta + \psi_1
$$

$$
\mathbf{q}_y = \mathbf{q} + \psi_2
$$

Where:
- $$\mathbf{J}$$: Inertial matrix of the CubeSat
- $$w$$: CubeSat angular rate with respect to the inertial frame  
- $$\mathbf{u}$$: Control input
- $$\mathbf{\Omega}$$: Matrix defined as

$$
\mathbf{\Omega} =
\begin{bmatrix}
g(\mathbf{q}) & -q_3 & q_2 \\
q_3 & g(\mathbf{q}) & -q_1 \\
-q_2 & q_1 & g(\mathbf{q})
\end{bmatrix}
\quad \text{with} \quad g(\mathbf{q}) = \sqrt{1 - q_1^2 - q_2^2 - q_3^2}.
$$
  
- $$\phi_1$$, $$\phi_2$$, $$\phi_3$$: Process Gaussian noise
- $$\beta$$: Drift in angular measurement
- $$\omega_y$$: Angular rate measurement
- $$\mathbf{q}_y$$: Reduced quaternion measurement
- $$\psi_1$$, $$\psi_2$$: Measurement Gaussian noise
