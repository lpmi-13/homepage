---
layout: post
title:  LoraDeviceLib now works with MBED
date:   2018-10-06
categories:
---

I recently completed an MBED wrapper for my [LoRaWAN implementation](https://github.com/cjhdev/lora_device_lib) (LDL). 
This follows on from the [Arduino wrapper]({% post_url 2018-08-23-build-your-own-experimental-lorawan-sensor %}) I wrote about just a few months ago. 
I thought it would be an interesting point of comparison to see the same demo application implemented for MBED as was implemented for Arduino.

## Application requirements

The requirements are the same as for the Arduino demo. The node shall power up, join [The Things Network](https://www.thethingsnetwork.org/), then transmit
temperature and humidity every 30 minutes.

## Hardware

I'm using an STM Nucleo F446RE board because that's what I had in my parts bin. I'm combining that
with the same radio and sensor used for the Arduino demo.

![assembled sensor]({{ "/assets/mbed_sensor.jpg" | absolute_url }})

## Application implementation

The demo application is implemented [here](https://github.com/cjhdev/lora_device_lib/tree/master/bindings/mbed/mbed_ldl_sensor) 
but be aware you need to run make from [this](https://github.com/cjhdev/lora_device_lib/tree/master/bindings/mbed) 
directory in order to put all the files in the right place.

The application code looks like this:

{% highlight c %}#include "mbed_ldl.h"
#include "DHT.h"

Semaphore pending_flag(1U);
Semaphore sample_flag(1U, 1U);

static void pending_handler()
{
    pending_flag.release();
}

static void sample_handler()
{
    sample_flag.release();    
    pending_flag.release();    
}

int main()
{
    static Ticker sample_ticker;

    static SPI spi(D11, D12, D13);
    
    static DHT sensor(D6, DHT::DHT11);
    
    static const uint8_t devEUI[] = DEV_EUI;
    static const uint8_t appEUI[] = APP_EUI;
    static const uint8_t appKey[] = APP_KEY;
    
    static MbedLDL ldl(
        callback(pending_handler),        
        devEUI,
        appEUI, 
        appKey, 
        EU_863_870,         /* region */
        LORA_RADIO_SX1272,  /* radio */
        spi,                /* SPI instance (possibly shared) */
        A0,                 /* radio reset */
        D10,                /* radio select */
        D2,                 /* DIO0 */
        D3                  /* DIO1 */
    );
    
    ldl.setRate(5U);
    ldl.setPower(5U);                
 
    sample_ticker.attach_us(callback(sample_handler), 30UL*60UL*1000000UL);
    
    while(true){
        
        pending_flag.wait();
        
        ldl.process();
        
        if(ldl.ready()){
    
            if(ldl.joined()){
                
                if(sample_flag.wait(0UL) > 0){
                
                    if(sensor.read() == DHT::SUCCESS){
                
                        float buf[] = {
                            sensor.getTemperature(), 
                            sensor.getHumidity()
                        };
                        
                        ldl.unconfirmedData(1U, buf, sizeof(buf));
                    }
                }
            }
            else{
                        
                ldl.otaa();
            }
        }
    }
}
{% endhighlight %}

The bulk of the application looks similar to the Arduino demo. The main difference is that the MBED
wrapper will notify the application by callback when an event is pending 
(i.e. when the application needs to call process())).

The demo application uses this callback to release the _pending_flag_ semphore. 
The _pending_flag_ semaphore is also released by _sample_handler_ which is called every 30 minutes
by _sample_ticker_. The main loop then waits on the _pending_flag_ and the effect is that the application will
sleep when there is no work to be done.

Building the application using the mbed-cli tool produces the following output:

{% highlight terminal %}
$ mbed compile

<snip>

Link: mbed_ldl_sensor
Elf2Bin: mbed_ldl_sensor
| Module                       |         .text |       .data |        .bss |
|------------------------------|---------------|-------------|-------------|
| [fill]                       |     134(+134) |       4(+4) |     25(+25) |
| [lib]/c.a                    | 27885(+27885) | 2472(+2472) |     89(+89) |
| [lib]/gcc.a                  |   3272(+3272) |       0(+0) |       0(+0) |
| [lib]/misc                   |     208(+208) |     12(+12) |     28(+28) |
| environment_sensor.o         |   1402(+1402) |       4(+4) | 1140(+1140) |
| mbed-dht/DHT.o               |     722(+722) |       0(+0) |       0(+0) |
| mbed-os/components           |       96(+96) |       0(+0) |       0(+0) |
| mbed-os/drivers              |   2072(+2072) |       4(+4) |   144(+144) |
| mbed-os/hal                  |   2073(+2073) |       4(+4) |     68(+68) |
| mbed-os/platform             |   3472(+3472) |   276(+276) |   173(+173) |
| mbed-os/rtos                 |   9747(+9747) |   168(+168) | 6053(+6053) |
| mbed-os/targets              | 15129(+15129) |       4(+4) | 1132(+1132) |
| mbed_ldl/lora_aes.o          |     835(+835) |       0(+0) |       0(+0) |
| mbed_ldl/lora_board.o        |       34(+34) |       0(+0) |       0(+0) |
| mbed_ldl/lora_cmac.o         |     560(+560) |       0(+0) |       0(+0) |
| mbed_ldl/lora_event.o        |     512(+512) |       0(+0) |       0(+0) |
| mbed_ldl/lora_frame.o        |   1654(+1654) |       0(+0) |       0(+0) |
| mbed_ldl/lora_mac.o          |   4354(+4354) |       0(+0) |       0(+0) |
| mbed_ldl/lora_mac_commands.o |     656(+656) |       0(+0) |       0(+0) |
| mbed_ldl/lora_radio.o        |   1247(+1247) |       0(+0) |       0(+0) |
| mbed_ldl/lora_region.o       |   1114(+1114) |       0(+0) |       0(+0) |
| mbed_ldl/lora_stream.o       |     106(+106) |       0(+0) |       0(+0) |
| mbed_ldl/mbed_ldl.o          |   1231(+1231) |       4(+4) |     32(+32) |
| Subtotals                    | 78515(+78515) | 2952(+2952) | 8884(+8884) |
Total Static RAM memory (data + bss): 11836(+11836) bytes
Total Flash memory (text + data): 81467(+81467) bytes

{% endhighlight %}

LDL appears to occupy around 11K bytes of flash memory and this includes run-time support 
for multiple regions and radios. 

RAM usage is no more than 1140 bytes in bss and some of that will be taken up by the other
objects in the demo application. 
 
## Flashing and running the demo

The MBED wrapper will print status information to the terminal as it runs. 
You can compile the application, flash the target, and open a terminal using the following
mbed-cli command:

{% highlight terminal %}
$ mbed compile --flash --sterm

<snip>

[mbed] Detecting connected targets/boards to your system...
[mbed] Opening serial terminal to "NUCLEO_F446RE"
--- Terminal on /dev/ttyACM2 - 9600,8,N,1 ---
[94]RESET
[21736]STARTUP
[60061482]TX_BEGIN: SZ=23 F=868300000 SF=7 BW=125 P=5
[60123557]TX_COMPLETE
[65118809]RX1_SLOT: F=868300000 SF=7 BW=125
[65196647]DOWNSTREAM: SZ=33 RSSI=-66 SNR=9
[65239633]JOIN_COMPLETE
[65287790]TX_BEGIN: SZ=21 F=867900000 SF=7 BW=125 P=5
[65344750]TX_COMPLETE
[66344194]RX1_SLOT: F=867900000 SF=7 BW=125
[67314471]RX2_SLOT: F=869525000 SF=9 BW=125
[67358533]DATA_COMPLETE
[1800023429]TX_BEGIN: SZ=21 F=867100000 SF=7 BW=125 P=5
[1800080384]TX_COMPLETE
[1801079827]RX1_SLOT: F=867100000 SF=7 BW=125
[1802050106]RX2_SLOT: F=869525000 SF=9 BW=125
[1802096183]DATA_COMPLETE
{% endhighlight %}

At the same time we can see activity on The Things Network console:

![sensor console]({{ "/assets/mbed_sensor_console.png" | absolute_url }}){: .center-image }

Nice.
