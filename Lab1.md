# Lab 1

## Table of Contents
- [Overview](#overview)
- [Task 1](#task-1)
- [Task 2](#task-2)

## Overview
This module is about using an FPGA-based System on a Chip, to deliver an embedded system at the end of the module.

This module uses a DE-10 Lite, using Quartus to deliver the bitstream to the FPGA, and Eclipse to write the software running on the board.

## Task 1
First things first I clicked New Project Wizard on the Quartus homescreen and started task 1.

![task1_1](images/Lab1/task1/task1_1.png)

Next up after clicking finish, I then need to select the correct FPGA device for the DE10-Lite which is 10M50DAF484C7G (Interpretation of the device code: 10M is Max10 family. 50 is the size of the device of around 50,000 logic elements. 484 is the number of pins. C7 is the speed grade.)

![task1_2](images/Lab1/task1/task1_2.png)

Next I click File -> New

![task1_3](images/Lab1/task1/task1_3.png)

Then select Verilog HDL File and click OK.

![task1_4](images/Lab1/task1/task1_4.png)

Then writing out the Verilog code into the window, then saving the file as hexto7seg.v.

![task1_5](images/Lab1/task1/task1_5.png)

To check if the file is free from any syntax errors, we can use Processing -> Analyze Current File.

![task1_6](images/Lab1/task1/task1_6.png)

The file is free from any errors, (Quartus produces a lot of useless warnings, most can be safely ignored).

![task1_7](images/Lab1/task1/task1_7.png)

Next up is making the top file for the system as seen below, HEX0 represents the pins connected to the 1st seven segment display, and SW represents 4 of the switches on the board from SW0-SW3.

![task1_8](images/Lab1/task1/task1_8.png)

Next up we need to select this file as the top level entity for this project by selecting Project -> Set as Top-Level Entity.

![task1_9](images/Lab1/task1/task1_9.png)

Then to check for any errors. I clicked Processing -> Start -> Start Analysis & Elaboration.

![task1_10](images/Lab1/task1/task1_10.png)
![task1_11](images/Lab1/task1/task1_11.png)

Then I went and used Assignments -> Pin Planner to select the I/O Standard (what kind of IO these pins use), and the Pin Location of the 7-segment display and Switches.

![task1_12](images/Lab1/task1/task1_12.png)
![task1_13](images/Lab1/task1/task1_13.png)

The result is seen below in the form of an SDC file containing the information about all the pins selected in Pin Planner.

![task1_14](images/Lab1/task1/task1_14.png)

Next up is using Compile Design on the left handside to compile the design into a bitstream that can be programmed onto the FPGA.

![task1_15](images/Lab1/task1/task1_15.png)

Then comes the time to flash the design onto the FPGA.

![task1_16](images/Lab1/task1/task1_16.png)

and as seen below it works!

![task1_17](images/Lab1/task1/task1_17.jpg)

## Task 2
To see the design's logic as compiled, I clicked Tools -> Netlist Views -> RTL Viewer. I then expanded the hex_to_7seg block by clicking on the plus sign.
![task2_1](images/Lab1/task2/task2_1.png)

As seen above a decoder splits up the 4-bit signal from the switch and each individual N-bit OR gate drives an individual line on the 7-segment display.

Below is the Logic Element view of the design.

![task2_2](images/Lab1/task2/task2_2.png)

There are 7 LEs on display each mapping to an individual line. This matches the code as the code can be split up into independent 4to1 multiplexers, which can fit into exactly one LE. (LEs can be thought of as 4 to 1 multiplexers).

The below image is the picture of the propogation delay from the Input to Output in nanoseconds. As seen here it takes around 8.5-9.5 nanoseconds for the input to match the output for each 7-segment line.

(This is the Slow 85C model).

![task2_3](images/Lab1/task2/task2_3.png)

(This is the Slow 0C model).

![task2_4](images/Lab1/task2/task2_4.png)

(This is the Fast 0C model).

![task2_5](images/Lab1/task2/task2_5.png)

In conclusion, the lower the temperature the FPGA is running at the faster the signals stabilise and input matches output. Lower temparture <-> Lower Propogation Delay.

Then I instantiated the hex_to_7seg module 2 more times to drive HEX1 and HEX2, to produce the following design.
![task2_6](images/Lab1/task2/task2_6.jpg)
