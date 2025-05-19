---
draft: false
date: 2025-05-15
categories:
  - Telemetrix Internals
comments: true
---

![](../assets/images/api.png){ width="450" }

## The Telemetrix Python Client API

This post will explore the implementation of the
[Telemetrix Python client API](https://mryslab.github.io/telemetrix-uno-r4/telemetrix_minima_reference/)
for the Arduino UNO R4 Minima. To help focus the discussion, 
we will again use the **HC-SR04 SONAR distance sensor** feature as an example.
Please search for the **_SONAR SIDEBAR_** heading to make the discussions about this 
feature easier to find.

Although this discussion is specific to the Arduino UNO R4 Minima client API, all
Telemetrix client APIs are very similar, so you should be able to apply the information 
in this post to any other Telemetrix client API.

The Telemetrix framework aims to provide an experience that is as close to 
real-time as possible.
To achieve this goal, a Telemetrix client implements concurrency and callback schemes.




### Concurrency

Concurrency refers to a system's ability to execute multiple tasks 
through simultaneous execution or time-sharing (context switching), 
sharing resources, and managing interactions. It improves responsiveness, 
throughput, and scalability.

A Telemetrix client has three primary operations competing for the processor's attention.

Those operations are:

* Process the on-demand API command requests.
* Receive and buffer reports sent from the Telemetrix server over the transport link.
* Process the data contained in the buffered reports.

Concurrency allows the client to send command messages to the server while retrieving 
and buffering server reports and processing the buffered 
reports. All these operations are performed with minimal blocking to 
ensure the application is highly reactive.

Concurrency is implemented using one of two Python concurrency schemes.

For the 
[TelemetrixUnoR4Minima](https://mryslab.github.io/telemetrix-uno-r4/telemetrix_minima_reference/)
threaded API, 
[Python threading](https://docs.python.org/3/library/threading.html)
and a 
Python [_deque_](https://docs.python.org/3/library/collections.html#collections.deque) 
are deployed. A deque, which stands for double-ended queue, is a 
data structure that allows you to add and remove items from either end of the queue. It
is thread-safe and part of the Python standard library.

For the
[TelemetrixUnoR4MinimaAio](https://mryslab.github.io/telemetrix-uno-r4/telemetrix_minima_reference_aio/)
asynchronous API, 
[Python asyncio](https://docs.python.org/3/library/asyncio.html) is used to implement 
concurrency.


<!-- more -->


### Minimizing Telemetrix Client/Server Messaging Overhead Via Callbacks

The Telemetrix protocol uses a callback scheme to minimize the data 
communication overhead between the client and server.

The alternative to using callbacks is to poll the server periodically 
for changes in input data.  This approach is unacceptable because it is highly inefficient.

Not only does a command message need to be formed and 
transmitted, but the client, without blocking,  must also wait for the server to create a 
response and send a report message back.

During all this messaging and waiting, a data change event on a 
pin or device may be missed.


Instead of polling, a Telemetrix server autonomously monitors input pins and only 
transmits a report to the client when the pin's state or value changes. 
Because the server continuously monitors its input pins and devices,
a data change is less likely to be missed. A report is immediately sent to the client 
when a change is detected.


When callbacks are used instead of polling, bidirectional communication between the 
client and server is significantly reduced by 50% or more when compared to polling.


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

Let's look at an example that sets a pin as a digital input pin and 
prints pin state changes.
The pin is connected to a pushbutton switch, and switch debouncing is provided within 
the callback function as part of this application.

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

    # if the time from the last event change is > .3 seconds, the input is debounced
    if data[CB_TIME] - debounce_time > .3:
        date = time.strftime('%Y-%m-%d %H:%M:%S', time.localtime(data[CB_TIME]))
        print(f'Pin: {data[CB_PIN]} Value: {data[CB_VALUE]} Time Stamp: {date}')
        debounce_time = data[CB_TIME]
```
All callback functions have the same signature. That signature is a single parameter 
called 
_**data**_. 

You are free to name the callback anything you like.
For this example, we called the callback function _the_callback_.

The data parameter is always a Python list, but the contents 
vary by report type. The last element of the list is always a time stamp, 
except for the loop_back report, which does not contain a time stamp.

When an API call requires a callback function, the list's contents are 
specified in the API documentation. For example, for _set_pin_mode_digital_input_,
data contains the following list:
[pin_type, pin_number, pin_value, raw_time_stamp]

The pin type identifies the pin type, such as DHT, analog, digital, etc. The pin 
number is the pin being monitored, the pin_value is 
the current reported value, and the raw_time_stamp is the time stamp of 
the event and the time that the client received the report.

In the example, we create offset variables to help dereference the list 

```aiignore
# Callback data indices
CB_PIN_MODE = 0
CB_PIN = 1
CB_VALUE = 2
CB_TIME = 3
```

Let's look at how the callback dereferences and uses the list contents.

The first thing the callback does is check if the time from the last event change is 
greater than .3 seconds. If it is, the input is debounced, and we can proceed.

Next, it converts the time stamp to a human-readable date and time.

Finally, it prints the pin number, the reported value, and the change's date and time.

We keep the callback code as simple as possible to avoid blocking.

#### Registering The Callback

```aiignore
# set the pin mode
    my_board.set_pin_mode_digital_input(pin, the_callback)
```

The call back is registered in the call to set_pin_mode_digital_input.

The callback function is automatically called when a digital input report is received.

The scope of a callback is only limited by how you wish to use it.
You may have a single callback function to handle all reports from 
a single pin_type or a callback function for each pin.

### The Telemetrix Client API Source Files

Let's look at the file tree for the telemetrix-uno-r4 project.

```aiignore
telemetrix_uno_r4
├── __init__.py
├── minima
│   ├── __init__.py
│   ├── telemetrix_uno_r4_minima
│   │   ├── __init__.py
│   │   ├── private_constants.py
│   │   └── telemetrix_uno_r4_minima.py
│   └── telemetrix_uno_r4_minima_aio
│       ├── __init__.py
│       ├── private_constants.py
│       ├── telemetrix_aio_serial.py
│       └── telemetrix_uno_r4_minima_aio.py
└── wifi
    ├── telemetrix_uno_r4_wifi
    │   ├── __init__.py
    │   ├── private_constants.py
    │   └── telemetrix_uno_r4_wifi.py
    └── telemetrix_uno_r4_wifi_aio
        ├── __init__.py
        ├── private_constants.py
        ├── telemetrix_aio_ble.py
        ├── telemetrix_aio_serial.py
        ├── telemetrix_aio_socket.py
        └── telemetrix_uno_r4_wifi_aio.py
```
This project consists of two main modules: one for the Arduino R4 Minima 
microcontroller and the other for the Arduino R4 WIFI microcontroller.

The minima module contains two submodules: _telemetrix_uno_r4_minima_ for 
the threaded API and _telemetrix_uno_r4_minima_aio_ 
for the asynchronous API.

The wifi module also contains two submodules: 
one for the threaded API, _telemetrix_uno_r4_wifi_, and the other for 
the asynchronous API, _telemetrix_uno_r4_wifi_aio_.

All four submodules contain a file called private_constants.py and a file that 
defines the API class definition.  For the threaded Minima API, 
this file is called _telemetrix_uno_r4_minima.py_, and we will look at 
it after discussing the private constants file.

The asyncio sub-modules contain one or more additional files that support a specific 
data-link transport.

This discussion will focus on the threaded API, 
_[telemetrix_uno_r4_minima](https://github.com/MrYsLab/telemetrix-uno-r4/tree/master/telemetrix_uno_r4/minima/telemetrix_uno_r4_minima)_, 
for the 
Arduino R4 Minima microcontroller.

#### The Private Constants File


All Telemetrix clients "define their constants" in a file called _private_constants.py_.

There isn't a built-in way to declare constants in Python like in some other languages 
(e.g., using const in C++ or final in Java). 
However, by convention, variables intended to be constants are named 
using all uppercase letters with underscores separating words. 
This convention serves as a signal these values should not be changed.

The _private_constants.py_ file is included with all Telemetrix clients and are specific
to the client API.


Let's look at
**_[private_constants.py](https://github.
com/MrYsLab/telemetrix-uno-r4/blob/master/telemetrix_uno_r4/minima/telemetrix_uno_r4_minima/private_constants.py)_**.


```aiignore
class PrivateConstants:
    """
    This class contains a set of constants for telemetrix internal use .
    """

    # commands
    # send a loop back request - for debugging communications
    LOOP_COMMAND = 0
    SET_PIN_MODE = 1  # set a pin to INPUT/OUTPUT/PWM/etc
    DIGITAL_WRITE = 2  # set a single digital pin value instead of entire port
    ANALOG_WRITE = 3
    MODIFY_REPORTING = 4
    GET_FIRMWARE_VERSION = 5
    ARE_U_THERE = 6  # Arduino ID query for auto-detect of telemetrix connected boards
    SERVO_ATTACH = 7
    SERVO_WRITE = 8
    SERVO_DETACH = 9
    I2C_BEGIN = 10
    I2C_READ = 11
    I2C_WRITE = 12
    SONAR_NEW = 13
    DHT_NEW = 14
    STOP_ALL_REPORTS = 15
    SET_ANALOG_SCANNING_INTERVAL = 16
    ENABLE_ALL_REPORTS = 17
    RESET = 18
    SPI_INIT = 19
    SPI_WRITE_BLOCKING = 20
    SPI_READ_BLOCKING = 21
    SPI_SET_FORMAT = 22
    SPI_CS_CONTROL = 23
    ONE_WIRE_INIT = 24
    ONE_WIRE_RESET = 25
    ONE_WIRE_SELECT = 26
    ONE_WIRE_SKIP = 27
    ONE_WIRE_WRITE = 28
    ONE_WIRE_READ = 29
    ONE_WIRE_RESET_SEARCH = 30
    ONE_WIRE_SEARCH = 31
    ONE_WIRE_CRC8 = 32
    SET_PIN_MODE_STEPPER = 33
    STEPPER_MOVE_TO = 34
    STEPPER_MOVE = 35
    STEPPER_RUN = 36
    STEPPER_RUN_SPEED = 37
    STEPPER_SET_MAX_SPEED = 38
    STEPPER_SET_ACCELERATION = 39
    STEPPER_SET_SPEED = 40
    STEPPER_SET_CURRENT_POSITION = 41
    STEPPER_RUN_SPEED_TO_POSITION = 42
    STEPPER_STOP = 43
    STEPPER_DISABLE_OUTPUTS = 44
    STEPPER_ENABLE_OUTPUTS = 45
    STEPPER_SET_MINIMUM_PULSE_WIDTH = 46
    STEPPER_SET_ENABLE_PIN = 47
    STEPPER_SET_3_PINS_INVERTED = 48
    STEPPER_SET_4_PINS_INVERTED = 49
    STEPPER_IS_RUNNING = 50
    STEPPER_GET_CURRENT_POSITION = 51
    STEPPER_GET_DISTANCE_TO_GO = 52
    STEPPER_GET_TARGET_POSITION = 53
    GET_FEATURES = 54
    SONAR_DISABLE = 55
    SONAR_ENABLE = 56
    BOARD_HARD_RESET = 57

    # reports
    # debug data from Arduino
    DIGITAL_REPORT = DIGITAL_WRITE
    ANALOG_REPORT = ANALOG_WRITE
    FIRMWARE_REPORT = GET_FIRMWARE_VERSION
    I_AM_HERE_REPORT = ARE_U_THERE
    SERVO_UNAVAILABLE = SERVO_ATTACH
    I2C_TOO_FEW_BYTES_RCVD = 8
    I2C_TOO_MANY_BYTES_RCVD = 9
    I2C_READ_REPORT = 10
    SONAR_DISTANCE = 11
    DHT_REPORT = 12
    SPI_REPORT = 13
    ONE_WIRE_REPORT = 14
    STEPPER_DISTANCE_TO_GO = 15
    STEPPER_TARGET_POSITION = 16
    STEPPER_CURRENT_POSITION = 17
    STEPPER_RUNNING_REPORT = 18
    STEPPER_RUN_COMPLETE_REPORT = 19
    FEATURES = 20
    DEBUG_PRINT = 99

    TELEMETRIX_VERSION = "1.1.1"

    # reporting control
    REPORTING_DISABLE_ALL = 0
    REPORTING_ANALOG_ENABLE = 1
    REPORTING_DIGITAL_ENABLE = 2
    REPORTING_ANALOG_DISABLE = 3
    REPORTING_DIGITAL_DISABLE = 4

    # Pin mode definitions
    AT_INPUT = 0
    AT_OUTPUT = 1
    AT_INPUT_PULLUP = 2
    AT_ANALOG = 3
    AT_SERVO = 4
    AT_SONAR = 5
    AT_DHT = 6
    AT_MODE_NOT_SET = 255

    # maximum number of digital pins supported
    NUMBER_OF_DIGITAL_PINS = 100

    # maximum number of analog pins supported
    NUMBER_OF_ANALOG_PINS = 20

    # maximum number of sonars allowed
    MAX_SONARS = 6

    # maximum number of DHT devices allowed
    MAX_DHTS = 6

    # DHT Report sub-types
    DHT_DATA = 0
    DHT_ERROR = 1

    # feature masks
    ONEWIRE_FEATURE = 0x01
    DHT_FEATURE = 0x02
    STEPPERS_FEATURE = 0x04
    SPI_FEATURE = 0x08
    SERVO_FEATURE = 0x10
    SONAR_FEATURE = 0x20
```

The "constants" are all defined as class variables of the PrivateConstants class.

##### Command IDs
The first section of the file defines the command IDs. These values must match those 
[defined in the server](https://github.com/MrYsLab/Telemetrix4UnoR4/blob/3629992d2c64da9b76eb5771d4c8933678149924/examples/Minima/Minima.ino#L128).
When adding a new feature, make sure that server is updated with the new value, and that
you add the new feature ID after the last command ID in the file.

###### _SONAR SIDEBAR_

There are three command IDs associated with the HC-SR04 device.

* SONAR_NEW - commands the server to add a new entry in its "sonar table" and to 
  immediately start monitoring device.
* SONAR_DISABLE - commands the server to stop sending sonar reports.
* SONAR_ENABLE - commands the server to send sonar reports.



