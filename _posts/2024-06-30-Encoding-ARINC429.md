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
* Discreta data encoding (won't talk about it here)

## 2.1 The Word format
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
  <tr style="font-size: 86%; text-align: center;">
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
As shown on the table, we can distingish(starting from the left):

* **Label**: the id of the information. Fixed for the system but it can change between aplications.
* **Source/Destination Identifiers(SDI)**: indicates the intended receiver/transmitting subsystem. Some times used to transfer more info.
* **Data**: the encoded data
* **Sign/Status Matrix**: indicates status or the sign of the data being sent
* **Parity**: error code. It uses the odd parity to make sure that the message information has not been corrupted.

The striking thing about ARINC429 is that when it is transmitted, we first send the label but reversed and then the rest of the message. 
The next table shows how it actually looks when we serialize a word into the bus. 

NOTE: this ARINC429 word is packed in little-endian format (LSB goes before MSB)
<div id = "second-table" >
<table class="wikitable" style="display: flex; border: none;">
  <tr>
    <th colspan="32">ARINC 429 Word Format</th>
  </tr>
  <tr>
    <th colspan="8" style="color: red; width: 12.500%; text-align: center; border: none;">Label</th>
    <th colspan="2" style="width: 6.250%;">SDI</th>
    <th colspan="4" style="width: 12.500%; font-size: 75%; text-align: right; border: none;">LSB</th>
    <th colspan="11" style="width: 34.375%; text-align: center; border: none;">Data</th>
    <th colspan="2" style="width: 6.250%;">SSM</th>
    <th colspan="4" style="width: 12.500%; font-size: 75%; text-align: left; border: none;">MSB</th>
    <th style="width: 3.125%;">P</th>
  </tr>
  <tr style="font-size: 86%; text-align: center;">
    <td>8</td>
    <td>7</td>
    <td>6</td>
    <td>5</td>
    <td>4</td>
    <td>3</td>
    <td>2</td>
    <td>1</td>
    <td>9</td>
    <td>10</td>
    <td>11</td>
    <td>12</td>
    <td>13</td>
    <td>14</td>
    <td>15</td>
    <td>16</td>
    <td>17</td>
    <td>18</td>
    <td>19</td>
    <td>20</td>
    <td>21</td>
    <td>22</td>
    <td>23</td>
    <td>24</td>
    <td>25</td>
    <td>26</td>
    <td>27</td>
    <td>28</td>
    <td>29</td>
    <td>30</td>
    <td>31</td>
    <td>32</td>
  </tr>
</table>
</div>

## 3 Encoding ARINC429
### 3.1 Encoding Binary information (BNR)
Now, imagine you're developing a successor to the Concorde and need to integrate an outside air temperature sensor to prevent the aircraft from overheating at high speeds. You'll need to capture the outside air temperature in Celsius, encoded as an A429 word by the sensor.

To encode this information accurately, consider several parameters to ensure precise data representation:

- **Scale:** What increments does your variable require? Is it sufficient to increase stepwise from 1 to 2 to 3, or is finer precision necessary?
- **Offset:** Is there a need to adjust the baseline value?
- **Most Significant Bit (MSB):** Are you utilizing all available data bits to store information?
- **Least Significant Bit (LSB):** This also concerns the use of data bits for information storage.

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

This setup ensures accurate and reliable data transmission critical for high-speed aviation.


So, lets start by calculating the encoded value:

$$ Val_{E} = \frac{value - offset}{scale} = \frac{200 - 0}{0.5} = 400 $$

Cool, now we can start the encoding process. 

1. The MSB is 28. Therefore, the 29 bit will be set as zero for padding.
2. The encoded value is 400 that is 0b110010000( 9 bits). We have  28-17=11  bits for our data field so we have extra two bits( yay!)
3. The SSM is 3. So 0b11.
4. The SDI is zero. So 0b00
5. Our label is 205(octal ). So it is 0b110010000
6. Our parity bit need to be 1( to be odd)

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
  <tr style="font-size: 86%; text-align: center;">
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
  <tr style="font-size: 86%; text-align: center;">
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
<table class="wikitable" style="border: none; margin-right: auto; margin-left: auto; display: block; width: max-content;">
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
  <tr style="font-size: 86%; text-align: center;">
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
### 3.2 Encoding discrete information (Discrete)
The example here is simpler. Let's say that you flight display needs to notify you what kind of fuel tank your aircraft using.

### 3.3 Encoding integers (Binary Coded Information(*BCD*))

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

