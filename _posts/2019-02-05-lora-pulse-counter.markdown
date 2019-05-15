---
layout: post
title:  "LoRa Pulse Counter Demo"
date:   2019-02-05
categories:
permalink: lpc
toc: true
---

![assembled board]({{ "/assets/lpc_0.2.0.jpg" | absolute_url }}){: .center-image }

{{ table_of_contents }}

Above is version [0.2.0](#020) of the LoRa Pulse Counter demonstration platform. The red and black wires are fitted only
for debug.

This project exists to demonstrate an end-to-end hardware/firmware/software project. It was
also an opportunity to evaluate [Kicad](http://kicad-pcb.org/).

## Application

The device can be used to count electrical pulses produced by 
instruments and/or detect level changes from switch based sensors. 
The count and level state is sent to a remote application over LoRaWAN.

This might seem dull but it is useful for things like:

- retrofitting remote reading to existing electricity and water meters
- tamper detection (e.g. someone opened a door)
- storage tank level monitoring (e.g. float switches)
- interactive demonstrations

## Features

### LoRaWAN

The device uses LoRaWAN to send data to a remote application. 
The stack makes use of over-the-air-activation (OTAA)
and adapative data rate (ADR) to keep setup and operation of the device
as simple as possible.

The implementation used in this project is bespoke and not based
on any of the existing permissively licensed projects.
You can have a peek at the API documentation [here](https://cjhdev.github.io/lora_device_lib_api/).

### Digital Inputs

The device has two inputs suitable for interfacing with open-collector or
dry-contact outputs. 

Each input has a configurable filter which allows the device to work with different
types of outputs (e.g. slow bouncy contacts vs. fast digital switches). 

There are three rise-time settings:

1. 0.250ms (2KHz)
2. 12.5ms (40Hz)
3. 125ms (4Hz)

### External Pickup Power Supply

The device can supply a small amount of power to external pickups 
required to interface with different equipment.

You may, for example, need to power a hall effect sensor in order to detect a 
rotating magnet.

### Status Button and LED

The device has a button which when pressed enables a status LED which
makes it possible to quickly inspect the status of the device.

The status codes are as follows:

number of flashes | status 
------------------|-----------------------------------------------------
0                 | dead
1                 | joined and downlink received in interval
2                 | joined but no downlink received in interval
3                 | joining
4                 | not-joined

### UART Connection

The device has a level-converted UART connection for the purpose of
performing configuration and applying updates. The level converter
is implemented in such a way that it doesn't consume any energy from
the battery.

The connector pin assignment is as follows:

pin | signal    | comment
----|-----------|-------------------------------------------------------
1   | GND       | mandatory  
2   | CTS       | connected to GND
3   | VCC       | mandatory 3V3 or 5V (this powers the level converter)
4   | TXD       | transmit line
5   | RXD       | receive line
6   | DTR       | not connected

This connector is compatible with USB-to-serial converters
made by FTDI, Sparkfun, and many others.

The [UART Communication Protocol](#uart-communication-protocol) runs
over this connection. 

### Battery Voltage Sensor

The device is able to sample battery voltage. This information is made
available to the remote application.

### Ambient Temperature Sensor

The device is able to sample ambient temperature. This information is made available to 
the remote application.

### Field Serviceable Primary Cell

The device is powered by a CR123A (Li-MnO2) primary cell which
is installed in a PCB mounted holder.

The hardware includes reverse polarity protection just in case the cell is
installed backwards.

### Low Power Design

Power consumption (and battery life) depends on three things: hardware implementation,
firmware implementation, and application requirements.

The hardware is designed to produce the lowest possible power consumption during active
and sleep modes. To achieve this the following decisions were made:

- direct unregulated power supply
- UART level converter is powered by
- status LED is active only when the user requests it via the user button
- STM32L051 MCU (using LSE + HSI oscillators)
- 4MHz system clock (HSI/4)

The firmware implementation is event driven and takes advantage of the MCU "wake from stop mode"
features. The MCU will wake from:

- external interrupts
- LPTIM backed application timer
- UART start bit

Average power consumption then depends on the demands of the application:

- resting state of digital inputs (active-low will produce longest battery life)
- frequency of pulse signals
- frequency of pulse count reporting (configurable)
- alerting strategy (configurable)
- link-check strategy (configurable)

### Configurable

While incomplete at this stage, the following characteristics are 
intended to be configurable:

- input filter mode
- input channel mode
    - counter
    - alert
- message formats
    - two counters
    - one counter per message
    - counter(s) plus alert status field
- counter push interval
- alert notification strategy

## LoRaWAN Message Format

LoRaWAN messages are differentiated by port number.

### Device-to-Application

#### Two-Counters (port 1)

[Octet Encoding Rules X.696](https://www.itu.int/rec/T-REC-X.696/en)

{% highlight asn1 %}
Two-Counters ::= SEQUENCE
{
    channel1    INTEGER (0..max-uint32),
    channel2    INTEGER (0..max-uint32)
}

max-uint32 Integer ::= 4294967295
{% endhighlight %}



## UART Communication Protocol

The following protocol allows the device to managed over the UART connection.

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
The CRC is calculated over the payload bytes prior to any escaping.

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
with [RFC 5424](https://tools.ietf.org/html/rfc5424).

#### Get-VBAT-And-Ambient

Return the battery voltage and ambient temperature.

#### Get-Counter-One and Get-Counter-Two

Return the counter register(s).

#### Clear-Counter-One and Clear-Counter-Two

Zero the counter register(s).

#### Goto-Boot

Put device into bootloader mode.

#### Goto-App

Put device into application mode.

#### Boot-Or-App

Indicate whether device is in bootloader or application mode.

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
            get-device-name         Get-Device-Name-Command,
            get-device-name-max-len Get-Device-Name-Max-Len-Command,
            set-device-name         Set-Device-Name-Command,
            set-log-severity-level  Set-Log-Severity-Level-Command,
            get-vbat-and-ambient    Get-VBat-And-Ambient-Command,
            reset                   Reset-Command,
            get-firmware-version    Get-Firmware-Version-Command,
            
            get-counter-one         Get-Counter-One-Command,
            get-counter-two         Get-Counter-Two-Command,
            
            clear-counter-one       Clear-Counter-One-Command,
            clear-counter-two       Clear-Counter-Two-Command,            
        
            erase-flash             Erase-Flash-Command,
            write-flash             Write-Flash-Command
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
            get-device-name         Get-Device-Name-Response,
            get-device-name-max-len Get-Device-Name-Max-Len-Response,
            set-device-name         Get-Device-Name-Response,
            set-log-severity-level  Set-Log-severity-Level-Response,
            get-vbat-and-ambient    Get-VBat-And-Ambient-Response,
            reset                   Reset-Response,
            get-firmware-version    Get-Firmware-Version-Response,
            
            get-counter-one         Get-Counter-One-Response,
            get-counter-two         Get-Counter-Two-Response,            
            
            clear-counter-one       Clear-Counter-One-Response,
            clear-counter-two       Clear-Counter-Two-Response            
        }
    }
    
    Alert ::= SEQUENCE
    {
        type CHOICE
        {
            invalid                 Invalid-Alert,
            not-implemented         Not-Implemented-Alert,
              
            log                     Log-Alert,
            
            reset                   Reset-Alert,
            startup                 Startup-Alert,
            chip-error              Chip-Error-Alert,
            link-status             Link-Status-Alert,
            tx-begin                TX-Begin-Alert,
            tx-complete             TX-Complete-Alert,
            rx1-slot                RX-Slot-Alert,
            rx2-slot                RX-Slot-Alert,
            downstream              Downstream-Alert,
            join-timeout            Join-Timeout-Alert,
            rx                      RX-Alert,
            data-complete           Data-Complete-Alert,
            data-timeout            Data-Timeout-Alert,
            data-nak                Data-NAK-Alert                       
        }
    }

    /* Commands, Response, Alert */

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

    Get-Device-Name-Command         ::= Empty-Command
    Get-Device-Name-Response        ::= SEQUENCE
    {
        counter         Invocation-Counter,
        device-name     VisibleString
    }
    
    Get-Device-Name-Max-Len-Command     ::= Empty-Command
    Get-Device-Name-Max-Len-Response    ::= SEQUENCE
    {
        counter         Invocation-Counter,
        len             INTEGER
    }
    
    Set-Device-Name-Command         ::= SEQUENCE
    {
        counter         Invocation-Counter,
        device-name     VisibleString (SIZE(0..36)) 
    }
    Set-Device-Name-Response        ::= SEQUENCE
    {
        counter         Invocation-Counter,
        result          CHOICE {
            ok          NULL,
            too-long    NULL
        }
    }
    
    Set-Log-Severity-Level-Command ::= SEQUENCE
    {
        counter         Invocation-Counter,
        severity-level  INTEGER (0..7)
    }
    Set-Log-Severity-Level-Response ::= Empty-Response

    Get-VBat-And-Ambient-Command ::= Empty-Command
    Get-VBat-And-Ambient-Response ::= SEQUENCE
    {
        counter         Invocation-Counter,
        vbat            INTEGER,
        ambient         INTEGER
    }
    
    Reset-Command ::= Empty-Command
    Reset-Response ::= Empty-Response
    
    Get-Firmware-Version-Command ::= Empty-Command
    Get-Firmware-Version-Response ::= SEQUENCE
    {
        counter         Invocation-Counter,
        version         VisibleString
    }
    
    Get-Counter-One-Command ::= Empty-Command
    Get-Counter-One-Response ::= SEQUENCE
    {
        counter     Invocation-Counter,
        value       INTEGER
    }
    
    Get-Counter-Two-Command ::= Empty-Command
    Get-Counter-Two-Response ::= Get-Counter-One-Response
    
    Clear-Counter-One-Command ::= Empty-Command
    Clear-Counter-One-Response ::= Empty-Response
    
    Clear-Counter-Two-Command ::= Empty-Command
    Clear-Counter-Two-Response ::= Empty-Response
    
    Erase-Flash-Command ::= SEQUENCE
    {
        counter     Invocation-Counter,
        password    OCTET STRING
    }
    Erase-Flash-Response ::= SEQUENCE
    {
        counter     Invocation-Counter,
        result      ENUMERATION
        {
            success,
            failure
        }
    }
    
    Write-Flash-Command ::= SEQUENCE
    {
        counter     Invocation-Counter,
        address     INTEGER,
        data        OCTET STRING
    }
    Write-Flash-Response ::= Empty-Response
    
    Invalid-Alert ::= NULL
    Not-Implemented-Alert ::= Invocation-Counter
    
    Log-Alert ::= VisibleString
    
    Reset-Alert ::= System-Time
    Chip-Error-Alert ::= System-Time    
    TX-Complete-Alert ::= System-Time
    Join-Timeout-Alert ::= System-Time
    Data-Complete-Alert ::= System-Time    
    Data-Timeout-Alert ::= System-Time    
    Data-NAK-Alert ::= System-Time
    
    Startup-Alert ::= SEQUENCE
    {
        time        System-Time,
        entropy     INTEGER
    }
    
    Link-Status-Alert ::= SEQUENCE
    {
        time    System-Time,
        gw      INTEGER (0..255),
        margin  INTEGER (0..255)
    }
    
    TX-Begin-Alert ::= SEQUENCE
    {
        time    System-Time,
        size    INTEGER (0..255),
        freq    INTEGER (0..max-uint32),
        sf      INTEGER,
        bw      INTEGER,
        power   INTEGER        
    }
    
    RX1-Slot ::= SEQUENCE
    {
        time System-Time        
    }
    
    RX-Slot-Alert ::= SEQUENCE    
    {
        time System-Time        
    }
    
    Downstream-Alert ::= SEQUENCE
    {
        time System-Time,
        size    INTEGER,
        rssi    INTEGER,
        snr     INTEGER        
    }
    
    RX-Alert ::= SEQUENCE
    {
        time System-Time,
        port INTEGER,
        count INTEGER,
        size INTEGER        
    }
    
    /* useful definitions */
    
    Empty-Command ::= Empty-Response
    Empty-Response ::= SEQUENCE
    {
        counter Invocation-Counter
    }
    
    Invocation-Counter ::= INTEGER (0..max-uint8)
    EUI ::= OCTET STRING (SIZE(8))
    Key ::= OCTET STRING (SIZE(16))
    
    System-Time ::= INTEGER (0 .. max-uint32)
    
    max-uint32 Integer ::= 4294967295
    max-uint16 Integer ::= 65535
    
END
{% endhighlight %}

## Hardware Revisions

### 0.3.0

Third revision. Changes from last revision include:

- clamp diodes added to UART level converter
- mounting holes now via reinforced
- one mounting hole can be tied to ground via resistor
- using spring loaded terminals instead of pluggable screw type
- keepout underneath radio module
- improved input circuit with configurable filter and surge protection
- 0805 packages replaced with 0603


![3d model]({{ "/assets/third_pass_model.png" | absolute_url }}){: .center-image }

### 0.2.0

Second revision. Changes from the last revision include:

- U.FL replaced with dual SMA/U.FL footprint
- removed DIO4 and DIO5 routing
- using single inverter for UART transmit line
- removed reset button
- using larger footprints for easy soldering
- enlarged M2 mounting holes to M3

This revision has been used to develop the firmware. A number of 
problems were discovered during the course of development:

- There should be a top-layer keepout underneath U3; trace isolation is currently dependent on soldermask integrity
- Pin assignments don't take into account external interrupt multiplexing of the STM32
- The design needs speed up capacitors on the level converter
- ADC result is measured against VDD and not the reference (therefore the VDD resistor divider is not required)

![schematic]({{ "/assets/second_pass_design.png" | absolute_url }}){: .center-image }

![3d model]({{ "/assets/second_pass_model.png" | absolute_url }}){: .center-image }

![assembled board]({{ "/assets/lpc_0.2.0.jpg" | absolute_url }}){: .center-image }

### 0.1.1

This was the first revision. A number of mistakes were made, namely:

- There was copper (and a via!) underneath the U.FL connector
- Pin assignments mean STM32L051K8 cannot be swapped for the new lower cost STM32L010K8
- The UART level converter power consumption could be improved by using the 
MCU UART signal invert feature
- The reset switch is redundant
- Standard footprints are too small for easy hand soldering
- Double row storage capacitors are difficult to solder by hand

This revision was swiftly replaced with [0.2.0](#020).

![schematic]({{ "/assets/first_pass_design.png" | absolute_url }}){: .center-image }

![3d model]({{ "/assets/first_pass_model.png" | absolute_url }}){: .center-image }

![assembled board]({{ "/assets/first_pass_assembled.jpg" | absolute_url }}){: .center-image }
