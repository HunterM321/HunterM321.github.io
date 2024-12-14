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
