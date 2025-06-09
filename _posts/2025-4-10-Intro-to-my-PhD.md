---
layout: post
title:  "Intro to my PhD: Building a Real-Time FPGA Control System for Fluid Dynamics"
subtitle: "Bridging Aerospace Engineering and Digital Hardware Design"
tags: [verilog,fpga,cfd,control,aerospace]
comments: true
---

# 1. Introduction

Recently, I started my PhD and with that the journey of learning on how to play with FPGAs. My PhD has a double focus:
- Aerospace: Active Flow Control
- Hardware: FPGA/MCU to accelerate this control

Therefore, I am in a weird spot that is forcing me to become an expert in combining both a highly specialized domain as fluid simulations and the incredible( and tough) world of digital design.

As someone said, Rome wasn't build in a day. Same is applicable to my expertise lol.

The first thing that I had to do (apart from learning how to code and think in Verilog) was to see what was the easiest model to implement to achive some kind of control of a simulation
output going into an FPGA.

To do this, I had(more like we, thanks Rodrigo) a couple of questions arised:
1. What is the easiest protocol to interact with an FPGA? -> A simple 
2. What is the simplest oscillator that I can control? -> The [Stuart-Landau oscillator](https://en.wikipedia.org/wiki/Stuart%E2%80%93Landau_equation)
3. How can I do arithmetic operations in the FPGA? -> ~~Using IP cores~~ Implement or use an existing [FPU](https://en.wikipedia.org/wiki/Floating-point_unit) implementation
4. How to combine everything to have a real hardware-in-the loop (HIL) setup?

The next sections will anwser the questions.

# 2. Answering the questions 

## 2.1 What is the easiest protocol to interact with an FPGA?

After thinking for bit, I realized that my best option was using an [UART](https://en.wikipedia.org/wiki/Universal_asynchronous_receiver-transmitter) communication protocol.

If you have some experience with electronics and MCUs, this protocol won't hold any mistery.
The UART protocol defines an asynchronous two way communcation that is simple and straight foward.

UART (Universal Asynchronous Receiver-Transmitter) operates with just two signal lines (TX and RX) for bidirectional communication, making it ideal for our initial FPGA interface. Unlike more complex protocols like SPI or I2C, UART doesn't require a clock signal, as timing is managed through predefined baud rates that both devices must agree upon.

But, the fun part with FPGAs is that you need to implement these protocols from scratch. There is not an a cool ```#include <uart.h>``` that we can use like in an ESP32 or an Arduino MCU. 

Apart from that, you need to control the state of the FPGA, and do some kind of arithmetic operation. Seems easy enough, but remember, we have no CPU! So we need to transfer this to a digital logic, or basically, configure the FPGA to use part of its resources as a CPU.

So far, we need the next things:
1. UART Tx module: to transmit information to the laptop
2. UART Rx module: to read information from the lapto

Additionally, we need dedicated buffer modules to handle data aggregation, since UART communication typically transmits only 1 byte of information per frame. This means that when working with single-precision (32-bit) or double-precision (64-bit) floating-point numbers, we'll need to assemble 4 or 8 frames respectively to reconstruct the complete value.

So, now we need an extra two modules:
- UART Rx buffer: stores the variable pieces until we have everything and we can send it to the arithcmetic module
- UART Tx buffer: transforms the result from the arithmetic module into frames that will send in order to the UART Tx module

Moreover, to make sure that the system is reliable, both buffers have a small PIDs implementation to make sure that each frame is well read and serialized


## 2.2 What is the simplest oscillator (equation) that I can control?

In the world of fluid mechanics, there is nothing that can be considered simple. Studying the behavoir of fluids is challenging and something that I deem difficult.
Therefore, I had to consult with my advisor Rodrigo Castellanos, that reccomended me to start with the Landau oscillator. Using the work from [] I finally got the references and the control model that I need.

ADD EQUATION AND MODEL HERE

Yet, here we enocunter the next question:

## 2.3 How can I do aritmetic operations in an FPGA?

Well, this one took me a bit of thinking. If you see the equation to control, there is actually nothing really weird there. Just an easy multiplication and some addition right? Well, that be easy if I had a CPU,
but I dont have access to one here... CPUs have something called Floating-Point Unit (FPU) that basically takes care of the operations related to floating-point numbers. 
Therefore, I want to use floating point units, I need to use an ip-core for FPGA or build my own FPU. OR, the third option, that my other advisor Francisco Barranco, told me, use fixed point arithmetic.

Fixed point arithmetic uses integers instead of floats, so there is no need for an FPU. Altough, we have a limited number of decimal precision. I have added a really good blog post that explains a bit on how to use fixed point arithmethic.

Now, we need to combine everything together:

## 2.4 How to combine everything to have a real HIL setup?

Well, we now have a interesting panorame of the different blocks that I need to build to make all this work. From an FPGA perspective, I have to build 5 blocks ( well, like 6 if I count the top one) and integrate
them with a simulation of the Landau oscillator, easy peasy.

The next sections will elaborate on the whole development of the FPGA blocks and the Python based simulation and interface.

# 3. FPGA: Building, simulating and sythintheisizen the blocks

## 3.1 Building and simulating the blocks

My chosen HDL is Verilog as it is the standard in the USA and I prefer the simplicity. To make things simpler, my modules will be pure verilog with no system verilog (for now). Also, I dont want to use
any IP cores unless necessary.

The five blocks to build are:
1. UART Rx
2. UART Rx PID Buffer
3. Control Law
4. UART TX PID Buffer
6. UART TX

All of this blocks will be referenced from a top module.

Concurrently, once a block is built, it is simulated and tested. To do this, I took advantage of [cocotb](https://www.cocotb.org/) as the Python interface with pytest
is way easier to use tha the verilog test benches.

### 3.1.2 UART Rx module

There is not much mistery in building an UART Rx and there are plenty of examples everywhere. You can check the module here LINK TO MODULE.

My implementation contains a couple of paremeters to change the baud rate, frame size, etc...


<script src="https://gist.github.com/alghanmi/c5d7b761b2c9ab199157.js"></script>

To test, same, you can look at the testing here LINK TO TEST

### 3.1.3 UART Rx PID Buffer module

This one took a bit more of effort. The idea behind is to load the frames in the correct place. I'm sending 32 bit numbers throuhg UART in 4 pieces. 

The number frame looks like this:
START_FRAME| PID | VALUE | END FRAME

The buffer needs to store all these values before sending the signla to the contro law module to read the buffer.

This module was hard to build. At first, I went with an easy implementation that kinda of worked when veryfing it, but when veryfhint the top module
I ran into a lot of race conditions and I had to rebuild the whole thing from top to bottom to make sure that the timing was correct. By far, the most
challening part to build.

### 3.1.4 Control Law module

As explained in the previous section, the control law is super easy to embed. It needs to calculate b which is calcualted:
b = a1*b1 + a2*b2

b1 and b2 are predifined paraemters will a static value(for now) and a1 and a2 are passed from the rx buffer.

Good thing about this block is that is purely combinationals. Therefore, I dont really need to think much about timing. Testing it was easy.

ADD LINKS HERE TOO

### 3.1.5 UART TX PID buffer

This module takes b and transform it into an UART frame as the one on 3.2. Waits for a signal from the RX buffer and then reads, and send the buffer info to the UART Tx module.

ADD LINK TO CODE

### 3.1.6 UART Tx

As the UART Rx, easy module and you can find a lot of examples online showcasing how to build this module

## 3.2 Verifying the top module

After weeks of trial and error, I finalized the blocks and started working on uniting everything together.
The verilog implementation does not have much of a mistery:
ADD CODE

The issue was the verification, if you check the test script, I had to test the normal operation mode and the test mode. Yes, I add a test mode to make sure that at least
the thing lives before trying to make it run with the simulation working on the background.

## 3.3 Synthesazing the modules

After the verification was complete, the next step was to load the code into the FPGA. To do this, first, the modules had to be transformed
into a binary containg the mapping of the logic cells into the FPGA.

ADD IMAGE HERE FROM THE POSTER

This is where another problem arised, if you see the figure under this text, you can see that I have an utilization of over 100% of some of the FPGA resources which

translates into:
a) You need an FPGA with more resources
b) You need to optimize your blocks

ADD IMAGE OF THE UTILIZATION HERE

I went with a), mainly because I tryed to optimize the code but I was unsecsfull and it was way too much work to figure out how use less registers.

With this solved, I was ready to start with the simulation!


# 4. Simulating: Running a Python based simulation and interfacing with the hardware

The FPGA part is done(lol, not really) and we can jump into the sim part.
By using some external libraries like py-serial and the famous scipy, I was able to create a nice code that was capable of solving the EDO for the oscillator
and at the same time, it was able to send through UART the result of the steps and read the value of the control law and compute the next step.

## 4.1 Py-Serial: Reading and sending through UART

Pyserial offers the possiblity of reading and sending UART through any of the available COM ports in the computer. This was trivial to implement with a bit of trial and error.

## 4.2 Scipy: Solving the differential equaiton

Scipy contains a good EDO solver that is able to run each step, and get the result of the step. I wrote some code with a couple of threads to ensure that running of this.

## 4.3 Tranforming floats into fixed point

Each step needs to packet the values received into a format called Q16.16 that then it is framed and send through UART and at teh same time, it needs to translate the incoming frames into a float.



# 5. Running the experimente

# 6. Future development

- RL in an FPGA: Training and inference
- Real CFD calculations, start with something simple and build from tehre

# References
[Fixed Point Arithmethic](https://vanhunteradams.com/FixedPoint/FixedPoint.html)
