---
layout: post
title:  "LoRa Pulse Counter Demo"
date:   2019-08-23
categories:
permalink: lpc
---

This is my LoRaWAN enabled business card project.

It's a four-channel 2KHz impulse counter based on an STM32L051K8. 

![assembled board]({{ "/assets/third_pass_assembled.jpg" | absolute_url }}){: .center-image }

![3d model]({{ "/assets/third_pass_model.png" | absolute_url }}){: .center-image }

![3d model]({{ "/assets/third_pass_model_back.png" | absolute_url }}){: .center-image }

![in case]({{ "/assets/top_crop_resize.jpg" | absolute_url }}){: .center-image }

![profile]({{ "/assets/activity.jpg" | absolute_url }}){: .center-image }

Hardware features include:

- STM32L051K8 (64K flash, 8K RAM, Cortex M0+)
- RFM95W (SX1276 based radio module from HopeRF)
- dual SMA/UFL footprint
- status button and LED indicator
- UART with level converter
- reverse cell protection
- power supply for accessories (via "V+")
- surge protection
- four inputs suitable for open-collector or dry-contact

The firmware is currently occupying half of available flash. Features include:

- low power operation (~2uA in stop mode)
- bespoke [LoRaWAN stack](https://github.com/cjhdev/lora_device_lib)
- [device failure alarms](#device-alarm)
- tickless application timers
- XTAL fail-safe
- 32 bit counters stored in no-init volatile memory
- count on rising/falling/both
- alert on rising/falling/both/none
- double-buffer configuration store with fail-safe defaults
- multi-function [status button with LED indicator](#status-button-and-led)
- [UART protocol](#uart-protocol)
- local and remote configuration/control via a common [register](#registers) system
- hard fault diagnostics

Configuration items include:

- (per input)
    - pull-up/pull-down
    - polarity
    - rise-time (250us to 10s)
    - counter trigger (rising/falling/both)
    - alert trigger (none/rising/falling/both)
    - publish enable/disable
- publish interval
- LoRaWAN parameters    
- radio enabled/disabled
- user defined serial number (device name)
- low-battery level

## Pin Map

### Inputs

pin | signal    | comment
----|-----------|-------------------------------------------------------
1   | V+        | weak power supply / reference
2   | 1         | channel 1
3   | 2         | channel 2
4   | GND       | 
5   | V+        | 
6   | 3         | channel 3
7   | 4         | channel 4
8   | GND       | 
{: style="font-size: 90%;" }

### UART

The pin layout is compatible with a large number of commercial USB-to-serial 
adapters:

pin | signal    | comment
----|-----------|-------------------------------------------------------
1   | GND       | mandatory  
2   | CTS       | connected to GND
3   | VCC       | mandatory 3V3 or 5V (this powers the level converter)
4   | TXD       | transmit line
5   | RXD       | receive line
6   | DTR       | not connected
{: style="font-size: 90%;" }

## Status Button and LED

The button has two actions depending on  how it is pressed:

- A short press will cause the status LED to blink the device status code.
- A long press (>5s) will enable or disable joining mode (i.e. toggle the setting)

The status codes are as follows:

number of flashes | status 
------------------|-----------------------------------------------------
0                 | dead
1                 | joined and downlink received in interval
2                 | joined but no downlink received in interval
3                 | joining
4                 | not-joined
{: style="font-size: 90%;" }

## LoRaWAN Message Format

| direction | port | name      |
|----------|------|-----------|
| up        | 1    | [power-up](#power-up) |
| up        | 2    | config-publish | 
| up        | 3    | [input-publish](#input-publish) | 
| up        | 4    | input-alert     | 
| down      | 1    | [remote-write-register](#remote-write-register) | 
{: style="font-size: 90%;" }

### Power-Up

Sent once immediately after the first join-accept is received after system start.
If MCU state is available from the time of reset, this is included in the message.

{% highlight asn1 %}
Counter-Publish ::= SEQUENCE
{
    device-reset-reason BIT STRING (SIZE(8)) {
        wdt,
        hard-fault,
        power,
        user,
        reserved-5,
        reserved-6,
        reserved-7,
        reserved-8,
    },
    device-alarms BIT STRING (SIZE(8)) {
        clock-failure,
        low-battery,
        over-temperature,
        reserved-4,
        reserved-5,
        reserved-6,
        reserved-7,
        reserved-8
    },
    mcu-state SEQUENCE
    {        
        r0  INTEGER (0..4294967295),
        r1  INTEGER (0..4294967295),
        r2  INTEGER (0..4294967295),
        r3  INTEGER (0..4294967295),
        r12 INTEGER (0..4294967295),
        lr  INTEGER (0..4294967295),
        pc  INTEGER (0..4294967295),
        sr  INTEGER (0..4294967295)
        
    } OPTIONAL
}
{% endhighlight %}

The encoding is [Octet Encoding Rules X.696](https://www.itu.int/rec/T-REC-X.696/en) with the following exceptions:

- OPTIONAL mcu-state presence is detected by size

### Input-Publish

Sent every [publish-interval](#publish-interval). The abstract structure is as follows:

{% highlight asn1 %}
Counter-Publish ::= SEQUENCE
{
    device-alarms BIT STRING (SIZE(8)) {
        clock-failure,
        low-battery,
        over-temperature,
        reserved-4,
        reserved-5,
        reserved-6,
        reserved-7,
        reserved-8
    },
    input-alarms BIT STRING (SIZE(8)) {
        input-1-active,
        input-2-active,
        input-3-active,
        input-4-active,
        reserved1,
        reserved2,
        reserved3,
        reserved4
    },
    counter-presence BIT STRING (SIZE(8)) {
        input-1-present,
        input-2-present,
        input-3-present,
        input-4-present,
        reserved-5,
        reserved-6,
        reserved-7,
        reserved-8,
    },    
    counter1 INTEGER (0..4294967295) OPTIONAL,
    counter2 INTEGER (0..4294967295) OPTIONAL,
    counter3 INTEGER (0..4294967295) OPTIONAL,
    counter4 INTEGER (0..4294967295) OPTIONAL  
}
{% endhighlight %}

The encoding is [Octet Encoding Rules X.696](https://www.itu.int/rec/T-REC-X.696/en) with the following exceptions:

- OPTIONAL counter fields are included according to corresponding counter-presence bit being set

### Remote-Write-Register

Works the same way as [Write-Register](#write-register). Use to modify [registers](#registers) remotely.

{% highlight asn1 %}
Remote-Write-Register ::= SEQUENCE OF command SEQUENCE
{
    id INTEGER (0..65535),
    // register type
}
{% endhighlight %}

Multiple commands can be packed into one message. [Config-Publish]() is
sent after processing a command regardless of success or failure.


## UART Protocol

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
| 0x0004 | [Write-Register](#write-register)
| 0x0005 | [Read-Register](#read-register)

Command/Response arguments are encoded according to [Octet Encoding Rules X.696](https://www.itu.int/rec/T-REC-X.696/en).

#### Write-Register

Write a value to a [register](#registers).

{% highlight asn1 %}
Command-Write-Register ::= SEQUENCE
{    
    id INTEGER (0..65535)
    // register type
}

Register-Access-Result ::= ENUMERATED {

    success,
    register-does-not-exist,
    access-denied,
    argument-error,
    range-error,
    not-authorised,
    temporary-failure        
}

Command-Write-Register :: SEQUENCE
{
    result Register-Access-Result
}
{% endhighlight %}

#### Read-Register

Read a value from a [register](#registers).

{% highlight asn1 %}
Command-Read-Register ::= SEQUENCE
{    
    id INTEGER (0..65535)
}

Command-Write-Register :: SEQUENCE
{
    result Register-Access-Result,
    // returned value
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

Used to push logging messages to a client depending on the [log severity level](#log-severity-level)
setting.

{% highlight asn1 %}
Alert-Log ::= SEQUENCE
{
    level   INTEGER (0..255),
    message VisibleString
}
{% endhighlight %}

### Registers

Registers can be read and written by the [Read-Register](read-register) and [Write-Register](write-register)
commands. Access is role based according to read-group and write-group.

Groups are:

- user
- owner
- lora
- ALL (user\|owner\|lora)

| id     | read-group | write-group | name | type |
|--------|--------|------|------|-------|
| 0x0000 | ALL | owner\|lora | [log-severity-level](#log-severity-level) | INTEGER(0..255)
| 0x0001 | ALL | | [boot-version](#boot-version) | OCTET STRING
| 0x0002 | ALL | | [app-version](#app-version) | OCTET STRING
| 0x0003 | ALL | | [factory-id](#factory-id) | OCTET STRING 
| 0x0004 | ALL | | [vbat](#vbat) | INTEGER(0..4294967295)
| 0x0005 | ALL | | [ambient](#ambient) | INTEGER(0..4294967295)
| 0x0006 | ALL | owner | [low-vbat-threshold](#low-vbat-threshold) | INTEGER(0..65535)
| 0x0007 | ALL | owner | [device-name](#device-name) | OCTET STRING(SIZE(0..36))
| 0x0008 | ALL | | [device-alarm](#device-alarm) | BIT STRING (SIZE(8))
| 0x0100 | ALL | owner | [dev-eui](#dev-eui) | OCTET STRING (SIZE(8))
| 0x0101 | ALL | owner | [app-eui](#app-eui) | OCTET STRING (SIZE(8))
| 0x0102 |  | owner | [app-key](#app-key) | OCTET STRING (SIZE(16))
| 0x0103 | ALL | | [enc-app-key](#enc-app-key) | OCTET STRING (SIZE(16))
| 0x0104 | ALL | owner\|lora | [join-enabled](#join-enabled) | BOOLEAN
| 0x0105 | ALL | owner\|lora | [tx-rate](#tx-rate) | INTEGER(0..255)
| 0x0106 | ALL | owner\|lora | [tx-power](#tx-power) | INTEGER(0..255)
| 0x0107 | ALL | owner\|lora | [adr-enabled](#adr-enabled) | BOOLEAN
| 0x0108 | ALL | | [joined](#joined) | BOOLEAN
| 0x0200 | ALL | owner\|lora | [counter1](#countern) | INTEGER(0..4294967295)
| 0x0201 | ALL | owner\|lora | [counter2](#countern)  | INTEGER(0..4294967295)
| 0x0202 | ALL | owner\|lora | [counter3](#countern)  | INTEGER(0..4294967295)
| 0x0203 | ALL | owner\|lora | [counter4](#countern)  | INTEGER(0..4294967295)
| 0x0204 | ALL | owner\|lora | [input1-polarity](#inputn-polarity)   | INTEGER(0..255)
| 0x0205 | ALL | owner\|lora | [input2-polarity](#inputn-polarity)   | INTEGER(0..255)
| 0x0206 | ALL | owner\|lora | [input3-polarity](#inputn-polarity)   | INTEGER(0..255)
| 0x0207 | ALL | owner\|lora | [input4-polarity](#inputn-polarity)   | INTEGER(0..255)
| 0x0208 | ALL | owner\|lora | [input1-pull](#inputn-pull)       | INTEGER(0..255)
| 0x0209 | ALL | owner\|lora | [input2-pull](#inputn-pull)       | INTEGER(0..255)
| 0x020a | ALL | owner\|lora | [input3-pull](#inputn-pull)       | INTEGER(0..255)
| 0x020b | ALL | owner\|lora | [input4-pull](#inputn-pull)       | INTEGER(0..255)
| 0x020c | ALL | owner\|lora | [input1-rise-time](#inputn-rise-time)     | INTEGER(0..255)
| 0x020d | ALL | owner\|lora | [input2-rise-time](#inputn-rise-time)     | INTEGER(0..255)
| 0x020e | ALL | owner\|lora | [input3-rise-time](#inputn-rise-time)     | INTEGER(0..255)
| 0x020f | ALL | owner\|lora | [input4-rise-time](#inputn-rise-time)     | INTEGER(0..255)
| 0x0210 | ALL | owner\|lora | [input1-alert-trigger](#inputn-alert-trigger) | INTEGER(0..255)
| 0x0211 | ALL | owner\|lora | [input2-alert-trigger](#inputn-alert-trigger) | INTEGER(0..255)
| 0x0212 | ALL | owner\|lora | [input3-alert-trigger](#inputn-alert-trigger) | INTEGER(0..255)
| 0x0213 | ALL | owner\|lora | [input4-alert-trigger](#inputn-alert-trigger) | INTEGER(0..255)
| 0x0214 | ALL | owner\|lora | [input1-alert-lockout-interval](#) | INTEGER(0..255)
| 0x0215 | ALL | owner\|lora | [input2-alert-lockout-interval](#) | INTEGER(0..255)
| 0x0216 | ALL | owner\|lora | [input3-alert-lockout-interval](#) | INTEGER(0..255)
| 0x0217 | ALL | owner\|lora | [input4-alert-lockout-interval](#) | INTEGER(0..255)
| 0x0218 | ALL | owner\|lora | [input1-publish-enabled](#inputn-publish-enable) | BOOLEAN
| 0x0219 | ALL | owner\|lora | [input2-publish-enabled](#inputn-publish-enable) | BOOLEAN
| 0x021a | ALL | owner\|lora | [input3-publish-enabled](#inputn-publish-enable) | BOOLEAN
| 0x021b | ALL | owner\|lora | [input4-publish-enabled](#inputn-publish-enable) | BOOLEAN
| 0x021c | ALL | owner\|lora | [publish-interval](#publish-interval) | INTEGER(0..65535)
| 0x021d | ALL | owner\|lora | [alert-interval](#) | INTEGER(0..255)
| 0x021e | ALL | | [input1-state](#inputn-state) | BOOLEAN
| 0x021f | ALL | | [input2-state](#inputn-state) | BOOLEAN
| 0x0220 | ALL | | [input3-state](#inputn-state) | BOOLEAN
| 0x0221 | ALL | | [input4-state](#inputn-state) | BOOLEAN    
| 0x0222 | ALL | owner\|lora | [counter1-trigger](#countern-trigger) | INTEGER(0..255)
| 0x0223 | ALL | owner\|lora | [counter2-trigger](#countern-trigger) | INTEGER(0..255)
| 0x0224 | ALL | owner\|lora | [counter3-trigger](#countern-trigger) | INTEGER(0..255)
| 0x0225 | ALL | owner\|lora | [counter4-trigger](#countern-trigger) | INTEGER(0..255)
{: style="font-size: 80%; text-align: center;"}

#### log-severity-level

The minimum level for log messages pushed by [Log alert](#log):

{% highlight asn1 %}
INTEGER(0..255) {

    emergency   (0),
    alert       (1),
    critical    (2),
    error       (3),
    warning     (4),
    notice      (5),
    info        (6),
    debug       (7)
}
{% endhighlight %}

#### boot-version

Bootloader version string.

#### app-version

Application version string.

#### factory-id

A read-only string unique to this LoRa Pulse Counter.

#### vbat

Battery voltage in millivolts.

#### ambient

Ambient temperature measured by integrated temperature sensor.

This value is used internally to determine when the operating temperature
is outside of the recommended range.

#### low-vbat-threshold

When vbat is below this value the low-battery [device alarm](#device-alarm)
will be active.

#### device-name

A user assigned name to help with inventory management.

#### ambient-offset

An offset correction applied to the ambient temperature measurement.

#### device-alarm

The device alarm field is used to indicate faults on the device:

{% highlight asn1 %}
BIT STRING (SIZE(8)) {
    clock-failure       (0),
    low-battery         (1),
    over-temperature    (2),
    reserved-4          (3),
    wdt-reset           (4),
    hard-fault          (5),
    reserved-7          (6),
    reserved-8          (7)
}
{% endhighlight %}


 fault | description 
-------|-------------
clock-failure       | device is unable to schedule precise time events
low-battery         | battery requires replacement
over-temperature    | ambient temperature is outside of the recommended window
power-reset         | power supply was interrupted either by tamper or brown-out
wdt-reset           | firmware malfunction
{: style="font-size: 90%;" }


#### dev-eui

LoRaWAN device EUI.

#### app-eui

LoRaWAN application EUI.

#### app-key

A write only register containing the LoRaWAN application key.

#### enc-app-key

A read only register containing a zero block which has been encrypted
with app-key. This can be used to verify [app-key](#app-key) without revealing it.

#### join-enabled

- Write true to start the OTAA process.
- Write false to forget the current network and prevent OTAA process

#### tx-rate

The rate setting to use when adr-enabled is set to false.

#### tx-power

The power setting to use when adr-enabled is set to false.

#### adr-enabled

Write 'true' to enable ADR or 'false' to disable ADR.

#### joined

Read to determine if device is joined to a network.

#### counter[n]

A 32 bit integer incremented according to [counter\[n\]-trigger](countern-trigger).

Write any value to reset to zero.

#### input[n]-polarity

{% highlight asn1 %}
INTEGER(0..255) 
{
    low     (0),
    high    (1)  
}
{% endhighlight %}

#### input[n]-pull

Pull-up/down setting.

{% highlight asn1 %}
INTEGER(0..255) 
{
    down    (0),
    up      (1)
}
{% endhighlight %}

#### input[n]-rise-time

{% highlight asn1 %}
INTEGER(0..255) 
{
    none    (0),
    25ms    (1),
    250ms   (2),
    1000ms  (3),
    5000ms  (4),
    10000ms (5)
}
{% endhighlight %}

#### input[n]-alert-trigger

{% highlight asn1 %}
INTEGER(0..255) 
{
    never                   (0),
    rising                  (1),
    falling                 (2),
    rising-falling          (3)
}
{% endhighlight %}

#### counter[n]-trigger

[counter[n]](#countern) is incremented according to this trigger.

{% highlight asn1 %}
INTEGER(0..255) 
{
    rising          (0),
    falling         (1),
    rising-falling  (2)
}
{% endhighlight %}

#### publish-interval

[Input-Publish](#input-publish) is sent every publish-interval minutes.

#### alert-interval
