---
layout: post
title: An ADS-B thought
subtitle: A security issue with ADS-B
tags: [ads-b, security]
comments: true
---  

Since the introduction of Automatic Dependent Surveillance-Broadcast (ADS-B) in the aviation industry, we have gained access to real-time information about how aircraft in the sky are organized. We have access to their position, speed, altitude, and other parameters that allow us to infer a lot of information from aircraft. However, in the wrong hands, this information could pose a security risk.

In case you're not familiar with how ADS-B works, it is a system that continuously sends information about an aircraft's situation. The aircraft itself "transmits" the information to a ground station, rather than using a Secondary Surveillance Radar (SSR) or Primary Surveillance Radar (PSR) to calculate the aircraft's position independently. For a more detailed explanation of how ADS-B works, I recommend reading "The 1090 Mhz Riddle" by Junzi Sun.

The main issue with this system is that the signal is not signed or encrypted, which means that someone could potentially falsify the signal. For example, if an individual were to create their own simulated signal using a cheap and easy-to-make ADS-B transmitter, they could falsify the position, altitude, timestamp, and registration of an aircraft, leading to confusion and potentially dangerous situations.

Another potential security concern is that it is possible to locate military aircraft. Normally, when they go on a mission, they can turn off their ADS-B transmitter to become invisible to civilian radar, but during maneuvers and other exercises, the transmitters are activated. This could be exploited by domestic terrorists to track military aircraft.

In conclusion, ADS-B is not as secure as it should be. One potential solution to many of its flaws would be to use asymmetric cryptography to digitally sign the messages. Every aircraft would have its own private key, and each ADS-B message could include a block containing the signature. By having the public key of the aircraft, the message could be verified. However, this would require a larger message size and a new protocol to be developed.
