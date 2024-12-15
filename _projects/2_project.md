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
[UBC Orbit Satellite Design Team](https://www.ubcorbit.com) is a student-run engineering design team, supported by UBC Faculty of Applied Science. Currently, we are working on our most challenging and most ambitious project called **[ALEASAT](https://www.esa.int/Education/CubeSats_-_Fly_Your_Satellite/Meet_the_team_Aleasat)**. ALEASAT is a low Earth orbit (LEO), Earth-observation **[1U CubeSAT](https://www.nasa.gov/what-are-smallsats-and-cubesats/)** designed for radio amateurs to support disaster relief efforts. It features an on-board camera, enabling users to request and directly receive targeted imagery of specific locations.

We are one of the four teams selected by **European Space Agency** (ESA) to participate in their [Fly Your Satellite](https://www.esa.int/Education/CubeSats_-_Fly_Your_Satellite) program, making us the first and only Canadian team in this program. ESA provides training and testing facilities. ALEASAT, **set to launch with ESA in 2027**, will glide through orbit like a celestial angel, gracefully watching over the Earth 🌎.

<div class="row justify-content-sm-center">
    <div class="col-sm-8 mt-3 mt-md-0">
        {% include figure.liquid path="assets/img/aleasat_1.png" title="aleasat" class="img-fluid rounded" %}
    </div>
    <div class="col-sm-4 mt-3 mt-md-0">
        {% include figure.liquid path="assets/img/aleasat_structure.png" title="aleasat structure" class="img-fluid rounded" %}
    </div>
</div>
<div class="caption">
    ALEASAT (left) and its CAD structural breakdown (right)
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
    ALEASAT assembly (left) and integration inside the TestPod (right) before undergoing vibration testing in ESA, Belgium
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

We are using unit quaternions (i.e., $$\|\mathbf{q}\| = 1$$) to represent attitude, hence we can reduce the quaternion representation further to only use the complex vector part, called the **reduced quaternion**. From now on, I will be using $$\mathbf{q}$$ to exclusively indicate reduced quaternion. _Moreover, in this section we will only use the **gyroscope** inside our IMU as our attitude determination instrument_. Now equipped with the knowledge of quaternions, we can proceed to write down the dynamics of our CubeSat:

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
- $$\boldsymbol{\psi}_1$$, $$\boldsymbol{\psi}_2$$: Measurement zero-mean Gaussian noise

We can thus easily convert this into a state space model for a more streamlined analysis later:

$$
\begin{aligned}
\dot{\mathbf{x}} &= \mathbf{f}(\mathbf{x}, \mathbf{u}, \boldsymbol{\phi}) \\
\mathbf{y} &= \mathbf{H} \mathbf{x} + \boldsymbol{\psi}
\end{aligned}
$$

Where 

$$
\mathbf{x} = \begin{bmatrix} 
\boldsymbol{\omega} \\ 
\mathbf{q} \\ 
\boldsymbol{\beta} 
\end{bmatrix}, \:
\mathbf{y} = \begin{bmatrix} 
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

Here $$\mathbf{x} \in \mathbb{R}^9$$ is the **state** of the system that matters to us the most and $$\mathbf{y} \in \mathbb{R}^6$$ is the measurement. The measurement is also very important to us as we will need it to correct our state later.

#### Wahba's problem
We are not quite done with the model dynamics as we are still missing one critical piece. Let's examine the state $$\mathbf{x} \in \mathbb{R}^9$$: we can easily obtain $$\boldsymbol{\omega}$$ and $$\boldsymbol{\beta}$$ from our gyro reading and the manufacturer datasheet respectively, but how do we get the reduced quaternion $$\mathbf{q}$$? Remeber so far we only used our gyro and have left the **accelerometer** and **magnetometer** untouched, so we better make use of them and not leave them to collect dust.

This issue has been generalized to a well-known mathematical problem called the **[Wahba's problem](https://en.wikipedia.org/wiki/Wahba%27s_problem)**. Specifically, this problem attempts to find a rotation matrix (or in our case, a reduced quaternion) between two coordinate systems from a set of weighted observation vectors. It does so by finding an orthogonal matrix $$\mathbf{R}$$ that minimizes the loss function:

$$
L(\mathbf{R}) = \frac{1}{2} \sum_{i=1}^n a_i \left\| \mathbf{\hat{w}}_i - \mathbf{R} \mathbf{\hat{v}}_i \right\|^2
$$

where $$a_i$$ are a set of non-negative weights such that $$\sum_{i=1}^n a_i = 1$$, $$\mathbf{\hat{v}}_i$$ are nonparallel **reference vectors**, and $$\mathbf{\hat{w}}_i$$ are the corresponding **observation vectors**, which we obtain through the accelerometer and magnetometer.

There exists many algorithms that approximate the solution, one of which is called **[Quaternion Estimator](https://ahrs.readthedocs.io/en/latest/filters/quest.html)** (QUEST), which is our algorithm of choice. It is widely used in the aerospace and robotics industry thanks to its highly efficient computation. QUEST's solution is daunting yet elegant, it uses methods like [Cayley-Hamilton theorem](https://en.wikipedia.org/wiki/Cayley–Hamilton_theorem) and [Newton-Raphson's root finding](https://en.wikipedia.org/wiki/Newton%27s_method) to simplify and solve the characteristic equation. I highly recommend checking out the link above to learn more. QUEST in the end will tell us an optimal solution to the reduced quaternion $$\mathbf{q}$$, which we can plug into our state vector $$\mathbf{x}$$ and feed into our control system.

#### Extended Kalman Filter
It wouldn't be a satellite's control system without the presence of **[Kalman Filter](https://en.wikipedia.org/wiki/Kalman_filter)** (KF). I have hyped things up so far just so I can introduce the bread and butter of AOCS. Originally intorduced in the 1960s, Kalman Filter's debut was to help the Americans be the first and only nation to set their foot on the lunar soil. More than 60 years later, KF has undergone countless iterations, yet it still stands as one of the most important inventions in control theory. The **[Extended Kalman Filter](https://en.wikipedia.org/wiki/Extended_Kalman_filter)** (EKF) is argubly one of the most important flavors of KF, which acknowledges the universe is in fact nonlinear and thus adds a linearization layer on top of KF. It was also technically the algorithm that was on-board Apollo's guidance and navigation computer.

_Note that even though KF bears the name "filter", this is still **NOT** to be confused with the traditional filter in signal processing (e.g., low-pass filter), despite it has the ability to smooth out noisy signal. They really don't have too much in common._

There are some important primises that help establish EKF. First and foremost, EKF is inherently a **[Markov process](https://en.wikipedia.org/wiki/Markov_chain)**, meaning the past is independent of the future given the present. In mathematical formulation, we can write it as:

$$
P(\mathbf{X}_{n+1} = \mathbf{x}_{n+1} \mid \mathbf{X}_n = \mathbf{x}_n, \dots, \mathbf{X}_1 = \mathbf{x}_1) = P(\mathbf{X}_{n+1} = \mathbf{x}_{n+1} \mid \mathbf{X}_n = \mathbf{x}_n), \quad \forall n \in \mathbb{N}
$$

The second key premise of EKF is that since it's a **probabilistic** approach, it assumes that all noises involved in the process are **zero-mean Gaussian distributions**. Whether this is an accurate assumption is up for debate, but modeling noise using Gaussian distribution definitely has its own benefits and this concept is widely adpated in many other fields.

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/kf.jpg" title="Kalman Filter" class="img-fluid rounded" %}
    </div>
</div>
<div class="caption">
    The essence of Kalman Filter: predict, correct, and repeat
</div>

In a nutshell, EKF operates in a **recursive** manner where in each iteration, it gives a **prediction** by passing the previous EKF ouput through the system dynamics model and attempts to perform **correcting** using the current measurement, all before passing the new output back to the input for the next loop. Specifically, we would compute:

**Prediction**

$$
\begin{aligned}
\hat{\mathbf{x}}_{k} & = \mathbf{F}(\hat{\mathbf{x}}_{k-1}, \mathbf{u}_{k-1}) \\
\hat{\mathbf{P}_{k}} & = \mathbf{F}_{k-1} \mathbf{P}_{k-1} \mathbf{F}_{k-1}^\top + \mathbf{L}_{k-1} \mathbf{Q}_{k-1} \mathbf{L}_{k-1}^\top
\end{aligned}
$$

**Correction**

$$
\begin{aligned}
\mathbf{v}_k & = \mathbf{y}_k - \mathbf{H} \hat{\mathbf{x}}_{k} \\
\mathbf{S}_k & = \mathbf{H} \hat{\mathbf{P}_{k}} \mathbf{H}^\top + \mathbf{R}_k \\
\mathbf{K}_k & = \hat{\mathbf{P}_{k}} \mathbf{H}^\top \mathbf{S}_k^{-1} \\
\mathbf{x}_{k} & = \hat{\mathbf{x}}_{k} + \mathbf{K}_k \mathbf{v}_k \\
\mathbf{P}_{k} & = (\mathbf{I} - \mathbf{K}_k \mathbf{H}) \hat{\mathbf{P}_{k}}
\end{aligned}
$$

The derivation of these equations are rather daunting and time-consuming (yet extremely interesting), so I won't go too in-depth here and instead will leave it to your discretion to decide. Section 8.4 of the [book](https://ntrs.nasa.gov/citations/20240009554) goes over the calculation details. I also find [AHRS's implementation](https://ahrs.readthedocs.io/en/latest/filters/ekf.html) to be pretty interesting, though they chose to use a different state vector $$\mathbf{x}$$ which changes their state space model, it is still a good read. I will however go over some of the more important steps in the calculation below.
