---
layout: post
title:  "LoRa Pulse Counter Demo"
date:   2019-02-05
categories:
permalink: lpc
toc: true
---

![assembled board]({{ "/assets/first_pass_assembled.jpg" | absolute_url }}){: .center-image }

{{ table_of_contents }}

Above is version [0.1.1](#011) of my LoRa Pulse Counter demonstration platform.

I put this together to demonstrate an end-to-end hardware/firmware/software project.
Also I wanted to evaluate [Kicad](http://kicad-pcb.org/) after not making a PCB for almost a decade.

## Application

The LoRa Pulse Counter can be used to count electrical pulses produced by flow meters 
and/or detect level changes from switch based sensors. 
The count and level state is sent to a remote application over LoRaWAN.

This is useful for:

- counting output produced by electricity meters, water meters, and other types of flow meters
- tamper detection
- storage tank monitoring

## Features

### LoRaWAN

The device uses LoRaWAN to send data to a remote application. 

The implementation is my own. Features include:

- low-resource design suitable for constrained devices 
- [API](https://cjhdev.github.io/lora_device_lib_api/) documentation
- OTAA only with full support for all network configurable parameters
- runtime multi-region support
- hardware emulator for debugging against network implementations

### Digital Inputs

The device has two inputs suitable for interfacing with open-collector or
dry-contact outputs. 

### External Pickup Power Supply

The device can supply power to external pickups required to interface
with different equipment. Typical external pickups include:

- photodiode to detect a flashing LED
- hall-effect sensor to detect a rotating magnet

### Status Button and LED

The device has a button which when pressed enables a status LED which
makes it possible to quickly inspect the status of the device.

The status codes are:

number of flashes | status 
------------------|-----------------------------------------------------
0                 | dead
1                 | not-joined
2                 | joining
3                 | joined but no downlink received in interval
4                 | joined and downlink received in interval

### UART Connection

The device has a level-converted UART connection to support:

- commissioning
- firmware update
- fault finding

The connector used is a 6 pin header with the following configuration:

pin | signal    | comment
----|-----------|-------------------------------------------------------
1   | GND       | mandatory  
2   | CTS       | connected to GND
3   | VCC       | mandatory 3V3 or 5V; battery is still required
4   | TXD       | transmit line
5   | RXD       | receive line
6   | DTR       | not connected

This configuration is compatible with USB to serial converters
made by FTDI, Sparkfun, and many others.

### Battery Voltage Sensor

The device is able to sample battery voltage through a switched resistive
divider. This information is made available to the remote application.

### Ambient Temperature Sensor

The device is able to monitor ambient temperature. This information is made available to 
the remote application.

### Field Serviceable Primary Cell

The device is powered by a CR123A (Li-MnO2) primary cell which
is installed in a PCB mounted holder. 

Having the cell easily replaceable is useful for shipping, and for 
allowing the device to be used in applications which require high frequency
reporting.

### Configurable

The following characteristics can be reconfigured:

- input behaviour
    - rising/falling/both edges
    - alerting
- reporting interval and alerting strategy
- link-check interval
- no-downlink-error interval

### Low Power Design

The device is designed for low power consumption:

- direct unregulated power supply
- switched battery voltage divider
- status LED active only when user button is pressed
- UART level converter powered by the USB to serial converter and not by the device
- event driven firmware

Average power consumption depends upon:

- resting state of digital inputs (active-low will produce longest battery life)
- frequency of pulse signals
- frequency of pulse count reporting
- alerting strategy
- link-check strategy

## Hardware Revisions

### 0.2.0

Changes from last revision:

- U.FL replaced with dual SMA/U.FL footprint
- removed DIO4 and DIO5 routing
- using single inverter for UART transmit line
- removed reset button
- using larger footprints for easy soldering
- enlarged M2 mounting holes to M3

![schematic]({{ "/assets/second_pass_design.png" | absolute_url }}){: .center-image }

![3d model]({{ "/assets/second_pass_model.png" | absolute_url }}){: .center-image }

#### Problems

- There should be a top-layer keepout underneath U3; trace isolation is currently dependent on soldermask integrity

### 0.1.1

First revision.

![schematic]({{ "/assets/first_pass_design.png" | absolute_url }}){: .center-image }

![3d model]({{ "/assets/first_pass_model.png" | absolute_url }}){: .center-image }

![assembled board]({{ "/assets/first_pass_assembled.jpg" | absolute_url }}){: .center-image }

#### Problems

Thoughts I had after having this PCB made:

- There should be a keepout underneath the U.FL connector
- The radio module is too far away from the U.FL connector
- UART pin assignment means the STM32L051K8 cannot be swapped for the lower cost STM32L010K8
- The UART level converter power consumption could be improved by using the 
MCU UART signal invert feature
- It is not necessary to connect radio module DIO4 and DIO5 
- The reset switch is redundant
- The footprints are too small for easy soldering
- Double row storage capacitors are difficult to solder by hand

## Communication Protocol

A protocol is required in order to commission and manage the LoRa Pulse Counter.

### UART Settings

- 38400 baud
- 8 data bits, 1 stop bit, no parity

### Framing

The protocol uses start, end, escape, and xor characters to mark start/end of 
messages and to remove start/end markers from the message payload.

Special characters are as follows:

| name   | value |
|--------|-------|
| start  | 0x7e  |
| end    | 0x7f  |
| escape | 0x7d  |
| xor    | 0x20  |

When a start, end, or escape character appears within a message payload, it is replaced
by an escape character followed by the original character XORed with the xor character.

The last two bytes of a frame is a 16 bit CRC ordered most significant byte first.
The CRC is calculated over the payload bytes as they appear on the wire (i.e. including escape characters).

CRC parameters are as follows:

| name          | CRC-16-CCITT |
| polynomial    | 0x1021 |
| initial value | 0xffff |

### Message Encoding Rules

Messages are encoded/decoded according to [Octet Encoding Rules X.696](https://www.itu.int/rec/T-REC-X.696/en).

### Commands

#### Commission

Device shall be commissioned by writing the following values to
non-volatile memory:

- Application EUI
- Application Key
- Device EUI

Following this the device shall forget the network and enter a non-joining
mode.

#### Join

Device shall forget the network and enter a joining mode.

#### Forget

Device shall forget the network and enter a non-joining mode.

#### Get-Device-EUI

Return the Device-EUI.

#### Get-Application-EUI

Return the Application-EUI.

#### Get-Encrypted-Application-Key

Return the result of performing AES-128 encryption of a zeroed block using the Application-Key.

#### Set-Log-Severity-Level

Set the severity level for log messages pushed by Log-Message alert in accordance
with RFC 5424.

### Alerts

#### Log-Alert

Used to push logging messages to a client depending on the severity level
set by the Set-Log-Severity-Level command.

### Message Structure

{% highlight asn1 %}
Pulse-Counter-Protocol
DEFINITIONS ::=
BEGIN
     
    PDU ::= CHOICE 
    {
        command     Command,
        response    Response,
        alert       Alert
    }

    Command ::= SEQUENCE
    {
        type CHOICE 
        {    
            commission              Commission-Command,
            join                    Join-Command,
            forget                  Forget-Command,
            get-deveui              Get-DevEUI-Command,
            get-appeui              Get-AppEUI-Command,
            get-encrypted-appkey    Get-Encrypted-AppKey-Command,
            set-log-severity-level  Set-Log-Severity-Level-Command
        }
    }
    
    Response ::= SEQUENCE
    {
        type CHOICE
        {
            commission              Commission-Response,
            join                    Join-Response,
            forget                  Forget-Response,
            get-deveui              Get-DevEUI-Response,
            get-appeui              Get-AppEUI-Response,
            get-encrypted-appkey    Get-Encrypted-AppKey-Response,
            set-log-severity-level  Set-Log-severity-Level-Response
        }
    }
    
    Alert ::= SEQUENCE
    {
        type CHOICE
        {
            log                     Log-Alert
        }
    }

    # 
    # Commands, Response, Alert
    #

    Commission-Command  ::= SEQUENCE
    {        
        counter Invocation-Counter,
        app-eui EUI,
        dev-eui EUI,
        app-key Key
    }
    Commission-Response ::= Empty-Response
    
    Join-Command    ::= Empty-Command
    Join-Response   ::= Empty-Response
    
    Forget-Command  ::= Empty-Command
    Forget-Response ::= Empty-Response
    
    Get-AppEUI-Command  ::= Empty-Command
    Get-AppEUI-Response ::= SEQUENCE
    {
        counter Invocation-Counter,
        eui     EUI
    }
    
    Get-DevEUI-Command  ::= Empty-Command
    Get-DevEUI-Response ::= SEQUENCE
    {
        counter Invocation-Counter,
        eui     EUI
    }
    
    Get-Encrypted-AppKey-Command    ::= Empty-Command
    Get-Encrypted-AppKey-Response   ::= SEQUENCE
    {
        counter         Invocation-Counter,
        encrypted-key   Key
    }

    Set-Log-Severity-Level-Command ::= SEQUENCE
    {
        counter         Invocation-Counter,
        severity-level  INTEGER (0..7)
    }
    Set-Log-Severity-Level-Response ::= Empty-Response

    Log-Alert ::= VisibleString

    #
    # useful definitions
    #
    
    Empty-Command ::= Empty-Response
    Empty-Response ::= SEQUENCE
    {
        counter Invocation-Counter
    }
    
    Invocation-Counter ::= INTEGER (0..max-uint8)
    EUI ::= OCTET STRING (SIZE(8))
    Key ::= OCTET STRING (SIZE(16))
    
    max-uint32 Integer ::= 4294967295
    max-uint16 Integer ::= 65535
    
END
{% endhighlight %}
