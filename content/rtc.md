# explaining GBA Real-Time Clock (RTC)

## intro 

there's not much documentation online about how the GBA RTC works, so this blog post will attempt to demystify that. to be honest, i still don't fully know how the RTC works. i'll try to explain what i *do* know, but DISCLAIMER: what i say may not be 100% accurate. you should be able to successfully emulate a working RTC though using this blogpost.

## gba gpio
if you already know how gba gpio works, then you can safely [skip this section](##gba-rtc).

basically, some gba cartridges contain extra circuits that the ROM communicates with using this interface called **GPIO**. examples of these extra circuits are the solar sensor, the gyro sensor, the gba rumble, and of course, the gba RTC. there's three mmio registers that the ROM can use to communicate with a GPIO device:

`0800'00C4 : IO Port Data`\
`0800'00C6 : IO Port Direction`\
`0800'00C8 : IO Port Control`

 the **Data** register is used to send and receive data from the GPIO device. the ROM can write to this register to send data over to the device, and read from the register to, well, receive data. this register is 4 bits wide.

 the **Direction** register is used to specify the direction for the **Data** register. the direction register is also 4 bits wide. each bit 0-3 in the **Direction** register corresponds to a bit in the **Data** register. a value of 0 means the associated data bit is "in", while a value of 1 means the associated data bit is "out". what does this mean? well, you can only read bit "x" in the **Data** register if that bit is specified as "out" in the **Direction** register. likewise, you can only write a bit "x" in the **Data** register if that data bit is "in". you can think of the **Direction** register as a sort of mask on the **Data** register. note that you, as the emulator developer, don't have to do any sort of writing to the **Direction** register. the ROM will write to it as it likes, and you should reflect the behavior of the **Data** register accordingly.

the **Control** register is a lot easier. it's 1 bit wide (bit 0), and if it has a value of 0, then the **all** GPIO registers become write-only (reads return 0). if it has a value of 1, you can read from the registers once more.

these registers will act as an interface to the GPIO device to allow the ROM to send and receive data to and from the device.

## gba rtc
the rtc is probably one of the bizarre things to wrap your head around at first. let's first start out with a higher level explanation of how a ROM can communicate with the RTC. essentially, the ROM can send two types of commands - read commands, and write commands. read commands allow the ROM to read the full value of some of the registers on the RTC. likewise, write commands allow the ROM to write to some of the registers on the RTC. here's a list of the RTC registers, along with their read/write behaviors.

| name     | length (in bytes)<img width=450/> | read behavior<img width=1500/> | write behavior | notes<img width=500/>
|--------  |-------------------|---------------|---------------|-
| Control  | 1 | **bit 1**: unknown, but it can be written to and read from, so you should preserve its value <br> **bit 3**: per minute IRQ (1 = fire a Gamepak IRQ every 30 seconds) <br>**bit 6**: 12/24 hour mode<br>**bit 7**: power off (cleared on read, 1 = failure / time lost) | same as read behavior, but bit 7 is read-only. | any unused bits are zero.
| Date/Time| 7 | **byte 0**: current year in BCD, 00h ~ 99h = 2000 ~ 2099<br>**byte 1**: current month in BCD, 01h ~ 12h = January ~ December (bits 5-7 unused) <br> **byte 2**: current day in BCD, 00h ~ 31h (bits 6-7 unused)<br>**byte 3**: day of week in BCD, 0h ~ 6h = Sunday ~ Saturday. (bits 3-7 unused)<br>**bytes 4 - 6**: see the Time register below. | according to GBATEK, these can all be written to. but both mGBA and NBA disallow writes. you can probably ignore writes to Date/Time, unless you're implementing some sort of time machine. | see above
| Time     | 3 | **byte 0**: current hour in BCD. 00h ~ 23h in 24 hour mode, 00h ~ 11h in 12 hour mode. bits 6-7 are unused. <br> **byte 1**: current minute in BCD, 00h ~ 59h. bit 7 is unused. <br> **byte 2**: current second in BCD, 00h ~ 59h. bit 7 is unused. | see above. | see above.
| Reset    | 0 | n/a | GBATEK says all registers are zeroed, except the month register which gets set to 01h. however, mGBA and NBA both only zero the control register, and that's what makes the most sense to me anyway.
| IRQ      | 0 | n/a | forces a gamepak interrupt.

 okay, now that you (hopefully) understand what the different registers are in the RTC and what happens if you read/write to them, i'm going to explain how the ROM reads/writes to these registers in the first place. so, the GPIO data register on the GBA has 4 bits. 3 of those bits are used for GBA RTC. here's the general mapping:

|bit|name | description
|:--|:----|:------------
| 0 | SCK | source clock
| 1 | SIO | serial IO
| 2 | CS  | chip select
| 3 |     | unused

the first key thing to understand is that the **SIO** bit is the only bit that encodes actual command data. the command data is sent bit-by-bit through the **SIO** bit to the chip. the **SCK** and **CS** bits are used in conjunction to tell the RTC chip when to sample the **SIO** bit. the ROM will use all three bits to send commands to the RTC chip.

here's how a typical transfer works, from the perspective of the ROM:
1) while **SCK** is high, **CS** rises. this indicates to the RTC chip the start of a new command. **CS** will now stay high until the command is completed.
2) **SCK** will now toggle between low and high. every time **SCK** rises, the RTC chip will sample the **SIO** bit. 8 such samples will occur to fill a byte.
3) if the command was a write command, then you should write n bytes to the RTC as described in step 2, where n is the size of the register. if the command was a read command, do the same, except instead of writing to SIO, you read from SIO.
4) **CS** goes low. this indicates that the command has completed.

you can sorta think of **CS** as a "command in progress" signal, and **SCK** as a "sample SIO" signal.

you may have noticed i neglected to mention whether or not the data in **SIO** is sent in LSB order of MSB order. this is where things get really stupid - it can be either. the **command** byte must have bits `0110` in bit positions 4-7. if it does not, then simply reverse the **command** byte (why does everything in emulation have to be so needlessly complicated and unnecessary oh my g-)

anyway, there are 4 bits remaining. if bit 0 is 0, this command is a write command. if its 1, this command is a read command. bits 1-3 specify the register that the command operates on:

|bits 1-3|register|
|--------|-
|0|Reset
|2|Date/Time
|3|Time
|4|Control
|6|IRQ

01101000
so, to summarize, here's an example: the ROM will write something like `16h` to **SIO** using the process described above. this commands bits are reversed because `6h` is in the lower nibble, so we reverse `16h` to get `68h`. bit `0` of `68h` is `0`, which means this is a write command. bits `1-3` of `68h` is `4h`, which means the ROM is trying to write to the **Control** register. the size of the control register is 1 byte, so we should expect 1 more byte of data to be sent through SIO before the ROM ends the command.

alright, that should about wrap it up! if you have any questions at all, feel free to create a github issue! :)

## sources (couldn't have done this without these)

fleroviux - [NanoBoyAdvance](https://github.com/nba-emu/NanoBoyAdvance)

endrift - [mGBA](https://github.com/mgba-emu/mgba)

Martin Korth - [GBATEK](https://problemkaputt.de/gbatek.htm)

pret team - [Pokemon Emerald Decompilation Project](https://github.com/pret/pokeemerald)
