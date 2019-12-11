---
layout: post
title:  "Conforming with LoRaWAN Fair Use Policies"
date:   2019-09-18
categories:
---

[The Things Network](https://www.thethingsnetwork.org/) (a LoRaWAN network provider with a free/community offering)
encourages its community users to meet a fair use policy. 

The policy is intended to prevent TTN users from overwhelming community provided gateways. 
The limit is set to no more than 30 seconds in 24 hours, so an effective duty cycle of 30/86400 (0.000347). 

One way to meet this limit is to design your application to
push data on a fixed interval calculated to not exceed the limit.

But what if:

- You want to push messages on demand
- You want to adjust send interval to a changing data rate (because you use ADR)
- You don't want to implement extra logic

The solution is to make use of the global duty cycle limit setting present
in most LoRaWAN implementations. 

![snippet]({{ "/assets/lorawan_duty_cycle.png" | absolute_url }})
*[https://lora-alliance.org/resource-hub/lorawanr-specification-v101](https://lora-alliance.org/resource-hub/lorawanr-specification-v101)*

This setting can be changed by the network using the DutyCycleReq command. 
On my [implementation](https://github.com/cjhdev/lora_device_lib) this 
setting can also be changed by the application through the 
LDL_MAC_setMaxDCycle() interface. 

The configuration item must be set to 12 in order to meet the TTN fair use policy.
This will limit maximum duty cycle to 1/4096 (0.000244). You may notice this
is more restrictive than what TTN requires, however this is as close as you can
get using this setting.

Since the [Arduino wrapper](https://github.com/cjhdev/lora_device_lib/tree/master/wrappers/arduino/output/arduino_ldl) 
in my implementation is intended for use with TTN, I've added
the duty cycle limit as a default setting. The end result is that a sketch like this meets
both regional and fair use duty cycle limits regardless of data rate:

{% highlight cpp %}
#include <arduino_ldl.h>

static void get_identity(struct lora_system_identity *id)
{       
    static const struct lora_system_identity _id = {
        .appEUI = {0x00U,0x00U,0x00U,0x00U,0x00U,0x00U,0x00U,0x00U},
        .devEUI = {0x00U,0x00U,0x00U,0x00U,0x00U,0x00U,0x00U,0x01U},
        .appKey = {0x2bU,0x7eU,0x15U,0x16U,0x28U,0xaeU,0xd2U,0xa6U,0xabU,0xf7U,0x15U,0x88U,0x09U,0xcfU,0x4fU,0x3cU}
    };

    memcpy(id, &_id, sizeof(*id));
}

ArduinoLDL& get_ldl()
{
    static ArduinoLDL ldl(
        get_identity,       /* specify name of function that returns euis and key */
        EU_863_870,         /* specify region */
        LORA_RADIO_SX1272,  /* specify radio */    
        LORA_RADIO_PA_RFO,  /* specify radio power amplifier */
        A0,                 /* radio reset pin */
        10,                 /* radio select pin */
        2,                  /* radio dio0 pin */
        3                   /* radio dio1 pin */
    );
    
    return ldl;
}

void setup() 
{
}

void loop() 
{ 
    ArduinoLDL& ldl = get_ldl();
    
    if(ldl.ready()){
    
        if(ldl.joined()){
        
            ldl.unconfirmedData(1U);                 
        }
        else{
         
            ldl.otaa();
        }
    }    
    
    ldl.process();        
}
{% endhighlight %}
