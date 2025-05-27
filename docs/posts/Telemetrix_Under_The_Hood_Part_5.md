---
draft: false
date: 2025-05-25
categories:
  - Telemetrix Internals
comments: true
---


![](../assets/images/summary.png)

## Summarizing - Adding New Features

Let's quickly summarize the steps needed to add new features to Telemetrix.

<!-- more -->

* Have a clear understanding of [Telemetrix messaging.](./Telemetrix_Under_The_Hood_Part_2.md)
* Select an [Appropriate support](./Telemetrix_Under_The_Hood_Part_3.md/#using-an-established-arduino-library-for-device-support) library for 
  your device.
### Adding A New Command
  * Select a new command ID for the device.
  * #### Server Changes
      * Add this ID to the [server code](https://github.com/MrYsLab/Telemetrix4UnoR4/blob/3629992d2c64da9b76eb5771d4c8933678149924/examples/Minima/Minima.ino#L126).
      * [Add a #define](https://github.com/MrYsLab/Telemetrix4UnoR4/blob/3629992d2c64da9b76eb5771d4c8933678149924/examples/Minima/Minima.ino#L35) for the new feature.
      * Select a name for the new command handler and add this to the [forward 
        references.](https://github.com/MrYsLab/Telemetrix4UnoR4/blob/3629992d2c64da9b76eb5771d4c8933678149924/examples/Minima/Minima.ino#L188)
      * Add the new command handler name to the [command descriptor table](https://github.com/MrYsLab/Telemetrix4UnoR4/blob/3629992d2c64da9b76eb5771d4c8933678149924/examples/Minima/Minima.ino#L324).
      * Write the [implementation code](https://github.com/MrYsLab/Telemetrix4UnoR4/blob/3629992d2c64da9b76eb5771d4c8933678149924/examples/Minima/Minima.ino#L651) for the new device.
      * [Update the firmware version number.](https://github.com/MrYsLab/Telemetrix4UnoR4/blob/3629992d2c64da9b76eb5771d4c8933678149924/examples/Minima/Minima.ino#L462)
  * #### Client API Changes
      * Add the new command ID to the [client's private constants file.](https://github.com/MrYsLab/telemetrix-uno-r4/blob/39f89aef39351ca339d3a9f42b240031e22a9b21/telemetrix_uno_r4/minima/telemetrix_uno_r4_minima/private_constants.py#L24)
      * Update the [client version number](https://github.com/MrYsLab/telemetrix-uno-r4/blob/39f89aef39351ca339d3a9f42b240031e22a9b21/telemetrix_uno_r4/minima/telemetrix_uno_r4_minima/private_constants.py#L107).
      * If the new feature requires instance-level storage, [allocate it in the _init_ 
      method.](https://github.com/MrYsLab/telemetrix-uno-r4/blob/39f89aef39351ca339d3a9f42b240031e22a9b21/telemetrix_uno_r4/minima/telemetrix_uno_r4_minima/telemetrix_uno_r4_minima.py#L45)
      * [Write and add](https://github.com/MrYsLab/telemetrix-uno-r4/blob/39f89aef39351ca339d3a9f42b240031e22a9b21/telemetrix_uno_r4/minima/telemetrix_uno_r4_minima/telemetrix_uno_r4_minima.py#L411) the new implementation code as a method to the API class.

### Adding A New Report
* Select a new report ID.
  * #### Server Changes
      * Add this ID to the [server code](https://github.com/MrYsLab/Telemetrix4UnoR4/blob/3629992d2c64da9b76eb5771d4c8933678149924/examples/Minima/Minima.ino#L398).
      * If the new [feature requires persistent storage,](https://github.com/MrYsLab/Telemetrix4UnoR4/blob/3629992d2c64da9b76eb5771d4c8933678149924/examples/Minima/Minima.ino#L571) add it to the server file.
      * To continuously monitor the device, write a 
[polling function](https://github.com/MrYsLab/Telemetrix4UnoR4/blob/3629992d2c64da9b76eb5771d4c8933678149924/examples/Minima/Minima.ino#L1836) using the device 
        library you selected above.
      * [Add the polling function to the loop function.](https://github.com/MrYsLab/Telemetrix4UnoR4/blob/3629992d2c64da9b76eb5771d4c8933678149924/examples/Minima/Minima.ino#L2063)
      * Optionally, you can write a one-shot function to poll the device.
  * #### Client API Changes
      * [Add the ID to the client API private constants file](https://github.com/MrYsLab/telemetrix-uno-r4/blob/39f89aef39351ca339d3a9f42b240031e22a9b21/telemetrix_uno_r4/minima/telemetrix_uno_r4_minima/private_constants.py#L85)
      * [Add any storage you might need to handle the report](https://github.com/MrYsLab/telemetrix-uno-r4/blob/39f89aef39351ca339d3a9f42b240031e22a9b21/telemetrix_uno_r4/minima/telemetrix_uno_r4_minima/telemetrix_uno_r4_minima.py#L184)
      * [Write a report handler method and add it to the API class.](https://github.com/MrYsLab/telemetrix-uno-r4/blob/39f89aef39351ca339d3a9f42b240031e22a9b21/telemetrix_uno_r4/minima/telemetrix_uno_r4_minima/telemetrix_uno_r4_minima.py#L2336)
      * [Add a new entry into the report dispatch dictionary.](https://github.com/MrYsLab/telemetrix-uno-r4/blob/39f89aef39351ca339d3a9f42b240031e22a9b21/telemetrix_uno_r4/minima/telemetrix_uno_r4_minima/telemetrix_uno_r4_minima.py#L134)


