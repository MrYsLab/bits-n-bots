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

Let's look at an example.

### Registering A Callback

When a pin is set to an input mode, or when a supported device is enabled to return
data, such as an HC-SR04 sensor distance sensor, the API requires that a call back 
function be specified.



The callback function, written by the user, is called when a report is received, and
the data contained in the report is passed to the callback function as a parameter.



<!-- more -->

## Design Features - Callbacks and Concurrency

### Callbacks

When we wish to interact with a pin set up as input, we might write an Arduino
sketch that looks something like this:

```aiignore
int ledPin = 6;    // LED connected to digital pin 13
int inPin = 12;    // pushbutton connected to digital pin 7
int val = 0;      // variable to store the read value

void setup() {
  pinMode(ledPin, OUTPUT);  // sets the digital pin 13 as output
  pinMode(inPin, INPUT);    // sets the digital pin 7 as input
}

void loop() {
  val = digitalRead(inPin);   // read the input pin
  digitalWrite(ledPin, val);  // sets the LED to the button's value
}
```

The pin mode for pin 7 is set to input mode in the _setup_ function, and the state
of the pin is continuously read using a call to _digital_read_.

To do something similar using Telemetrix, the Python application might look something 
like:


When establishing a pin as an input pin, the API requires you to specify a callback
function or method. For example, let's look at _set_pin_mode_digital_input_.

```aiignore
def set_pin_mode_digital_input(self, pin_number, callback=None):
    """
    Set a pin as a digital input.

    :param pin_number: arduino pin number

    :param callback: callback function


    callback returns a data list:

    [pin_type, pin_number, pin_value, raw_time_stamp]

    The pin_type for all digital input pins = 2

    """
```
When a Telemetrix server detects a change in state for an input pin, it generates a
report with the state change information for that pin.



To process report messages received from the Telemetrix server, a Telemetrix client
application employs a callback mechanism to process the data contained in the report 
message.

#### What Is A Callback





A Telemetrix server transmits information asynchronously to a Telemetrix client via
report messages. To process the information for these messages, you
must provide a callback function. The callback function is registered
typically when a pin mode established.

For example, when setting a pin as a digital input, the callback function
is registered as part of the call to 
[set_pin_mode_digital_input.](https://mryslab.github.io/telemetrix-uno-r4/telemetrix_minima_reference/#telemetrix_uno_r4_minima.TelemetrixUnoR4Minima.set_pin_mode_digital_input)

The callback data provided by the report is a list described in the API when 
registering the callback.

```aiignore
def set_pin_mode_digital_input(self, pin_number, callback=None):
        """
        Set a pin as a digital input.

        :param pin_number: arduino pin number

        :param callback: callback function


        callback returns a data list:

        [pin_type, pin_number, pin_value, raw_time_stamp]

        The pin_type for all digital input pins = 2

        """
```
The report callback data contains the pin type being reported, the pin number,
the reported changed pin value, and a timestamp for the report occurence.


[An example](https://github.com/MrYsLab/telemetrix-uno-r4/blob/master/telemetrix_uno_r4/r4_minima_examples/threaded/mst_digital_input.py) 
illustrates the use of callback.

Here is the callback code within the example:

```aiignore
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
```
The example provides for input "debouncing" often needed when monitoring a device
such as a switch.

#### Callbacks Versus Polling

So why have the perceived added complexity of having to register and write a callback
function? Why not just request the current value of the pin?

The way Telemetrix is designed, once a pin is established as an input pin, the 
server automatically monitors the pin, and only alerts the client when the state of the
pin changes.

Requesting a direct read or polling of a pin takes the time to build a poll request
to be sent to the server. There is also the transmission time for the message to 
arrive at the server, the time for the server to process the message, and build reply
message. Finally, there is the time for the message to arrive back to the client and the
client to process that message.

It is therefore possible to miss a data change that occured during the process described
above. By using a callback, you only receive information when a change happens and you 
need
not be wasting processor cycles used for polling.


### Concurrency

Concurrency refers to the ability of a system to execute 
multiple tasks through simultaneous execution or time-sharing.

The Telemetrix API can send commands to the server and, 
at the same time, receive and process report data from the 
server in a non-blocking fashion.

Telemetrix uses Python threaded concurrency or Python asyncio concurrency depending
upon the client API chosen.
#### Threaded Concurrency


To accomplish this Telemetrix employs Python threads in conjunction with a Python
[deque](https://docs.python.org/3/library/collections.html#collections).

![](../assets/images/deque.png)

Writing threading code that shares a buffer between multiple 
threads can be complex. It usually employs thread locks and/or 
semaphores to make sure the buffer is "thread safe" so that when 
one thread is reading the data, it is not overwritten by another 
thread, which would result in data corruption.

A deque can be used as a shared buffer. It has the advantage
of being a thread-safe data structure. This means that we can 
safely add data to one end of the deque using one thread and 
remove data from the opposite end using another thread, all
without the complexity of locks and semaphores.

A Telemetrix application runs in three threads.
The main thread receives API calls from the application, then forms and sends 
command messages to the server.

A second thread, the serial_receiver thread, polls the transport 
interface to determine if report data is available from the server.

```aiignore
    def _serial_receiver(self):
        """
        Thread to continuously check for incoming data.
        When a byte comes in, place it onto the deque.
        """
        self.run_event.wait()

        # Don't start this thread if using a tcp/ip transport

        while self._is_running() and not self.shutdown_flag:
            # we can get an OSError: [Errno9] Bad file descriptor when shutting down
            # just ignore it
            try:
                if self.serial_port.inWaiting():
                    c = self.serial_port.read()
                    self.the_deque.append(ord(c))
                    # print(ord(c))
                else:
                    time.sleep(self.sleep_tune)
                    # continue
            except OSError:
                pass
```
It simply
retrieves the next byte from the transport interface. It appends that byte  
into a Python deque
without any further processing.

A third thread, the reporter thread, "pops" bytes off 
the deque to retrieve an entire report message and then processes it.

```aiignore
    def _reporter(self):
        """
        This is the reporter thread. It continuously pulls data from
        the deque. When a full message is detected, that message is
        processed.
        """
        self.run_event.wait()

        while self._is_running() and not self.shutdown_flag:
            if len(self.the_deque):
                # response_data will be populated with the received data for the report
                response_data = []
                packet_length = self.the_deque.popleft()
                if packet_length:
                    # get all the data for the report and place it into response_data
                    for i in range(packet_length):
                        while not len(self.the_deque):
                            time.sleep(self.sleep_tune)
                        data = self.the_deque.popleft()
                        response_data.append(data)

                    # print(f'response_data {response_data}')

                    # get the report type and look up its dispatch method
                    # here we pop the report type off of response_data
                    report_type = response_data.pop(0)
                    # print(f' reported type {report_type}')

                    # retrieve the report handler from the dispatch table
                    dispatch_entry = self.report_dispatch.get(report_type)

                    # if there is additional data for the report,
                    # it will be contained in response_data
                    # noinspection PyArgumentList
                    dispatch_entry(response_data)
                    continue
                else:
                    if self.shutdown_on_exception:
                        self.shutdown()
                    raise RuntimeError(
                        'A report with a packet length of zero was received.')
            else:
                time.sleep(self.sleep_tune)
```

Having the ability to gather data from the data transport as fast as possible,
while simultaneously being able to assemble the data as report messages gives
the best possible performance available.

The threading mechanisms are built into a Telemetrix API, and the developer
does not need to be concerned with them, either when creating an application
or extending Telemetrix.

#### Asyncio Concurrency

Every Telemetrix library contains two different APIs. If the name of the library
ends in **_-aio_**, that library uses Python asyncio to accomplish concurrency.

Typically, using asyncio results in better performance, but at the price of having
to understand the complexities of asyncio. I have created many asyncio Telemetrix
applications. 

Having the choice of using threading or asyncio gives the developer the 
flexibility to choose what works best for their application.


### Server Report Callbacks

Let's now take a look at the code that implements the Python Telemetrix API
for The Arduino UNO R4 Minima.

The API is implemented within two files. 

APIs for all Telemetrix supported microprocessors are very similar in nature.

A Telemetrix API consists of two files each containing a class definition. One that 
contains the 
[class](https://github.com/MrYsLab/telemetrix-uno-r4/blob/master/telemetrix_uno_r4/minima/telemetrix_uno_r4_minima/telemetrix_uno_r4_minima.py) that implements
the API. The other is a supporting file containing a set of "constants"
utilized by the class.





