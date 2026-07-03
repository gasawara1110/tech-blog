---

title: "Starting a Robotics Tech Blog from a Hardware-Oriented Perspective"
emoji: "🤖"
type: "idea"
topics: ["robotics", "ros2", "mujoco", "control", "hardware"]
published: false
----------------

> This article is an English translation of the original Japanese version.

# Introduction

Hi, I'm Kaz.

Since this is my first post, I would like to briefly introduce myself and explain what I plan to write about in this blog.

## Background

As a student, I mainly studied control engineering and was involved in robot contest activities.

Since starting my career, I have worked in several engineering fields:

Automotive-related system development at a major electrical equipment manufacturer
↓
Manufacturing engineering for circuit production at a startup
↓
Research and development of robot arms at a precision machinery manufacturer
↓
Development of powered prosthetic legs at a startup
↓
Development related to autonomous delivery robots in my current role
↓
Robot system development in the field of Physical AI in my current role

## Areas of Strength

I have worked across mechanical design, circuit design, embedded systems, control engineering, and ROS 2.

Among these, I am especially comfortable with control engineering, circuit design, and embedded systems.

In a positive sense, I can cover a fairly wide range of areas.
In a less positive sense, I may be a bit of a generalist.

# Why I Started Writing

## Sharing Robotics Knowledge from a Hardware-Oriented Perspective

When I first started learning about robotics as a student, the Maker movement was gaining momentum.

Arduino, Raspberry Pi, 3D printers, and similar tools were becoming more accessible, and there was a growing sense that individuals could take on robotics and electronics projects more easily.

More than ten years have passed since then, and I feel that the environment surrounding robotics has changed significantly.

AI, sensors, motors, edge devices, simulation environments, and manufacturing technologies are all evolving very quickly.

To be honest, I am often surprised by the speed of overseas robotics and hardware companies.
Seeing high-quality motor modules, easy-to-use development kits, and companies actively sharing real-world robot demonstrations has been both inspiring and motivating.

It reminds me that I also need to keep building, testing, and learning with my own hands.

Robots cannot be built with software alone, nor with hardware alone.

Mechanical design, circuits, embedded systems, control, power systems, communication, sensors, ROS 2, and many other elements need to come together before a robot finally works as one system.

I am more of a hardware-oriented engineer, but I find this boundary area especially interesting in robotics development.

When a robot does not work properly, the cause may be control, communication, power, mechanical design, or something else entirely.
At least for now, I do not think this is an area where AI alone can neatly isolate every problem.

But that process of investigating each possibility one by one, and finally seeing the robot work properly, is one of the most enjoyable parts of robotics.

In this blog, I would like to organize and share robotics-related technologies from that kind of hardware-oriented perspective.

Rather than only introducing parts or explaining theory, I want to write about why a certain configuration was chosen, where things became difficult, and what actually turned out to be challenging when trying to make things work.

## Building What I Want to Build and Putting It into Words

Robotics is not something that can be fully learned just by reading.

You build a simulation, implement a controller, run it, fail, check the logs, revise your hypothesis, and try again.

I believe understanding deepens little by little through that process.

In this blog, I do not want to stop at simply saying, "I made this."

I want to write about why I built it that way, where I got stuck, and what I learned from the process.

## Keeping a Record of Technical Trial and Error

I also want to create a place where I can look back on what I thought about, what I built, and how I verified things.

Work-related projects cannot be published as they are.
Also, a resume alone cannot fully communicate daily learning, small experiments, or the thoughts behind design decisions.

That is why I want to gradually build up public technical notes and experiment records under my own name.

# Topics I Want to Write About

### Robot Simulation with MuJoCo

Topics such as inverted pendulums, legged robots, and simulations involving contact dynamics.

### Control Engineering and Robot Control

PD control, state feedback, LQR, modeling, and the practical difficulties that appear during implementation.

### Motors and Power Electronics

Motors, motor drivers, regenerative braking, EDLCs, and power system design.

### Robot System Design with ROS 2

Topic design, node separation, debuggable system structures, and how system design connects with engineering documentation.

### Animal Locomotion and Legged Robots

Gait, hopping, ground contact, and energy efficiency.

# Closing

I plan to start by writing technical notes on MuJoCo simulations, robot control, motors, and power systems.

I do not think I can write perfectly polished articles from the beginning.
Instead, I want to honestly record what I built, what I tried, what failed, and what I learned.

When working with robots, unexpected things happen all the time once you actually try to make them move.

That is why I believe it is valuable to write not only about theory, but also about the implementation and validation process.

I hope this blog becomes a useful record for people who want to build robots, people interested in control and hardware, and also for my future self.
