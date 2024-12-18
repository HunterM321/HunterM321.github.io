---
layout: page
title: Echocardiogram Segmentation
description: Robotics and Control Lab, UBC
img: assets/img/echo.jpg
importance: 1
category: work
related_publications: false
---

This page is still being updated, please check back later. For now, I recommend checking out my [CubeSat Control System](../2_project) page and my [Deep Learning-Based Dynamical System Solver](../4_project) page, some really cool stuff happening there 😃.

#### Overview
Throughout the 4 months I stayed in UBC's [Robotics and Control Lab](https://rcl.ece.ubc.ca) (RCL) as a research intern, I worked on image and video segmentation of [echocardiogram](https://www.mayoclinic.org/tests-procedures/echocardiogram/about/pac-20393856). Specifically, I focused on chamber segmentation. The ultimate goal of this project is to teach a single agent to **simultaneously** segment all 4 chambers, which could be done by incorporating semantic information or scene graph, instead of training individual agent for each chamber segmentation. I mainly worked on two sub-projects. The first one is baseline testing for [MedSAM](https://www.nature.com/articles/s41467-024-44824-z) and [SAM 2](https://ai.meta.com/sam2/). The second one is video object tracking and segmentation using [SAMURAI](https://yangchris11.github.io/samurai/), which is also a derivative of SAM 2. I would like to focus more on discussing the SAMURAI project as I think it is the more interesting one.

#### What is SAMURAI
SAMURAI stands for _SAM-based Unified and Robust zero-shot visual tracker with motion-Aware Instance-level memory_. Specifically, SAMURAI is a derivative of the **video segmentation** functionality introduced in SAM 2. 
