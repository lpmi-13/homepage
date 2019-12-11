---
layout: post
title:  "LDL vs. LoRaMac-Node: Size and Complexity"
date:   2019-12-11
categories:
---

In this post I compare the apparent [cyclomatic complexity](https://en.wikipedia.org/wiki/Cyclomatic_complexity) and compiled
size of two LoRaWAN implementations: LDL and LoRaMac-Node.

[LDL](https://github.com/cjhdev/lora_device_lib) is my own implementation. [LoRaMac-Node](https://github.com/Lora-net/LoRaMac-node) 
appears to be a Semtech sponsored LoRaWAN reference implementation.

LoRaMac-Node has been around for a while and is probably a sensible choice for 
commercial projects. LDL is experimental and should not be trusted.

## Method

The comparison is between LDL [0.2.3](https://github.com/cjhdev/lora_device_lib/releases/tag/0.2.3)
and LoRaMac-Node [4.4.2](https://github.com/Lora-net/LoRaMac-node/releases/tag/v4.4.2). 

Source to be compared is grouped into one of three domains:

<table>
    <caption>Domain Definitions</caption>
    <theader>
        <tr>
            <th>Domain</th>
            <th>Description</th>
        </tr>
    </theader>
    <tbody>
        <tr>
            <td>Region</td>
            <td>region specific features</td>
        </tr>
        <tr>
            <td>Radio</td>
            <td>radio drivers</td>
        </tr>
        <tr>
            <td>MAC</td>
            <td>all platform agnostic code which does not fall into the region or radio domains, excluding cryptographic modes</td>
        </tr>
    </tbody>
</table>

The following source files are selected from each project for each domain:

<table>
    <caption>Radio Domain Files</caption>
    <theader>
        <tr>
            <th>LDL</th>
            <th>LoRaMac-Node</th>
        </tr>
    </theader>
    <tbody>
        <tr>
            <td>ldl_radio.c/h</td>
            <td>radio/radio.h</td>
        </tr>
        <tr>
            <td>ldl_radio_defs.h</td>
            <td>radio/sx1272/sx1272.c/h</td>
        </tr>
        <tr>
            <td>ldl_chip.h</td>
            <td>radio/sx1276/sx1276.c/h</td>
        </tr>
        <tr>
            <td></td>
            <td>radio/sx1272/sx1272Regs-LoRa.h</td>
        </tr>
        <tr>
            <td></td>
            <td>adio/sx1272/sx1272Regs-Fsk.h</td>
        </tr>
        <tr>
            <td></td>
            <td>radio/sx1276/sx1276Regs-LoRa.h</td>
        </tr>
        <tr>
            <td></td>
            <td>radio/sx1276/sx1276Regs-Fsk.h</td>
        </tr>
    </tbody>
</table>

<table>
    <caption>Region Domain Files</caption>
    <theader>
        <tr>
            <th>LDL</th>
            <th>LoRaMac-Node</th>
        </tr>
    </theader>
    <tbody>
        <tr>
            <td>ldl_region.c/h</td>
            <td>region/Region.c/h</td>
        </tr>
        <tr>
            <td></td>
            <td>region/RegionCommon.c/h</td>
        </tr>
        <tr>
            <td></td>
            <td>region/RegionEU868.c/h</td>
        </tr>
        <tr>
            <td></td>
            <td>region/RegionUS915.c/h</td>
        </tr>
        <tr>
            <td></td>
            <td>region/RegionEU433.c/h</td>
        </tr>
        <tr>
            <td></td>
            <td>region/RegionAU915.c/h</td>
        </tr>        
    </tbody>
</table>

<table>
    <caption>MAC Domain Files</caption>
    <theader>
        <tr>
            <th>LDL</th>
            <th>LoRaMac-Node</th>
        </tr>
    </theader>
    <tbody>
        <tr>
            <td>ldl_mac.c/h</td>
            <td>mac/LoRaMac.c/h </td>
        </tr>     
        <tr>
            <td>ldl_mac_commands.c/h</td>
            <td>mac/LoRaMacCommands.c/h</td>
        </tr>     
        <tr>
            <td>ldl_frame.c/h </td>
            <td>mac/LoRaMacConfirmQueue.c/h</td>
        </tr>     
        <tr>
            <td>ldl_ops.c/h</td>
            <td>mac/LoRaMacSerializer.c/h</td>
        </tr>     
        <tr>
            <td>ldl_sm.c/h</td>
            <td>mac/LoRaMacAdr.c/h</td>
        </tr>     
        <tr>
            <td>ldl_sm_internal.h </td>
            <td>mac/LoRaMacParser.c/h</td>
        </tr>     
        <tr>
            <td>ldl_stream.c/h </td>
            <td>mac/LoRaMacCrypto.c/h</td>
        </tr>     
        <tr>
            <td>ldl_system.c/h</td>
            <td>system/timer.c/h</td>
        </tr>     
        <tr>
            <td></td>
            <td>peripherals/soft-se/soft-se.c/h</td>
        </tr>     
        <tr>
            <td></td>
            <td> mac/LoRaMacHeaderTypes.h</td>
        </tr>     
        <tr>
            <td></td>
            <td>mac/LoRaMacTypes.h</td>
        </tr>     
        <tr>
            <td></td>
            <td>mac/secure-element.h</td>
        </tr>     
        <tr>
            <td></td>
            <td>mac/LoRaMacTest.h</td>
        </tr>     
        
    </tbody>
</table>

The comparison excludes:

- cryptographic implementations (AES-ECB, AES-CMAC, AES-CTR)
- target specific code (e.g. hardware abstraction layer)
- mismatched LoRaWAN features which are implemented in separate files

LoRaMac-Node implements more regions, radios, and modes than LDL. Where possible 
these extra features have been excluded in order to create a more like for like
comparison. 

Unfortunately not all mismatched features can be excluded. For example, 
LoRaMac-Node class C is inseparable from class A, while LDL only implements class A.
This difference means we should expect LoRaMac-Node to have more LoC and associated
scores.

The table below gives a high-level view of the features implemented by
the files sorted into the domain tables above. Mismatched features
are indicated by a 'yes' in one column and a 'no' in the other.

<table>
    <caption>Feature Matrix</caption>
    <theader>
        <tr>
            <th>Domain</th>
            <th>Feature</th>
            <th>LDL</th>
            <th>LoRaMac-Node</th>
        </tr>        
    </theader>
    <tbody>
        <tr>
            <td>MAC</td>
            <td>Class A</td>
            <td class="table-success">yes</td>
            <td class="table-success">yes</td>
        </tr>
        <tr>
            <td>MAC</td>
            <td>Class C</td>
            <td class="table-danger">no</td>
            <td class="table-success">yes</td>
        </tr>
        <tr>
            <td>MAC</td>
            <td>LoRaWAN 1.1 key derivation and handling</td>
            <td class="table-success">yes</td>
            <td class="table-danger">no</td>
        </tr>
        <tr>
            <td>MAC</td>
            <td>OTAA</td>
            <td class="table-success">yes</td>
            <td class="table-success">yes</td>
        </tr>
        <tr>
            <td>MAC</td>
            <td>ABP</td>
            <td class="table-danger">no</td>
            <td class="table-success">yes</td>
        </tr>        
        <tr>
            <td>MAC</td>
            <td>ADR</td>
            <td class="table-success">yes</td>
            <td class="table-success">yes</td>
        </tr>
        <tr>
            <td>MAC</td>
            <td>Confirmed/Unconfirmed Data</td>
            <td class="table-success">yes</td>
            <td class="table-success">yes</td>
        </tr>        
        <tr>
            <td>MAC</td>
            <td>ioctl style access to all parameters</td>
            <td class="table-danger">no</td>
            <td class="table-success">yes</td>
        </tr>        
        <tr>
            <td colspan="4"></td>            
        </tr>
        <tr>
            <td>Radio</td>
            <td>SX1272</td>
            <td class="table-success">yes</td>
            <td class="table-success">yes</td>
        </tr>
        <tr>
            <td>Radio</td>
            <td>SX1276</td>
            <td class="table-success">yes</td>
            <td class="table-success">yes</td>
        </tr>
        <tr>
            <td>Radio</td>
            <td>FSK Modulation</td>
            <td class="table-danger">no</td>
            <td class="table-success">yes</td>
        </tr>
        <tr>
            <td colspan="4"></td>            
        </tr>
        <tr>
            <td>Region</td>
            <td>EU_868_870</td>
            <td class="table-success">yes</td>
            <td class="table-success">yes</td>
        </tr>        
        <tr>
            <td>Region</td>
            <td>EU_433</td>
            <td class="table-success">yes</td>
            <td class="table-success">yes</td>
        </tr>        
        <tr>
            <td>Region</td>
            <td>US_902_928</td>
            <td class="table-success">yes</td>
            <td class="table-success">yes</td>
        </tr>        
        <tr>
            <td>Region</td>
            <td>AU_915_928</td>
            <td class="table-success">yes</td>
            <td class="table-success">yes</td>
        </tr>        
    </tbody>
</table>

The tool [pmccabe](https://people.debian.org/~bame/pmccabe/overview.html) is used to measure:

- apparent lines of code (includes comments and whitespace)
- number of statements
- number of functions
- modified McCabe complexity (per function and per source file)

The higher the McCabe score, the more complicated the code. Functions with scores
above 10 are considered complicated.

To determine compiled size, the source is compiled for an ARM Cortex M0+ using 
arm-none-aebi-gcc 8.2.1.  All compiler settings are in the [makefile](https://github.com/cjhdev/lorawan_comparison/blob/master/makefile).
Size optimisation (-Os) is used in an effort to produce the smallest possible compiled size.

The objects are grouped into archive files and the size of each archive is
determined using arm-none-aebi-size. The objects are not linked.

Both projects are vendored into [this](https://github.com/cjhdev/lorawan_comparison) repository.
The tools pmccabe and gcc are orchestrated from [this](https://github.com/cjhdev/lorawan_comparison/blob/master/comparison.rb) script.

Data is always presented in the order of LDL first.
LDL is plotted in blue, LoRaMac-Node is plotted in orange.


## Overall Scores

<table><caption>LDL PMCCABE Results</caption><header><tr><th>Domain</th><th>LoC</th><th>Functions</th><th>Statements</th><th>Complexity</th></tr></header><tbody><tr><td>Mac</td><td>6933</td><td>149</td><td>1812</td><td>462</td></tr><tr><td>Radio</td><td>1205</td><td>36</td><td>289</td><td>72</td></tr><tr><td>Region</td><td>1127</td><td>24</td><td>418</td><td>83</td></tr><tr><td></td><td><b>9265</b></td><td><b>209</b></td><td><b>2519</b></td><td><b>617</b></td></tr></tbody></table>

<table><caption>LoRaMAC-Node PMCCABE Results</caption><header><tr><th>Domain</th><th>LoC</th><th>Functions</th><th>Statements</th><th>Complexity</th></tr></header><tbody><tr><td>Mac</td><td>13798</td><td>189</td><td>3381</td><td>862</td></tr><tr><td>Radio</td><td>8129</td><td>71</td><td>1358</td><td>345</td></tr><tr><td>Region</td><td>9915</td><td>155</td><td>2347</td><td>512</td></tr><tr><td></td><td><b>31842</b></td><td><b>415</b></td><td><b>7086</b></td><td><b>1719</b></td></tr></tbody></table>

![]({{ "/assets/lorawan_comparison/domain_loc.jpg" | absolute_url }}){: .center-image }

![]({{ "/assets/lorawan_comparison/domain_functions.jpg" | absolute_url }}){: .center-image }

![]({{ "/assets/lorawan_comparison/domain_statements.jpg" | absolute_url }}){: .center-image }

![]({{ "/assets/lorawan_comparison/domain_complexity.jpg" | absolute_url }}){: .center-image }

## Function Complexity

Here I've plotted the frequency of function complexity scores for each domain.

![]({{ "/assets/lorawan_comparison/function_mac.jpg" | absolute_url }}){: .center-image }

![]({{ "/assets/lorawan_comparison/function_radio.jpg" | absolute_url }}){: .center-image }

![]({{ "/assets/lorawan_comparison/function_region.jpg" | absolute_url }}){: .center-image }

## Compiled Size

<table><caption>LDL Compiled Size</caption><theader><tr><th>Object File (from *.c)</th><th>Text (bytes)</th><th>BSS+Data (bytes)</th></tr></theader><tbody><tr><td>ldl_region.o</td><td>1500</td><td>0</td></tr><tr><td>ldl_radio.o</td><td>1437</td><td>0</td></tr><tr><td>ldl_dummy_radio.o</td><td>20</td><td>16</td></tr><tr><td>ldl_mac.o</td><td>7982</td><td>0</td></tr><tr><td>ldl_mac_commands.o</td><td>782</td><td>0</td></tr><tr><td>ldl_frame.o</td><td>888</td><td>0</td></tr><tr><td>ldl_ops.o</td><td>1340</td><td>0</td></tr><tr><td>ldl_sm.o</td><td>262</td><td>0</td></tr><tr><td>ldl_stream.o</td><td>504</td><td>0</td></tr><tr><td>ldl_system.o</td><td>18</td><td>0</td></tr><tr><td>ldl_dummy_mac.o</td><td>36</td><td>992</td></tr><tr><td></td><td><b>14769</b></td><td><b>1008</b></td></tr></tbody></table>

<table><caption>LoRaMAC-Node Compiled Size</caption><theader><tr><th>Object File (from *.c)</th><th>Text (bytes)</th><th>BSS+Data (bytes)</th></tr></theader><tbody><tr><td>Region.o</td><td>1372</td><td>0</td></tr><tr><td>RegionAU915.o</td><td>2810</td><td>917</td></tr><tr><td>RegionCommon.o</td><td>1184</td><td>0</td></tr><tr><td>RegionEU433.o</td><td>2675</td><td>212</td></tr><tr><td>RegionEU868.o</td><td>2883</td><td>292</td></tr><tr><td>RegionUS915.o</td><td>3000</td><td>920</td></tr><tr><td>sx1276.o</td><td>6092</td><td>284</td></tr><tr><td>sx1272.o</td><td>5517</td><td>284</td></tr><tr><td>LoRaMac.o</td><td>12100</td><td>1644</td></tr><tr><td>LoRaMacCommands.o</td><td>608</td><td>256</td></tr><tr><td>LoRaMacConfirmQueue.o</td><td>580</td><td>42</td></tr><tr><td>LoRaMacSerializer.o</td><td>484</td><td>0</td></tr><tr><td>LoRaMacAdr.o</td><td>204</td><td>0</td></tr><tr><td>LoRaMacParser.o</td><td>334</td><td>0</td></tr><tr><td>LoRaMacCrypto.o</td><td>2292</td><td>84</td></tr><tr><td>timer.o</td><td>530</td><td>4</td></tr><tr><td>soft-se.o</td><td>1040</td><td>968</td></tr><tr><td></td><td><b>43705</b></td><td><b>5907</b></td></tr></tbody></table>

![]({{ "/assets/lorawan_comparison/flash_size.jpg" | absolute_url }}){: .center-image }

![]({{ "/assets/lorawan_comparison/ram_size.jpg" | absolute_url }}){: .center-image }
