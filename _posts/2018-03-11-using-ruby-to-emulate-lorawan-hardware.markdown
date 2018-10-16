---
layout: post
title:  "Using Ruby to Emulate LoRaWAN Hardware"
date:   2018-03-11
categories:
---

*update: LDL source code is no longer publicly accessible*

For the past year I’ve been working on a background project called [LoraDeviceLib](https://github.com/cjhdev/lora_device_lib_api) (LDL for short). 

Since this is not my day job, I’ve been focusing on the parts of the problem that interest me (the MAC layer) and avoiding anything that seems tedious (buying a gateway, setting up a gateway, debugging radio/hardware related problems, complaining about late kickstarter projects). 

To make this work, I've been exercising all features on my development machine using small tests with varying degrees of mocking vs. integration. This is effective up until the point where all the parts are ready to come together to talk to a network service. For me, I got to this point at the beginning of the year when I was ready to start talking to [The Things Network](https://www.thethingsnetwork.org) (TTN).

To continue with my "no hardware" approach, I decided to write a software layer in Ruby that would emulate the hardware components required to connect a free-running instance of LDL up to TTN. Basically, everything in the box marked "emulator":

![emulator classes]({{ "/assets/high_level.png" | absolute_url }}){: .center-image }

# Implementation

The emulator is not one complete program, rather a collection of classes that are composed into a scenario. The classes developed for the emulator live [here]() and everything is arranged in the usual Rubygems [pattern](http://guides.rubygems.org/make-your-own-gem/). 

The classes developed for the emulator look a bit like this:

![emulator classes]({{ "/assets/emulator.png" | absolute_url }}){: .center-image }

- ExtMAC is a native extension (Ruby written in C) that wraps up LDL.

- MAC is a pure Ruby subclass that extends and improves the basic interfaces in ExtMAC. Native extensions are more difficult to write and maintain than pure
  Ruby so this pattern helps to keep ExtMAC simple.

- Device combines a MAC and a Radio to emulate a LoRaWAN device. It runs an application which is passed in as a block upon initialisation. 

- Radio contains the minimum amount of logic required to send and receive LoRaWAN frames via an instance of Broker. It can detect overlapping transmit and receive windows but doesn’t go so far as to simulate the effect of distance and obstacles on signal quality. 

- Gateway is a fully functional LoRaWAN gateway. It implements the Semtech forwarder protocol to connect up to TTN. To avoid dealing with configuration, the gateway will send and receive frames on any frequency. Also, since I’m not simulating radio, every frame relayed upstream has perfect signal quality.

- Clock implements a event queue which drives Device and Gateway instances. It ensures that system time stands still until an event is complete. This means that timing sensitive code in LDL is never late even if the garbage collector kicks in.

- FrameLogger logs radio frames which it subscribes to via Broker. It includes a packet dissector built from another native extension (not shown).

- Broker implements publish/subscribe messaging used to pass frames and events.

# Running a Scenario

The first thing I wanted to try on the emulator was a scenario in which the device would join and then send and receive data. In bullet point form:

- device starts
- device joins
- device sends a short unique message upstream as often as duty cycle limits permit
- while running, device receives downstream messages in the receive slots
- device prints messages to the terminal as it runs so we can see what is happening

Implemented as a scenario:

{% highlight ruby %}
require 'ldl'

include LDL

LDL::Logger = CompositeLogger
LDL::Logger << ::Logger.new(STDOUT).tap do |log|
  log.formatter = LDL::LOG_FORMATTER
  log.sev_threshold = ::Logger::INFO
end

LDL::SystemTime = Clock.new

broker = Broker.new

FrameLogger.new(broker, File.open('frame.log', 'w'))

gateway = Gateway.new(broker, EUI64.new(ENV["GATEWAY_EUI"]), 
    name: 'gateway'
)

device = Device.new(broker, 
    devEUI: EUI64.new(ENV["DEV_EUI"]),
    appEUI: EUI64.new(ENV["APP_EUI"]), 
    appKey: Key.new(ENV["APP_KEY"]), 
    name: "device"
) do |d|

  sleep 1

  d.mac.on_receive { |port, message| puts "port#{port}: #{message}" }
          
  begin   
      
    puts "joining..."
    
    d.mac.join
    
    puts "join complete"

    counter = 0; port = 1
    
    loop do
    
      SystemTime.wait(d.mac.ticksUntilNextChannel)
      
      puts "sending message '#{counter}' to port #{port}..."
  
      d.mac.data port, counter.to_s
      
      puts "send complete"
      
      counter += 1
        
    end
      
  rescue JoinTimeout            
  
    puts "join failed"
      
  end
    
end

[LDL::SystemTime, gateway, device].each(&:start)
    
begin
  sleep 
rescue Interrupt
end

[LDL::SystemTime, gateway, device].each(&:stop)

exit
{% endhighlight %}

Running the scenario for a few seconds produces the following lines on the terminal:

{% highlight terminal %}
$ bundle exec ruby -Ilib examples/one_device_one_gateway.rb
joining...
INFO  [2018-03-10 00:16:37]        231 gateway: sending PullData (keep alive)
INFO  [2018-03-10 00:16:37]       2263 gateway: received PullAck
INFO  [2018-03-10 00:16:37]       5780 gateway: received frame
INFO  [2018-03-10 00:16:37]       5836 gateway: sending PushData (forward)
INFO  [2018-03-10 00:16:37]       5836 gateway: received PushAck
INFO  [2018-03-10 00:16:41]     100014 gateway: received PullResp
INFO  [2018-03-10 00:16:41]     100014 gateway: transmit frame in 4057660us (tmst=5057800)
INFO  [2018-03-10 00:16:41]     100014 gateway: send up TXAck (error=NONE)
INFO  [2018-03-10 00:16:42]     505780 gateway: transmitting frame
INFO  [2018-03-10 00:16:42]     512460 device: received message
join complete
sending message '0' to port 1...
INFO  [2018-03-10 00:16:42]     516707 gateway: received frame
INFO  [2018-03-10 00:16:42]     516796 gateway: sending PushData (forward)
INFO  [2018-03-10 00:16:42]     516796 gateway: received PushAck
INFO  [2018-03-10 00:16:42]     516796 gateway: received PullResp
INFO  [2018-03-10 00:16:42]     516796 gateway: transmit frame in 999110us (tmst=6167070)
INFO  [2018-03-10 00:16:42]     516796 gateway: send up TXAck (error=NONE)
INFO  [2018-03-10 00:16:43]     616707 gateway: transmitting frame
INFO  [2018-03-10 00:16:43]     621340 device: received message
port1: hello
send complete
sending message '1' to port 1...
INFO  [2018-03-10 00:16:43]     625547 gateway: received frame
INFO  [2018-03-10 00:16:43]     625617 gateway: sending PushData (forward)
INFO  [2018-03-10 00:16:43]     625617 gateway: received PushAck
send complete
sending message '2' to port 1...
INFO  [2018-03-10 00:16:46]     928837 gateway: received frame
INFO  [2018-03-10 00:16:46]     928897 gateway: sending PushData (forward)
INFO  [2018-03-10 00:16:46]     928897 gateway: received PushAck
send complete
sending message '3' to port 1...
INFO  [2018-03-10 00:16:48]    1133042 gateway: received frame
INFO  [2018-03-10 00:16:48]    1133125 gateway: sending PushData (forward)
INFO  [2018-03-10 00:16:48]    1133125 gateway: received PushAck
send complete
sending message '4' to port 1...
INFO  [2018-03-10 00:16:50]    1340969 gateway: received frame
INFO  [2018-03-10 00:16:50]    1341054 gateway: sending PushData (forward)
INFO  [2018-03-10 00:16:50]    1341054 gateway: received PushAck
send complete
sending message '5' to port 1...
INFO  [2018-03-10 00:16:52]    1545174 gateway: received frame
INFO  [2018-03-10 00:16:52]    1545245 gateway: sending PushData (forward)
INFO  [2018-03-10 00:16:52]    1545245 gateway: received PushAck
send complete
{% endhighlight %}

The following CSV format lines are logged by FrameLogger:

{% highlight terminal %}
$ cat frame.log
system_time, eui, message_type, frequency, spreading_factor, bandwidth, power, air_time, size, message
0.00123, 75-0A-FA-1F-8F-01-57-8C, JoinReq, 868300000, 7, 125, 0, 0.05657, 23, AJRFAPB+1bNwjFcBjx/6CnUhYsMTQI8=
5.0578, EA-15-79-3E-11-2C-7F-93, JoinAccept, 868300000, 7, 125, 0, 0.06681, 33, IN7FEkE2/DzEr4ofNWbrMAiSxmxW7LdiGwzQWW3pcUqI
5.12586, 75-0A-FA-1F-8F-01-57-8C, UnconfirmedDataUp, 867700000, 7, 125, 0, 0.04121, 14, QBknASYAAAABU523sLo=
6.16707, EA-15-79-3E-11-2C-7F-93, UnconfirmedDataDown, 867700000, 7, 125, 0, 0.04633, 18, YBknASYAAAABA/v+WucSM4Z1
6.21426, 75-0A-FA-1F-8F-01-57-8C, UnconfirmedDataUp, 868500000, 7, 125, 0, 0.04121, 14, QBknASYAAQABtYpgU6Q=
9.24716, 75-0A-FA-1F-8F-01-57-8C, UnconfirmedDataUp, 867900000, 7, 125, 0, 0.04121, 14, QBknASYAAgABrhd/ihw=
11.28921, 75-0A-FA-1F-8F-01-57-8C, UnconfirmedDataUp, 868300000, 7, 125, 0, 0.04121, 14, QBknASYAAwABwOB/BJo=
13.36848, 75-0A-FA-1F-8F-01-57-8C, UnconfirmedDataUp, 867100000, 7, 125, 0, 0.04121, 14, QBknASYABAABK/zPSX4=
15.41053, 75-0A-FA-1F-8F-01-57-8C, UnconfirmedDataUp, 868100000, 7, 125, 0, 0.04121, 14, QBknASYABQAB4PZJfdY=
{% endhighlight %}

We can see the same thing on the TTN data console:

![emulator classes]({{ "/assets/ttn_console.png" | absolute_url }}){: .center-image }

# Was It Worth The Trouble?

Yes. It's just as useful as I thought it would be.

The emulator helps me to:

- Try out complete instances of LDL against TTN without any physical setup
- Iterate through development/debug cycles just as quickly as a non-embedded software project
- Conserve the ISM band by not blasting out useless "hello world" messages
- Run LDL in different regions (LoRaWAN works differently in EU compared to US, for example)
- Setup automated end-to-end tests

It's not perfect. There are plenty of wrinkles and missing features, the most obvious being that the emulator doesn't simulate the effect of distance and obstacles on radio signals. 
This means that at this time it's not useful for debugging ADR or for observing the effect of many devices and gateways operating in the same space.
