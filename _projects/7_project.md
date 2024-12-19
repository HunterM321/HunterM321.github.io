---
layout: page
title: Echocardiogram Segmentation
description: Robotics and Control Lab, UBC
img: assets/img/echo.jpg
importance: 1
category: work
related_publications: false
---

This page is still being updated, please check back later. I also recommend checking out my [CubeSat Control System](../2_project) page and my [Deep Learning-Based Dynamical System Solver](../4_project) page, some really cool stuff happening there 😃.

#### Overview
Throughout the 4 months I stayed in UBC's [Robotics and Control Lab](https://rcl.ece.ubc.ca) (RCL) as a research intern, I worked on image and video segmentation of [echocardiogram](https://www.mayoclinic.org/tests-procedures/echocardiogram/about/pac-20393856). Specifically, I focused on chamber segmentation. The ultimate goal of this project is to teach a single agent to **simultaneously** segment all 4 chambers, which could be done by incorporating semantic information or scene graph, instead of training individual agent for each chamber segmentation. I mainly worked on two sub-projects. The first one is baseline testing for [MedSAM](https://www.nature.com/articles/s41467-024-44824-z) and [SAM 2](https://ai.meta.com/sam2/). The second one is video object tracking and segmentation using [SAMURAI](https://yangchris11.github.io/samurai/), which is also a derivative of SAM 2. I would like to focus more on discussing the SAMURAI project as I think it is the more interesting one.

#### What is SAMURAI
SAMURAI stands for _SAM-based Unified and Robust zero-shot visual tracker with motion-Aware Instance-level memory_. Specifically, SAMURAI is a derivative of the **video object segmentation** (VOS) functionality introduced in SAM 2, seeking to improve the **video object tracking** (VOT) performance without a huge architectural modification. Maybe before we introduce SAMURAI's novelty in this domain of knowledge, a quick background of SAM 1 and SAM 2's architectures would come in handy.

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

So how does SAMURAI improve upon the model proposed in SAM 2? In short, since SAM 2 already has very robust segmentation performance, SAMURAI focuses on improving the **tracking** quality of the segmented object. It does so by introducing two novelties—both of which do **NOT** require any additional learnable parameters. The first contribution is the addition of a **motion modeling module**. This module trakcs and predicts the bounding box surrounding the segmented object; specifically, it tracks and predicts the bounding box's top left coordinate, height and width, as well as their corresponding rate of change (i.e., speed). Using this motion cue, the motion modeling module is able to handle difficult scenarios, such as self-occlusions and fast-moving objects. The second contribution is the addition of **motion-aware memory selection** from the memory bank. The new proposed scheme selects high-quality memories from the memory bank. instead of selecting the latest memories using a fix window design. The higher quality memories will directly have an influence during cross-attention with the current frame.

<div class="row">
    <div class="col-sm-4 mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/first_frame.jpg" title="Sample first frame" class="img-fluid rounded z-depth-1" %}
    </div>
    <div class="col-sm-4 mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/middle_frame.jpg" title="Sample middle frame" class="img-fluid rounded z-depth-1" %}
    </div>
    <div class="col-sm-4 mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/last_frame.jpg" title="Sample last frame" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Frame extractions of video segmention of LV chamber using SAMURAI on EchoNet-Dynamic dataset
</div>
