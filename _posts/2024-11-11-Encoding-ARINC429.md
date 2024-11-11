---
layout: post
title:  Encoding ARINC429
subtitle: A guide and resource
tags: [a429,wasm,cpp,c]
comments: true
---
*Note*: click [here](#enc-calc) to go directly to the encoding calculator.
# 1. Introduction

If you have ever dealt with avionics in aircraft, you've definitely crossed paths with ARINC429.
This protocol is the standard used for transferring data between different systems in aviation, and it can be challenging to work with at times.
Designed in the 1970s, it has a unique way of transmitting data and is limited in the quantity of information it can send. Despite its age, ARINC429 remains widely used in modern aircraft due to its reliability and established infrastructure.

This post intends to help the newbies or not-so-newbies on how it works from a software perspective. If you want more details, check these links:
* [Wikipedia](https://en.wikipedia.org/wiki/ARINC_429)
* [AIM](https://www.aim-online.com/wp-content/uploads/2019/07/aim-tutorial-oview429-190712-u.pdf)

Also, at the end of this post, you'll find a handy tool for calculating ARINC429 encoding values.

# 2. ARINC429 101: Basics

The A429 format is based on transferring fixed-length 32 bit frames referred to as "words". These words contain an id, its value and some extra information.

We can distinguish three main types of encoding:

* Binary (BNR): normally used to encode floating point values
* Binary Coded Decimal (BCD): normally used to encode integers
* Discrete (DSC): used to encode state values ( ON/OFF values for example)

There are more encodings but these are the most common ones.

As for the transmission speeds (which we won't be discussing in detail), ARINC429 operates at two standard rates:

* High Speed: 100 kHz (100,000 bits per second)
* Low Speed: 12.5 kHz (12,500 bits per second)


## 2.1 The Word format
An ARINC429 word is made up of 5 different parts:

* **Label**: the id of the information. Fixed for the system but it can change between applications.
* **Source/Destination Identifiers(SDI)**: indicates the intended receiver/transmitting subsystem. 
* **Data**: the encoded data
* **Sign/Status Matrix(SSM)**: indicates status or the sign of the data being sent. It is an optional field.
* **Parity(P)**: error code. It uses the odd parity to make sure that the message information has not been corrupted.

The below table shows a visual representation of the word.
<div id="first-table">
<table class="wikitable" style="display: flex; border: none;"> 
  <tr>
    <th colspan="32">ARINC 429 Word Format</th>
  </tr>
  <tr>
    <th style="width: 3.125%;">P</th>
    <th colspan="2" style="width: 6.250%;">SSM</th>
    <th colspan="4" style="width: 12.500%; font-size: 75%; text-align: left; border: none;">MSB</th>
    <th colspan="11" style="width: 34.375%; text-align: center; border: none;">Data</th>
    <th colspan="4" style="width: 12.500%; font-size: 75%; text-align: right; border: none;">LSB</th>
    <th colspan="2" style="width: 6.250%;">SDI</th>
    <th colspan="2" style="width: 6.250%; font-size: 75%; text-align: left; border: none;">LSB</th>
    <th colspan="4" style="width: 12.500%; text-align: center; border: none;">Label</th>
    <th colspan="2" style="width: 6.250%; font-size: 75%; text-align: right; border-left: none;">MSB</th>
  </tr>
  <tr style="font-size: 60%; text-align: center;">
    <td>32</td>
    <td>31</td>
    <td>30</td>
    <td>29</td>
    <td>28</td>
    <td>27</td>
    <td>26</td>
    <td>25</td>
    <td>24</td>
    <td>23</td>
    <td>22</td>
    <td>21</td>
    <td>20</td>
    <td>19</td>
    <td>18</td>
    <td>17</td>
    <td>16</td>
    <td>15</td>
    <td>14</td>
    <td>13</td>
    <td>12</td>
    <td>11</td>
    <td>10</td>
    <td style="width: 3.125%;">9</td>
    <td>8</td>
    <td>7</td>
    <td>6</td>
    <td>5</td>
    <td>4</td>
    <td>3</td>
    <td>2</td>
    <td>1</td>
  </tr>
</table>
</div>


One interesting aspect of ARINC429 is that when transmitted, the label is reversed (e.g., binary 01101010 becomes 01010110). 
This is due to the early days of A429, when the hardware would use shift registers to receive the data. By reversing the label, the hardware could easily reject words that were not meant for it or weren't part of its grouping.

It is important to note that while there are standard label values (e.g., label 205 for Mach number, 310 for latitude), these assignments are not strictly enforced and may vary between different systems and applications.

The following table illustrates how a word appears when serialized onto the bus:


| P  | SSM |     Data   | SDI | Label | 
|---|------|------------|-----|------|
|32 |31 - 30  | 29 <---> 11| 10 - 9   | 1 <---> 8  |

## 2.2 Understanding SDI (Source/Destination Identifier)
The SDI can be thought of as a way to differentiate between multiple sources or destinations for the same type of data. Here's a practical example:
Imagine an aircraft with two flight computers, both capable of providing position information to the Multi-Function Display (MFD). 
To distinguish between these sources we will set different SDIs to their messages. Its usage is a incosistent across different systems, so it is important to check the system documentation to understand how it is used.

The SDI field, while optional, can serve dual purposes. In some cases, it's used to extend the data field, effectively providing additional bits for the message payload. This is particularly common when encoding latitude (label 310) or longitude (label 311) values, where every available bit is valuable for maintaining precision.
## 2.3 SSM states
The SSM (Sign/Status Matrix) bits have different meanings depending on the encoding type used:


### BNR:

| Bit: 31 | Bit: 30 | Value in Hex | Decoded info         |
|----|----|-----------|--------------------|
| 0  | 0  | 0x0       | Failure Warning    |
| 0  | 1  | 0x1       | No computed data   |
| 1  | 0  | 0x2       | Functional Test    |
| 1  | 1  | 0x3       | Normal Operation   |

### BCD

| Bit: 31 | Bit: 30 | Value in Hex | Decoded info         |
|----|----|-----------|--------------------|
| 0  | 0  | 0x0       | Plus, North, East, Right, To, Above    |
| 0  | 1  | 0x1       | No computed data   |
| 1  | 0  | 0x2       | Functional Test    |
| 1  | 1  | 0x3       | Minus, South, West, Left, From, Below|

### DSC

| Bit: 31 | Bit: 30 | Value in Hex | Decoded info         |
|----|----|-----------|--------------------|
| 0  | 0  | 0x0       | Verified data, Normal Operation      |
| 0  | 1  | 0x1       | No computed data   |
| 1  | 0  | 0x2       | Functional Test    |
| 1  | 1  | 0x3       | Failure Warning   |


These SSM states provide important context about the data being transmitted, 
such as its validity, directionality, or operational status.


# 3 Encoding ARINC429
## 3.1 Encoding Binary information (BNR)
Now, imagine you're developing a successor to the Concorde and need to integrate an outside air temperature sensor to prevent the aircraft from overheating at high speeds. You'll need to capture the outside air temperature in Celsius, encoded as an A429 word by the sensor.

To encode this information accurately, consider several parameters to ensure precise data representation:

- **Scale:** What increments does your variable require? Is it sufficient to increase stepwise from 1 to 2 to 3, or is finer precision necessary?
- **Offset:** Is there a need to adjust the baseline value?
- **Most Significant Bit (MSB):** Are you utilizing all available data bits to store information?
- **Least Significant Bit (LSB):** This also concerns the use of data bits for information storage.
- **Sign**: Is it positive or negative? The sign is stored in the bit 29.

MSB and LSB are crucial when determining the scale and precision of the data.

With these parameters defined, you can proceed to compute the encoded value. In this example, we use:

- **Label:** `0211` (Outside Air Temperature, typically represented in octal format)
- **SSM:** `3` (Normal Operation for BNR data)
- **SDI:** `0` 
- **Value:** `200°C` (assuming supersonic flight conditions)
- **Scale:** `0.5`
- **Offset:** `0`
- **MSB:** `28`
- **LSB:** `11`
- **Sign:** 0 (None)


This setup ensures accurate and reliable data transmission critical for high-speed aviation.


So, let's start by calculating the encoded value:

$$ Val_{E} = \frac{value - offset}{scale} = \frac{200 - 0}{0.5} = 400 $$

Now we can start the encoding process. 

1. The MSB is 28. Therefore, the 29 bit will be set as zero for the sign.
2. The encoded value is 400 that is 0b110010000( 9 bits). We have  28-17=11  bits for our data field so we have extra two bits(yay!)
3. The SSM is 3. So 0b11.
4. The SDI is zero. So 0b00
5. Our label is 205(octal). So it is 0b10000101
6. Our parity bit needs to be 1( to be odd)




Finally, we reverse the label and put it out front. Note that the next table is a bit odd. 

| P  | SSM |     Data   | SDI | Label | 
|---|------|------------|-----|------|
|32 |31 - 30  | 29          <--->                  11| 10 - 9   | 1 <---> 8  |
|1 |1     1  | 0 0 0 0 0 0 0 0 0 0 1 1 0 0 1 0 0 0 0| 0 0   | 1 0 1 0 0 0 0 1 |


Now, we need to transform this binary codes into the hexadecimal word format.
To do so, we group in 4 bytes that generate our words(32 bits). 

* The first byte of our message includes bits from 32 to 25 : 0xE0(0b11100000)
* The second byte of our message includes bits from 24 to 17 : 0x06(0b0000110)
* The third byte of our message includes bits from 16 to 9: 0x40 (0b100000)
* The fourth byte of our message is the label reversed: 0xA1 (0b10100001)

Our final word would like this: 0xE0, 0x06, 0x40, 0xA1

Now, lets check if we are correct in our calculations. Test it [here](#enc-calc).
## 3.2 Encoding discrete information (Discrete)

Let's consider a more straightforward example. 
Imagine your flight display needs to notify you about the type of fuel tank your aircraft is using: normal range or extended range. We'll also explore how to reuse a label for additional information.

In this scenario, we'll use label 207 to encode both the fuel tank type and the quantity of fuel reported by the fuel sensor. Our specification states that:

- Byte 28 indicates the type of fuel tank
- Bytes 27 to 11 represent the fuel quantity


For the fuel quantity, we'll use the following parameters:: Offset: 0 and Scaling: 0.5

Now, let's proceed with the calculations:

1. Encoding the fuel tank type (byte 28):
   - 0: Normal range tank
   - 1: Extended range tank

Let's assume we have an extended range tank, so we'll set byte 28 to 1.
(If we were sending only this information, we would finish here. Add the correct SSM, SDI and calculate the partity; and send it. )

2. Encoding the fuel quantity (bytes 27 to 11):
Suppose our fuel sensor reports 1000 liters. We'll use the formula:

$$ Val_{E} = \frac{value - offset}{scale} = \frac{1000 - 0}{0.5} = 2000 $$

The binary representation of 2000 is 0b11111010000 (11 bits).

3. Constructing the word:
   - Label: 207 (octal) = 0b10000111
   - SDI: 00 
   - Data: 0b1 (tank type) + 0b011111010000 (fuel quantity)
   - SSM: 0x03 (0b11)
   - Parity: To be calculated


This finally yields our data:

| P  | SSM |     Data   | SDI | Label | 
|---|------|------------|-----|------|
|32 |31 - 30  | 29          <--->                  11| 10 - 9   | 1 <---> 8  |
|1 |1     1  | 0 1 0 1 1 1 1 1 0 1 0 0 0 0 0 0 0 0 | 0 0   | 1 1 1 0 0 0 0 1 |


You can verify this operation in the calculator by entering both variables separately and examining the byte values on 
the binary side. All values should match, except for the data bytes. The sum of the data bytes will produce the encoding we obtained.

When combining BNR and DSC encodings in the same word, the data field is divided into two sections: the lower bits are reserved for discrete values, while bits 29 down to the remaining higher bits are used for the signed binary data. This arrangement allows both types of data to coexist within the same ARINC429 word while maintaining their respective functionalities.

## 3.3 Encoding BCD

Binary Coded Decimal is another way to transmit data. It is specially useful when we want to transmit 
integers. In this case, we can encode up to 4 full digits or 5 digits. 
Let's break down how BCD encoding works in ARINC429:

The data field provides 19 bits (bits 11-29) for encoding decimal digits. Since each decimal digit (0-9) requires 4 bits, we can encode digits as follows:

- First digit (rightmost): bits 11-14 (4 bits)
- Second digit: bits 15-18 (4 bits)
- Third digit: bits 19-22 (4 bits)
- Fourth digit: bits 23-26 (4 bits)
- Fifth digit (leftmost): bits 27-29 (3 bits)

Due to having only 3 bits for the most significant digit, the maximum value that can be encoded is 79999. Any value above this will be truncated, as the leftmost digit can only represent values 0-7 using 3 bits.

If the value we want to represent is greater than 79999(80001 for example), bits 27 to 29 will be padded with zeros and the most significant digit will pass to be the third most significant bit (08000) and we will lose the last digit(1).

As an example of encoding the data, we could think of the value 79876. It´s less than 79999 so we will be able to encode it:
- 29 to 27: ```1 1 1```
- 26 to 23: ```1 0 0 1```
- 22 to 19: ```1 0 0 0```
- 18 to 15: ```0 1 1 1```
- 14 to 11: ```0 1 1 0```

Now, if we wanted to encode the value 80001 (80001 > 80000):
- 29 to 27: ```0 0 0```
- 26 to 23: ```1 0 0 0```
- 22 to 19: ```0 0 0 0```
- 18 to 15: ```0 0 0 0```
- 14 to 11: ```0 0 0 0```

We will lose the last digit and the value will be 80000.

# 4. Final Remarks
ARINC429 is an old but established standard in avionics, and it seems like it will continue to haunt us for the next few decades. As technology and computing evolve, the limitations of ARINC429 in terms of data capacity and speed present ongoing challenges.

To overcome these bottlenecks, we have to find imaginative solutions, such as using multiple labels to send larger data sets, like files, across several ARINC429 words. This allows us to stretch the protocol's capabilities while staying within its constraints.

While ARINC429 may not be the most advanced protocol, its reliability ensures that it will stick around in aviation systems. The challenge is finding innovative ways to make it work alongside new technologies, ensuring it remains relevant as avionics systems evolve.


# Encoding calculator {#enc-calc}
<div id="wasm-container">

<style>
      .game {
        top: 0;
        left: 0;
        margin: 0;
        border: 0;
        width: 120%;
        height: 100%;
        overflow: hidden;
        display: block;
        image-rendering: optimizeSpeed;
        image-rendering: -moz-crisp-edges;
        image-rendering: -o-crisp-edges;
        image-rendering: -webkit-optimize-contrast;
        image-rendering: optimize-contrast;
        image-rendering: crisp-edges;
        image-rendering: pixelated;
        -ms-interpolation-mode: nearest-neighbor;
      }
</style>
<canvas
      class="game"
      id="canvas"
      oncontextmenu="event.preventDefault()"
></canvas>
</div>
<script>
  var Module = {
    preRun: [],
    postRun: [],
    print: function (o) {
      (o = Array.prototype.slice.call(arguments).join(" ")), console.log(o);
    },
    printErr: function (o) {
      (o = Array.prototype.slice.call(arguments).join(" ")),
        console.error(o);
    },
    canvas: document.getElementById("canvas"),
    setStatus: function (o) {
      console.log("status: " + o);
    },
    monitorRunDependencies: function (o) {
      console.log("monitor run deps: " + o);
    },
  };
  window.onerror = function () {
    console.log("onerror: " + event.message);
  };
</script>
<script async src="../assets/js/demo.js"></script>

