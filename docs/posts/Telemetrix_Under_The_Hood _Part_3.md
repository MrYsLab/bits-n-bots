---
draft: false
date: 2024-12-12
categories:
  - Telemetrix Internals
comments: true
---

![](../assets/images/under_the_hood.png){ width="450" }


# Telemetrix Server File Layout

All Telemetrix servers use a very similar file layout. Let's begin by exploring the code 
for a typical Telemetrix server. For 
discussion 
purposes, we
will be using the server built for the
[Arduino UNO R4 Minima](https://github.com/MrYsLab/Telemetrix4UnoR4/blob/master/examples/Minima/Minima.ino).

<!-- more -->
The server code is considered "fixed" in that it is 
uploaded to the microcontroller and left unchanged. The user programs the 
microcontroller by writing a Python script constructed 
from a  [Telemetrix Python API](https://mryslab.github.io/telemetrix-uno-r4/telemetrix_minima_reference/).

The server sketch is only modified to add functionality or when 
it is first created to support a new microcontroller.

Now, let's explore the code.

## Telemetrix Server Code Sections


All Telemetrix servers are [implemented using the following common sections](https://github.com/MrYsLab/Telemetrix4UnoR4/blob/3629992d2c64da9b76eb5771d4c8933678149924/examples/Minima/Minima.ino#L19):

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

Disabling a built-in feature can be handy. 
 For example, you may wish to disable certain features when debugging a
modified server.

Or perhaps the server code's current size limits the addition of a new feature. 
You can limit the server's footprint by 
removing support for unneeded features.

A typical set of server features include:

* Support SPI device communication.
* Support for i2c device communication.
* Support for 1-Wire device communication.
* Support for servo motors.
* Support for stepper motors.
* Support for HC-SR04 type ultrasonic distance sensors.
* Support for DHT-type temperature/humidity sensors.

Let's look at the [feature enabling defines](https://github.com/MrYsLab/Telemetrix4UnoR4/blob/3629992d2c64da9b76eb5771d4c8933678149924/examples/Minima/Minima.ino#L35).

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
the AccelStepper library used to implement this feature does not yet function with Arduino UNO R4 boards.

We will implement the feature in a future article
using a different stepper motor library. 

Note that feature defines conditionally 
[include the required header files](https://github.com/MrYsLab/Telemetrix4UnoR4/blob/3629992d2c64da9b76eb5771d4c8933678149924/examples/Minima/Minima.ino#L74) for
the feature support libraries.

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

### Arduino ID
For microcontrollers that use a serial/USB data transport, 
Telemetrix defaults to using an auto-discovery scheme to find the 
COM port to which the microcontroller is connected.

The Arduino ID helps to automatically discover and connect to a 
specific microcontroller.

You may also manually specify the COM port in your application, 
but this is not always effective since the operating system's COM port 
assignment may dynamically change from run to run.


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

For the [For the Arduino UNO R4 Minima](https://github.com/MrYsLab/Telemetrix4UnoR4/blob/3629992d2c64da9b76eb5771d4c8933678149924/examples/Minima/Minima.ino#L119), 58 commands are defined.
A command ID is defined for each command that the server supports.

Each command has an associated function used to process it. 
Each command handler is initially specified using [forward referencing](https://github.com/MrYsLab/Telemetrix4UnoR4/blob/3629992d2c64da9b76eb5771d4c8933678149924/examples/Minima/Minima.ino#L188)
to simplify compilation. The actual handlers are defined further down the 
file. By using forward referencing, a [forward referencing](https://github.com/MrYsLab/Telemetrix4UnoR4/blob/3629992d2c64da9b76eb5771d4c8933678149924/examples/Minima/Minima.ino#L188) consisting of an array 
of pointers to the command functions can be built at the top of the file.


**_IMPORTANT NOTE:_**

**The Command IDs serve as an index into the command table. Therefore, when adding a
new command, add a new ID at the bottom of the command defines.**


### Server Report Related Defines

A server report transmits information, such as an input value change or a reply 
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
add it after the FEATURES report and NEW_REPORT would be assigned an ID of 21.

```aiignore
#define STEPPER_RUNNING_REPORT 18
#define STEPPER_RUN_COMPLETE_REPORT 19
#define FEATURES 20
#define NEW_REPORT 21  // The newly added report
#define DEBUG_PRINT 99
```

### I2C Related Defines
This [section](https://github.com/MrYsLab/Telemetrix4UnoR4/blob/3629992d2c64da9b76eb5771d4c8933678149924/examples/Minima/Minima.ino#L485) 
contains defines used for managing i2c ports. It specifies 
the i2c SDA and SCL pins.

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
contains the definitions for arrays of pin descriptors for both analog and 
digital pins.

Each pin descriptor contains information such as the pin number, if its mode is
input or output, whether the pin is enabled to generate a report, and the last
value reported.

```aiignore
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

```
### Feature Related Defines, Data Structures, And Storage Allocation

This [section of code](https://github.com/MrYsLab/Telemetrix4UnoR4/blob/3629992d2c64da9b76eb5771d4c8933678149924/examples/Minima/Minima.ino#L567)
contains the code to manage features such as servos, DHT temperature/humidity
devices, sonar distance sensors, stepper motors, and onewire devices.

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



After setting a pin as an input pin, Telemetrix polls the pin 
for any changes. It does so for
[digital inputs](https://github.com/MrYsLab/Telemetrix4UnoR4/blob/3629992d2c64da9b76eb5771d4c8933678149924/examples/Minima/Minima.ino#L1751), 
[analog inputs](https://github.com/MrYsLab/Telemetrix4UnoR4/blob/3629992d2c64da9b76eb5771d4c8933678149924/examples/Minima/Minima.ino#L1785),
[HC-SR04 distance sensors](https://github.com/MrYsLab/Telemetrix4UnoR4/blob/3629992d2c64da9b76eb5771d4c8933678149924/examples/Minima/Minima.ino#L1836),
and [DHT temperature sensors](https://github.com/MrYsLab/Telemetrix4UnoR4/blob/3629992d2c64da9b76eb5771d4c8933678149924/examples/Minima/Minima.ino#L1873),
and if required, 
will keep a stepper motor running.

Let's look at the code for the digital input scanner.

```aiignore
void scan_digital_inputs()
{
    byte value;

    // report message

    // byte 0 = packet length
    // byte 1 = report type
    // byte 2 = pin number
    // byte 3 = value
    byte report_message[4] = {3, DIGITAL_REPORT, 0, 0};

    for (int i = 0; i < MAX_DIGITAL_PINS_SUPPORTED; i++)
    {
        if (the_digital_pins[i].pin_mode == INPUT ||
            the_digital_pins[i].pin_mode == INPUT_PULLUP)
        {
            if (the_digital_pins[i].reporting_enabled)
            {
                // if the value changed since last read
                value = (byte)digitalRead(the_digital_pins[i].pin_number);
                if (value != the_digital_pins[i].last_value)
                {
                    the_digital_pins[i].last_value = value;
                    report_message[2] = (byte)i;
                    report_message[3] = value;
                    Serial.write(report_message, 4);
                }
            }
        }
    }
}
```
The scanner loops through all possible digital pins, checking whether a 
pin is configured as a digital input. 
If it is, the scanner next checks to see if 
reporting is enabled for the pin, and if it is, the pin is read.
The value read is compared to the last changed value, 
and if they differ, the new value is stored in the 
[digital pins table](https://github.com/MrYsLab/Telemetrix4UnoR4/blob/3629992d2c64da9b76eb5771d4c8933678149924/examples/Minima/Minima.ino#L546).
Finally, a report message is constructed and sent over the 
serial link to the Python client.
The code is similar to the other scanners.


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

This completes the discussion of the Telemetrix server file. In the next post,
we will discuss the Python API class.