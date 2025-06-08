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
1. What is the easiest protocol to interact with an FPGA? -> A simple [UART](https://en.wikipedia.org/wiki/Universal_asynchronous_receiver-transmitter) protocol
2. What is the simplest oscillator that I can control? -> The [Stuart-Landau oscillator](https://en.wikipedia.org/wiki/Stuart%E2%80%93Landau_equation)
3. How can I do arithmetic operations in the FPGA? -> ~~Using IP cores~~ Implement or use an existing [FPU](https://en.wikipedia.org/wiki/Floating-point_unit) implementation


With that in mind, I could start thinking about the next steps.
## 1.1 UART interfacing

If you have some experience with electronics and MCUs, this protocol won't hold any mistery.
The UART protocol defines an asynchronous two way communcation that is simple and straight foward.

UART (Universal Asynchronous Receiver-Transmitter) operates with just two signal lines (TX and RX) for bidirectional communication, making it ideal for our initial FPGA interface. Unlike more complex protocols like SPI or I2C, UART doesn't require a clock signal, as timing is managed through predefined baud rates that both devices must agree upon.

But, the fun part with FPGAs is that you need to implement these protocols from scratch. There is not an a cool ```#include <uart.h>``` that we can use like in an ESP32 or an Arduino MCU. 
Apart from that, you need to control the state of the FPGA, and do some kind of arithmetic operation. Seems easy enough, but remember, we have no CPU! So we need to transfer this to a digital logic, or basically, configure the FPGA to use part of its resources as a CPU.

So far, we need the next things:
1. UART Tx module: to transmit information to the laptop
2. UART Rx module: to read information from the lapto

Additionally, we need dedicated buffer modules to handle data aggregation, since UART communication typically transmits only 1 byte of information per frame. This means that when working with single-precision (32-bit) or double-precision (64-bit) floating-point numbers, we'll need to assemble 4 or 8 frames respectively to reconstruct the complete value.


## 1.2 The Stuart-Landau oscillator

In the world of fluid mechanics, 

# References

