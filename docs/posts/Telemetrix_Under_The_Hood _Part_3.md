---
draft: false
date: 2025-05-22
categories:
  - Telemetrix Internals
comments: true
---

![](../assets/images/server.png)

# Understanding The Telemetrix Server

## Introduction

Let's examine the 
[Telemetrix server code](https://github.com/MrYsLab/Telemetrix4UnoR4/blob/master/examples/Minima/Minima.ino) 
for the Arduino UNO R4 Minima,


Why select the server for the Minima and not another board?

The Minima is one of the newer boards in the Arduino family, 
so we chose it to highlight a Telemetrix server design.
However, all Telemetrix servers are remarkably similar. 
The discussion below would be almost identical if any other server were chosen.
Once you learn about one Telemetrix server, you are fully prepared to understand them all.
This post will examine the file's structure, major data structures, and internal workings.



### A Feature Reference Point

![](../assets/images/HC-SR04.jpeg)

One main reason for understanding the internal workings of a Telemetrix 
server is to be able to add support for a new sensor or actuator.

To this end, the HC-SR04 SONAR distance sensor illustrates the 
areas of server code affected by adding sensor support. Adding actuator support
is very similar to adding sensor support. 

Why the HC-SR04? Because it highlights some of the finer points
of adding a new feature, such as:

* Device instantiation.
* Continuous non-blocking timed polling of the device.
* Generating reports for data changes.

All these things will be uncovered as we proceed with the discussion.
Search for the heading _**SONAR SIDEBAR**_ to make the HC-SR04-specific 
discussions easier to find within the post.


<!-- more -->

### Using An Established Arduino Library For Device Support

When adding support for a new device, you have the choice to create your own 
Arduino library or use an existing library. Using an 
existing support library has many advantages. 
You can evaluate the code to ensure it does not 
block for long periods and provides all the desired functionality.

#### _SONAR SIDEBAR_

The [NewPing](https://bitbucket.org/teckel12/arduino-new-ping/wiki/Home) 
library was chosen to support the HC-SR04 SONAR feature. 
It is mostly non-blocking and has a straightforward API, 
making its integration into Telemetrix fairly simple.

Let's take a look at a basic example provided by NewPing.

```aiignore
// ---------------------------------------------------------------------------
// Example NewPing library sketch that does a ping about 20 times per second.
// ---------------------------------------------------------------------------

#include <NewPing.h>

#define TRIGGER_PIN  12  // Arduino pin tied to trigger pin on the ultrasonic sensor.
#define ECHO_PIN     11  // Arduino pin tied to echo pin on the ultrasonic sensor.
#define MAX_DISTANCE 200 // Maximum distance we want to ping for (in centimeters). Maximum sensor distance is rated at 400-500cm.

NewPing sonar(TRIGGER_PIN, ECHO_PIN, MAX_DISTANCE); // NewPing setup of pins and maximum distance.

void setup() {
  Serial.begin(115200); // Open serial monitor at 115200 baud to see ping results.
}

void loop() {
  delay(50);                     // Wait 50ms between pings (about 20 pings/sec). 29ms should be the shortest delay between pings.
  Serial.print("Ping: ");
  Serial.print(sonar.ping_cm()); // Send ping, get distance in cm and print result (0 = outside set distance range)
  Serial.println("cm");
}
```
Essentially, an instance of NewPing is created for each SONAR sensor. 
Then, a NewPing method, **_ping_cm()_**, reads the sensor.

For HC-SR04 sensors, a read may not be performed more than 
once every 29 milliseconds to get accurate reads. 
That is the reason for the 50-millisecond delay in the example above.

Telemetrix provides a non-blocking scheme to read the sensor continuously. 
This scheme will be covered in the
[Scanning Inputs](#scanning-inputs-generating-reports-and-running-steppers)
section of this document.



## Telemetrix Server File Layout

All Telemetrix servers use a very similar file layout. 
We will use the server built for the 
[Arduino UNO R4 Minima](https://github.com/MrYsLab/Telemetrix4UnoR4/blob/master/examples/Minima/Minima.ino)
for discussion purposes.

The server code is considered "fixed" because it is 
uploaded to the microcontroller and left unchanged. 

To change the behavior of the Arduino or other microcontroller, the 
Telemetrix client sends messages to the server. 
The server is implemented to wait for and interpret
commands from the client and then act upon them.
A command may result in continuously monitoring 
a pin or device for data changes. When a data change is detected, 
the server may form a report message and autonomously send the report to the client.

We will cover the client in the next post.

Let's explore the code.

## Telemetrix Server Code Sections


All Telemetrix servers are [implemented using the following "common" sections](https://github.com/MrYsLab/Telemetrix4UnoR4/blob/3629992d2c64da9b76eb5771d4c8933678149924/examples/Minima/Minima.ino#L19):

1. [Feature Enabling Defines](#feature-enabling-defines)
2. [Arduino ID](#arduino-id)
3. [Client Command Related Defines and Support](#client-command-related-defines-and-support)
4. [Server Report Related Defines](#server-report-related-defines)
5. [i2c Related Defines](#i2c-related-defines)
6. [Pin-Related Defines And Data Structures](#pin-related-defines-and-data-structures)
7. [Feature Related Defines, Data Structures, and Storage Allocation](#feature-related-defines-data-structures-and-storage-allocation)
8. [Command Functions](#command-functions)
9. [Scanning Inputs, Generating Reports, and Running Steppers](#scanning-inputs-generating-reports-and-running-steppers)
10. [Setup and Loop](#setup-and-loop)

**Both code snippets and links to the code will be used 
when discussing these sections.**

### Feature Enabling Defines

The ability to turn a built-in feature on or off can be handy. 
For example, you may turn off certain features when debugging a modified server.


 
If the new feature causes the server to run out of programming memory space, 
you can control
the server's footprint by removing support for unneeded features.

A typical set of server features include:

* Support SPI device communication.
* Support for i2c device communication.
* Support for 1-Wire device communication.
* Support for servo motors.
* Support for stepper motors.
* Support for HC-SR04 type ultrasonic distance sensors.
* Support for DHT-type temperature/humidity sensors.

Let's look at the [feature-enabling #defines](https://github.com/MrYsLab/Telemetrix4UnoR4/blob/3629992d2c64da9b76eb5771d4c8933678149924/examples/Minima/Minima.ino#L35).

```aiignore
/* %%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%*/
/*                    FEATURE ENABLING DEFINES                      */
/* %%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%*/



// To disable a feature, comment out the desired enabling define or defines

// This will allow SPI support to be compiled into the sketch.
// Comment this out to save sketch space for the UNO
#define SPI_ENABLED 1

// This will allow OneWire support to be compiled into the sketch.
// Comment this out to save sketch space for the UNO
#define ONE_WIRE_ENABLED 1

// This will allow DHT support to be compiled into the sketch.
// Comment this out to save sketch space for the UNO
#define DHT_ENABLED 1

// This will allow sonar support to be compiled into the sketch.
// Comment this out to save sketch space for the UNO
#define SONAR_ENABLED 1

// This will allow servo support to be compiled into the sketch.
// Comment this out to save sketch space for the UNO
#define SERVO_ENABLED 1

// This will allow stepper support to be compiled into the sketch.
// Comment this out to save sketch space for the UNO

// Accelstepper is currently not compatible with the UNO R4 Minima
// #define STEPPERS_ENABLED 1

// This will allow I2C support to be compiled into the sketch.
// Comment this out to save sketch space for the UNO
#define I2C_ENABLED 1

```
All the features are enabled for the UNO R4 Minima except stepper
motor support. The #define for this feature is commented out because 
the AccelStepper library used to implement this feature does not 
yet function with Arduino UNO R4 boards.

We will implement the feature in a future article
using a different stepper motor library. 



Note that when a feature is activated, the 
[required header files](https://github.com/MrYsLab/Telemetrix4UnoR4/blob/3629992d2c64da9b76eb5771d4c8933678149924/examples
/Minima/Minima.ino#L74) are #included.

```aiignore
#ifdef SERVO_ENABLED
#include <Servo.h>
#endif

#ifdef SONAR_ENABLED
#include <NewPing.h>
#endif

#ifdef I2C_ENABLED
#include <Wire.h>
#endif

#ifdef DHT_ENABLED
#include <DHTStable.h>
#endif

#ifdef SPI_ENABLED
#include <SPI.h>
#endif

#ifdef ONE_WIRE_ENABLED
#include <OneWire.h>
#endif

#ifdef STEPPERS_ENABLED
#include <AccelStepper.h>
#endif

```

#### _SONAR SIDEBAR_
For the **HC-SR04**, the feature is enabled with:

```aiignore
// This will allow sonar support to be compiled into the sketch.
// Comment this out to save sketch space for the UNO
#define SONAR_ENABLED 1
```

With the **SONAR** feature enabled, the server includes the
[NewPing](https://bitbucket.org/teckel12/arduino-new-ping/wiki/Home) Arduino library
for feature support.

```aiignore
#ifdef SONAR_ENABLED
#include <NewPing.h>
#endif
```

When adding a new feature, you must add the new feature #define and use the #define to 
include any required support libraries.

### Arduino ID
For microcontrollers that use a serial/USB data transport, 
Telemetrix defaults to using an auto-discovery scheme to find the 
COM port to which the microcontroller is connected.

The Arduino ID aids in automatic microcontroller discovery and connection.

Optionally, you may also manually specify the COM port in your application. 
However, this is not always effective because the operating system's COM port 
assignment may change dynamically from run to run.


```aiignore
/* %%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%*/
/*                    Arduino ID                      */
/* %%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%*/

// This value must be the same as specified when instantiating the
// telemetrix client. The client defaults to a value of 1.
// This value is used for the client to auto-discover and to
// connect to a specific board regardless of the current com port
// it is currently connected to.

#define ARDUINO_ID 1
```

### Client Command Related Defines and Support

The [Arduino UNO R4 Minima](https://github.com/MrYsLab/Telemetrix4UnoR4/blob/3629992d2c64da9b76eb5771d4c8933678149924/examples/Minima/Minima.ino#L119)
has 58 client commands defined. 
Each command that the server supports has a command ID.

The command IDs must match those mirrored in the Python client API code.

```aiignore

/* %%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%*/
/*         Client Command Related Defines and Support               */
/* %%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%*/

// Commands Sent By The Client


// Add commands retaining the sequential numbering.
// The order of commands here must be maintained in the command_table.
#define SERIAL_LOOP_BACK 0
#define SET_PIN_MODE 1
#define DIGITAL_WRITE 2
#define ANALOG_WRITE 3
#define MODIFY_REPORTING 4 // mode(all, analog, or digital), pin, enable or disable
#define GET_FIRMWARE_VERSION 5
#define ARE_U_THERE 6
#define SERVO_ATTACH 7
#define SERVO_WRITE 8
#define SERVO_DETACH 9
#define I2C_BEGIN 10
#define I2C_READ 11
#define I2C_WRITE 12
#define SONAR_NEW 13
#define DHT_NEW 14
#define STOP_ALL_REPORTS 15
#define SET_ANALOG_SCANNING_INTERVAL 16
#define ENABLE_ALL_REPORTS 17
#define RESET 18
#define SPI_INIT 19
#define SPI_WRITE_BLOCKING 20
#define SPI_READ_BLOCKING 21
#define SPI_SET_FORMAT 22
#define SPI_CS_CONTROL 23
#define ONE_WIRE_INIT 24
#define ONE_WIRE_RESET 25
#define ONE_WIRE_SELECT 26
#define ONE_WIRE_SKIP 27
#define ONE_WIRE_WRITE 28
#define ONE_WIRE_READ 29
#define ONE_WIRE_RESET_SEARCH 30
#define ONE_WIRE_SEARCH 31
#define ONE_WIRE_CRC8 32
#define SET_PIN_MODE_STEPPER 33
#define STEPPER_MOVE_TO 34
#define STEPPER_MOVE 35
#define STEPPER_RUN 36
#define STEPPER_RUN_SPEED 37
#define STEPPER_SET_MAX_SPEED 38
#define STEPPER_SET_ACCELERATION 39
#define STEPPER_SET_SPEED 40
#define STEPPER_SET_CURRENT_POSITION 41
#define STEPPER_RUN_SPEED_TO_POSITION 42
#define STEPPER_STOP 43
#define STEPPER_DISABLE_OUTPUTS 44
#define STEPPER_ENABLE_OUTPUTS 45
#define STEPPER_SET_MINIMUM_PULSE_WIDTH 46
#define STEPPER_SET_ENABLE_PIN 47
#define STEPPER_SET_3_PINS_INVERTED 48
#define STEPPER_SET_4_PINS_INVERTED 49
#define STEPPER_IS_RUNNING 50
#define STEPPER_GET_CURRENT_POSITION 51
#define STEPPER_GET_DISTANCE_TO_GO 52
#define STEPPER_GET_TARGET_POSITION 53
#define GET_FEATURES 54
#define SONAR_SCAN_OFF 55
#define SONAR_SCAN_ON 56
#define BOARD_HARD_RESET 57
```

Each command ID has an associated command handler to process the command. 
Each command handler is initially specified using [forward referencing](https://github.com/MrYsLab/Telemetrix4UnoR4/blob/3629992d2c64da9b76eb5771d4c8933678149924/examples/Minima/Minima.ino#L188)
to simplify compilation. 


```
/* Command Forward References*/

// If you add a new command, you must add the command handler
// here as well.

extern void serial_loopback();

extern void set_pin_mode();

extern void digital_write();

extern void analog_write();

extern void modify_reporting();

extern void get_firmware_version();

extern void are_you_there();

extern void servo_attach();

extern void servo_write();

extern void servo_detach();

extern void i2c_begin();

extern void i2c_read();

extern void i2c_write();

extern void sonar_new();

extern void dht_new();

extern void stop_all_reports();

extern void set_analog_scanning_interval();

extern void enable_all_reports();

extern void reset_data();

extern void init_pin_structures();

extern void init_spi();

extern void write_blocking_spi();

extern void read_blocking_spi();

extern void set_format_spi();

extern void spi_cs_control();

extern void onewire_init();

extern void onewire_reset();

extern void onewire_select();

extern void onewire_skip();

extern void onewire_write();

extern void onewire_read();

extern void onewire_reset_search();

extern void onewire_search();

extern void onewire_crc8();

extern void set_pin_mode_stepper();

extern void stepper_move_to();

extern void stepper_move();

extern void stepper_run();

extern void stepper_run_speed();

extern void stepper_set_max_speed();

extern void stepper_set_acceleration();

extern void stepper_set_speed();

extern void stepper_get_distance_to_go();

extern void stepper_get_target_position();

extern void stepper_get_current_position();

extern void stepper_set_current_position();

extern void stepper_run_speed_to_position();

extern void stepper_stop();

extern void stepper_disable_outputs();

extern void stepper_enable_outputs();

extern void stepper_set_minimum_pulse_width();

extern void stepper_set_3_pins_inverted();

extern void stepper_set_4_pins_inverted();

extern void stepper_set_enable_pin();

extern void stepper_is_running();

extern void get_features();

extern void sonar_disable();

extern void sonar_enable();

extern void board_hard_reset();
```

The actual handlers are defined further down the 
file. 







**_IMPORTANT NOTE:_**

**The command IDs are used as an index to reference a command in the command 
table array. Therefore, when adding a
new command, add a new ID at the bottom of the command defines.**

The command IDs are mirrored in the Telemetrix client Python API.

#### The Command Table

We must add each new command to the command table.
```aiignore
// When adding a new command update the command_table.
// The command length is the number of bytes that follow
// the command byte itself, and does not include the command
// byte in its length.

// The command_func is a pointer the command's function.
struct command_descriptor
{
    // a pointer to the command processing function
    void (*command_func)(void);
};


// An array of pointers to the command functions.
// The list must be in the same order as the command defines.

command_descriptor command_table[] =
        {
                {&serial_loopback},
                {&set_pin_mode},
                {&digital_write},
                {&analog_write},
                {&modify_reporting},
                {&get_firmware_version},
                {&are_you_there},
                {&servo_attach},
                {&servo_write},
                {&servo_detach},
                {&i2c_begin},
                {&i2c_read},
                {&i2c_write},
                {&sonar_new},
                {&dht_new},
                {&stop_all_reports},
                {&set_analog_scanning_interval},
                {&enable_all_reports},
                {&reset_data},
                {&init_spi},
                {&write_blocking_spi},
                {&read_blocking_spi},
                {&set_format_spi},
                {&spi_cs_control},
                {&onewire_init},
                {&onewire_reset},
                {&onewire_select},
                {&onewire_skip},
                {&onewire_write},
                {&onewire_read},
                {&onewire_reset_search},
                {&onewire_search},
                {&onewire_crc8},
                {&set_pin_mode_stepper},
                {&stepper_move_to},
                {&stepper_move},
                {&stepper_run},
                {&stepper_run_speed},
                {&stepper_set_max_speed},
                {&stepper_set_acceleration},
                {&stepper_set_speed},
                (&stepper_set_current_position),
                (&stepper_run_speed_to_position),
                (&stepper_stop),
                (&stepper_disable_outputs),
                (&stepper_enable_outputs),
                (&stepper_set_minimum_pulse_width),
                (&stepper_set_enable_pin),
                (&stepper_set_3_pins_inverted),
                (&stepper_set_4_pins_inverted),
                (&stepper_is_running),
                (&stepper_get_current_position),
                {&stepper_get_distance_to_go},
                (&stepper_get_target_position),
                (&get_features),
                (&sonar_disable),
                (&sonar_enable),
                (&board_hard_reset),
        };


```

#### _SONAR SIDEBAR_

Three methods support the SONAR feature: **_sonar_new_**, **_sonar_disable_**, and **_sonar_enable_**.
The [Command Functions](#command-functions) section of this post will discuss these.


### Server Report Related Defines

A _server report_ transmits information, such as an input value change or a reply 
to a client informational request.

Each report contains a report ID. These IDs are defined
[here](https://github.com/MrYsLab/Telemetrix4UnoR4/blob/3629992d2c64da9b76eb5771d4c8933678149924/examples/Minima/Minima.ino#L398).

When adding an ID for a new report, add it before DEBUG_PRINT_REPORT. 

```aiignore
// Reports sent to the client

#define DIGITAL_REPORT DIGITAL_WRITE
#define ANALOG_REPORT ANALOG_WRITE
#define FIRMWARE_REPORT 5
#define I_AM_HERE 6
#define SERVO_UNAVAILABLE 7
#define I2C_TOO_FEW_BYTES_RCVD 8
#define I2C_TOO_MANY_BYTES_RCVD 9
#define I2C_READ_REPORT 10
#define SONAR_DISTANCE 11
#define DHT_REPORT 12
#define SPI_REPORT 13
#define ONE_WIRE_REPORT 14
#define STEPPER_DISTANCE_TO_GO 15
#define STEPPER_TARGET_POSITION 16
#define STEPPER_CURRENT_POSITION 17
#define STEPPER_RUNNING_REPORT 18
#define STEPPER_RUN_COMPLETE_REPORT 19
#define FEATURES 20
#define DEBUG_PRINT 99
```

For example, if we want to create a new report called NEW_REPORT, we would
add it after the FEATURES report, and NEW_REPORT would be assigned an ID of 21.

```aiignore
#define STEPPER_RUNNING_REPORT 18
#define STEPPER_RUN_COMPLETE_REPORT 19
#define FEATURES 20
#define NEW_REPORT 21  // The newly added report
#define DEBUG_PRINT 99
```

Like command IDs, report IDs are mirrored in the Telemetrix Python API.

### I2C Related Defines
This [section](https://github.com/MrYsLab/Telemetrix4UnoR4/blob/3629992d2c64da9b76eb5771d4c8933678149924/examples/Minima/Minima.ino#L485) 
contains #defines used for managing i2c ports. It specifies 
the i2c SDA and SCL pins.

Note that the Arduino UNO R4 Minima has only one I2C port. Its one 
I2C bus is marked with SCL and SDA. It is shared with A4 (SDA) 
and A5 (SCL), which owners of previous UNOs are familiar with.

The #defines for a second I2C port are included for possible future development, but 
are not supported by the current version of the Arduino UNO R4 Minima.

```aiignore
/* %%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%*/
/*                     i2c Related Defines*/
/* %%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%*/

/**********************************/
/* i2c defines */

#ifdef I2C_ENABLED
// uncomment out the next line to create a 2nd i2c port
// #define SECOND_I2C_PORT

#ifdef SECOND_I2C_PORT
// Change the pins to match SDA and SCL for your board
#define SECOND_I2C_PORT_SDA PB3
#define SECOND_I2C_PORT_SCL PB10

TwoWire Wire2(SECOND_I2C_PORT_SDA, SECOND_I2C_PORT_SCL);
#endif

// a pointer to an active TwoWire object
TwoWire *current_i2c_port;
#endif
```

### Pin Related Defines And Data Structures
This [section](https://github.com/MrYsLab/Telemetrix4UnoR4/blob/3629992d2c64da9b76eb5771d4c8933678149924/examples/Minima/Minima.ino#L509)
contains the definitions for arrays of pin descriptors for analog and 
digital pins.

Each pin descriptor contains information such as the pin number, if its mode is
input or output, whether the pin is enabled to generate a report, and the last
value reported.

```aiignore
/* %%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%*/
/*           Pin Related Defines And Data Structures                */
/* %%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%*/


// Pin mode definitions

// INPUT defined in Arduino.h = 0
// OUTPUT defined in Arduino.h = 1
// INPUT_PULLUP defined in Arduino.h = 2
// The following are defined for arduino_telemetrix (AT)
#define AT_ANALOG 3
#define AT_MODE_NOT_SET 255

// maximum number of pins supported
#define MAX_DIGITAL_PINS_SUPPORTED 14
#define MAX_ANALOG_PINS_SUPPORTED 6


// Analog input pins are defined from
// A0 - A5.


// To translate a pin number from an integer value to its analog pin number
// equivalent, this array is used to look up the value to use for the pin.
int analog_read_pins[6] = {A0, A1, A2, A3, A4, A5};

// a descriptor for digital pins
struct pin_descriptor
{
    byte pin_number;
    byte pin_mode;
    bool reporting_enabled; // If true, then send reports if an input pin
    int last_value;         // Last value read for input mode
};

// an array of digital_pin_descriptors
pin_descriptor the_digital_pins[MAX_DIGITAL_PINS_SUPPORTED];

// a descriptor for analog pins
struct analog_pin_descriptor
{
    byte pin_number;
    byte pin_mode;
    bool reporting_enabled; // If true, then send reports if an input pin
    int last_value;         // Last value read for input mode
    int differential;       // difference between current and last value needed
                            // to generate a report
};

// an array of analog_pin_descriptors
analog_pin_descriptor the_analog_pins[MAX_ANALOG_PINS_SUPPORTED];

unsigned long current_millis;  // for analog input loop
unsigned long previous_millis; // for analog input loop
uint8_t analog_sampling_interval = 19; // in milliseconds

```
### Feature Related Defines, Data Structures, And Storage Allocation

This [section](https://github.com/MrYsLab/Telemetrix4UnoR4/blob/3629992d2c64da9b76eb5771d4c8933678149924/examples/Minima/Minima.ino#L567) contains the code to manage features such as 
servos, DHT temperature/humidity devices, sonar distance sensors, 
stepper motors, and onewire devices.


```aiignore
/* %%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%*/
/*  Feature Related Defines, Data Structures and Storage Allocation */
/* %%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%*/

// servo management
#ifdef SERVO_ENABLED
Servo servos[MAX_SERVOS];

// this array allows us to retrieve the servo object
// associated with a specific pin number
byte pin_to_servo_index_map[MAX_SERVOS];
#endif

// HC-SR04 Sonar Management
#define MAX_SONARS 6

#ifdef SONAR_ENABLED
struct Sonar
{
    uint8_t trigger_pin;
    unsigned int last_value;
    NewPing *usonic;
};

// an array of sonar objects
Sonar sonars[MAX_SONARS];

byte sonars_index = 0; // index into sonars struct

// used for scanning the sonar devices.
byte last_sonar_visited = 0;
#endif //SONAR_ENABLED

unsigned long sonar_current_millis;  // for analog input loop
unsigned long sonar_previous_millis; // for analog input loop

#ifdef SONAR_ENABLED
uint8_t sonar_scan_interval = 33;    // Milliseconds between sensor pings
// (29ms is about the min to avoid = 19;
#endif

// DHT Management
#define MAX_DHTS 6                // max number of devices
#define READ_FAILED_IN_SCANNER 0  // read request failed when scanning
#define READ_IN_FAILED_IN_SETUP 1 // read request failed when initially setting up

#ifdef DHT_ENABLED
struct DHT
{
    uint8_t pin;
    uint8_t dht_type;
    unsigned int last_value;
    DHTStable *dht_sensor;
};

// an array of dht objects
DHT dhts[MAX_DHTS];

byte dht_index = 0; // index into dht struct

unsigned long dht_current_millis;      // for analog input loop
unsigned long dht_previous_millis;     // for analog input loop
unsigned int dht_scan_interval = 2000; // scan dht's every 2 seconds
#endif // DHT_ENABLED


/* OneWire Object*/

// a pointer to a OneWire object
#ifdef ONE_WIRE_ENABLED
OneWire *ow = NULL;
#endif

#define MAX_NUMBER_OF_STEPPERS 4

// stepper motor data
#ifdef STEPPERS_ENABLED
AccelStepper *steppers[MAX_NUMBER_OF_STEPPERS];

// stepper run modes
uint8_t stepper_run_modes[MAX_NUMBER_OF_STEPPERS];
#endif

### 
```

#### _SONAR SIDEBAR_
Specific to HC-SR04 management is this section:

```aiignore
// HC-SR04 Sonar Management
#define MAX_SONARS 6

#ifdef SONAR_ENABLED
struct Sonar
{
    uint8_t trigger_pin;
    unsigned int last_value;
    NewPing *usonic;
};

// an array of sonar objects
Sonar sonars[MAX_SONARS];

byte sonars_index = 0; // index into sonars struct

// used for scanning the sonar devices.
byte last_sonar_visited = 0;
#endif //SONAR_ENABLED

#ifdef SONAR_ENABLED
uint8_t sonar_scan_interval = 33;    // Milliseconds between sensor pings
// (29ms is about the min to avoid = 19;
#endif
```

The server supports up to 6 sensors. Each sensor has an entry 
into an array that stores its trigger pin, the last value read, 
and a pointer to its associated NewPing instance.

The scan interval is set to 33 milliseconds for accurate distance measurement.

### Command Functions

This [section of code](https://github.com/MrYsLab/Telemetrix4UnoR4/blob/3629992d2c64da9b76eb5771d4c8933678149924/examples/Minima/Minima.ino#L651) 
contains the implementations of the command handlers.

Each handler dereferences the data specific to each command.

Let's look at the handler for a digital write.

```aiignore
void digital_write()
{
    byte pin;
    byte value;
    pin = command_buffer[0];
    value = command_buffer[1];
    digitalWrite(pin, value);
}
```
The command buffer contains the information sent from the client, including the pin 
number and the requested value for that pin.

The Arduino Core digital write command, digitalWrite,  is called using the
information sent from the client.

### Scanning Inputs, Generating Reports, And Running Steppers



After setting a pin as input, Telemetrix polls the pin for any changes. It does so for
[digital inputs](https://github.com/MrYsLab/Telemetrix4UnoR4/blob/3629992d2c64da9b76eb5771d4c8933678149924/examples/Minima/Minima.ino#L1751), 
[analog inputs](https://github.com/MrYsLab/Telemetrix4UnoR4/blob/3629992d2c64da9b76eb5771d4c8933678149924/examples/Minima/Minima.ino#L1785),
[HC-SR04 distance sensors](https://github.com/MrYsLab/Telemetrix4UnoR4/blob/3629992d2c64da9b76eb5771d4c8933678149924/examples/Minima/Minima.ino#L1836),
and [DHT temperature sensors](https://github.com/MrYsLab/Telemetrix4UnoR4/blob/3629992d2c64da9b76eb5771d4c8933678149924/examples/Minima/Minima.ino#L1873),
and if required, 
will keep a stepper motor running.

#### _SONAR SIDEBAR_

Let's look at the code for the _sonar_ scanner.

```aiignore
void scan_sonars()
{
#ifdef SONAR_ENABLED
    unsigned int distance;

    if (sonars_index)
    {
        sonar_current_millis = millis();
        if (sonar_current_millis - sonar_previous_millis > sonar_scan_interval)
        {
            sonar_previous_millis = sonar_current_millis;
            distance = sonars[last_sonar_visited].usonic->ping_cm();
            if (distance != sonars[last_sonar_visited].last_value)
            {
                sonars[last_sonar_visited].last_value = distance;

                // byte 0 = packet length
                // byte 1 = report type
                // byte 2 = trigger pin number
                // byte 3 = distance high order byte
                // byte 4 = distance low order byte
                byte report_message[5] = {4, SONAR_DISTANCE, sonars[last_sonar_visited].trigger_pin,
                                          (byte)(distance >> 8), (byte)(distance & 0xff)
                };
                Serial.write(report_message, 5);
            }
            last_sonar_visited++;
            if (last_sonar_visited == sonars_index)
            {
                last_sonar_visited = 0;
            }
        }
    }
#endif
}
```
The _scan_sonars_ function first checks to see if any sensors were enabled by checking 
the _sonars_index_ variable.

Next, it determines whether the scanning interval has elapsed and updates the 
sonar_previous_millis variable to the current time.

It then calls the _ping_cm_ function of the _usonic_ instance associated with the 
sensor to retrieve the distance.

If the current distance read differs from the last distance read, it updates 
the _last_value_ variable to the current distance read, assembles, and sends a 
report message to the client.

Finally, it updates the sonars_index to the next sensor to be scanned.


### Setup and Loop

```aiignore
/* %%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%*/
/*                    Setup And Loop                                */
/* %%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%*/

void setup()
{
    // set up features for enabled features
#ifdef ONE_WIRE_ENABLED
    features |= ONEWIRE_FEATURE;
#endif

#ifdef DHT_ENABLED
    features |= DHT_FEATURE;
#endif

#ifdef STEPPERS_ENABLED
    features |= STEPPERS_FEATURE;
#endif

#ifdef SPI_ENABLED
    features |= SPI_FEATURE;
#endif

#ifdef SERVO_ENABLED
    features |= SERVO_FEATURE;
#endif

#ifdef SONAR_ENABLED
    features |= SONAR_FEATURE;
#endif

#ifdef I2C_ENABLED
    features |= I2C_FEATURE;
#endif

#ifdef STEPPERS_ENABLED

    for ( int i = 0; i < MAX_NUMBER_OF_STEPPERS; i++) {
        stepper_run_modes[i] = STEPPER_STOP ;
    }
#endif

    init_pin_structures();

    Serial.begin(115200);

    pinMode(13, OUTPUT);
    for( int i = 0; i < 4; i++){
        digitalWrite(13, HIGH);
        delay(250);
        digitalWrite(13, LOW);
        delay(250);
    }
}

void loop()
{
    // keep processing incoming commands
    get_next_command();

    if (!stop_reports)
    { // stop reporting
        scan_digital_inputs();
        scan_analog_inputs();

#ifdef SONAR_ENABLED
        if(sonar_reporting_enabled ){
            scan_sonars();
        }
#endif

#ifdef DHT_ENABLED
        scan_dhts();
#endif
#ifdef STEPPERS_ENABLED
        run_steppers();
#endif
    }
}
```
#### Setup

The [setup function](https://github.com/MrYsLab/Telemetrix4UnoR4/blob/3629992d2c64da9b76eb5771d4c8933678149924/examples/Minima/Minima.ino#L2012) checks for all enabled features and stores then stores
the information in the [_features_ byte](https://github.com/MrYsLab/Telemetrix4UnoR4/blob/3629992d2c64da9b76eb5771d4c8933678149924/examples/Minima/Minima.ino#L481).

It builds the [pin structures](https://github.com/MrYsLab/Telemetrix4UnoR4/blob/3629992d2c64da9b76eb5771d4c8933678149924/examples/Minima/Minima.ino#L1724),
starts the serial link, and then flashes the board LED 4 times.

#### Loop

The [loop function](https://github.com/MrYsLab/Telemetrix4UnoR4/blob/3629992d2c64da9b76eb5771d4c8933678149924/examples/Minima/Minima.ino#L2063) checks to see if a client command needs to be processed
by calling [get_next_command](https://github.com/MrYsLab/Telemetrix4UnoR4/blob/3629992d2c64da9b76eb5771d4c8933678149924/examples/Minima/Minima.ino#L1631).

If reporting is enabled, it calls the various scanners and, if necessary, sends the requested commands 
to the configured stepper motors.

## The Next Posting

We've completed the discussion of the Telemetrix server file. 
In the next post, we will discuss the Python API class.

## Summary For Adding New Features

* Identify an existing device library or write your own.
* Verify that the library is mostly non-blocking and meets your needs.
* Add a [feature enabling #define to the server file.](#feature-enabling-defines)
* Add [client command related defines.](#client-command-related-defines-and-support)
* Add [report defines](#server-report-related-defines) if necessary.
* Add feature-specific [data structures and storage allocation.](#feature-related-defines-data-structures-and-storage-allocation)
* Add [command handlers](#command-functions) for each command.
* Add [scanning functions](#scanning-inputs-generating-reports-and-running-steppers) 
  for each feature if required.
* Update [setup and loop functions](#setup-and-loop) to support the feature.