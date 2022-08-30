---
layout:     post
title:      ICS Security Architecture
date:       2022-08-26 23:40:00
summary:    A broad introduction to ICS security architecture
categories: security
comments: true
---
Recently I've become very interested in Operational Technology (OT) security, more specifically Industrial Control Systems (ICS) security. Systems such as these require a unique perspective when it comes to securing them, particularly because they are so different to the standard kind of systems we secure in IT.

The scope of this post is to define a basic understanding of what an ICS is, and what the security architecture for one of these systems might look like. Please note that I am still learning myself, so my apologies for any mistakes and feedback is more than welcome.

### What is an ICS?

![](https://www.bgigurtsis.com/pictures/posts/otarch/controlloop.PNG)

Rather than butchering the definition, here's how Marty Edwards, ICS/OT Security SME, [describes what ICS are](https://youtu.be/k7qNCU8_Wpc?list=PL8OWO1qWXF4qRHrSTpwFbuLUL-bOrGn4y&t=119): `All Industrial Control Systems are designed for one primary purpose - and that is to automate a physical process. They accomplish this through sensors to measure physical properties and actuators to manipulate those properties`.

##### Programmable Logic Controllers (PLCs)

![](https://www.bgigurtsis.com/pictures/posts/otarch/plc.png)

One example Programmable Logic Controllers (PLCs) - PLCs are ruggedized industrial computers that can act as a middleman. They modulate outputs such as actuators depending on how they are programmed and what input they receive (generally input from sensors).

### ICS Security Architecture
