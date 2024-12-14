---
layout: page
title: Deep Learning-based Dynamical System Solver
description: UBC CPEN 355 Machine Learning self-directed course project
img: assets/img/pde.jpg
importance: 3
category: personal
---

#### Overview
This project explores the use of **[State Space Models](https://huggingface.co/blog/lbourdois/get-on-the-ssm-train)** (SSMs), specifically the **[Mamba](https://arxiv.org/pdf/2312.00752)** architecture, as advanced solvers for dynamical systems governed by partial differential equations (PDEs). Unlike traditional methods, SSMs offer **theoretically infinite context windows** and **linear scalability**. When combined with spatial feature learning techniques such as Convolutional Neural Networks (CNNs) or image patching, SSMs excel at capturing complex **spatio-temporal** dependencies in dynamical systems, making them a powerful alternative to traditional methods.

_If you are interested in learning more about this project, you can check out our full report by clicking [here](https://drive.google.com/file/d/1Y4HOeHQ3rJqTKTam8SC0FJ8IltrXjNWM/view?usp=sharing)._

#### Background
Dynamical systems are typically modeled by PDEs who parameterize the state of the system, $$u$$, as a function of the state itself and its spatial derivative. Using a simple first order system with two spatial dimensions as an exmaple, we can describe such PDEs in the form of:

$$
\frac{\partial u}{\partial t} = f\left(u, t, \frac{\partial u}{\partial x}, \frac{\partial u}{\partial y}\right)
$$

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/dynamical.png" title="Sample dynamical system" class="img-fluid rounded" %}
    </div>
</div>
<div class="caption">
    Sample dynamical system and its temporal derivative evolving with time
</div>

Traditionally, PDEs are solved numerically using iterative methods that approximate $$\frac{\partial u}{\partial t}$$ over time. A common example is **Euler’s method**:

$$
u_{t+1} = u_t + f(u_t, t)\Delta t
$$

While this approach can achieve accurate results by using fine time steps or combining multiple evaluations of $$f(u_t, t)$$ (e.g., [Runge-Kutta methods](https://en.wikipedia.org/wiki/Runge–Kutta_methods)), it comes with significant computational and resource demands. Neural networks are thus potential candidates to mitigate lots of the disadvantages.

Our idea was originally inspired by the current SOTA method introduced in [NeuralPDE](https://arxiv.org/pdf/2111.07671), which essentially uses [Method of Lines](https://en.wikipedia.org/wiki/Method_of_lines) (MOL) to convert a PDE into multiple ODEs, then solves each one of them using [NeuralODE](https://arxiv.org/pdf/1806.07366)—a **differentiable** ODE solver. We were amazed that they are able to parameterize $$\frac{\partial u}{\partial t}$$ using nothing but CNN, essentially learning temporal features using a tried-and-true spatial feature learner:

$$
\frac{\partial u}{\partial t} = f(u, t) \approx \text{CNN}_{\phi}(u)
$$

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/neuralpde.png" title="NeuralPDE architecture" class="img-fluid rounded" %}
    </div>
</div>
<div class="caption">
    NeuralPDE architecture, notice how they were able to parameterize temporal derivative simply by using CNN
</div>

This is where we wanted to introduce our method's novelty: instead of purely relying on spatial features, we must also take advantage of the temporal features in dynamical systems. This is especially true considering the majority of dynamical systems are **[chaotic](https://en.wikipedia.org/wiki/Chaos_theory)**, which means unlike speech or LLM sequence modeling, a small perturbation to the initial state of the system can result in a vastly different state in a short amount of time.

#### Proposed models
##### MambaCNNMOL

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/cnn_mamba.jpg" title="MambaCNNMOL architecture" class="img-fluid rounded" %}
    </div>
</div>
<div class="caption">
    Model architecture
</div>

**MambaCNNMOL** is an architecture designed to combine the strengths of CNNs and SSMs to learn both spatial and temporal dependencies in dynamical systems. Its structure works as follows:

1. **Input CNN Module**:
   - The spatial features of the input data are processed using a convolutional neural network.
   - This module reduces the 2D spatial resolution of the data into a lower-dimensional grid while retaining the essential spatial information, which reduces computational overhead.

2. **Flattening and Temporal Feature Learning**:
   - The reduced spatial grid is flattened into a 1-dimensional vector and passed to the Mamba module.
   - The Mamba module then captures the temporal structure of the data, leveraging its ability to model long-range dependencies efficiently.

3. **Output CNN Module**:
   - The 1-dimensional output of the Mamba module is reshaped back into a 2D grid.
   - This reshaped data is fed into the output CNN module, which reconstructs and upsamples the spatial resolution to match the original input dimensions and produces the final prediction.

By separating spatial and temporal feature learning into distinct stages, MambaCNNMOL reduces computational costs while maintaining a robust ability to learn spatio-temporal relationships in dynamical systems.

##### MambaPatchMOL

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/patch_mamba.jpg" title="MambaPatchMOL architecture" class="img-fluid rounded" %}
    </div>
</div>
<div class="caption">
    Model architecture
</div>

**MambaPatchMOL** is a novel architecture that addresses the balance between spatial and temporal dependencies in dynamical systems by utilizing a patch-based approach. The key features are:

1. **Patch-Based Spatial Division**:
   - The spatial dimensions of the input grid are divided into smaller, non-overlapping patches. Each patch represents a localized spatial neighborhood, preserving local spatial relationships while keeping the feature dimensions manageable.

2. **Temporal Modeling with Mamba**:
   - Each patch is treated as its own temporal sequence over time and processed independently using the Mamba module.
   - Two configurations are available:
     - **Sequential Processing**: A single Mamba block processes the temporal evolution of each patch, reusing the same parameters across patches, ensuring efficiency through parameter sharing.
     - **Parallel Processing**: Each patch is assigned its own Mamba series, allowing the temporal evolution of each patch to be modeled independently. This approach better captures localized behavior but is computationally more expensive.

3. **Reassembly of Spatial Data**:
   - After processing, the temporal sequences from the patches are reassembled into the original spatial grid format.

##### Key Differences
- **MambaCNNMOL** uses parameterized CNN layers for spatial feature extraction, which can improve performance in capturing global spatial relationships, particularly in longer-range predictions.
- **MambaPatchMOL**, inspired by patch-based approaches like the Vision Transformer, focuses on efficiently preserving local spatial features and offers flexibility in handling complex spatio-temporal tasks.

Both architectures leverage the strengths of the Mamba module for temporal modeling, effectively overcoming the limitations of most NN-based PDE solvers that focus solely on learning spatial features while neglecting temporal dependencies, thereby demonstrating improved performance in capturing spatio-temporal features in dynamical systems.

#### Dataset
The project uses the [DynaBench](https://dynabench.github.io) Advection Equation dataset, which features sparsely distributed data in a $$15 \times 15$$ grid structure. The advection equation is a linear first-order equation:

$$
\frac{\partial u}{\partial t} = -\nabla \cdot (\mathbf{c}u)
$$

We define lookback $$L$$ as the number of previous inputs we use for prediciotn, and rollout $$R$$ as the number of ouputs to predict. Additionally, we defined the following two tasks:
- Sequence-to-One (**S2O**): Predicting the immediate next state provided with $$L$$ timesteps of the state.
- Sequence-to-Sequence (**S2S**): Predicting the $$R$$ next timesteps provided with $$L$$ timesteps of the state.

#### Evaluation
Mean squared error (MSE) was the primary metric, calculated across all spatial and temporal dimensions:

$$
\text{MSE} = \frac{1}{B \cdot T \cdot F \cdot H \cdot W} \sum_{b,t,f,h,w} \left( \hat{y}_{b,t,f,h,w} - y_{b,t,f,h,w} \right)^2
$$

Where:
- $$B$$: Batch size  
- $$T$$: Number of time steps  
- $$F$$: Number of features  
- $$H$$, $$W$$: Spatial dimensions  
- $$\hat{y}$$: Predicted value  
- $$y$$: Ground truth value

For S2O, a closed-loop setting was employed where the next $$R$$ states are predicted **autoregressively** by appending previous predictions to the past input. Specifically, if the context is given by $$X_{t-L+1}, ..., X_t$$ and the label is $$X_{t+1}, ..., X_{t+R}$$, we generate predictions:

$$
\begin{aligned}
\hat{X}_{t+1} &= m_\phi \left( X_t \| X_{t-1} \| ... \| X_{t-L+1} \right) \\
\hat{X}_{t+2} &= m_\phi \left( \hat{X}_{t+1} \| X_t \| ... \| X_{t-L+2} \right) \\
\hat{X}_{t+3} &= m_\phi \left( \hat{X}_{t+2} \| \hat{X}_{t+1} \| ... \| X_{t-L+3} \right) \\
\vdots & \\
\hat{X}_{t+R} &= m_\phi \left( \hat{X}_{t+R-1} \| \hat{X}_{t+R-2} \| ... \| \hat{X}_t \right)
\end{aligned}
$$

For S2S, this autoregressive setting is not applicable, so we show the mean MSE on the test epoch.


#### Results
Both MambaCNNMOL and MambaPatchMOL demonstrated strong performance in learning spatio-temporal structures in complex dynamical systems. MambaCNNMOL excelled in **long-range predictions**, while MambaPatchMOL showed promise in capturing **local spatial dynamics**.

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/s2o.jpg" title="S2O loss" class="img-fluid rounded" %}
    </div>
</div>
<div class="caption">
    S2O loss comparison between MambaCNNMOL and MambaPatchMOL at various rollouts
</div>

As shown above, MambaPatchMOL shows lower MSE at low rollouts. However, the advantage decreases as we start to predict more future timesteps, and was eventually outperformed by MambaCNNMOL at rollout equals to 5.

<div align="center">
  <table style="border: 1px solid black; border-collapse: collapse; text-align: center;">
    <tr>
      <th style="border: 1px solid black; padding: 8px;">Model</th>
      <th style="border: 1px solid black; padding: 8px;">Final Test Loss</th>
    </tr>
    <tr>
      <td style="border: 1px solid black; padding: 8px;">MambaCNNMOL</td>
      <td style="border: 1px solid black; padding: 8px;">0.0097</td>
    </tr>
    <tr>
      <td style="border: 1px solid black; padding: 8px;">MambaPatchMOL - Sequential</td>
      <td style="border: 1px solid black; padding: 8px;">0.091</td>
    </tr>
    <tr>
      <td style="border: 1px solid black; padding: 8px;">MambaPatchMOL - Parallel</td>
      <td style="border: 1px solid black; padding: 8px;">0.0093</td>
    </tr>
  </table>
</div>

<br>The above table shows the S2S MSE comparison between the two models. Here, MambaPatchMOL takes the throne, specifically when it was running in **parallel** mode introduced in the model section above. This shows that when a Mamba SSM is in charge of its own patch instead of all the patches, the parameters can be tuned for that specific patch without being polluted by patches that are further away, allowing greater flexibility and boosting the model accuracy.
