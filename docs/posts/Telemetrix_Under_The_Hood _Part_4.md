---
draft: false
date: 2025-05-15
categories:
  - Telemetrix Internals
comments: true
---

![](../assets/images/api.png){ width="450" }

## The Telemetrix Python Client API

In this post, we will explore the Python files that 
implement the Telemetrix Python API. To help focus the discussion, 
we will use the **HC-SR04 SONAR distance sensor** feature.

However, before getting to the actual code, 
let's explore two important design features that 
are integral to a Telemetrix Python API.


<!-- more -->

## Design Features
### Concurrency

#### Threaded Concurrency

Concurrency refers to the ability of a system to execute 
multiple tasks through simultaneous execution or time-sharing.

The Telemetrix API can send commands to the server and, 
at the same time, receive and process report data from the 
server in a non-blocking fashion.

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





