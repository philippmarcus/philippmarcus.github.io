---
layout: post
title: Behaviour Planning for a Highway Pilot based on C++ and State Machines 
date: 2021-01-1 13:32:20 +0300
description: A behaviour planner for self-drivinig cars in a highway scenario, written in C++.
img: ad-behaviour-planning/banner.png # Add image post (optional)
fig-caption: # Add figcaption (optional)
excerpt_separator: <!--more-->
tags: [C++, Behaviour Planning, Self-driving Cars, Highway Pilot]
---

In this article, I would like to revisit the Path Planning project from the Self-driving Car Nanodegree Program and redesign it towards a modularized, object-oriented architecture. This project uses a given simulator based on the Unity Engine, which simulates a round track three-lane highway with dynamic traffic and in which the own vehicle called 'Ego' is located. The task was to write a client in C++ that connects to the simulator via web sockets and receives a JSON data structure every 20ms, calculates a path that Ego should follow next, and return it to the simulator as a pearl-chain of waypoints. The data structure received by the client is created within the sensor fusion module inside the perception layer of Ego's onboard self-driving car stack and represents a state estimation of the detected other cars. In reality, such data created by the perception layer would be based on observed camera, lidar, or radar measurements. Finally, the JSON received in the C++ client contains one data record for each recognized vehicle and Ego itself, describing the measured coordinates and speed vectors.

The original project was monolithic and procedural, with no clear module structure or data types. This restricted its readability and expandability. The idea of ​​refactoring has therefore attracted me again and again since I completed the original project. Therefore, in this article, I will first describe a redesigned architecture made up of modules. Its dependencies point from unstable to stable, and from low-level to high-level modules. The individual modules are developed independently of one another and validated with unit testing. The actual path planning algorithm is then designed using a state machine design pattern and allows much better traceability than the original, if-then-else-based solution.