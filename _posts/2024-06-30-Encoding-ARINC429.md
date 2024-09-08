---
layout: post
title:  Encoding ARINC429
subtitle: A guide and resource
tags: [a429,wasm,cpp,c]
comments: true
image2: /assets/img/hello_pegaso_post/trayectory.png
---
# 1. Introduction
*Note*: press [here](#enc-calc) to go directly to the encoding calculator.

If you have ever had to deal with avionics in aircraft, you'd definityle had to cross paths with ARINC429.
This protocol is the standard used for transferring data between different systems and honestly, it is painful to use sometimes.
Designed in the 70s, it has an odd way of sending stuff areound and it is also limited in the quantity of information that you can send around. 

This post intends to help the newbies/not-so-newbies on how it worsk from a software perspective. If you want a more details, check this links:
* [Wikipedia](https://en.wikipedia.org/wiki/ARINC_429)
* [AIM](https://www.aim-online.com/wp-content/uploads/2019/07/aim-tutorial-oview429-190712-u.pdf)


# 2. ARINC429 101: Basics

The A429 format is based in transferring fixed-lenght 32 bit frames refereds as "words". These words contain an id, its value and some extra information.

We can distingish three main different types of encoding:

* Binary (BNR): normally used to encode floating point values
* Binary Coded Decimal (BCD): normally used to encode integers
* Discrete (DSC): used to encode state values ( ON/OFF values for example)

## 2.1 The Word format
As shown on the table, we can distingish(starting from the left):

* **Label**: the id of the information. Fixed for the system but it can change between aplications.
* **Source/Destination Identifiers(SDI)**: indicates the intended receiver/transmitting subsystem. 
* **Data**: the encoded data
* **Sign/Status Matrix**: indicates status or the sign of the data being sent
* **Parity**: error code. It uses the odd parity to make sure that the message information has not been corrupted.

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


The striking thing about ARINC429 is that when it is transmitted, we reverse the label( ie: 76543210 to 01234567)
The next table shows how it actually looks when we serialize a word into the bus. 

NOTE: this ARINC429 word is packed in little-endian format (LSB goes before MSB)
<div id="second-table">
<table class="wikitable" style="width:100%; border-collapse: collapse; table-layout: fixed;">
  <tr>
    <th colspan="32">ARINC 429 Word Format</th>
  </tr>
  <tr>
    <th style="width: 3.125%;">P</th>
    <th colspan="2" style="width: 6.25%; font-size: 75%;">SSM</th>
    <th colspan="4" style="width: 12.5%; font-size: 75%; text-align: left; border: none;">MSB</th>
    <th colspan="19" style="width: 59.375%; text-align: center; border: none;">Data</th>
    <th colspan="2" style="width: 6.25%; font-size: 75%; text-align: right; border: none;">LSB</th>
    <th colspan="2" style="width: 6.25%; font-size: 75%; text-align: left; border: none;">SDI</th>
    <th colspan="4" style="width: 12.5%; color: red; text-align: center; border: none;">Label</th>
    <th colspan="2" style="width: 6.25%; font-size: 75%; text-align: right; border: none;">MSB</th>
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
    <td>9</td>
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

| P  | SSM |     Data   | SDI | Label | 
|---|------|------------|-----|------|
|32 |31 - 30  | 29 <---> 11| 10 - 9   | 8 <---> 1  |


## 2.2 Using the SDI
To understand with the SDI does it’s interesting to think of it as a personalised label. Imagine we have two flight computers that are giving showing us the position in our MFD. The MFD will receive through ARINC429 the same label but with different SDIs so its knows from where each values comes from.
## 2.3 SSM states
The SSM status meaning will change depending on how we encode the value:

- BNR:

| Bit: 31 | Bit: 30 | Value in Hex | Decoded info         |
|----|----|-----------|--------------------|
| 0  | 0  | 0x0       | Failure Warning    |
| 0  | 1  | 0x1       | No computed data   |
| 1  | 0  | 0x2       | Functional Test    |
| 1  | 1  | 0x3       | Normal Operation   |

- BCD

| Bit: 31 | Bit: 30 | Value in Hex | Decoded info         |
|----|----|-----------|--------------------|
| 0  | 0  | 0x0       | Plus, North, East, Right, To, Above    |
| 0  | 1  | 0x1       | No computed data   |
| 1  | 0  | 0x2       | Functional Test    |
| 1  | 1  | 0x3       | Minus, South, West, Left, From, Below|



# 3 Encoding ARINC429
## 3.1 Encoding Binary information (BNR)
Now, imagine you're developing a successor to the Concorde and need to integrate an outside air temperature sensor to prevent the aircraft from overheating at high speeds. You'll need to capture the outside air temperature in Celsius, encoded as an A429 word by the sensor.

To encode this information accurately, consider several parameters to ensure precise data representation:

- **Scale:** What increments does your variable require? Is it sufficient to increase stepwise from 1 to 2 to 3, or is finer precision necessary?
- **Offset:** Is there a need to adjust the baseline value?
- **Most Significant Bit (MSB):** Are you utilizing all available data bits to store information?
- **Least Significant Bit (LSB):** This also concerns the use of data bits for information storage.
- **Sign**: Is it positive or negative? The sign is store in the bit 29.

MSB and LSB are crucial when there's a lot of data to transmit but insufficient labels. As you might realize, a single byte (allowing for 255 different labels) isn't enough without reusing labels to encode more information across various bits of the message.

With these parameters defined, you can proceed to compute the encoded value. In this example, we use:

- **Label:** `0205` (Outside Air Temperature, typically represented in octal format)
- **SSM:** `3` (Normal Operation for BNR data)
- **SDI:** `0` (not utilized in this context)
- **Value:** `200°C` (assuming supersonic flight conditions)
- **Scale:** `0.5`
- **Offset:** `0`
- **MSB:** `28`
- **LSB:** `11`
- **Sign:** 0 (None)


This setup ensures accurate and reliable data transmission critical for high-speed aviation.


So, lets start by calculating the encoded value:

$$ Val_{E} = \frac{value - offset}{scale} = \frac{200 - 0}{0.5} = 400 $$

Cool, now we can start the encoding process. 

1. The MSB is 28. Therefore, the 29 bit will be set as zero for the sign.
2. The encoded value is 400 that is 0b110010000( 9 bits). We have  28-17=11  bits for our data field so we have extra two bits( yay!)
3. The SSM is 3. So 0b11.
4. The SDI is zero. So 0b00
5. Our label is 205(octal ). So it is 0b110010000
6. Our parity bit needs to be 1( to be odd)

With this in mind, we can add this to the below table:

<div id="third-table">
<table class="wikitable" style="display: flex; border: none;">
  <tr>
    <th colspan="32">OAT </th>
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
  <tr style="font-size: 66%; text-align: center;">
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
  <tr style="font-size: 66%; text-align: center;">
    <td>1</td>
    <td>1</td>
    <td>1</td>
    <td>0</td>
    <td>0</td>
    <td>0</td>
    <td>0</td>
    <td>0</td>
    <td>0</td> 
    <td>0</td>
    <td>0</td>
    <td>0</td> 
    <td>0</td> 
    <td>1</td> 
    <td>1</td> 
    <td>0</td> 
    <td>0</td> 
    <td>1</td> 
    <td>0</td> 
    <td>0</td> 
    <td>0</td> 
    <td>0</td> 
    <td>0</td> 
    <td>0</td> 
    <td>0</td> 
    <td>1</td> 
    <td>0</td> 
    <td>0</td> 
    <td>0</td> 
    <td>1</td> 
    <td>0</td> 
    <td>1</td> 
    </tr>
</table>
</div>

Finally, we reverse the label and put it out front. Note that the next table is a bit odd. 
I show the label as the first element and then I add the rest of the word. 

This is to ease for the reader the extraction of the words as we packing the word in little endian. Therefore, the first elements will be

<div id = "fourth-table" >
<table class="wikitable" style="display: flex; border: none;">
  <tr>
    <th colspan="32">OAT Label fliped and add before</th>
  </tr>
  <tr>
    <th colspan="8" style="width: 10.500%; text-align: center; border: none;">Label</th>
    <th style="width: 3.125%;">P</th>
    <th colspan="2" style="width: 6.250%;">SSM</th>
    <th colspan="4" style="width: 12.500%; font-size: 75%; text-align: left; border: none;">MSB</th>
    <th colspan="11" style="width: 34.375%; text-align: center; border: none;">Data</th>
    <th colspan="4" style="width: 12.500%; font-size: 75%; text-align: right; border: none;">LSB</th>
    <th colspan="2" style="width: 6.250%;">SDI</th>
  </tr>
  <tr style="font-size: 66%; text-align: center;">
    <td>1</td>
    <td>2</td>
    <td>3</td>
    <td>4</td>
    <td>5</td>
    <td>6</td>
    <td>7</td>
    <td style="color: red">8</td>
    <td>32</td>
    <td>31</td>
    <td>30</td>
    <td>29</td>
    <td>28</td>
    <td>27</td>
    <td>26</td>
    <td style="color:red">25</td>
    <td>24</td>
    <td>23</td>
    <td>22</td>
    <td>21</td>
    <td>20</td>
    <td>19</td>
    <td>18</td>
    <td style="color:red">17</td>
    <td>16</td>
    <td>15</td>
    <td>14</td>
    <td>13</td>
    <td>12</td>
    <td>11</td>
    <td>10</td>
    <td style="color: red">9</td>
  </tr>
  <tr>
    <td>1</td> 
    <td>0</td> 
    <td>1</td> 
    <td>0</td> 
    <td>0</td> 
    <td>0</td> 
    <td>0</td> 
    <td>1</td> 
    <td>1</td>
    <td>1</td>
    <td>1</td>
    <td>0</td>
    <td>0</td> 
    <td>0</td>
    <td>0</td>
    <td>0</td>
    <td>0</td>
    <td>0</td>
    <td>0</td>
    <td>0</td> 
    <td>0</td> 
    <td>1</td> 
    <td>1</td> 
    <td>0</td> 
    <td>0</td> 
    <td>1</td> 
    <td>0</td> 
    <td>0</td> 
    <td>0</td> 
    <td>0</td> 
    <td>0</td> 
    <td>0</td> 
  </tr>
</table>
</div>

Now, we need to transform this binary codes into the hexadecimal word format.
To do so, we group in 4 bytes that generate our Words(32 bits). 

* The first word  of our message includes bits from 32 to 25 : 0xE0(0b11100000)
* The second word  of our message includes bits from 24 to 17 : 0x06(0b0000110)
* The third word of our message includes bits from 16 to 9: 0x40 (0b100000)
* The fourth word  of our message is the label reversed: 0xA1 (0b10100001)

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
(If we were sending only this information, we would finished here. Add the correct SSM, SDI and calcualte the partity; and send it. )

2. Encoding the fuel quantity (bytes 27 to 11):
Suppose our fuel sensor reports 1000 liters. We'll use the formula:

$$ Val_{E} = \frac{value - offset}{scale} = \frac{1000 - 0}{0.5} = 2000 $$

The binary representation of 2000 is 0b11111010000 (11 bits).

3. Constructing the word:
   - Label: 207 (octal) = 0b10000111
   - SDI: 00 (not used in this example)
   - Data: 0b1 (tank type) + 0b011111010000 (fuel quantity)
   - SSM: 0x03 ( it the SSM for BNR data)
   - Parity: To be calculated


This finally yields with our data:

<div id="fuel-table">
<table class="wikitable" style="display: flex; border: none;">
  <tr>
    <th colspan="32">Fuel Tank Type and Quantity</th>
  </tr>
  <tr>
    <th style="width: 3.125%;">P</th>
    <th colspan="2" style="width: 6.250%;">SSM</th>
    <th colspan="4" style="width: 12.500%; font-size: 75%; text-align: left; border: none;">MSB</th>
    <th colspan="11" style="width: 34.375%; text-align: center; border: none;">Data</th>
    <th colspan="4" style="width: 12.500%; font-size: 75%; text-align: right; border: none;">LSB</th>
    <th colspan="2" style="width: 6.250%;">SDI</th>
    <th colspan="8" style="width: 25.000%; text-align: center; border: none;">Label</th>
  </tr>
  <tr style="font-size: 86%; text-align: center;">
    <td>1</td>
    <td>1</td>
    <td>1</td>
    <td>0</td>
    <td>1</td>
    <td>0</td>
    <td>1</td>
    <td>1</td>
    <td>1</td>
    <td>1</td>
    <td>1</td>
    <td>0</td>
    <td>1</td>
    <td>0</td>
    <td>0</td>
    <td>0</td>
    <td>0</td>
    <td>0</td>
    <td>0</td>
    <td>0</td>
    <td>0</td>
    <td>0</td>
    <td>0</td>
    <td>0</td>
    <td>0</td>
    <td>0</td>
    <td>1</td>
    <td>0</td>
    <td>0</td>
    <td>0</td>
    <td>1</td>
    <td>1</td>
  </tr>
</table>
</div>

## 3.3 Encoding BCD

Binary Coded Decimal is another way to transmit data. It is specially useful when we want to transmit 
integers. In this case, we can encode up to 4 full digits or 5 digits. 
There are 18 data bits, and in order to represent a value from 0 to 9 we need 4 bits. Therefore, we have 4 bits for the first four digits but then we only have 3 for our most significant digit. Threfore, the last three digist will only be able to represent up to 7.

There are some rules regarding how to do this in ARINC429:

1. If the value is bigger than 79999(80000 for example), bits 27 to 29 will be padded with zeros and the most significant digit will pass to be the third most significant bit (0800). 

As an example of encoding the data, we could think of the value 79876. It´s less than 7999 so we will be able to encode it:
- 29 to 27: ```1 1 1```
- 26 to 23: ```1 0 0 1```
- 22 to 19: ```1 0 0 0```
- 18 to 15: ```0 1 1 1```
- 14 to 11: ```0 1 1 0```



### Encoding calculator {#enc-calc}
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

