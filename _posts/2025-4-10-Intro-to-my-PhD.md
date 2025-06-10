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

To do this, I had(more like we, thanks Rodrigo and Francisco) a couple of questions arised:
1. What is the easiest protocol to interact with an FPGA? 
2. What is the simplest oscillator that I can control? 
3. How can I do arithmetic operations in the FPGA? 
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
Therefore, I had to consult with my advisor Rodrigo Castellanos, that reccomended me to start with the Landau oscillator. Using the work from Isaac Robledo [1] I finally got the references and the control model that I need.

The selected model is the function that represents the oscillatory motion of the von Karman vortex shedding behind a cylinder:

$$
\begin{cases}
\dot{a}_1 = (1 - a_1^2 - a_2^2) a_1 - a_2, \\
\dot{a}_2 = (1 - a_1^2 - a_2^2) a_2 + a_1 + b(a_1, a_2),
\end{cases}
$$

The control is going to be characterize by the function b, which it will be calculated as:

$$
b(a_1,a_2) = a_1 \cdot b_1 + a_2 \cdot b_2
$$

I'll get in more detail in the next section.
    
## 2.3 How can I do arithmetic operations in an FPGA?

This challenge required a fundamental shift in my thinking. Looking at the Landau oscillator equations, you might think: "It's just multiplication and addition—how hard could it be?" But there's a critical difference between software and hardware implementations.

In a CPU, floating-point operations are handled by dedicated Floating-Point Units (FPUs) that efficiently manage the complex IEEE-754 standard with its mantissa, exponent, and sign bit. On an FPGA, I had three options:

1. **Use an IP core**: Many FPGA vendors offer floating-point IP cores, but these are often proprietary, resource-intensive, and limit portability.

2. **Build my own FPU**: Theoretically possible, but would require significant development time and FPGA resources that could be better allocated to the control system itself.

3. **Use fixed-point arithmetic**: The approach recommended by my advisor Francisco Barranco, which offers an elegant compromise.

### Fixed-Point Arithmetic: The Elegant Solution

Fixed-point representation uses integers to approximate real numbers by implicitly placing a decimal point at a predetermined position. For example, in a Q16.16 format:

- 16 bits for the integer part
- 16 bits for the fractional part
- Total: 32 bits (standard integer width)

This approach offers several advantages for my FPGA implementation:

- **Resource efficiency**: Uses standard integer DSP blocks already available in the FPGA
- **Deterministic timing**: Operations complete in a fixed number of clock cycles
- **Predictable precision**: Error bounds can be mathematically determined
- **Simpler implementation**: Requires only shifts, additions, and multiplications

The trade-off is reduced dynamic range and precision compared to floating-point, but for my control application, a Q16.16 format provides sufficient precision (±0.0000152 resolution) while maintaining a reasonable range (±32,768).

### Implementation Challenges

Converting the Landau oscillator equations to fixed-point wasn't trivial. Key considerations included:

1. **Overflow prevention**: The term $$(1 - a_1^2 - a_2^2)$$ could potentially overflow during intermediate calculations
2. **Multiplication precision**: Products of two Q16.16 numbers result in Q32.32 format, requiring careful scaling
3. **Division operations**: These are resource-intensive and were avoided through mathematical reformulation

The resulting arithmetic module processes the control law $$b(a_1,a_2) = a_1 \cdot b_1 + a_2 \cdot b_2$$ in just 4 clock cycles, achieving the performance needed for real-time control.

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

There is not much mistery in building an UART Rx and there are plenty of examples everywhere. You can check the module [here](https://github.com/jaimebw/verilog_modules/blob/main/src/uart_rx.v) My implementation contains a couple of paremeters to change the baud rate, frame size, etc...

The verification phase is done using coco-tb, you can check my test bench script [here](https://github.com/jaimebw/verilog_modules/blob/main/tb/test_uart_rx.py)

### 3.1.3 UART Rx PID Buffer module

This module, UartRxPidBuffer, is responsible for reconstructing two 32-bit fixed-point control inputs (a1 and a2) from UART packets using a custom framing protocol. Each value is transmitted in four separate packets, and the expected structure is:
START_FRAME | PID | VALUE | END_FRAME.

Each byte is associated with a specific PID that identifies which part of a1 or a2 it belongs to. The module uses a finite state machine to parse incoming bytes, capture valid data, and assert a one-cycle ready pulse once both a1 and a2 are fully assembled. A special TEST_PID is handled separately, allowing for quick injection of test data.

Building this module was particularly challenging. My initial approach worked in isolated tests, but integration with the top-level design exposed race conditions and incorrect timing behavior. I had to completely redesign the FSM, add explicit tracking of received byte lanes (a1_written, a2_written), and manage the control logic to guarantee a clean, single-cycle handshake. The final design is stable, timing-safe, and cleanly interfaces with the control law block.
<details>
<summary>UART Rx PID Buffer Module (Click to expand)</summary>
<iframe frameborder="0" scrolling="no" style="width:100%; height:3355px;" allow="clipboard-write" src="https://emgithub.com/iframe.html?target=https%3A%2F%2Fgithub.com%2Fjaimebw%2Fverilog_modules%2Fblob%2Fmain%2Fsrc%2Fuart_rx_pid_buffer.v&style=github-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></iframe>
</details>

### 3.1.4 Control Law module

Once the fixed point arithmetic was dealt with, we coul
 
b1 and b2 are predifined paraemters will a static value(for now) and a1 and a2 are passed from the rx buffer.

Good thing about this block is that is purely combinationals. Therefore, I dont really need to think much about timing. Testing it was easy.

<iframe frameborder="0" scrolling="no" style="width:100%; height:814px;" allow="clipboard-write" src="https://emgithub.com/iframe.html?target=https%3A%2F%2Fgithub.com%2Fjaimebw%2Fverilog_modules%2Fblob%2Fmain%2Fsrc%2Fcontrol_law.v&style=github-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on"></iframe>

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

1. Robledo Martin, I. (2025). HyGO: A Python toolbox for Hybrid Genetic Optimization. *Journal of Open Source Software*.

2. [Fixed Point Arithmethic](https://vanhunteradams.com/FixedPoint/FixedPoint.html)

