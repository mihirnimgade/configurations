
# Telemetry board (TEL) firmware architecture

## Overview

The minimum required functionality of the telemetry board is described below:

- Transmitting CAN message data over cellular/radio to Solar's telemetry interface
- Transmitting GPS information to Solar's telemetry interface
- Transmitting IMU (inertial measurement unit) data to the telemetry interface

Some optional/experimental features include:

- A sensor fusion algorithm to combine the IMU and GPS data to give us the absolute orientation of the car in 3D space.
- Receiving firmware updates via the radio module.

## Architecture diagram

```
┌────────────────────────────────────────────────────────────────────────────────────+─────────────────────────┐
│                                                                                    +                         │
│                                             ON-BOARD                               +        OFF-BOARD        │
│    ┌────────────┐      ┌─────────────┐                        ┌────────────────┐   +                         │
│    │            │      │             │                        │ * Cellular     │   +                         │
│    │  * GPS     │      │   * IMU     │                        │                │   +                         │
│    │            │      │             │           ┌────────────┤ DOUT           │   +                         │
│    └──┬──────▲──┘      └──┬───────▲──┘           │            │                │   +                         │
│       │      │            │       │              │ ┌──────────► DIN            │   +                         │
│       │ UART │            │  I2C  │              │ │          │                │   +                         │
│       │      │            │       │              │ │ ┌────────► !RESET         │   + ┌─── HTTP Request       │
│    ┌──▼──────┴────────────▼───────┴───────────┐  │ │ │        │                │   + │                       │
│    │  │                   │                   │  │ │ │ ┌──────┤ RSSI PWM       │     ▼ ┌─────────────────┐   │
│    │ ┌▼────────┐     ┌────▼────┐  ┌─────────┐ ◄──┘ │ │ │      │                │~~~~~~>│ * Telemetry     │   │
│    │ │   NMQ   │     │   IMQ   │  │   TMQ   │ │    │ │ │ ┌────► DI8            │~~~~~~>│   interface     │   │
│    │ │         │     │         │  │         ├─►────┘ │ │ │    │                │       │   server        │   │
│    │ └───┬─────┘     └────┬────┘  │         │ │      │ │ │ ┌──► !RTS           │   +   │                 │   │
│    │     │                │       │         │ ├──────┘ │ │ │  │                │   +   │                 │   │
│    │     └────────────────┼───────►         │ │        │ │ │  └────────────────┘   +   └────────^────────┘   │
│    │                      │       │         │ ◄────────┘ │ │                       +            |            │
│    │ ┌┬───────┬┐     ┌┬───▼───┬┐  │         │ │          │ │                       +            +~~~~~~~~~+  │
│    │ ││       ││     ││       ││  │         │ ├──────────┘ │                       +                      |  │
│    │ ││       ││     ││  SFA  ││  │         │ │            │  ┌────────────────┐       ┌───────────────┐  |  │
│    │ ││  GPC  ││     ││       ││  │         │ ├────────────┘  │ * Radio 1      │~~~~~~>│ * Radio 2     │  |  │
│    │ ││       ││     └┴───┬───┴┘  │         │ │               │                │~~~~~~>│               │  |  │
│    │ ││       ││          │       │         │ ◄───────────────┤ DOUT           │       └───────┬───────┘  |  │
│    │ └┴───────┴┘          │       │         │ │               │                │   +           │          |  │
│    │                      │       │         ├─►───────────────► DIN            │   +           │ UART     |  │
│    │ ┌─────────┐     ┌┬───▼───┬┐  └────▲────┘ │               │                │   +           │          |  │
│    │ │   CMQ   │     ││       ││       │      ├───────────────► !RTS           │   +    ┌──────▼──────┐   |  │
│    │ │         ├─────►│ NTAA  │┼───────┘      │               │                │   +    │ * Host PC   │   |  │
│    │ │         │     ││       ││              ├───────────────► RST            │   +    │             │~~~+  │
│    │ └──┬───▲──┘     └┴───────┴┘              │               │                │   +    │             │      │
│    │    │   │                       * STM32   ◄───────────────┤ PWM            │   +    └─────────────┘      │
│    └────┼───┼─────────────────────────────────┘               └────────────────┘   +                         │
│         │   │                                                                      +++++++++++++++++++++++++++
│   CAN Tx│   │CAN Rx                                                                                          │
│         ▼   │                                                                                                │
│  >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>   │
│  <<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<   │
│                                                                                                              │
│                                          Central CAN bus                                                     │
│                                                                                                              │
└──────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

## Architecture description 

### Peripherals

- **GPS**: this peripheral provides us with the location of the car at all time along with various other pieces of information.
- **IMU**: stands for inertial measurement unit. Provides instantaneous accelerations and angular momentum in 3 dimensions.
- **Radio**: uses RF magic to transit information to secondary radio receiever. This will be the main communication method used by the board.  
- **Cellular**: uses RF magic + cellular networks to transmit information directly to the telemetry interface server over the internet using HTTP. Will be used as more of a fail-safe communication method.

### Queues

This a list of the software queues that will be used for the telemetry board firmware. CMSIS-RTOS v2 provides API calls for creating these queues.

- **NMEA message queue (NMQ)**: this queue stores pointers to NMEA message structs. These NMEA messages are produced by the GPS and are transmitted to the microcontroller over UART.
- **IMU message queue (IMQ)**: this queue stores pointers to IMU message structs. These IMU messages need to be built up by reading from the IMU registers since the communication protocol is I2C.
- **CAN message queue (CMQ)**: this queue stores pointers to CAN message structs. These CAN message structs also need to be built up as CAN messages come in.
- **Transmission message queue (TMQ)**: this queue holds all of the processed messages (i.e., they are in a serial character stream) that have been queued up for transmission over radio/cellular.

### Algorithms/methods

**General peripheral control (GPS)**

The main function for this block is to keep track of the status of transmitting peripherals (i.e., radio and cellular) and data peripherals (i.e., GPS and IMU). This includes any initialization work when the board powers up, ensuring the cellular, radio, and GPS modules are receiving a signal, performing test transmissions, etc. 

**Sensor fusion algorithm (SFA)**

This algorithm is responsible for combining the data produced by the IMU and the GPS and calculating the absolute orientation of the car in 3D space. This is by no means essential functionality but I thought it would be a fun little project. Who knows if we can even transmit the orientation data fast enough for it to be useful. 

**Number-to-ASCII algorithm (NTAA)**

Since we have to communicate with the radio and the cellular module over UART, we can only send serial characters to them. We have three different kinds of messsages to send: NMEA messages, CAN messages, and IMU messages. NMEA messages already come in as a serial character stream so they make their way from the NMQ to the TMQ without any additional processing. CAN messages and IMU messages on the other hand need to be converted to a serial character stream and that is what this algorithm is in charge of.

## Message types

### NMEA messages

The general structure of an NMEA message is shown below:

```
$GPZDA,141644.00,22,03,2002,00,00*67<CR><LF>
```

Breaking that down:

```
       talker ID                                 checksum

           │                                         │
           │                                         │
           ▼                                         ▼
        $ GP  ZDA,    141644.00,22,03,2002,00,00    *67  <CR><LF>
        ▲      ▲                  ▲                          ▲
        │      │                  │                          │
        │      │                  │                          │
NMEA ───┘      │                  │                          │
msg            │                  │                          │
identifier     │
               │               data field              carriage return
               │        (delimited with commas)         + line feed

            defines
          msg contents
```

### CAN messages

### IMU messages

