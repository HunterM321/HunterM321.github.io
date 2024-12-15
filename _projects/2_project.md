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

We are one of the four teams selected by **European Space Agency** (ESA) to participate in their [Fly Your Satellite](https://www.esa.int/Education/CubeSats_-_Fly_Your_Satellite) program, making us the first and only Canadian team in this program.

<div class="row justify-content-sm-center">
    <div class="col-sm-8 mt-3 mt-md-0">
        {% include figure.liquid path="assets/img/aleasat_1.png" title="aleasat" class="img-fluid rounded" %}
    </div>
    <div class="col-sm-4 mt-3 mt-md-0">
        {% include figure.liquid path="assets/img/aleasat_structure.png" title="aleasat structure" class="img-fluid rounded" %}
    </div>
</div>
<div class="caption">
    ALEASAT (left) and its CAD breakdown structure (right)
</div>

I have been with the team since 2021 and I currently lead the **Attitude and Orbit Control System** (AOCS) subteam. The AOCS subteam is in charge of determining and stabilizing the orientation of the satellite. The positioning of the satellite is determined using a variety of sensors, such as sun sensors and an Inertial Measurement Unit (IMU). The gathered data is then fed into our custom control system which controls the satellite's actuators, including magnetorquers and reaction wheels.

<div class="row">
    <div class="col-sm-6 mt-3 mt-md-0">
        {% include figure.liquid path="assets/img/aleasat_assembly.png" title="aleasat assembly" class="img-fluid rounded z-depth-1" %}
    </div>
    <div class="col-sm-6 mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/aleasat_testpod.jpg" title="ALEASAT integrated inside the TestPod" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    ALEASAT assembly (left) and integration inside the TestPod (right) before undergoing vibration testing in ESA
</div>

_Our control system design was inspired by [Spacecraft Modeling, Attitude Determination, and Control: Quaternion-Based Approach](https://ntrs.nasa.gov/citations/20240009554), written by Dr. Yang, a NASA research scientist. Additionally, we used [Attitude and Heading Reference Systems](https://ahrs.readthedocs.io/en/latest/) (AHRS) Python library as another source of reference._

#### Spacecraft dynamics
To come up with a solution to a control problem, we always need to first start with the **dynamics** of the model. This is nothing but a glorified way of describing $$\mathbf{F}=m\mathbf{a}$$, Newton's second law of motion which we learned in high school or first-year university physics. This model characterizes how does the state of our system change with time given our existing measurements. Our CubeSat has very complex dynamics as we utilize and fuse multiple sensors to obtain the observed state, yet each sensor has its distinct noise distribution.

To fully characterize the state of our CubeSat, we _simply_ need to know the orientation, or in a more professional way, the **attitude**, of the CubeSat. We chose to use the industry-standard **[quaternions](https://en.wikipedia.org/wiki/Quaternions_and_spatial_rotation)** to represent the attitude of the CubeSat. Quaternions can be thought as an extension of complex numbers and have the form:

$$
\mathbf{q} = w + x\mathbf{i} + y\mathbf{j} + z\mathbf{k}
$$

Notice that in this definition, the real scalar part comes before the complex vector part; some literatures prefer the other way around. Recall in complex analysis, a multiplication of $$i$$ corresponds to a counter-clockwise rotation of 90 degrees in the complex plane. Quaternions generalize this idea into 3D space while delivering several advantages, including:
- Quaternions don't suffer limitations such as **[gimbal lock](https://en.wikipedia.org/wiki/Gimbal_lock)** which arises from using Euler angles (row, pitch, yaw) to represent 3D rotations.
- When representing 3D rotation using Euler angles, we need to establish a $$3 \times 3$$ rotation matrix which consists of 9 entries. A quaternion with the same rotation effect only uses 4 entries, saving half of the space which accumulates in large computations.

Since we are using unit quaternions (i.e., $$\|\mathbf{q}\| = 1$$) to represent to represent attitude, we can reduced the quaternion further to only using the complex vector part, called the **reduced quaternion**. From now on, I will be using $$\mathbf{q}$$ to exclusively indicate reduced quaternion. _Moreover, in this section we will only use the **gyroscope** inside our IMU as our attitude determination instrument_. Equipped with the knowledge of quaternions, we can proceed to writing down the dynamics of our CubeSat:

$$
\begin{aligned}
\dot{\boldsymbol{\omega}} &= -\mathbf{J}^{-1} \boldsymbol{\omega} \times (\mathbf{J} \boldsymbol{\omega}) + \mathbf{J}^{-1} \mathbf{u} + \boldsymbol{\phi}_1 \\
\dot{\mathbf{q}} &= \frac{1}{2} \mathbf{\Omega}(\boldsymbol{\omega} + \boldsymbol{\phi}_2) \\
\dot{\boldsymbol{\beta}} &= \boldsymbol{\phi}_3 \\
\boldsymbol{\omega}_y &= \boldsymbol{\omega} + \boldsymbol{\beta} + \boldsymbol{\psi}_1 \\
\mathbf{q}_y &= \mathbf{q} + \boldsymbol{\psi}_2
\end{aligned}
$$

Where:
- $$\mathbf{J}$$: Inertial matrix of the CubeSat
- $$\boldsymbol{\omega}$$: CubeSat angular rate with respect to the inertial frame  
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
  
- $$\boldsymbol{\phi}_1$$, $$\boldsymbol{\phi}_2$$, $$\boldsymbol{\phi}_3$$: Process zero-mean Gaussian noise
- $$\boldsymbol{\beta}$$: Drift in gyro angular measurement
- $$\boldsymbol{\omega}_y$$: Gyro angular rate measurement
- $$\mathbf{q}_y$$: Reduced quaternion measurement
- $$\boldsymbol{\phi}_1$$, $$\boldsymbol{\phi}_2$$: Measurement zero-mean Gaussian noise

We can thus easily convert this into a state space model for a more streamlined analysis later:

$$
\begin{aligned}
\dot{\mathbf{x}} &= \mathbf{f}(\mathbf{x}, \mathbf{u}, \boldsymbol{\phi}) \\
\mathbf{z} &= \mathbf{H} \mathbf{x} + \boldsymbol{\psi}
\end{aligned}
$$

Where 

$$
\mathbf{x} = \begin{bmatrix} 
\boldsymbol{\omega} \\ 
\mathbf{q} \\ 
\boldsymbol{\beta} 
\end{bmatrix}, \:
\mathbf{z} = \begin{bmatrix} 
\boldsymbol{\omega}_y \\ 
\mathbf{q}_y 
\end{bmatrix}, \:
\boldsymbol{\phi} = \begin{bmatrix} 
\boldsymbol{\phi}_1 \\ 
\boldsymbol{\phi}_2 \\ 
\boldsymbol{\phi}_3 
\end{bmatrix}, \:
\boldsymbol{\psi} = \begin{bmatrix} 
\boldsymbol{\psi}_1 \\ 
\boldsymbol{\psi}_2 
\end{bmatrix}, \:
\mathbf{H} =
\begin{bmatrix}
\mathbf{I}_3 & \mathbf{0}_3 & \mathbf{I}_3 \\
\mathbf{0}_3 & \mathbf{I}_3 & \mathbf{0}_3
\end{bmatrix}.
$$

Here $$\mathbf{x} \in \mathbb{R}^9$$ is the **state** of the system that matters to us the most and $$\mathbf{z} \in \mathbb{R}^6$$ is the measurement. The measurement is also very important to us as we will need it to correct our state later.

#### Wahba's problem
We are not quite done with the model dynamics as we are still missing one critical piece. Let's examine the state $$\mathbf{x} \in \mathbb{R}^9$$: we can easily obtain $$\boldsymbol{\omega}$$ and $$\boldsymbol{\beta}$$ from our gyro reading and the manufacturer datasheet, but how do we get the reduced quaternion $$\mathbf{q}$$? Remeber so far we only used our gyro and have left the accelerometer and magnetometer untouched, so we better make use of them and not leave them to collect dust.

This issue has been generalized into a well-known mathematical problem called the **[Wahba's problem](https://en.wikipedia.org/wiki/Wahba%27s_problem)**. In short, this problem attempts to find a rotation matrix (or in our case, a reduced quaternion) between two coordinate systems from a set of weighted observation vectors. It does so by finding an orthogonal matrix $$\mathbf{R}$$ that minimizes the loss function:

$$
L(\mathbf{A}) = \frac{1}{2} \sum_{i=1}^n a_i \left\| \mathbf{\hat{w}}_i - \mathbf{A} \mathbf{\hat{v}}_i \right\|^2
$$

where $$a_i$$ are a set of non-negative weights such that $$\sum_{i=1}^n a_i = 1$$, $$\mathbf{\hat{v}}_i$$ are nonparallel **reference vectors**, and $$\mathbf{\hat{w}}_i$$ are the corresponding **observation vectors**, which we obtain through the accelerometer and magnetometer.

#### Extended Kalman Filter
