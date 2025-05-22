---
title: Building a quadcopter flight controller from scratch, part 0
tags: [FPV, STM32, Embedded]
style: border
color: 
description: Introduction to my programming journey.
---

# Brief description of the project

My goal is to create a functional piece of software, that runs on a STM32 microcontroller and allows me to control, a 5 inch RC quadcopter.

Also, as you can probably notice I'm writing this entry, around half a year into the project, as only now I found it actually worth sharing. I will still try to, present my journey up to this point in some orderly manner, but i can't promise it will be perfect. Thought, I still hope that any person reading this will find it, at least a little bit interesting and maybe even helpful.

# Ok, but why just not use BetaFlight

The purpose of the project is not to make an extremely well flying drone nor to develop software that's in any way better than one found on commercially available flight controllers.

You can say that the actual purpose is not the end goal but the journey itself.
The whole project requires understanding a fairly broad range of topics (mostly DSP, control theory and electronics), most of which perfectly correspond with my current field of study.
Because of that, this project will serve as a way to gather all of the theoretical knowledge, I gathered over last few years and finally put it to some practical use.

# Getting the hardware together

At the depths of my heart I'm mostly a programmer. I do like to stay close to the hardware, developing for embedded or FPGA platforms but most of my experience in in front of the computer. But to write software for controlling a quadcopter, I still need a actual quadcopter. I mean yes I could make a MatLab model and call it a day, but where's the fun in that (I might still make one at later stages of the project, though).

So I have two options, I can either buy a fully functional quad or collect a few parts and build one myself.
The natural choice is to of course build it myself, so lets quickly get over the hardware that I got.

Before you start criticizing the build, yes I could do some more research and maybe find some parts that fit a little better together, but you also need to take into consideration that my budget wasn't too big, and availability of FPV parts in Poland is also kinda low.

## Let's get through some of the parts then
### [Frame](https://www.amazon.pl/dp/B07X9TPC3H?ref=ppx_yo2ov_dt_b_fed_asin_title&th=1)
Just a random cheap, frame from Amazon. The arms on this one are 6" ones, I believe but I don't care that much about weight and some extra space definitely won't hurt.

### Motors
KO Technologies Method R, 2000KV 2 motors. (I only bought them because there were discounted). I should've bought something with lower KV, as I intend to use 6s battery but that's going to be a problem for future me.

I've never heard of the brand, but the motors seem to be fairly well made, for the price.\
Still I noticed some things that I don't really like. When turned by hand the motors are very "notchy". By itself is not really a bad thing, but the strength of those notches is very inconsistent. One of the motors makes 1 or 2 full spins, when turned by hand before stopping while the other doesn't even finnish one. This can indicate some noticeably big manufacturing quality variations in individual products.\
On top of that one of the motors gets noticeably hotter the the others, witch seems like a obvious manufacturing flaw.
Last thing is complete lack of any kind of technical documentation, the only thing i found about theme is the [manufacturer site](https://www.ko-technologies.com/ko-technolgies-method-r), which gives us some information about most important specifications, but I find it very lacking.

### Propellers
HQ Prop R38C - basically one of 2 different props available for me at a time, so I bought the cheaper ones.

### ESC
I went with ReadyToSky, BLHeliS 35A ESC's. And that model was literally the only singe ESC in the entire store, so at least the choice was easy.

I chose to use 4 independent ESC's, as most most of the 4 in 1 models are made to work with, specific FC's and I just wast'n sure how tricky using them will be with a custom one, as the documentation for those ESC's lacks any specific usage info beside how to wire them up to a specific FC of the same brand (and that's sadly a oftenly occurring theme with FPV components).

## And of course the electronics

As my main goal is, to build a flight controller we will need some electronics to run the code, and gather all necessary data. Here again we have two main options. A good one and the easy one.

The best option would of course be to just make a custom PCB and put all the necessary components on it. But that solution poses a few big difficulties for me. First I honestly lack the ability to design one myself, and I find learning that far outside the scope of what this project was supposed to be about. On top of that i don't have any easy access to tools needed for soldiering small SMD components, so making the board would cost me much more than I can let myself spend on it, as I would need to pay somebody else to do it all.

So the obvious option will be the easy one, I can always get back to the PCB if I ever make version 2.\
This solution relies on using well known, and loved development boards. The biggest limitation to that idea is going to be how little space is left on the frame after mounting all previously mentioned components. Because of that I need to limit myself to boards in Nano format. 

## Microcontroller

After some research I found that the best STM32 MCU I can get, for reasonable amount of money is going to be the F411. Of course, it can be found on the BlackPill board.\
I bought a slightly [modified version](https://wiki.kamamilabs.com/index.php?title=KAmod_BlackPill_411) of it, sold by one of Polish shops.

The board should provide enough processing power for all DSP, control and data acquisition algorithms that I intend to use.\
Some noteworthy features that were most important to me in making the choice:
- CPU Clock - 100 Mhz on the f411, important because of how many and how often differnt algoriths will run (like filters and PIDs)

- FPU unit - Necessary if I want to work on floating point numbers

- DMA - Nice to have to offload the main CPU form copying all data between memory and peripherals (Mostly form serial interfaces)

- Serial interfaces - needed to handle communication with RC receiver and IMU.

## IMU

To keep the drone stable I will need to get some feedback about it current state. The solution for that will be whats called an Inertial Measurement Unit, or just IMU. Its a combination of 3-axis rate gyroscope and accelerometer. With that we can measure rotational velocity needed for flying in so called ACRO model. After adding accelerometer data to it we can also estimate the drone attitude to control not only its rotation velocity but also position in relation to ground

I chose the MPU6050 IMU unit, mostly because it's cheap and it does its job.\
Yes it is a noisy, and not the most precise sensor but at lest it will give me more incentive to talk a little bit more about proper digital filter design.

You can find the exact board [here](https://kamami.pl/czujniki-6dof-9dof-10dof/565671-modgy-87-modul-10dof-z-akcelerometrem-zyroskopem-magnetometrem-oraz-barometrem-5906623456154.html), it does contain a few more sensors that I may use some time in the future.


