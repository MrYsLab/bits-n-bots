---
draft: false
date: 2025-05-15
categories:
  - Telemetrix Internals
comments: true
---

![](../assets/images/api.png){ width="450" }

## The Telemetrix Python Client API

In this post, we will explore the Telemetrix Python client API implementation
for the Arduino UNO R4 Minima. To help focus the discussion, 
we will again use the **HC-SR04 SONAR distance sensor** feature as an example.
To make it easier to find the discussions about this feature, search for the _**SONAR 
SIDEBAR**_ heading.

However, before getting to the actual code, 
let's explore two important design features that 
are integral to a Telemetrix Python API.

A goal of the Telemetrix framework is to provide an experience that is close to 
real-time as possible.

A Telemetrix client has three major operations all competing for the attention
of the processor.

Those operations are:

* Process the on-demand API command requests.
* Receive and buffer reports and associated data coming from the Telemetrix server.
* Process the data contained in the buffered reports.

### Concurrency

To manage these competing interests, Telemetrix uses a Python concurrency scheme.

For the synchronous API, [TelemetrixUnoR4Minima](https://mryslab.github.io/telemetrix-uno-r4/telemetrix_minima_reference/),
[Python threading](https://docs.python.org/3/library/threading.html) 
is used, and for the asynchronous API, 
[telemetrix_uno_r4_minima_aio](https://mryslab.github.io/telemetrix-uno-r4/telemetrix_minima_reference_aio/),
[Python asyncio](https://docs.python.org/3/library/asyncio.html) is used.

For the synchronous API, a [Python deque](https://docs.python.org/3/library/collections.html#deque-objects)
is used. A deque supports thread-safe, memory efficient appends and pops from either 
side of the deque to buffer and retrieve Telemetrix server reports.

<!-- more -->


### Minimizing Telemetrix Client/Server Messaging Overhead Via Callbacks

The Telemetrix protocol is designed to minimize the data communication overhead
between the client and server by using a callback scheme.

The alternative to using callbacks is to have the client poll the server for
data. This approach is not only more complex, but also less efficient.
For example, if we want to continuously monitor the state of a pin, we would have to 
poll the
server for data at a regular interval.

A Telemetrix server autonomously monitors input pins and only transmits a report to 
the client for processing when the state or value of the pin has changed.

What is even worse is that polling increases the possibility of missing a pin value
change event as a result of the added messaging and processing overhead.

#### What Is A Callback?

Simply put, a callback is a function 
passed as an argument to another function. This action is sometimes referred
to as _registering_ a callback.

Callback functions are user-written pieces of code that process
server report data and are part of a Telemetrix client application.

Let's look at an example application that sets a pin as a digital input pin
and reports the state of the pin when the pin state has been reported as changed.

```aiignore
import sys
import time

from telemetrix_uno_r4.minima.telemetrix_uno_r4_minima import telemetrix_uno_r4_minima
"""
Monitor a digital input pin
"""

"""
Setup a pin for digital input and monitor its changes
"""

# Set up a pin for analog input and monitor its changes
DIGITAL_PIN = 12  # arduino pin number

# Callback data indices
CB_PIN_MODE = 0
CB_PIN = 1
CB_VALUE = 2
CB_TIME = 3

# variable to hold the last time a button state changed
debounce_time = time.time()


def the_callback(data):
    """
    A callback function to report data changes.
    This will print the pin number, its reported value and
    the date and time when the change occurred

    :param data: [pin, current reported value, pin_mode, timestamp]
    """
    global debounce_time

    # if the time from the last event change is > .2 seconds, the input is debounced
    if data[CB_TIME] - debounce_time > .3:
        date = time.strftime('%Y-%m-%d %H:%M:%S', time.localtime(data[CB_TIME]))
        print(f'Pin: {data[CB_PIN]} Value: {data[CB_VALUE]} Time Stamp: {date}')
        debounce_time = data[CB_TIME]


def digital_in(my_board, pin):
    """
     This function establishes the pin as a
     digital input. Any changes on this pin will
     be reported through the call back function.

     :param my_board: a telemetrix instance
     :param pin: Arduino pin number
     """

    # set the pin mode
    my_board.set_pin_mode_digital_input(pin, the_callback)
    # time.sleep(1)
    # my_board.disable_all_reporting()
    # time.sleep(4)
    # my_board.enable_digital_reporting(12)

    # time.sleep(3)
    # my_board.enable_digital_reporting(pin)
    # time.sleep(1)

    print('Enter Control-C to quit.')
    # my_board.enable_digital_reporting(12)
    try:
        while True:
            time.sleep(.0001)
    except KeyboardInterrupt:
        board.shutdown()
        sys.exit(0)


board = telemetrix_uno_r4_minima.TelemetrixUnoR4Minima()
try:
    digital_in(board, DIGITAL_PIN)
except KeyboardInterrupt:
    board.shutdown()
    sys.exit(0)

```

