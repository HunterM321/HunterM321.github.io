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
[UBC Orbit Satellite Design Team](https://www.ubcorbit.com) is a student-run engineering design team supported by UBC Faculty of Applied Science and various industry partners. Currently, we are working on our most challenging and most ambitious project called **[ALEASAT](https://www.esa.int/Education/CubeSats_-_Fly_Your_Satellite/Meet_the_team_Aleasat)**. ALEASAT is a low Earth orbit (LEO), Earth-observation **[1U CubeSAT](https://www.nasa.gov/what-are-smallsats-and-cubesats/)** designed for radio amateurs to support disaster relief efforts. It features an on-board camera, enabling users to request and directly receive targeted imagery of specific locations.

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

I have been with the team since 2021 and I currently lead the **Attitude and Orbit Control System** (AOCS) subteam. The AOCS subteam is in charge of determining and stabilizing the orientation of the satellite. The positioning of the satellite is determined using a variety of sensors, such as **[sun sensors](https://en.wikipedia.org/wiki/Sun_sensor)** and an **[Inertial Measurement Unit](https://en.wikipedia.org/wiki/Inertial_measurement_unit)** (IMU). The gathered data is then fed into our custom control system which controls the satellite's actuators, including **[magnetorquers](https://en.wikipedia.org/wiki/Magnetorquer)** and **[reaction wheels](https://en.wikipedia.org/wiki/Reaction_wheel)**.

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

_Our control system design was inspired by [Spacecraft Modeling, Attitude Determination, and Control: Quaternion-Based Approach](https://ntrs.nasa.gov/citations/20240009554), written by Dr. Yang, a NASA research scientist. Additionally, we used [Attitude and Heading Reference Systems](https://ahrs.readthedocs.io/en/latest/) (AHRS) Python library as a secondary source of reference._

#### Spacecraft dynamics
To come up with a solution to a control problem, we always need to first start with the **dynamics** of the model. This is nothing but a glorified way of describing $$\mathbf{F}=m\mathbf{a}$$, Newton's second law of motion which we learned in high school or first-year university physics. This model characterizes how the state of our system changes with time given our system inputs and existing measurements. Our CubeSat has very complex dynamics as we utilize and fuse multiple sensors to obtain the observed state, yet each sensor has its distinct noise distribution.

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
- $$\mathbf{\Omega}$$: [Skew-symmetric](https://en.wikipedia.org/wiki/Skew-symmetric_matrix#:~:text=In%20mathematics%2C%20particularly%20in%20linear,whose%20transpose%20equals%20its%20negative.) matrix defined as

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

Here $$\mathbf{x} \in \mathbb{R}^9$$ is the **state** of the system and $$\mathbf{y} \in \mathbb{R}^6$$ is the measurement. The measurement's primary goal is to correct our state, which I will explain later. One caveat of this state space model is that it is still in the form of **continuous-time ordinary differential equations**, but our computer would much appreciate if we can convert this into **discrete-time algebric equations**. This shouldn't be too hard since we already knew the sampling period $$\Delta t$$ and parameterized the state derivative with $$\mathbf{f}(\mathbf{x}, \mathbf{u}, \boldsymbol{\phi})$$, a **first-order approximation**, synonymous to **[Euler's method](https://en.wikipedia.org/wiki/Euler_method)**, would suffice:

$$
\begin{aligned}
\mathbf{x}_{k+1} & = \mathbf{F}(\mathbf{x}_k, \mathbf{u}_k) + \mathbf{G}(\mathbf{x}_k, \boldsymbol{\phi}_k) \\
\mathbf{y}_k & = \mathbf{H} \mathbf{x}_k + \boldsymbol{\psi}_k
\end{aligned}
$$

Here I won't go in-depth into the derivation as section 8.4 of the [book](https://ntrs.nasa.gov/citations/20240009554) shows the interesting math behind. In short, the **nonlinear** state transition function $$\mathbf{F}(\mathbf{x}_k, \mathbf{u}_k)$$ advances the present state $$\mathbf{x}_{k}$$ through $$\Delta t$$, and the **nonlinear** noise transition function $$\mathbf{G}(\mathbf{x}_k, \boldsymbol{\phi}_k)$$ makes sure the process noises are also propagated to together produce the next state $$\mathbf{x}_{k+1}$$. $$\mathbf{H}$$ is called the **observation model matrix** which simply links the present state and measurement.

#### Wahba's problem
It may seem like we have everything in place to talk about the control system, but we are still missing one critical piece. Let's examine the initialization state $$\mathbf{x}_0 \in \mathbb{R}^9$$: we can easily obtain $$\boldsymbol{\omega}_0$$ from our gyro reading and $$\boldsymbol{\beta}_0$$ by setting it to $$\mathbf{0}_3$$, but how do we initialize the reduced quaternion $$\mathbf{q}_0$$? Remember so far we only used our gyro and have left the **accelerometer** and **magnetometer** untouched, so we better make use of them and not leave them to collect dust.

This issue has been generalized to a well-known mathematical problem called the **[Wahba's problem](https://en.wikipedia.org/wiki/Wahba%27s_problem)**. Specifically, this problem attempts to find a rotation matrix (or in our case, a reduced quaternion) between two coordinate systems from a set of weighted observation vectors. It does so by finding an [orthogonal matrix](https://en.wikipedia.org/wiki/Orthogonal_matrix) $$\mathbf{R}$$ that minimizes the loss function:

$$
L(\mathbf{R}) = \frac{1}{2} \sum_{i=1}^n a_i \left\| \mathbf{\hat{w}}_i - \mathbf{R} \mathbf{\hat{v}}_i \right\|^2
$$

where $$a_i$$ are a set of non-negative weights such that $$\sum_{i=1}^n a_i = 1$$, $$\mathbf{\hat{v}}_i$$ are nonparallel **reference vectors**, and $$\mathbf{\hat{w}}_i$$ are the corresponding **observation vectors**, which we obtain through the accelerometer and magnetometer.

There exists many algorithms that approximate the solution, one of which is called **[Quaternion Estimator](https://ahrs.readthedocs.io/en/latest/filters/quest.html)** (QUEST), which is our algorithm of choice. It is widely used in the aerospace and robotics industry thanks to its highly efficient computation. QUEST's solution is daunting yet elegant, it uses methods like [Cayley-Hamilton theorem](https://en.wikipedia.org/wiki/Cayley–Hamilton_theorem) and [Newton-Raphson's root finding](https://en.wikipedia.org/wiki/Newton%27s_method) to simplify and solve the characteristic equation. I highly recommend checking out the link above to learn more. QUEST in the end will tell us an optimal solution to the reduced quaternion $$\mathbf{q}_0$$, which we can plug into our state vector $$\mathbf{x}_0$$ to complete the initialization. Furthermore, this optimal quaternion is also our measurement quaternion $$\mathbf{q}_{y_k} \: \forall k \in \mathbb{N}$$. Together with the gyro measurement $$\mathbf{w}_{y_k}$$, this forms our measurement vector $$\mathbf{y}_k$$.

#### Extended Kalman Filter
It wouldn't be a satellite's control system without the presence of **[Kalman Filter](https://en.wikipedia.org/wiki/Kalman_filter)** (KF). I have hyped things up so far just so I can introduce the bread and butter of AOCS. Originally intorduced in the 1960s, Kalman Filter's debut was to help the Americans be the first and only nation to set their feet on the lunar soil. More than 60 years later, KF has undergone countless iterations, yet it still stands as one of the most important inventions in control theory. The **[Extended Kalman Filter](https://en.wikipedia.org/wiki/Extended_Kalman_filter)** (EKF) is argubly one of the most important flavors of KF, which acknowledges the universe is in fact **nonlinear** and thus adds a linearization layer on top of KF. It was also technically the algorithm that was on-board Apollo's guidance and navigation computer.

_Note that even though KF bears the name "filter", it is still **NOT** to be confused with the traditional filter in signal processing (e.g., low-pass filter), despite it has the ability to smooth out noisy signal. They really don't have too much in common._

There are some important primises that help establish EKF. First and foremost, EKF is inherently a **[Markov process](https://en.wikipedia.org/wiki/Markov_chain)**, meaning _the past is independent of the future given the present_. In mathematical formulation, we can write it as:

$$
P(X_{n+1} = \mathbf{x}_{n+1} \mid X_n = \mathbf{x}_n, \dots, X_1 = \mathbf{x}_1) = P(X_{n+1} = \mathbf{x}_{n+1} \mid X_n = \mathbf{x}_n), \: \forall n \in \mathbb{N}
$$

The second key premise of EKF is that since it's a **probabilistic** approach, it assumes that all noises involved in the process are **zero-mean Gaussian distributions**. Whether this is an accurate assumption is up for debate, but modeling noises using Gaussian distributions definitely has its own benefits and this concept is widely adpated in many other fields.

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/kf.jpg" title="Kalman Filter" class="img-fluid rounded" %}
    </div>
</div>
<div class="caption">
    The essence of Kalman Filter: predict, correct, and repeat
</div>

In a nutshell, EKF operates in a **recursive** manner where in each iteration, it gives a **prediction** by passing the previous EKF ouput through the system dynamics model, then it attempts to perform **correction** using the current measurement, all before passing the new output back to the input for the next loop. Specifically, we would compute:

**Prediction**

$$
\begin{aligned}
\hat{\mathbf{x}}_{k} & = \mathbf{F}(\mathbf{x}_{k-1}, \mathbf{u}_{k-1}) + \mathbf{G}(\mathbf{x}_{k-1}, \boldsymbol{\phi}_{k-1}) \\
\hat{\mathbf{P}}_{k} & = \mathbf{F}_{k-1} \mathbf{P}_{k-1} \mathbf{F}_{k-1}^\top + \mathbf{L}_{k-1} \mathbf{Q}_{k-1} \mathbf{L}_{k-1}^\top
\end{aligned}
$$

**Correction**

$$
\begin{aligned}
\mathbf{v}_k & = \mathbf{y}_k - \mathbf{H} \hat{\mathbf{x}}_{k} \\
\mathbf{S}_k & = \mathbf{H} \hat{\mathbf{P}}_{k} \mathbf{H}^\top + \mathbf{R}_k \\
\mathbf{K}_k & = \hat{\mathbf{P}}_{k} \mathbf{H}^\top \mathbf{S}_k^{-1} \\
\mathbf{x}_{k} & = \hat{\mathbf{x}}_{k} + \mathbf{K}_k \mathbf{v}_k \\
\mathbf{P}_{k} & = (\mathbf{I} - \mathbf{K}_k \mathbf{H}) \hat{\mathbf{P}}_{k}
\end{aligned}
$$

The derivation of these equations are rather daunting and time-consuming (yet extremely interesting), so I won't go too in-depth here and instead will leave it to your discretion to decide. Aagin, section 8.4 of the [book](https://ntrs.nasa.gov/citations/20240009554) goes over the calculation details. I also find [AHRS's implementation](https://ahrs.readthedocs.io/en/latest/filters/ekf.html) to be pretty interesting and relatively easy to digest, though they chose to use a different state vector $$\mathbf{x}$$ which changes their state space model, it is still a good read. I will however go over some of the more important steps and concepts in the calculations below to build some intuitions.

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/kf_demo_1.jpg" title="Kalman Filter demo" class="img-fluid rounded" %}
    </div>
</div>
<div class="caption">
    Kalman Filter ouputs another Gaussian distribution by combining the prediction and measurement Gaussian distributions (note that the notation used here is a bit different than ours)
</div>

Going back to the EKF equations above, there are two very important **[covariance matrices](https://en.wikipedia.org/wiki/Covariance_matrix)** that dictate the confidence of EKF's final output value. They are the **process covariance matrix** $$\mathbf{Q}$$ and the **measurement covariance matrix** $$\mathbf{R}$$. _The process covariance matrix captures the uncertainty arising from system dynamics, while the measurement covariance matrix captures the uncertainty arising from the measuring instruments_. They are **diagonal matrices** since we made the assumption that the **random variables** in each matrix are independent of each other, resulting in a covariance of 0 in the non-diagonals. For example, it is fair to assume that the gyroscope measurement in each axis is **independent** and bears no impact on the other axes. We additionally made the assumption that the process noises are also independent of the measurement noises. In reality, these assumptions may very well not be the case, but it is the best we could do. The matrices are derived from the process and measurement noises introduced in the spacecraft dynamics section. We can formulate $$\mathbf{Q}$$ and $$\mathbf{R}$$ in the following way:

$$
\mathbb{E}[\boldsymbol{\phi}_k] = 0, \: \mathbb{E}[\boldsymbol{\psi}_k] = 0, \: \forall k
$$

$$
\mathbb{E}[\boldsymbol{\phi}_k \boldsymbol{\phi}_k^\top] = \mathbf{Q}_k, \: 
\mathbb{E}[\boldsymbol{\psi}_k \boldsymbol{\psi}_k^\top] = \mathbf{R}_k, \: 
\mathbb{E}[\boldsymbol{\psi}_i \boldsymbol{\phi}_j^\top] = 0, \: \forall i, j, k
$$

$$
\mathbb{E}[\boldsymbol{\phi}_i \boldsymbol{\phi}_j^\top] = 0, \: 
\mathbb{E}[\boldsymbol{\psi}_i \boldsymbol{\psi}_j^\top] = 0, \: \forall i \neq j
$$

Note that $$\mathbf{Q}$$ and $$\mathbf{R}$$ are not to be confused with the **state covariance matrix** $$\mathbf{P}$$. One noticeable difference is that there is no restriction on whether $$\mathbf{P}$$ should be a diagonal matrix. In fact, it is highly unlikely that $$\mathbf{P}$$ is a diagonal matrix since it is constantly getting updated in each iteration. The state covariance matrix undergoes the same prediction and correction procedure as its buddy, the state vector $$\mathbf{x}$$. The predicted $$\hat{\mathbf{P}}_{k}$$ is directly influenced by $$\mathbf{Q}_{k-1}$$, and the value is corrected indirectly by $$\mathbf{R}_{k}$$ to yield the final state covariance $$\mathbf{P}_{k}$$.

So far our discussion only involves things that are common between KF and EKF, and we have not gone into what makes EKF special—handling nonlinear system with proper linearization. In fact, our system dynamics model is indeed a nonlinear system, as reflected by $$\mathbf{F}$$ and $$\mathbf{G}$$. To obtain the predicted state covariance matrix $$\hat{\mathbf{P}}_k$$, we need to first linearize both $$\mathbf{F}$$ and $$\mathbf{G}$$. This is done through **[Taylor expansion](https://en.wikipedia.org/wiki/Taylor_series)** about the point $$(\mathbf{x}_{k-1}, \mathbf{u}_{k-1})$$, equivalent to finding the **[Jacobians](https://en.wikipedia.org/wiki/Jacobian_matrix_and_determinant)** of $$\mathbf{F}$$ and $$\mathbf{G}$$. This first-order approximation is computationally efficient and precise enough for our application, especially considering we don't have the luxury to run heavy tasks on our MCU. The linearized $$\mathbf{F}_{k-1}$$ and $$\mathbf{L}_{k-1}$$ can be computed by:

$$
\begin{aligned}
\mathbf{F}_{k-1} &= \left. \frac{\partial \mathbf{F}}{\partial \mathbf{x}} \right|_{\mathbf{x}_{k-1}, \mathbf{u}_{k-1}} \\
\mathbf{L}_{k-1} &= \left. \frac{\partial \mathbf{G}}{\partial \boldsymbol{\phi}_k} \right|_{\mathbf{x}_{k-1}, \mathbf{u}_{k-1}}
\end{aligned}
$$

Lastly, I will introduce the two quantities that help us correct our prediction and output the final Gaussian distribution composed of $$\mathbf{x}_k$$ and $$\mathbf{P}_k$$, which are the **innovation** $$\mathbf{v}_k$$ and the **Kalman gain** $$\mathbf{K}_k$$. The innovation is sometimes referred to as the **measurement residual**. In short, it measures the gap between the predicted state and measurement. The bigger the gap, the more _off_ our measurement is compared to our system dynamics model, and the more correction we need to apply to our predicted state. The Kalman gain measures how much we **DO NOT** trust our system dynamics model, or conversely how much we **DO** trust our measurement. To simplify the math, let's consider an 1D model with scalar equations. Imagine if $$K_k=0$$, that means our predicted process variance $$\hat{P}_k$$ must be 0; in other words, the **probability distribution function** (PDF) of the prediction is a **[Dirac delta function](https://en.wikipedia.org/wiki/Dirac_delta_function)** $$\delta(\hat{x}_k)$$. Our EKF would thus fully trust our system dynamics model:

$$
x_k = \hat{x}_k
$$

At the other end of the spectrum, if $$K_k=1$$, that means our measurement variance $${R}_k$$ must be 0. The PDF of the measurement would then be a Dirac delta function $$\delta(y_k)$$. Our EKF would then fully trust our measurement instead:

$$
x_k = y_k
$$

In reality, it is pretty much guaranteed that $$\mathbf{K}_k$$ would be somewhere in between 0 and 1, and the output state $$\mathbf{x}_k$$ would be somewhere in between the prediction and the measurement.
