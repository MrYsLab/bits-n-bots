---
draft: false
date: 2025-05-15
categories:
  - Telemetrix Internals
comments: true
---

![](../assets/images/api.png){ width="450" }

## The Telemetrix Python Client API

This post will explore the Telemetrix Python client API 
implementation for the Arduino UNO R4 Minima. To help focus the discussion, 
we will again use the **HC-SR04 SONAR distance sensor** feature as an example.
Please search for the **_SONAR SIDEBAR_** heading to make the discussions about this 
feature easier to find.

Before getting to the actual code, 
let's explore two crucial design features
integral to a Telemetrix Python API.

The Telemetrix framework aims to provide an experience as close 
to real-time as possible. 
To achieve this goal, the Telemetrix client implements concurrency and callback schemes.




### Concurrency

Concurrency refers to a system's ability to execute multiple tasks 
through simultaneous execution or time-sharing (context switching), 
sharing resources, and managing interactions. It improves responsiveness, 
throughput, and scalability.

A Telemetrix client has three primary operations competing for the processor's attention.

Those operations are:

* Process the on-demand API command requests.
* Receive and buffer reports and their associated data coming from the Telemetrix server.
* Process the data contained in the buffered reports.

To manage these competing interests, Telemetrix uses a Python concurrency scheme.

For the synchronous API, [TelemetrixUnoR4Minima](https://mryslab.github.io/telemetrix-uno-r4/telemetrix_minima_reference/),
[Python threading](https://docs.python.org/3/library/threading.html) 
is used, and for the asynchronous API, 
[telemetrix_uno_r4_minima_aio](https://mryslab.github.io/telemetrix-uno-r4/telemetrix_minima_reference_aio/),
[Python asyncio](https://docs.python.org/3/library/asyncio.html) is used.

A [Python deque](https://docs.python.org/3/library/collections.html#deque-objects)
is used. It supports thread-safe, memory-efficient appends and pops from either side of 
the deque to buffer and retrieve Telemetrix server reports.

<!-- more -->


### Minimizing Telemetrix Client/Server Messaging Overhead Via Callbacks

The Telemetrix protocol uses a callback scheme to minimize the data 
communication overhead between the client and server.

The alternative to using callbacks is for the client to poll the server for input data 
changes, but this approach is highly inefficient.

A message must be sent to the server, and the client must wait for the response.
In addition, polling increases the chances of missing a pin value change event.

Instead of polling, a Telemetrix server autonomously monitors input pins and only 
transmits a report to the client when the pin's state or value has changed.

The amount of bidirectional communication between the client and 
server is significantly reduced.

Telemetrix uses a callback scheme to process incoming reports asynchronously.

#### What Is A Callback?

Simply put, a callback is a function 
passed as an argument to another function. This action is sometimes referred
to as _registering_ a callback.

Callbacks are registered in the following telemetrix API calls:

* [i2c_read](https://mryslab.github.io/telemetrix-uno-r4/telemetrix_minima_reference/#telemetrix_uno_r4_minima.TelemetrixUnoR4Minima.i2c_read)
* [i2c_read_restart_transmission](https://mryslab.github.io/telemetrix-uno-r4/telemetrix_minima_reference/#telemetrix_uno_r4_minima.TelemetrixUnoR4Minima.i2c_read_restart_transmission)
* [loop_back](https://mryslab.github.io/telemetrix-uno-r4/telemetrix_minima_reference/#telemetrix_uno_r4_minima.TelemetrixUnoR4Minima.loop_back)
* [set_pin_mode_analog_input](https://mryslab.github.io/telemetrix-uno-r4/telemetrix_minima_reference/#telemetrix_uno_r4_minima.TelemetrixUnoR4Minima.set_pin_mode_analog_input)
* [set_pin_mode_dht](https://mryslab.github.io/telemetrix-uno-r4/telemetrix_minima_reference/#telemetrix_uno_r4_minima.TelemetrixUnoR4Minima.set_pin_mode_dht)
* [set_pin_mode_digital_input](https://mryslab.github.io/telemetrix-uno-r4/telemetrix_minima_reference/#telemetrix_uno_r4_minima.TelemetrixUnoR4Minima.set_pin_mode_digital_input)
* [set_pin_mode_digital_input_pullup](https://mryslab.github.io/telemetrix-uno-r4/telemetrix_minima_reference/#telemetrix_uno_r4_minima.TelemetrixUnoR4Minima.set_pin_mode_digital_input)
* [set_pin_mode_sonar](https://mryslab.github.io/telemetrix-uno-r4/telemetrix_minima_reference/#telemetrix_uno_r4_minima.TelemetrixUnoR4Minima.set_pin_mode_sonar)
* [spi_read_blocking](https://mryslab.github.io/telemetrix-uno-r4/telemetrix_minima_reference/#telemetrix_uno_r4_minima.TelemetrixUnoR4Minima.spi_read_blocking)

Callback functions are user-written pieces of code that process
server report data and are part of a Telemetrix client application.

Let's look at an example application that sets a pin as a digital input pin and 
reports the pin state when it has been reported as changed.
The pin is connected to a pushbutton switch, and switch debouncing is provided
as part of this application.

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

    :param data: [pin_type, pin_number, pin_value, raw_time_stamp]
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
Let's begin by looking at the callback function.
```aiignore
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
```
All callback functions have the same signature. That signature is a single parameter 
called 
_**data**_. 

You are free to name the callback anything you like.
For this example, we named the callback function _the_callback_.

The data parameter is always a Python list, but the contents of the list vary by the 
report type. The last element of the list is always a time stamp except 
for the loop_back report which does not contain a time stamp.

When an API call requires a callback function, the 
contents of list
are specified in the API documentation. For example, for _set_pin_mode_digital_input_,
data contains the following list:
[pin_type, pin_number, pin_value, raw_time_stamp]

The pin type identifies the pin type such as dht, analog, digital, etc. The pin number
is the pin that is being monitored, the pin_value is the current reported value, and 
the raw_time_stamp is the time stamp of the event, and the raw_time_stamp is the time 
that the client received the report.

In the example, we create variables that are offsets into the list, to help
dereference the list contents.

```aiignore
# Callback data indices
CB_PIN_MODE = 0
CB_PIN = 1
CB_VALUE = 2
CB_TIME = 3
```

Now, let's look at how the callback dereferences and uses the list contents.

The first thing the callback does is check if the time from the last event change is 
greater than .3 seconds. If it is, the input is debounced and we can proceed.

Next, it converts the time stamp to a human-readable date and time.

Finally, it prints the pin number, its reported value and the date and time when the 
change occurred.

We try to keep the callback code as simple as possible, to avoid blocking.

#### Registering The Callback

```aiignore
# set the pin mode
    my_board.set_pin_mode_digital_input(pin, the_callback)
```

The call back is registered in the call to set_pin_mode_digital_input.

When a digital input report is received, the callback function is automatically called.

The scope of a callback is only limited by how you wish to use it.
You may have a single callback function to handle all reports from a single pin_type,
or you may have a callback function for each pin.
