---
layout: page
title: Echocardiogram Segmentation
description: Robotics and Control Lab, UBC
img: assets/img/echo.jpg
importance: 1
category: work
related_publications: false
---

#### Overview
Throughout the 4 months I stayed in UBC's [Robotics and Control Lab](https://rcl.ece.ubc.ca) (RCL) as a research intern, I worked on image and video segmentation of [echocardiogram](https://www.mayoclinic.org/tests-procedures/echocardiogram/about/pac-20393856). Specifically, I focused on chamber segmentation. The ultimate goal of this project is to teach a single agent to **simultaneously** segment all 4 chambers, which could be done by incorporating semantic information, instead of training individual agent for each chamber segmentation. I mainly worked on two sub-projects. The first one is baseline testing for [MedSAM](https://www.nature.com/articles/s41467-024-44824-z) and [SAM 2](https://ai.meta.com/sam2/) and fine-tuning [Florence 2](https://huggingface.co/microsoft/Florence-2-large). The second one is visual object tracking and segmentation using [SAMURAI](https://yangchris11.github.io/samurai/), which is also a derivative of SAM 2, as well as fine-tuning SAMURAI for echocardiogram video segmentation. All fine-tunings were done using [Low-Rank Adaptation](https://arxiv.org/pdf/2106.09685) (LoRA). I would like to focus more on discussing the SAMURAI project as I think it is the more interesting one.

#### What is SAMURAI
SAMURAI stands for _SAM-based Unified and Robust zero-shot visual tracker with motion-Aware Instance-level memory_. Specifically, SAMURAI is a derivative of the **video object segmentation** (VOS) functionality introduced in SAM 2, seeking to improve the **visual object tracking** (VOT) performance without a huge architectural modification. Maybe before we introduce SAMURAI's novelty in this domain of knowledge, a quick background of SAM 1 and SAM 2's architectures would come in handy.

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/sam2_architecture.jpg" title="SAM 2 architecture" class="img-fluid rounded" %}
    </div>
</div>
<div class="caption">
    SAM 2 architecture, all modules related to memory are new compared to the original SAM 1
</div>

Back when Meta introduced SAM 1, it quickey gained popularity in the image segmentation domain due to its robust performance. SAM 1 receives an image which goes through an **image encoder**, and a prompt which goes through a **prompt encoder**. Additionally, the image and prompt **embeddings** both go through a **mask decoder** which performs **cross-attention** between the prompt and image, as well as **self-attention** of the prompts, to finally output the predicted masks. However, SAM 1 lacks the ability to perform VOS since it does not have the ability to encode any memory information. This is where SAM 2 came in to fill in this gap. Specifically, SAM 2 has the same image encoder, prompt encoder, and mask decoder found in SAM 1, it also adds the ability to use previous mask outputs as reference to aid the current prediction. It does so by the following 3 additions:
- **Memory attention**: takes the current frame embedding as input and performs cross-attention between the current frame embedding and previous embeddings from past frames using several **transformer** blocks.
- **Memory encoder**: encodes the predicted masks of the current frame as embeddings using CNN and saves it as a memory.
- **Memory bank**: retains fixed-size memories of the past predictions in a First-In-First-Out format which will be used to aid the current mask prediction.

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/samurai_architecture.png" title="SAMURAI architecture" class="img-fluid rounded" %}
    </div>
</div>
<div class="caption">
    SAMURAI architecture proposes a motion modeling module and a motion-aware memory selection
</div>

So how does SAMURAI improve upon the model proposed in SAM 2? In short, since SAM 2 already has very robust segmentation performance, SAMURAI focuses on improving the **tracking** quality of the segmented object. It does so by introducing two novelties—**neither** of which requires any additional learnable parameters. The first contribution is the addition of a **motion modeling module**. This module trakcs and predicts the bounding box surrounding the segmented object; specifically, it tracks and predicts the bounding box's top left coordinate, height and width, as well as their corresponding rate of change (i.e., speed). Using this motion cue, the motion modeling module is able to handle difficult scenarios, such as self-occlusions and fast-moving objects. The second contribution is the addition of **motion-aware memory selection** for memory attention. The new proposed scheme selects high-quality memories from the memory bank. instead of selecting the latest memories using a fix window design. The higher quality memories will directly have an influence during cross-attention with the current frame.

<div class="row">
    <div class="col-sm-4 mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/0X1E67F4F64EDDA6ED_out.gif" title="Sample output 1" class="img-fluid rounded z-depth-1" %}
    </div>
    <div class="col-sm-4 mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/0X59E3868FDA8BC941_out.gif" title="Sample output 2" class="img-fluid rounded z-depth-1" %}
    </div>
    <div class="col-sm-4 mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/0XFA2F04D15D5D222_out.gif" title="Sample output 3" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Frame extractions of video segmention of LV chamber using SAMURAI on EchoNet-Dynamic dataset
</div>

##### Motion modeling
The addition of the motion modeling module gives SAM 2 another reference to look at when selecting the best mask from all its predictions. The original SAM 2 only generates an affinity score $$s_{\text{aff}}$$ and an object score $$s_{\text{obj}}$$ for each predicted mask, the best mask is selected by choosing the highest $$s_{\text{aff}}$$ such that the corresponding $$s_{\text{obj}}$$ is positive:

$$
\mathbb{M} = \{M_0, M_1, \dots, M_{N-1}\}
$$

$$
M_i = \underset{M_i \in \mathbb{M}}{\mathrm{argmax}} \ s_{\text{aff}}(M_i) \quad \text{where} \quad s_{\text{obj}}(M_i) > 0
$$

SAMURAI generates another score called the **motion** score $$s_{\text{kf}}$$. This score is calculated based on the **IoU** score of the predicted bounding box and each mask generated from SAM 2. In the end, a weighted sum of $$s_{\text{aff}}$$ and $$s_{\text{kf}}$$ is used to determine the mask with the best score:

$$
M_i = \underset{M_i \in \mathbb{M}}{\mathrm{argmax}} \ \alpha_{\text{aff}} \cdot s_{\text{aff}}(M_i) + (1 - \alpha_{\text{aff}}) \cdot s_{\text{kf}}(M_i) \quad \text{where} \quad s_{\text{obj}}(M_i) > 0
$$

You might be interested in why the newly introduced motion tracking score $$s_{\text{kf}}$$ bears the subscript _kf_, that's because SAMURAI tracks and predicts the bounding box of the segmented object using **[Kalman Filter](https://en.wikipedia.org/wiki/Kalman_filter)** (KF). Kalman Filter has no learnable parameters and was originally introduced in control theory to estimate the **state** of a dynamical system. We can always write down the state of a dynamical system in either **continuous-time ordinary differential equation** (ODE) or **discrete-time algebraic equation** format:

$$
\dot{\mathbf{x}} = A \mathbf{x} \quad \text{for continuous time}
$$

$$
\mathbf{x}_{t+1} = F \mathbf{x}_{t} \quad \text{for discrete time}
$$

For obvious reasons our computers always prefer to work with discrete-time systems. I went in-depth into KF in my [CubeSat Control System](../2_project) page, so please check it out if you are interested in learning more about it. Here I will go through some of the essential pieces of KF that are unique to SAMURAI instead.

As we mentioned earlier, the motion modeling module outputs an estimated position and speed of the bounding box, which we call the **state** of the system. We can encode this state into a vector format:

$$
\mathbf{x} =
\begin{bmatrix}
x & y & w & h & \dot{x} & \dot{y} & \dot{w} & \dot{h}
\end{bmatrix}^\top
$$

KF needs to know the dynamics of the system, or in other words, how the system propagates with time. To achieve this, we need the **linear system transition matrix**, or $$F$$, in the discrete-time equation above. Note that we consider this to be a simple linear system, otherwise we would need to utilize the Extended Kalman Filter (EKF), which will be a lot more work. Assuming all velocities are constant and the period between two consecutive frames is $$\Delta t$$, and using the x-coordinate as an example:

$$
\begin{aligned}
x_{t+1} &= x_t + \dot{x}_t \Delta t \\
\dot{x}_{t+1} &= \dot{x}_t
\end{aligned}
$$

The rest coordinates all follow, and we can wrtie down $$F$$:

$$
F = 
\begin{bmatrix}
1 & 0 & 0 & 0 & \Delta t & 0 & 0 & 0 \\
0 & 1 & 0 & 0 & 0 & \Delta t & 0 & 0 \\
0 & 0 & 1 & 0 & 0 & 0 & \Delta t & 0 \\
0 & 0 & 0 & 1 & 0 & 0 & 0 & \Delta t \\
0 & 0 & 0 & 0 & 1 & 0 & 0 & 0 \\
0 & 0 & 0 & 0 & 0 & 1 & 0 & 0 \\
0 & 0 & 0 & 0 & 0 & 0 & 1 & 0 \\
0 & 0 & 0 & 0 & 0 & 0 & 0 & 1
\end{bmatrix}
$$

The final corrected prediction of the bounding box can be computed as follow:

$$
\mathbf{x}_{t+1} = \hat{\mathbf{x}}_{t+1} + \mathbf{K}_{t+1} (\mathbf{y}_{k+1} - \mathbf{H} \hat{\mathbf{x}}_{k+1})
$$

Where:
- $$\mathbf{x}_{k+1}$$: Final corrected bounding box prediction
- $$\hat{\mathbf{x}}_{k+1}$$: Uncorrected bounding box prediction from the discrete-time linear system equation above
- $$\mathbf{K}_{k+1}$$: Kalman gain
- $$\mathbf{y}_{k+1}$$: Measurement, derived from the selected mask from SAM 2
- $$\mathbf{H}$$: Observation model matrix

Again, all the derivation of these matrices and vectors can be found in my [CubeSat Control System](../2_project) page, so take a look if you are interested in knowing the math and theory behind. One main advantage of using Kalman Filter to assist in SAM 2's VOS, apart from no additional training parameters and ease of use, is the ability to make modifications to the linear system. In SAMURAI, the authors chose to track the eight elements presented above in the state of the dynamical system, but one can easily adjust the state vector $$\mathbf{x}$$ to incorporate different information. For example, we may find tracking the top left and bottom right corners of the bounding box more effective than the top left corner plus its height and width. Basically, as long as we can come up with a correct representation of the proposed dynamical system, we can track the bounding box however we desire.

##### Motion-aware memory selection
In the original SAM 2, the preparation of memory for attention is done through selecting the previous $$N$$ frames. Specifically, SAM 2 simply selects the most recent $$N$$ frames as masks to feed into the memory attention module. This approach does not care about the quality of the memory selection. In SAMURAI, the authors addressed this issue by assigning scores to each mask at every frame and intelligently selecting the previous $$N$$ masks whose scores surpass the corresponding threshold.

$$
B_t = \{ m_i \mid s_{\text{mask}} > \tau_{\text{mask}}, \: s_{\text{obj}} > \tau_{\text{obj}}, \: s_{\text{kf}} > \tau_{\text{kf}}, \: t - N_{\text{max}} \leq i < t \}
$$

Similar to motion modeling, each mask is assigned an affinity score $$s_{\text{mask}}$$, an object occurrence score $$s_{\text{obj}}$$, and a motion score $$s_{\text{kf}}$$. SAMURAI has a maximum number of frames $$N_{\text{max}}$$ to look back, for all frames within this period, only $$N$$ frames are selected if their masks' $$s_{\text{mask}}$$, $$s_{\text{obj}}$$, and $$s_{\text{kf}}$$ pass the predetermined thresholds $$\tau_{\text{mask}}$$, $$\tau_{\text{obj}}$$, and $$\tau_{\text{kf}}$$. This appraoch encompasses motion cue into memory selection, hence significantly enhancing the VOT performance of SAM 2.

##### Fine-tuning
The addition of motion modeling and motion-aware memory selection already proves SAMURAI can greatly increase the performance of SAM 2's VOS. However, SAMURAI still uses the pre-trained weights from SAM 2 and SAM 2 was built for a more general task. SAMURAI, or the concept behind SAMURAI, has the ability to be virtually plugged into any other vision language models. At the time of this project, [Medical SAM 2](https://supermedintel.github.io/Medical-SAM2/) (MedSAM 2), which is built on top of SAM 2 specifically for segmenting medical images and also supports VOS, was not open-sourced yet. This made fine-tuning SAMURAI our only option, and also a useful learning journey. We opted for LoRA for fine-tuning due to its significant reduction of trainable parameters while retaining effective tuning results. Here is a short summary of how LoRA works:

###### 1. Full Weight Update
In standard fine-tuning, a weight matrix $$\mathbf{W} \in \mathbb{R}^{d \times k}$$ is updated by learning an additive term $$\Delta \mathbf{W}$$:

$$\mathbf{W}' = \mathbf{W} + \Delta \mathbf{W}$$

###### 2. Low-Rank Factorization
LoRA replaces the large $$\Delta \mathbf{W}$$ with a factorized version $$\mathbf{A}\mathbf{B}$$, where:

$$\mathbf{A} \in \mathbb{R}^{d \times r}, \quad
\mathbf{B} \in \mathbb{R}^{r \times k}, \quad
r \ll \min(d,k)$$

Thus, 

$$\Delta \mathbf{W} = \mathbf{A}\mathbf{B}$$

This drastically reduces the number of trainable parameters.

###### 3. LoRA Update Rule
The new weight matrix is:

$$\mathbf{W}' = \mathbf{W} + \alpha \mathbf{A}\mathbf{B}$$

where $$\alpha$$ is a scalar that scales the low-rank update.

- **Original weights $$\mathbf{W}$$** remain frozen.
- Only the low-rank factors $$\mathbf{A}$$ and $$\mathbf{B}$$ are trained.

###### 4. Parameter Efficiency
- **Full Fine-Tuning:** $$\Delta \mathbf{W}$$ has $$d \times k$$ parameters.
- **LoRA:** $$\mathbf{A}$$ and $$\mathbf{B}$$ have $$d \times r + r \times k$$ parameters, with $$r$$ being far smaller than $$d$$ or $$k$$.

This substantially lowers memory and computational cost.

###### 5. Why It Works
- **Base Model Knowledge:** The original $$\mathbf{W}$$ is already well-trained.
- **Lightweight Adaptation:** A low-rank update $$\mathbf{AB}$$ is enough to tailor the model to new tasks, serving as a regularized and efficient update.

Acccording to LoRA's paper, the intermediate rank $$r$$ can be as low as 1 or 2 and the result would be still on par with if not outperforming the full rank of $$\Delta \mathbf{W}$$ when fine-tuning large language models. In our application, we specifically targeted important layers, namely the mask decoder and memory attention layers, and froze the other layers.

```python
lora_config = LoraConfig(
    r=8,
    lora_alpha=8,
    target_modules=[
        # Mask decoder layers
        "sam_mask_decoder.transformer.layers.0.self_attn.q_proj",
        "sam_mask_decoder.transformer.layers.0.self_attn.k_proj",
        "sam_mask_decoder.transformer.layers.0.self_attn.v_proj",
        "sam_mask_decoder.transformer.layers.0.self_attn.out_proj",
        "sam_mask_decoder.transformer.layers.0.cross_attn_token_to_image.q_proj",
        "sam_mask_decoder.transformer.layers.0.cross_attn_token_to_image.k_proj",
        "sam_mask_decoder.transformer.layers.0.cross_attn_token_to_image.v_proj",
        "sam_mask_decoder.transformer.layers.0.cross_attn_token_to_image.out_proj",
        "sam_mask_decoder.transformer.layers.0.cross_attn_image_to_token.q_proj",
        "sam_mask_decoder.transformer.layers.0.cross_attn_image_to_token.k_proj",
        "sam_mask_decoder.transformer.layers.0.cross_attn_image_to_token.v_proj",
        "sam_mask_decoder.transformer.layers.0.cross_attn_image_to_token.out_proj",
        "sam_mask_decoder.transformer.layers.1.self_attn.q_proj",
        "sam_mask_decoder.transformer.layers.1.self_attn.k_proj",
        "sam_mask_decoder.transformer.layers.1.self_attn.v_proj",
        "sam_mask_decoder.transformer.layers.1.self_attn.out_proj",
        "sam_mask_decoder.transformer.layers.1.cross_attn_token_to_image.q_proj",
        "sam_mask_decoder.transformer.layers.1.cross_attn_token_to_image.k_proj",
        "sam_mask_decoder.transformer.layers.1.cross_attn_token_to_image.v_proj",
        "sam_mask_decoder.transformer.layers.1.cross_attn_token_to_image.out_proj",
        "sam_mask_decoder.transformer.layers.1.cross_attn_image_to_token.q_proj",
        "sam_mask_decoder.transformer.layers.1.cross_attn_image_to_token.k_proj",
        "sam_mask_decoder.transformer.layers.1.cross_attn_image_to_token.v_proj",
        "sam_mask_decoder.transformer.layers.1.cross_attn_image_to_token.out_proj",
        # Memory attention layers
        "memory_attention.layers.0.self_attn.q_proj",
        "memory_attention.layers.0.self_attn.k_proj",
        "memory_attention.layers.0.self_attn.v_proj",
        "memory_attention.layers.0.self_attn.out_proj",
        "memory_attention.layers.0.cross_attn_image.q_proj",
        "memory_attention.layers.0.cross_attn_image.k_proj",
        "memory_attention.layers.0.cross_attn_image.v_proj",
        "memory_attention.layers.0.cross_attn_image.out_proj",
        "memory_attention.layers.1.self_attn.q_proj",
        "memory_attention.layers.1.self_attn.k_proj",
        "memory_attention.layers.1.self_attn.v_proj",
        "memory_attention.layers.1.self_attn.out_proj",
        "memory_attention.layers.1.cross_attn_image.q_proj",
        "memory_attention.layers.1.cross_attn_image.k_proj",
        "memory_attention.layers.1.cross_attn_image.v_proj",
        "memory_attention.layers.1.cross_attn_image.out_proj"
    ],
    lora_dropout=0.0,
    bias="none",
)
```

The above code snippet is the LoRA configuration we chose to perform fine-tuning on SAMURAI using the CAMUS dataset. We chose the rank of the low-rank matrices $$r$$ to be 8. Several mask decoder and memory attention layers are chosen to be fine-tuned to perform echocardiogram video segmentation due to the following reasons:

- **Mask Decoder Focus**
  - Direct Impact on Segmentation Quality: The mask decoder is the final component responsible for generating precise object masks.
  - Task-Specific Refinement: Fine-tuning via LoRA here refines how the model attends to features for domain-specific or specialized segmentation tasks.
- **Memory Attention for Temporal Consistency**
  - Frame-to-Frame Tracking: Memory attention layers store and reference information from prior frames, improving object continuity over time.
  - Adaptation to Video Dynamics: LoRA-based updates to memory attention enable better handling of changes in object appearance, motion, or occlusions across frames.
- **Parameter-Efficient Fine-Tuning**
  - Low-Rank Updates Only: LoRA imposes rank constraints on attention projections, reducing the number of trainable parameters.
  - Preservation of Generality: Keeping the encoders frozen maintains broad, robust feature extraction while still allowing targeted improvements for video segmentation performance.
