---
draft: false
date: 2024-12-10
categories:
  - Telemetrix Internals
comments: true
---

![](../assets/images/under_the_hood.png){ width="450" }


Over the past few years, I've been developing the 
Telemetrix family of libraries. These libraries, 
designed to facilitate microcontroller programming, allow you to 
control and monitor a variety of microcontrollers through a 
standardized set of Python3 client APIs and associated microcontroller 
servers written in C++.

<!-- more -->

The Telemetrix architecture, with its simplicity and consistency, 
is highly extensible. It empowers you to easily add new functionality and 
support for future microprocessors and hardware devices, giving you complete 
control over your development process.

Except for the Raspberry Pi Pico, all Telemetrix servers are based upon
Arduino Cores. This includes the server for
the Raspberry Pi 
Pico W. 

An Arduino Core contains all the software and tools to provide a software 
abstraction 
layer for a particular processor. 
It includes various tools, like the gcc compiler tools, to 
compile and link  the code.
Using Arduino Cores allows for a high degree of commonality between 
the servers, simplifies adding support for a new microprocessor, and allows for 
integrating additional Arduino device libraries.

The client APIs are designed to focus on efficiency
and productivity, sharing many standard features.
This makes porting code written from one microprocessor
to another a breeze, saving time and effort.

Currently, Telemetrix supports the:

* [Arduino ATMega boards(UNO, Leonardo, Mega2560)](https://mryslab.github.io/telemetrix/)
* [Arduino UNO R4 Minima and WIFI](https://mryslab.github.io/telemetrix-uno-r4/)
* [Arduino Nano RP2040 Connect ](https://mryslab.github.io/telemetrix-nano-2040-wifi/)
* [ESP8266](https://mryslab.github.io/telemetrix/)
* [ESP32](https://mryslab.github.io/telemetrix-esp32/)
* [Raspberry Pi Pico (Raspberry Pi C++ SDK-based)](https://mryslab.github.io/telemetrix-rpi-pico/)
* [Raspberry Pi Pico-W](https://mryslab.github.io/telemetrix-rpi-pico-w/)
* [STM32 Boards (i.e. Blackpill)](https://mryslab.github.io/telemetrix/)

To uncover Telemetrix internals, we will be using
the HC-SR04 ultrasonic sensor feature to explore the following:

* Telemetrix Command And Message Structure.
* Telemetrix Server File Layout
* Telemetrix Client File Layout



In the next post we will explore the Telemetrix Command And Message Structure.

