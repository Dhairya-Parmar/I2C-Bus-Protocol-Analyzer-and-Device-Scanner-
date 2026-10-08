# I2C-Bus-Protocol-Analyzer-and-Device-Scanner-

A Microcontroller Based project for scanning an I2C bus, detecting connected slave devices, identifying their 7-bit addresses and observing basic I2C transactions using a protocol analyzer in **Proteus**. 

The Project is implemented using the **MSP430G2553** microcontroller and demonstrates practical concepts of **i2C communication, device addressing, ACK/NACK detection, and embedded debugging** 

# Project Overview 

This project scans the standard 7-bit I2C address range and determines whether a slave responds to each address. 

when a device acknowledges the address, the microcontroller reports the detected address through a serial terminal. 

*For example*- a **DS1307 RTC** uses the 7-bit address: 0x68 
