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

Above is version [0.2.0](#020) of the LoRa Pulse Counter demonstration platform. 

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

The device has four inputs suitable for interfacing with open-collector, 
dry-contact, or push-pull outputs. In the case of push-pull, the voltage applied
must not exceed the battery voltage, which can be referenced from the "V+" terminals.

Rise-time of the active signal is limited to 250us the RC filter. The fall-time
will be around 500us for open-collector outputs.

The rise-time can be further limited by a configurable software filter. This allows
each input to be adapted to different types of outputs (e.g. slow bouncy contacts vs. fast digital switches).

Each input has a configurable pull up/down resistor. This is useful
for adapting the average energy consumption to the resting state of the input
signal. 

### External Pickup Power Supply

The device can supply a small amount of power to external sensors through
the "V+" terminals. 

You may, for example, need to power a photodiode detector circuit in order
to detect a flashing LED.

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

- resting state of digital inputs (vs. the pull up/down configuration)
- frequency of pulse signals
- frequency of pulse count reporting (configurable)
- alerting strategy (configurable)
- link-check strategy (configurable)

### Configurable

While incomplete at this stage, the following characteristics are 
intended to be configurable:

- input filter mode
- input trigger mode
    - rising
    - falling
    - both
- input pullup/pulldown mode
- alert mode
    - active high/low
- message formats
    - two counters
    - one counter per message
    - counter(s) plus alert status field
- counter push interval
- alert notification strategy

## LoRaWAN Message Format

LoRaWAN messages are differentiated by port number.

### Device-to-Application

[Octet Encoding Rules X.696](https://www.itu.int/rec/T-REC-X.696/en)

#### One-Counter1 (port 1)

#### Two-Counters (port 1)

{% highlight asn1 %}
Two-Counters ::= SEQUENCE
{
    channel1    INTEGER (0..max-uint32),
    channel2    INTEGER (0..max-uint32)
    channel3    INTEGER (0..max-uint32)
    channel4    INTEGER (0..max-uint32)
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
| reflect input | no |
| reflect output | no |
| xor output | no |

### Commands

Commands are constructed as a sequence of a Command-Header
followed by a command specific argument. Similarly Responses
are constructed as a sequence of a Command-Header followed by
a response specific Argument.

{% highlight asn1 %}
Command-Header ::= [0] SEQUENCE 
{
    token   INTEGER (0..255),
    c-id    INTEGER (0..65535)
}

Response-Header ::= [1] SEQUENCE 
{
    token   INTEGER (0..255),
    c-id    INTEGER (0..65535)
}
{% endhighlight %}


| c-id     | name
|--------|-------
| 0x0000 | [Boot-Or-App](#boot-or-app)
| 0x0001 | [Goto-Boot](#goto-boot)
| 0x0002 | [Goto-App](#goto-app)
| 0x0003 | [Restart](#restart)
| 0x0004 | [Get-Boot-Version](#get-boot-version)
| 0x0005 | [Get-App-Version](#get-app-version)
| 0x0010 | [Set-Dev-EUI](#set-dev-eui)
| 0x0011 | [Set-App-EUI](#set-app-eui)
| 0x0012 | [Set-App-Key](#set-app-key)
| 0x0013 | [Get-Dev-EUI](#get-dev-eui)
| 0x0014 | [Get-App-EUI](#get-app-eui)
| 0x0015 | [Get-Encrypted-App-Key](#get-encrypted-app-key)
| 0x0016 | [Join-Network](#join-network)
| 0x0017 | [Forget-Network](#forget-network)
| 0x0018 | [Get-Counter](#get-counter)
| 0x0019 | [Clear-Counter](#clear-counter)
| 0x001a | [Set-Pull](#set-pull)
| 0x001b | [Set-Filter](#set-filter)
| 0x001c | [Get-Device-Name](#get-device-name)
| 0x001d | [Set-Device-Name](#set-device-name)
| 0x001e | [Set-Log-Severity-Level](#set-log-severity-level)
| 0x001f | [Get-Vbat-And-Ambient](#get-vbat-and-ambient)


Command/Response arguments are encoded according to [Octet Encoding Rules X.696](https://www.itu.int/rec/T-REC-X.696/en).

#### Set-Dev-EUI

Write Dev-EUI to non-volatile memory. 

Following this the device shall forget the network and enter a non-joining
mode.

{% highlight asn1 %}
Command-Set-Dev-EUI ::= OCTET STRING (SIZE(8))

Response-Set-Dev-EUI ::= NULL
{% endhighlight %}

#### Set-App-EUI

Write App-EUI to non-volatile memory. 

Following this the device shall forget the network and enter a non-joining
mode.

{% highlight asn1 %}
Command-Set-App-EUI ::= OCTET STRING (SIZE(8))

Response-Set-App-EUI- ::= NULL
{% endhighlight %}

#### Set-App-Key

Write App-Key to non-volatile memory. 

Following this the device shall forget the network and enter a non-joining
mode.

{% highlight asn1 %}
Command-Set-App-Key ::= OCTET STRING (SIZE(16))

Response-Set-App-Key ::= NULL
{% endhighlight %}

#### Get-Dev-EUI

Return the Dev-EUI.

{% highlight asn1 %}
Command-Get-Dev-EUI ::= NULL

Response-Get-Dev-EUI ::= OCTET STRING (SIZE(8))
{% endhighlight %}

#### Get-App-EUI

Return the App-EUI.

{% highlight asn1 %}
Command-Get-App-EUI ::= NULL

Response-Get-App-EUI ::= OCTET STRING (SIZE(8))
{% endhighlight %}

#### Get-Encrypted-App-Key

Return the result of performing AES-128 encryption of a zeroed block using the App-Key.

{% highlight asn1 %}
Command-Get-Encrypted-App-Key ::= NULL

Response-Get-Encrypted-App-Key ::= OCTET STRING (SIZE(16))
{% endhighlight %}

#### Join-Network

Device shall forget the network and enter a joining mode.

{% highlight asn1 %}
Command-Join ::= NULL

Response-Join ::= NULL
{% endhighlight %}

#### Forget-Network

Device shall forget the network and enter a non-joining mode.

{% highlight asn1 %}
Command-Forget ::= NULL

Response-Forget ::= NULL
{% endhighlight %}

#### Set-Log-Severity-Level

Set the severity level for log messages pushed by Log-Message alert in accordance
with [RFC 5424](https://tools.ietf.org/html/rfc5424).

{% highlight asn1 %}
Command-Set-Log-Severity-Level ::= INTEGER (0..255)

Response-Set-Log-Severity-Level ::= NULL
{% endhighlight %}

#### Get-VBAT-And-Ambient

Return the battery voltage and ambient temperature.

{% highlight asn1 %}
Command-Get-Vbat-And-Ambient ::= NULL

Response-Get-Vbat-And-Ambient ::= SEQUENCE
{
    vbat INTEGER (0..65535),
    ambient INTEGER (0..65535)
}
{% endhighlight %}

#### Get-Counter

Return a counter register.

{% highlight asn1 %}
Command-Get-Counter ::= SEQUENCE
{
    index INTEGER (0..255)
}

Response-Get-Counter ::= CHOICE
{
    success                 INTEGER (0..4294967295),
    index-out-of-range      NULL    
}
{% endhighlight %}

#### Clear-Counter

Zero a counter register.

{% highlight asn1 %}
Command-Clear-Counter ::= SEQUENCE
{
    index INTEGER (0..255)
}

Response-Clear-Counter ::= CHOICE
{
    success                 NULL,
    index-out-of-range      NULL    
}
{% endhighlight %}

#### Set-Pull

Set a pull-resistor.

{% highlight asn1 %}
Command-Set-Pull ::= SEQUENCE
{
    index   INTEGER (0..255),
    setting ENUMERATED {
    
        pull-up,
        pull-down    
    }
}

Response-Set-Pull ::= CHOICE
{
    success                 NULL,
    index-out-of-range      NULL,
    setting-out-of-range    NULL    
}
{% endhighlight %}

#### Set-Filter

Set an input filter.

{% highlight asn1 %}
Command-Set-Filter ::= SEQUENCE
{
    index   INTEGER (0..255),
    setting ENUMERATED {
    
        none
    }
}

Response-Set-Filter ::= CHOICE
{
    success                 NULL,
    index-out-of-range      NULL,
    setting-out-of-range    NULL    
}
{% endhighlight %}

#### Goto-Boot

Put device into bootloader mode.

{% highlight asn1 %}
Command-Goto-Boot ::= NULL

Response-Goto-Boot ::= NULL
{% endhighlight %}

#### Goto-App

Put device into application mode.

{% highlight asn1 %}
Command-Goto-App ::= NULL

Response-Goto-App ::= NULL
{% endhighlight %}

#### Boot-Or-App

Indicate whether device is in bootloader or application mode.

{% highlight asn1 %}
Command-Boot-Or-App ::= NULL

Response-Boot-Or-App ::= ENUMERATED { boot (0), app (1) }
{% endhighlight %}

#### Get-Boot-Version

Get bootloader version string.

{% highlight asn1 %}
Command-Get-Boot-Version ::= NULL

Response-Get-Boot-Version ::= VisibleString
{% endhighlight %}

#### Get-App-Version

Get application version string.

{% highlight asn1 %}
Command-Get-App-Version ::= NULL

Response-Get-App-Version ::= VisibleString
{% endhighlight %}

#### Restart

User request to restart the device.

{% highlight asn1 %}
Command-Restart ::= NULL

Response-Restart ::= NULL
{% endhighlight %}

#### Get-Device-Name

Get the unique device name string.

{% highlight asn1 %}
Command-Get-Device-Name ::= NULL

Response-Get-Device-Name ::= VisibleString (SIZE(0..36))
{% endhighlight %}

#### Set-Device-Name

Set the unique device name string.

{% highlight asn1 %}
Command-Set-Device-Name ::= VisibleString (SIZE(0..36))

Response-Set-Device-Name ::= CHOICE 
{
    success,
    too-long
}
{% endhighlight %}

### Alerts

Alerts are constructed as a sequence of a Alert-Header
followed by an alert specific argument

{% highlight asn1 %}
Alert-Header ::= [2] SEQUENCE 
{
    a-id    INTEGER (0..65535)
}
{% endhighlight %}

| a-id     | name
|--------|-------
| 0x0000 | [Unknown-Message](#unknown-message)
| 0x0001 | [Unknown-Command](#unknown-command)
| 0x0002 | [Invalid-Command-Argument](#invalid-command-argument)
| 0x0010 | [Log](#log)

Alert arguments are encoded according to [Octet Encoding Rules X.696](https://www.itu.int/rec/T-REC-X.696/en).

#### Unknown-Message

Pushed by device which has received a series of bytes which do
not match any expected patterns.

{% highlight asn1 %}
Alert-Unkown-Message ::= NULL
{% endhighlight %}

#### Unknown-Command

Pushed by a device which has decoded an unimplemented c-id.

{% highlight asn1 %}
Alert-Unknown-Command ::= SEQUENCE
{
    token INTEGER (0..255),
    c-id  INTEGER (0..65535)
}
{% endhighlight %}

#### Invalid-Command-Argument

Pushed by a device which has decoded an invalid c-id specific argument.

{% highlight asn1 %}
Alert-Invalid-Command-Argument ::= SEQUENCE 
{
    token INTEGER (0..255),
    c-id  INTEGER (0..65535)
}
{% endhighlight %}

#### Log

Used to push logging messages to a client depending on the severity level
set by [Set-Log-Severity-Level](#set-log-severity-level) command.

{% highlight asn1 %}
Alert-Log ::= SEQUENCE
{
    level   INTEGER (0..255),
    message VisibleString
}
{% endhighlight %}

## Hardware Revisions

### 0.4.0

Third revision. Changes from last revision include:

- clamp diodes and TVS surge protection added to UART level converter
- via reinforced mounting holes with optional chassis ground
- optional external button header
- simplified input circuit with TVS diode for surge protection
- four inputs (up from two) each with configurable pull up/down resistors
- keepout underneath radio module
- 0805 packages replaced with 0603
- larger pitch 6 way SWD header

![3d model]({{ "/assets/third_pass_model.png" | absolute_url }}){: .center-image }

![3d model]({{ "/assets/third_pass_model_back.png" | absolute_url }}){: .center-image }

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

![3d model]({{ "/assets/first_pass_model.png" | absolute_url }}){: .center-image }

![assembled board]({{ "/assets/first_pass_assembled.jpg" | absolute_url }}){: .center-image }
