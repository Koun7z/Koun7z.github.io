---
title: Building a quadcopter flight controller from scratch [Part 1]
tags: [FPV, STM32, Embedded, ExpressLRS]
style: border
color:
description: Setting up a radio link between ground and the RC quadcopter.
---

# Introduction
Now that I have the drone all built and wired up, I can start writing some code to bring it to life.
First thing that will be necessary is a way to tell the drone what I want it to do. For that purpose, I will use a radio link to control it remotely from the ground.

With the hardware that I'm using ([RadioMaster Pocket](https://www.radiomasterrc.com/collections/pocket-radio/products/pocket-crush-radio-controller?variant=50101307310311) and [RP3 receiver](https://www.radiomasterrc.com/products/rp3-expresslrs-2-4ghz-nano-receiver?variant=46464183107815)), establishing a radio link is as easy as setting the same connection string on both pieces of equipment. The more difficult part comes after, as I need to send all of that data from the receiver to my main control board.

The RP3 receiver has a single UART data interface. It can send data using a few [different protocols](https://www.expresslrs.org/software/serial-protocols/). They differ in types of packets that they can send, transfer speeds, bidirectional communication capabilities, and a few other aspects. I ended up using the CRSF protocol, as it's the most capable and flexible one. It's also very well integrated with EdgeTX firmware that most FPV related radios are using.

# CRSF protocol

This protocol is quite a simple one, so let's quickly get through some of its core characteristics.
More information about the protocol, that I haven't mentioned can be found in the [official documentation](https://github.com/tbs-fpv/tbs-crsf-spec/blob/main/crsf.md), I wouldn't say its more detailed than my explanations though.

> There seem to be two CRSF version (v2 and v3). V2 is the one integrated with ExpressLRS so I will focus mostly on that one.

# Physical layer

As mentioned before, the receiver uses UART as its serial interface. It can communicate in both full and half duplex modes.\
Most devices use baud rates ranging from 115.2k to 5.25M bps. There are no parity checks and a single stop bit is used.

> CRSF can also use I2C as its physical layer, but I haven't seen any actual hardware using it.

# Frame format

Each frame starts with a sync byte `0xC8` (official documentation states that EdgeTX's outgoing channel/telemetry packets start with `0xEE` byte, but I never encountered it during my testing despite using a handset running EdgeTX).

The second byte informs us about the length of the message (including the frame type and CRC bytes). We need the data length field as different packet types will have different lengths. The maximum length of the packet is 64 bytes, so we expect a value no bigger than 62 here.

The third byte contains the frame type. It tells us what kind of data is in the following bytes.

Next comes the actual data, which is the part of the message we're most interested about. The frame is finalized with an 8-bit CRC value calculated using `0xD5` polynomial.
<div style="text-align: center;">
  <table class="center-table">

    <thead>
      <tr>
        <th>Sync</th><th>Message Length</th><th>Frame Type</th><th>Data[0]</th><th>...</th><th>...Data[N]</th><th>CRC</th>
      </tr>
    </thead>
  </table>
  <div> Fig. 1: CRSF frame structure</div>
</div>

> There's also a extended frame type used by CRSF v3. It uses two first bytes of the data section for specifying destination and source of the packet. I never had a need to use any features that require this extended format though.

# Sending data to the quadcopter

To pilot the quadcopter we need to send it control commands, for example desired throttle value or pitch rate/value based on flying mode. Giving it some information about link quality would also be nice, it's not necessary, but it allows for implementation of some additional safety features.

To do that we will need to handle exactly two frame types:
- [**RC Channels Packed**](https://github.com/crsf-wg/crsf/wiki/CRSF_FRAMETYPE_RC_CHANNELS_PACKED) - Which is basically a packed struct of 16, 11-bit unsigned integers containing values of corresponding channels.

> CRSF v3 uses a different packet type for sending channels data. It's called Subset RC Channels Packed and as the name suggest, it doesn't send all channels at ones but splits them over multiple frames.

- [**Link Statistics**](https://github.com/crsf-wg/crsf/wiki/CRSF_FRAMETYPE_LINK_STATISTICS) - Another packed struct, this time constructed out of 8-bit integers of signed and unsigned variety. In this packet we can find information about packet loss, RSSI, SNR, etc. for both uplink and down link.

>Although I'm focusing mostly on those two, there are many more frame types in the protocol. Description for all of them can be found [here](https://github.com/crsf-wg/crsf/wiki/Packet-Types).

## Establishing project requirements

Now that we know what can be expected on the serial interface, we can write some code to handle it.\
All software used in this project will be written to run on STM32 MCU's and use the HAL library. Using HAL simplifies a lot of steps like handling peripheral or interrupts, while still allowing for a lot of control over the hardware. On top of that it allows me to create a common codebase that will work on most STM32 MCU's without a need to alter code.

When interfacing with any peripherals through HAL, we can choose from 3 different approaches.
- **Blocking mode** - In this mode, the CPU waits until the peripheral finishes its job. It means that program execution is stopped until, for example UART peripheral receives all specified bytes. It's a big drawback of this mode, especially when receiving many or long massages. There are some upsides to this mode though, all communication is basically a single function call, and we don't need to bother with interrupts or race conditions.

- **Interrupt mode** - In this mode, CPU tells the peripheral to start doing its job and goes back to executing code normally. Later when the peripheral is done it triggers an interrupt to tell the CPU that it can come back and get the results. At this moment the CPU stops normal code execution, handles the interrupt and return to normal code execution. This means that during the time which the peripheral needs to complete its job, the CPU can do something else. It's great, as it doesn't waste any time just waiting, but as with any type of asynchronous operation it introduces race conditions to the program that we need to be wary of. It also complicates code a bit more.


> When receiving through UART in interrupt mode, an interrupt is triggered after receiving each byte of the message, so that the CPU can copy it to memory. HAL library handles those interrupts for us, and only triggers the transfer complete callback after the whole message has been received, but it's still good to know that the CPU does all of that work.


- **DMA mode** - This mode works very similarly to the IT mode, with a single exception that makes a big difference. In this mode, all data transfer between memory and peripherals is handled by a dedicated coprocessor called DMA unit([Every stm32 microcontroller should have at least one of those](https://www.st.com/resource/en/application_note/an2548-introduction-to-dma-controller-for-stm32-mcus-stmicroelectronics.pdf)). This means that the CPU no longer wastes time on copying data between those two. It can focus on doing different tasks in the meantime and, after receiving an interrupt return to find all data already in the memory. This mode of operation works perfectly for any jobs requiring transfer of big amounts of data. The only downside of it, that I can think of right now, is the fact that we need to use another peripheral (assuming that the MCU has one) and that's another hardware resource that we need to allocate for a specific task. So it's best to use it when we actually need it.

So, knowing all of that we can make a decision about which mode to use. ExpressLRS can send from 50 to 1000 packets per second, which should mean exactly this many Channels Packed packets (each 26 bytes long) received. This gives us around 1.3 kB to 16 kB of data per second. On top of that, there should be some Link Statistics packets mixed in.

Sending a single Channels Packed packet with baud rate of 115200 b/s takes around 80-90 &micro;s. We can increase the baud rate to reduce this time, but that will have a negative effect on connection stability and noise rejection. With this in mind receiving in blocking mode, using the slowest packet rate of 50Hz would take over 10% of total CPU time not even mentioning any CRC calculations and data parsing that need to be done after. This leaves us with only two options left, and after considering the amounts of data that needs to be copied, using DMA seems to be the most sensible idea.

## Peripheral configuration
As I expect that I or maybe someone else will use this code in some other projects, I want to make it into an independent library to make it reusable. HAL library requires you to include platform dependent drivers into your project. My CRSF library will expect that those drivers will be located in the main project. Then it can be included in source code form and built together using the same drivers.

Let's start with setting up the main project, then. Its main structure and configuration can be writen by hand, but I will use this handy tool called [CubeMX](https://www.st.com/en/development-tools/stm32cubemx.html). Basically, it's a graphical interface for setting up STM32 projects. It will download all required drivers and create a basic project structure. ST microelectronics also provides a dedicated IDE for programming STM32 MCUs which makes it even simpler, but I will stick to using CMake and CLion as I find Eclipse based editors annoying to use.

There are a few things needed for configuration before we can generate the code. First I need to select MCU model (for me, it's F411CEUx). Next is the UART peripheral, and It needs quite a lot of configuration compared to what you may be used to with platforms like Arduino.\
Firstly:
- Baud Rate    - I eventually went with 420k (default expressLRS receiver value)
- Word Length  - 8 bit
- Parity Check - No
- Stop bits    - 1

We also need to enable the UART global interrupt, as well as add DMA requests (only to RX, for now).
>There are also some advanced options like oversampling, overrun detection or even auto baud selection. There may or may not be available on specific MCUs.

Next, it would be nice to see what we're actually receiving, so let's use the usb interface and configure it to work in device mode. For it to work, we also need to add additional middleware - `USB_DEVICE` and configure it as Virtual Port Com.

Last thing is the CPU clock, it can also be adjusted. The board that I'm using has an external oscillator, so let's use that as the clock source. It will always be more precise than the internal one. With all configuration done, I can finally generate the project files.

> At this moment in configuration I found a small oversight in the board design. I need to supply 48MHz signal for USB communication and I would also like to run the board on its full design clock speed - 100Mhz. The problem  is that I cannot configure the PLL in a way to achieve both of those frequencies (CubeMX solver also gave up) when using external 25Mhz clock. For now I will need to settle for 96Mhz main clock then.

Schematic of the assembled circuit can be seen on Fig 2. VCC input on the board accepts 5V. It powers the board through an internal voltage regulator, it can also be used to distribute voltage to other devices (assuming they don't draw too much current), like our receiver for example. VCC and GND pins are also connected to USB port so we can power the board through it. Those devices don't draw much current, so powering them from a PC shouldn't be a problem, but I'm still using an isolated hub just in case. 
{% include elements/figure.html image="\assets\images\CRSF_schematic.png" caption="Fig 2. Electrical connection schematic" %}

## Finally some code

The first problem that we will need to solve is the fact that the messages will be variable length, so we cannot just tell the UART to read N bytes and be done with it. Luckily, HAL provides us with a simple solution - Idle Line Detection API. It's a set of functions that will read UART data until an idle event is detected on data line or specified buffer is filled (max length packet). Idle event occurs when transmission stops and the data line remains pulled high for at lest 1 UART frame.

Reception initialization looks like this:

```c
// CRSF_Connection.c

UART_HandleTypeDef* uart;
uint8_t CRSF_RX_Buffer[64];

void CRSF_Init(UART_HandleTypeDef* huart)
{
    uart = huart;

    HAL_UARTEx_ReceiveToIdle_DMA(uart, CRSF_RX_Buffer, 64);
    __HAL_DMA_DISABLE_IT(uart->hdmarx, DMA_IT_HT);
}
```

Before receiving any data
Reception begins through the `HAL_UARTEx_ReceiveToIdle_DMA` function. After this call, when the next CRSF frame finishes transmitting we will get an interrupt related to that event. The last line in the function disables the DMA Half Complete interrupt, as it's handled through the same callback. It triggers when half of the specified buffer gets filled. It's useful when working with circular buffers, but as I expect fairly long pauses between frames this kind of buffer is not necessary.

Now we need to handle the UART callbacks. Based on the message length, we can expect either `HAL_UARTEx_RxEventCallback` or `HAL_UART_RxCpltCallback` (the latter will trigger only after receiving exactly 64 bytes).

Let's define a `CRSF_HandleRX` function:
```c
// CRSF_Connection.c

// RX buffer
#define CRSF_RX_SYNC_BYTE  CRSF_RX_Buffer[0]
#define CRSF_RX_MSG_LEN    CRSF_RX_Buffer[1]
#define CRSF_RX_FRAME_TYPE CRSF_RX_Buffer[2]
#define CRSF_RX_DATA_BEGIN &(CRSF_RX_Buffer[3])
#define CRSF_RX_DATA_LEN   (CRSF_RX_MSG_LEN - 2)
#define CRSF_RX_CRC_BEGIN  &(CRSF_RX_Buffer[2])

.
.
.

void CRSF_HandleRX(void)
{
    if(CRSF_RX_SYNC_BYTE != CRSF_SYNC_DEFAULT && CRSF_RX_SYNC_BYTE != CRSF_SYNC_EDGE_TX)
    {
        HAL_UARTEx_ReceiveToIdle_DMA(uart, CRSF_RX_Buffer, 64);
        __HAL_DMA_DISABLE_IT(uart->hdmarx, DMA_IT_HT);
        return;
    }

    // Check CRC
    // Parse data

    printf("T: %u, Sync: 0x%hhx, Len: %hhu, Type: 0x%hhx\n\r", HAL_GetTick(),
           CRSF_RX_SYNC_BYTE, CRSF_RX_MSG_LEN, CRSF_RX_FRAME_TYPE);

    HAL_UARTEx_ReceiveToIdle_DMA(uart, CRSF_RX_Buffer, 64);
    __HAL_DMA_DISABLE_IT(uart->hdmarx, DMA_IT_HT);
}
```

The only check that I'm doing, is checking if the frame starts with a correct value. If not, I discard the frame and restart the reception. Instead of a proper parsing, let's just print some values to the console for now.

Callbacks will be handled in the main file:
```c
// main.c
#include "main.h"
#include "usb_device.h"
#include "usbd_cdc_if.h"
#include "CRSF_Connection.h"

#include <stdbool.h>


volatile NewRCData = false;

int _write(int file, char* ptr, int len)
{
    CDC_Transmit_FS(ptr, len);
    return len;
}

void HAL_UARTEx_RxEventCallback(UART_HandleTypeDef* huart, uint16_t Size)
{
    NewRCData = true;
}

void HAL_UART_RxCpltCallback(UART_HandleTypeDef* huart)
{
    NewRCData = true;
}

int main(void)
{
    /*
    **
    ** Some HAL init stuff should be here
    **
    */

    while(1)
    {
        if (NewRcData)
        {
            NewRCData = false;
            CRSF_HandleRX()
        }
    }
}
```

> This codeblock skips most of the CubeMX autogenerated code as it would make this post twice as long if I included it in.

A good practice when working with interrupts is to do as little work inside the actual interrupt as possible. The goal of that is to not delay the execution of other interrupts, if they happened one after another. Arguably this is not that big of a problem on Cortex-M because of the NVIC, which allows for setting priorities for the interrupts, but let's still abide by this rule.\
Both `Event` and `Cplt` callbacks set the `NewRCData` flag to true. Inside `main`, when the flag is true received data is processed and the flag is cleared.\
`NewRCData` variable is marked as volatile to ensure that the compiler won't just optimize it out. From the compiler point of view, the `NewRCData` flag is false and the functions that set it to true are unreachable from `main`. Because of that, it may assume that deleting the whole if statement won't change anything, as the statement would evaluate to false anyway. I don't think I ever encountered an issue like that even when compiling with `-O3` or `-Ofast` flags, but knowing about bugs like these can save you many hours on debugging in the future if they eventually happen.

Last interesting thing about this code is the usage of `printf()` statement to send data through USB interface. The default implementation of `_write` in the GCC standard C library is defined with the `weak` attribute, making it easy to overwrite by simply declaring our own implementation. The `printf` function in the library calls `_write` after correctly formatting the message.

The only thing left now is to wire everything up and see if the code works. For the first test, I will just power up the receiver without connecting my radio to it and see if it sends any data. Console output can be found below. I also added one more column to the output, which display time passed since receiving a packet of that type. All timings should be interpreted as time from MCU power up in milliseconds, with accuracy of $\pm1 ms$.

```console
Console output with no radio link:
T: 17866, Sync: c8, Len: 12, Type: 14, dT: 17866
T: 17967, Sync: c8, Len: 12, Type: 14, dT: 101
T: 21388, Sync: c8, Len: 12, Type: 14, dT: 3421
T: 21488, Sync: c8, Len: 12, Type: 14, dT: 100
T: 21589, Sync: c8, Len: 12, Type: 14, dT: 101
T: 21690, Sync: c8, Len: 12, Type: 14, dT: 101
T: 21918, Sync: c8, Len: 12, Type: 14, dT: 228
T: 22018, Sync: c8, Len: 12, Type: 14, dT: 100
T: 22119, Sync: c8, Len: 12, Type: 14, dT: 101
T: 22220, Sync: c8, Len: 12, Type: 14, dT: 101
T: 22321, Sync: c8, Len: 12, Type: 14, dT: 101
T: 22422, Sync: c8, Len: 12, Type: 14, dT: 101
T: 22977, Sync: c8, Len: 12, Type: 14, dT: 555
T: 23077, Sync: c8, Len: 12, Type: 14, dT: 100
T: 24035, Sync: c8, Len: 12, Type: 14, dT: 958
T: 24135, Sync: c8, Len: 12, Type: 14, dT: 100
T: 25444, Sync: c8, Len: 12, Type: 14, dT: 1309
T: 25544, Sync: c8, Len: 12, Type: 14, dT: 100
T: 27791, Sync: c8, Len: 12, Type: 14, dT: 2247
T: 27891, Sync: c8, Len: 12, Type: 14, dT: 100
T: 31312, Sync: c8, Len: 12, Type: 14, dT: 3421
T: 31412, Sync: c8, Len: 12, Type: 14, dT: 100
T: 34833, Sync: c8, Len: 12, Type: 14, dT: 3421
T: 34934, Sync: c8, Len: 12, Type: 14, dT: 101
T: 35035, Sync: c8, Len: 12, Type: 14, dT: 101
T: 35136, Sync: c8, Len: 12, Type: 14, dT: 101
T: 35363, Sync: c8, Len: 12, Type: 14, dT: 227
T: 35464, Sync: c8, Len: 12, Type: 14, dT: 101
T: 35564, Sync: c8, Len: 12, Type: 14, dT: 100
.
.
.
T: 54782, Sync: c8, Len: 12, Type: 14, dT: 100
T: 58203, Sync: c8, Len: 12, Type: 14, dT: 3421
T: 58303, Sync: c8, Len: 12, Type: 14, dT: 100
```
This particular receiver starts transmitting after around 18 second after power up. The only frame type sent is the Link Statistics packet (0x14). The timing of the frames may seem random at first, but this patters can actually tell us a bit about how the receiver works internally. Firstly, the timing pattern on the frames is consistent between all power ups. Lowest timing interval is 100ms and the highest consistently around 3421 ms. The standard Link Statistics reporting rate seems to be 10 times per second (every 100ms) and every time there's a bigger delay the receiver is probably busy with something else. The most likely answer is that the receiver tries to find a transmitter to connect to. At first, it tries for a short period of time (around 100ms). After failing, it tries for longer and longer. After reaching the ~3.5s mark twice, it goes back to the short 100ms period. Sadly, this experiment won't tell us anything about what's happening before the 18s mark.

Let's establish the radio link and see how the output changes. In the ExpressLRS script on my radio, I've set the packet rate at 50Hz and turned off telemetry for now.
```console
Console output with radio link established:
T: 3776, Sync: c8, Len: 12, Type: 14, dT: 3180
T: 6996, Sync: c8, Len: 12, Type: 14, dT: 3220
T: 7056, Sync: c8, Len: 24, Type: 16, dT: 7056
T: 7076, Sync: c8, Len: 24, Type: 16, dT: 20
T: 7096, Sync: c8, Len: 24, Type: 16, dT: 20
T: 7097, Sync: c8, Len: 12, Type: 14, dT: 101
T: 7116, Sync: c8, Len: 24, Type: 16, dT: 20
T: 7136, Sync: c8, Len: 24, Type: 16, dT: 20
T: 7156, Sync: c8, Len: 24, Type: 16, dT: 20
T: 7176, Sync: c8, Len: 24, Type: 16, dT: 20
T: 7196, Sync: c8, Len: 24, Type: 16, dT: 20
T: 7198, Sync: c8, Len: 12, Type: 14, dT: 101
T: 7216, Sync: c8, Len: 24, Type: 16, dT: 20
T: 7236, Sync: c8, Len: 24, Type: 16, dT: 20
T: 7256, Sync: c8, Len: 24, Type: 16, dT: 20
T: 7276, Sync: c8, Len: 24, Type: 16, dT: 20
T: 7296, Sync: c8, Len: 24, Type: 16, dT: 20
T: 7299, Sync: c8, Len: 12, Type: 14, dT: 101
T: 7316, Sync: c8, Len: 24, Type: 16, dT: 20
T: 7336, Sync: c8, Len: 24, Type: 16, dT: 20
T: 7356, Sync: c8, Len: 24, Type: 16, dT: 20
T: 7376, Sync: c8, Len: 24, Type: 16, dT: 20
T: 7396, Sync: c8, Len: 24, Type: 16, dT: 20
T: 7400, Sync: c8, Len: 12, Type: 14, dT: 101
T: 7416, Sync: c8, Len: 24, Type: 16, dT: 20
T: 7436, Sync: c8, Len: 24, Type: 16, dT: 20
T: 7456, Sync: c8, Len: 24, Type: 16, dT: 20
T: 7476, Sync: c8, Len: 24, Type: 16, dT: 20
T: 7496, Sync: c8, Len: 24, Type: 16, dT: 20
T: 7501, Sync: c8, Len: 12, Type: 14, dT: 101
T: 7516, Sync: c8, Len: 24, Type: 16, dT: 20
T: 7536, Sync: c8, Len: 24, Type: 16, dT: 20
T: 7556, Sync: c8, Len: 24, Type: 16, dT: 20
T: 7576, Sync: c8, Len: 24, Type: 16, dT: 20
T: 7596, Sync: c8, Len: 24, Type: 16, dT: 20
T: 7602, Sync: c8, Len: 12, Type: 14, dT: 101
```

This time first packet came in after just ~3s from the power up, this number is also fairly consistent between a few tries. This means that the receiver tries to establish connection for some time before it starts reporting anything. After that it does some more work establishing the link. It takes another 3 seconds before sending next Link Statistics packet and another second before we finally get out first packet of different type. It has a code of 0x16 which indicates the Channels packed packet. After that, we can see that the channel value packets are sent in equal intervals every 20ms which corresponds with the handset side packet rate setting. We can also tell that the Link Statistics are sent every ~100ms. That value is independent of the set packet rate and I didn't stumble upon a way to change it (I didn't look for it too hard though).

## Error handling

It seems we have a working program, so let's start interpreting the data sent to us. After we do one more important thing first. As mentioned before, the `HAL_UARTEx_ReceiveToIdle_DMA` function eventually triggers either `event` or `cplt` callback, but that assumes the transmission actually completes without any problems. In HAL, when a peripheral encounters any error state it halts its operation, and raises an error callback so we can handle it accordingly. In our case, if at any point the communication fails (due to noise, overrun, timing issues, etc.) the reception will stop and the `HAL_UART_ErrorCallback` will be raised. When that happens, we need to manually restart the communication if we want to receive any more data.

To handle those errors, I added a dedicated function in my CRSF library. I also added the `receptionReset` function to wrap the reception initialization and reduce code repetitions.

```c
// CRSF_Connection.c

#if SERIAL_DEBUG
#  include <stdio.h>
#  define DEBUG_LOG(...) printf(__VA_ARGS__)
#else
#  define DEBUG_LOG(...)
#endif

static void receptionReset(void)
{
    HAL_UARTEx_ReceiveToIdle_DMA(uart, CRSF_RX_Buffer, 64);
    __HAL_DMA_DISABLE_IT(uart->hdmarx, DMA_IT_HT);
}

void CRSF_HandleErr(void)
{
	DEBUG_LOG("err: %lu\n\r", uart->ErrorCode);
	receptionReset();
}
```
The only thing left now, is to call the function after the error callback. This time I've put it inside the callback directly, as this function doesn't do any complicated logic or calculations. I also added an option to turn the serial debugging on and off, as we don't want it slowing down our program during normal operation.

```c
// main.c
void HAL_UART_ErrorCallback(UART_HandleTypeDef* huart)
{
    CRSF_HandleErr();
}
```
---
Now we can handle errors recognized by the UART peripheral. But if we don't get one, it doesn't automatically mean that the received data is correct. Due to electromagnetic interference some bits could get flipped during transmission and due to that the data that we've read is different from one sent by the receiver.

To detect those errors, we use what's called a Cyclic Redundancy Check, or just CRC. The idea behind it is fairly simple. It's basically a polynomial division where our transmitted data is the dividend, and the divisor is what we call a generating polynomial. The generating polynomials are fixed in value and are chosen in such a way that the algorithm has the most chance at detecting most errors. As the result of the algorithm, we take not the quotient but the remainder of the division operation and append it to the end of the message before sending. Later, on the receiving end, we perform the calculation again using the same generating polynomial including the CRC at the end of the message as part of our data. This way, if the message is correct, the remainder of this division should equal to zero. The CRSF protocol uses the same CRC generating polynomial as [DVB-S2](https://en.wikipedia.org/wiki/DVB-S2) system, with a value of 0xD5.

The Default CRC algorithm requires looping over every bit in the message, which gives us a time complexity of $O(n)$, where n is the message length in bits. This implementation is simple and as the messages are not that long, I could possibly go with that implementation and call it a day. But I really don't want for the library to consume any more processing power than necessary. Let's look for a better solution, then.

Instead of updating the remainder for every bit of data, let's precompute every possible outcome for a whole byte. This still has the time complexity of $O(n)$ but this time the n is 8 times smaller. The only downside of this approach is the fact that we need to store the precomputed outcomes somewhere, but as our CRC is only 8-bit long the amount of required data totals to $2^8$ = 256 bytes. My MCU has a total of 128 kB of RAM, so the table will occupy a terrifying 0.002% of total memory, and I don't expect that somebody would use a much weaker hardware for a purpose of a flight controller.

There's also one more option. Most STM32 MCUs come with a hardware CRC peripheral, so we could possibly use that. The problem with this approach is that some MCUs, despite having this peripheral, don't allow for any additional configuration, and only calculate a 32-bit CRC with a "default poly". The F411 is one of those as well. I will still implement both of those options so anybody using the library can choose one that suits their needs most.

Let's take a look at the code:
```c
// CRSF_CRC_8.h

#define CRC8_POLYNOMIAL 0xd5
#define CRC8_INIT       0x00
.
.
.
```

```c
// CRSF_CRC_8.c

#include "CRSF_CRC_8.h"

#ifdef CRSF_CRC_HARD
CRC_HandleTypeDef hcrc;
#else
static uint8_t CRC8Table[256];
#endif

void CRSF_CRC_Init(void)
{
#if CRSF_CRC_HARD
    hcrc.Instance = CRC;
    CRC->POL = CRC8_POLYNOMIAL;
    CRC->INIT = CRC8_INIT;
#else
    for(size_t i = 0; i < 256; i++)
    {
        uint8_t crc = i;
        for(uint8_t j = 0; j < 8; j++)
        {
            if(crc & 0x80)
            {
                crc = (crc << 1) ^ CRC8_POLYNOMIAL;
            }
            else
            {
                crc <<= 1;
            }
        }
        CRC8Table[i] = crc & 0xff;
    }
#endif
}

uint8_t CRSF_CRC_Calculate(const uint8_t* data, uint8_t length)
{
#if CRSF_CRC_HARD
    return HAL_CRC_Calculate(&hcrc, data, length);
#else
    uint8_t crc = CRC8Table[0 ^ data[0]];

    for(uint8_t i = 1; i < length; i++)
    {
        crc = CRC8Table[crc ^ data[i]];
    }
    return crc;
#endif
}
```

For CRC checks, we need only two functions. First one initializes the CRC module. When using hardware implementation it just configures the peripheral. When using the software one, it generates a look-up table that will be used in CRC calculation later. Calculating the table during startup allows us to save some flash memory. It also makes changing the CRC to work with a different poly extremely simple. Just change a value of single define, that's it.

The second function calculates the CRC value from a given data buffer. Again, its behavior changes based on the chosen implementation. In hardware one, it calls a HAL function which handles the calculation, and in software version it calculates the CRC one byte at a time using the lookup table initialized before.


Now the only thing left is to update `CRSF_HandleRX` and `CRSF_Init` functions:
```c
void CRSF_Init(UART_HandleTypeDef* huart)
{
    uart = huart;

    CRSF_CRC_Init();

    CRSF_RX_SYNC_BYTE = 0;
    CRSF_RX_MSG_LEN   = 0;
    receptionReset();
}

void CRSF_HandleRX(void)
{
    if(CRSF_RX_SYNC_BYTE != CRSF_SYNC_DEFAULT && CRSF_RX_SYNC_BYTE != CRSF_SYNC_EDGE_TX)
    {
        receptionReset();
        return;
    }

    if(CRSF_CRC_Calculate(CRSF_RX_CRC_BEGIN, CRSF_RX_MSG_LEN) != 0)
    {
        DEBUG_LOG("CRC\n");
        receptionReset();
        return;
    }

    // Parse data

    receptionReset();
}
```

After receiving the data, CRC is calculated and if it doesn't pass the check, the analyzed frame is discarded.

## Parsing the data

Now that we are almost 100% sure that the data in messages is correct, we can finally look what's inside them.\
The first step is to recognize what kind of information is in the packet. For that, a switch statement can be used.

First, let's parse the Link Statistics message. This one is very simple, as every parameter in this packet is 8-bit long. That means we can easily mimic its memory layout with a struct like this one:

```c
typedef struct __attribute__((packed)) /* <-- I don't think this attribute does anything here,
                                              but it wont hurt either so... */
{
	uint8_t UplinkRSSI_Ant1;
	uint8_t UplinkRSSI_Ant2;
	uint8_t UpLinkQuality;
	int8_t UplinkSNR;
	uint8_t DivActiveAnt;
	uint8_t RFMode;
	uint8_t UplinkTXPow;
	uint8_t DownlinkRSSI;
	uint8_t DownlinkQuality;
	int8_t DownlinkSNR;
} CRSF_LinkStatistics;
```

With this, the only thing left is to do is to copy a corresponding part of the message into preallocated struct instance. This way accessing struct members will read correct parts of memory.

Parsing RC Channels is a bit trickier. Every element of this packet is on 11-bit long, and computers don't really like working with anything that's not a multiple of eight. So unpacking that will require a bit of bit manipulation.

> In perfect word we could just used bit fields to make a packed struct of 11-bit integers and parse it in the same way as the Link Statistics packet, but that comes with a few problem. Don't get me wrong it does indeed work, at least on my machine. Implementation of bit fields in C are both platform and, whats worse compiler dependent, so just because it works on Cortex-M4 with ARM-GCC toolchain it does't mean it will, on something like Cortex-M0 (I believe, it's big endian) or when using a different compiler. Good old bit manipulation it is then.

```c
// CRSF_Connection.c

// Global data
uint32_t CRSF_Channels;
CRSF_LinkStatistics CRSF_LinkState;

static void unpackBitField(uint32_t* dst, const uint8_t* src)
{
    uint32_t read_bits = 0;
    uint32_t src_ptr = 0;
    uint32_t val = 0;
    for (uint32_t i = 0; i < 16; i++)
    {
        while (read_bits < 11)
        {
            val |= src[src_ptr++] << read_bits;
            read_bits += 8;
        }

            dst[i] = val & 0x7FF;
            val >>= 11;
            read_bits -= 11;
    }
}

static void parseData(void)
{
    switch(CRSF_RX_FRAME_TYPE)
    {
        case CRSF_FRAMETYPE_LINK_STATISTICS:
            /* I didn't use Message Length here to avoid writing to random memory,
                if I got a VERY corrupted packet. */
            memcpy(&CRSF_LinkState, CRSF_RX_DATA_BEGIN, sizeof(CRSF_LinkStatistics));

            DEBUG_LOG("RQly: %lu\n", CRSF_LinkState.UplinklinkQuality);
            break;

        case CRSF_FRAMETYPE_RC_CHANNELS_PACKED:
            unpackBitField(CRSF_Channels, CRSF_RX_DATA_BEGIN);
            break;

        default:
            DEBUG_LOG("How did we get there? - %hhx\n", CRSF_RX_FRAME_TYPE);
            break;
    }
}
```

RC Channels unpacking function works by OR'ing next message bytes until it has enough to extract a whole value. Then it discards already used bits and reads more data into this register until all 16 channels are decoded.

Let's quickly add one more logging function to finally see the all of that code in action.
```c
void CRSF_DEBUG_PrintChannels(uint32_t n_channels)
{
    if (n_channels > 16)
    {
        n_channels = 16;
    }

    printf("T: %lu", HAL_GetTick());
    for (uint32_t i = 0; i < n_channels; i++)
    {
        printf(", Ch%lu: %4lu", i, CRSF_Channels[i]);
    }
    printf("\n");
}
```
A small chunk of what got dumped over to my TeraTerm console:
```console
T: 152709, Ch0: 1189, Ch1: 1699, Ch2:  951, Ch3: 1696
T: 152729, Ch0: 1314, Ch1: 1649, Ch2:  951, Ch3: 1598
T: 152749, Ch0: 1430, Ch1: 1583, Ch2:  954, Ch3: 1483
T: 152769, Ch0: 1513, Ch1: 1499, Ch2:  992, Ch3: 1337
T: 152789, Ch0: 1551, Ch1: 1388, Ch2: 1036, Ch3: 1183
RQly: 100
T: 152829, Ch0: 1511, Ch1: 1096, Ch2: 1109, Ch3:  971
T: 152849, Ch0: 1457, Ch1:  984, Ch2: 1120, Ch3:  949
T: 152869, Ch0: 1356, Ch1:  955, Ch2: 1156, Ch3:  943
T: 152889, Ch0: 1356, Ch1:  955, Ch2: 1156, Ch3:  943
RQly: 100
T: 152909, Ch0:  949, Ch1:  838, Ch2: 1247, Ch3:  952
T: 152929, Ch0:  752, Ch1:  757, Ch2: 1292, Ch3:  999
T: 152949, Ch0:  528, Ch1:  678, Ch2: 1383, Ch3: 1023
T: 152969, Ch0:  367, Ch1:  621, Ch2: 1471, Ch3: 1076
T: 152989, Ch0:  284, Ch1:  611, Ch2: 1548, Ch3: 1161
RQly: 100
T: 153009, Ch0:  249, Ch1:  667, Ch2: 1617, Ch3: 1257
T: 153029, Ch0:  267, Ch1:  818, Ch2: 1665, Ch3: 1345
T: 153049, Ch0:  342, Ch1:  996, Ch2: 1708, Ch3: 1398
T: 153069, Ch0:  436, Ch1: 1120, Ch2: 1708, Ch3: 1425
T: 153089, Ch0:  592, Ch1: 1273, Ch2: 1700, Ch3: 1428
RQly: 100
CRC
T: 153149, Ch0:  890, Ch1: 1407, Ch2: 1518, Ch3: 1279
T: 153169, Ch0:  931, Ch1: 1345, Ch2: 1401, Ch3: 1173
T: 153189, Ch0:  992, Ch1: 1153, Ch2: 1269, Ch3: 1040
RQly: 100
T: 153209, Ch0:  992, Ch1:  997, Ch2: 1088, Ch3:  991
T: 153229, Ch0:  992, Ch1:  984, Ch2:  875, Ch3:  987
T: 153249, Ch0:  992, Ch1:  989, Ch2:  552, Ch3:  994
T: 153269, Ch0:  992, Ch1:  992, Ch2:  544, Ch3:  992
T: 153289, Ch0:  992, Ch1:  992, Ch2:  544, Ch3:  992
```
The printed output contains values of a few channels and some connection info. I printed only 4 RC channels, as the rest is basically the same. Those channel correspond to sticks deflection on the controller. First thing to observe is that the values are in a pretty weird range (172 - 1811, yes I know you cannot tell that from the example, I read it from docs), so we don't even get a full 11-bit resolution :pouting_cat:. We of course also need to map those values into some useful range as well (most likely $\pm100$).

Next thing, we get one Link Statistics packet per 4 to 5 RC Channels packets at 50Hz packet rate. This ratio will depend on that setting, of course. You can also see that I actually got one CRC error and that packet got discarded.

Last thing. If you look closely at the timings of the packets, something is off. We're missing two packets that should be there (at tick 152809 and 153129 exactly). First though would be, that it's caused by some oversight in the code, but the real reason is that I needed some source of magnetic interference, so I can actually get some CRC errors. This led me to turning on the telemetry feature for a while (as a radio transmitter is a pretty good EM noise source).\
Two things to note here. First, the receiver sends back telemetry even when we don't specifically give it any data. If that's the case, only Link Statistics packets are sent. Second thing, ExpressLRS radio transmission works in half-duplex mode (both RC data and telemetry are sent on the same radio channel). Because of that, the RC transmitter takes a break every few packets and allows the receiver to send something back. With my settings, it should happen every 16-th RC packet, and the example confirms that. 

Last piece of data I got is after power cycling the receiver and MCU. We can see how the UART keeps throwing errors while struggling to read the frames correctly.

```console
// After power cycling

err: 4
err: 4
err: 4
err: 4
err: 4
err: 4
err: 4
err: 4
err: 4
err: 4
err: 4
err: 4
err: 4
err: 4
err: 4
err: 2 <- This one, most likely got incorrectly recognized by the peripheral
err: 4
err: 4
err: 4
err: 4
err: 4
err: 4
err: 4
err: 4
err: 4
err: 4
err: 2 <- this one too
err: 4
err: 4
err: 4
err: 4
err: 4
err: 4
err: 4
RQly: 0
RQly: 0
T: 5857, Ch0:  992, Ch1:  992, Ch2:  544, Ch3:  992
T: 5877, Ch0:  992, Ch1:  992, Ch2:  544, Ch3:  992
T: 5897, Ch0:  992, Ch1:  991, Ch2:  544, Ch3:  992
T: 5917, Ch0:  992, Ch1:  992, Ch2:  544, Ch3:  992
T: 5937, Ch0:  992, Ch1:  991, Ch2:  544, Ch3:  992
```

This piece of code, taken directly from HAL source tells us exactly what happened:
```c
/** @defgroup UART_Error_Code UART Error Code
* @{
*/
#define HAL_UART_ERROR_NONE              0x00000000U   /*!< No error            */
#define HAL_UART_ERROR_PE                0x00000001U   /*!< Parity error        */
#define HAL_UART_ERROR_NE                0x00000002U   /*!< Noise error         */
#define HAL_UART_ERROR_FE                0x00000004U   /*!< Frame error         */
#define HAL_UART_ERROR_ORE               0x00000008U   /*!< Overrun error       */
#define HAL_UART_ERROR_DMA               0x00000010U   /*!< DMA transfer error  */
```
It indicates that the UART peripheral struggles to sync up to the transmission, as the reception started not before but somewhere in the middle of a UART frame. I didn't append time to those errors (my bad), but I am 100% sure that all of them happened in the duration of a single CRSF frame. After this unfortunately timed frame ended, the UART got some time to initialize and read the next frame on time, so the reception got back to normal. Over all, this error is a good demonstration of how this library maintains stable connection despite those errors.

> It may be easy to mistake the two so to avoid unnecessary confusion:\
**UART frame** is a single byte of data (most of the time) with a start and stop bit appended, to make syncing up the transmitter and receiver together.\
**CRSF frame** on the other hand contains dozens of UART frames, one for each byte of data.

# What's next
With all that, I finally have the bare minimum to send enough data to pilot basically any RC model. It's of course not the end of development of that library. Next I need to add support for sending telemetry back to the ground, so that I can, at least check my remaining battery voltage. Receiving end of the library also needs some improvements and a few more features, mostly some data collection and additional safety measures.

The most recent release of my CRSF library can be found [here](https://github.com/Koun7z/CRSF-for-stm32).

*[UART]: Universal Asynchronous Receiver Transmitter
*[CRSF]: Cross Fire
*[TBS]: Team Black Sheep
*[SNR]: Signal-to-Noise ratio
*[RSSI]: Received Signal Strength Indicator
*[MCU]: Microcontroller Unit
*[HAL]: Hardware Abstraction Layer
*[IT]: Interrupt
*[DMA]: Direct Memory Access
*[NVIC]: Nested Vectored Interrupt Controller
*[CRC]: Cyclic Redundancy Check
