---
draft: false
date: 2024-12-14
categories:
  - Telemetrix Internals
comments: true
---

![](../assets/images/under_the_hood.png){ width="450" }

Let's now take a look at the code that implements the Python Telemetrix API
for The Arduino UNO R4 Minima.

The API is implemented within two files. 

APIs for all Telemetrix supported microprocessors are very similar in nature.

A Telemetrix API consists of two files each containing a class definition. One that 
contains the 
[class](https://github.com/MrYsLab/telemetrix-uno-r4/blob/master/telemetrix_uno_r4/minima/telemetrix_uno_r4_minima/telemetrix_uno_r4_minima.py) that implements
the API. The other is a supporting file containing a set of "constants"
utilized by the class.





<!-- more -->