# 03/10/

Beginning the competition at 10PM.
Why join. Because I want to get more comfortable with systemverilog.
Mainly used VHDL before, but I believe SV has more features and is more
widely used in the world.
I also wanted to do discover more technical problems and understand how good
architecture leads to good RTL.

I decided not to use AI to write any code at all in order to improve.
AI can be used to write the blogposts later or build schematics.
AI is also used for research, helping me directionally sometimes, etc.
Everytime AI was used, it will be documented.

I have a artix 7, wukong board from qmtech.
I bought a elitedesk g4 mini to run vivado and other tools.
I mainly code from my mac on VSCode that ssh into the elitedesk.

----
Use of AI (Claude Opus 5.5 Medium) to make sure I understand the problem.

Seems to me like something similar to a CPU needs to be designed.
Jane street mentions RP2040 so let's check the datasheet and the instructions.

First little problem.
I want to download store the docs I used but i don't have a way to do that
easily via GUI to the elitedesk.

Use of AI to setup the SMB server as it has nothing to do with the challenge
itself.
(Codex 6.1 XHigh)

worked.

----
## RP2040 datasheet
the PIO execute sequential programs to manipulate GPIO and transfer data.
it's specialised for IOs, focus on determinism, precise timing.

---
 9 instructions:
JMP, WAIT, IN, OUT, PUSH, PULL, MOV, IRQ, and SET

---
 How things work:

every clock cycle, each state machines do:
fetch, decode, execute (of one instruction)
an instruction takes one cycle
(unless some like WAIT, since it's on purpose)

the PC points at the current instruction being executed on this cycle.
it increments by one every cycle, and wraps around when he finishes the
instruction memory.
(unless jump instructions, make sense)

you have a CLK division.
for example
if clk division chosen to be 2.5, then if your program takes like 4 clock
cycle, 4x2.5=10 cycles
this is very useful for UART ! to set a precise baud rate

this CLKdivider slows the state machine itself !
in order to process the program slower.

there is the SMx INSTR special configuration register.
this permits execution of instruction not in the instruction memory
for example you can place a jmp instruction inside this SMx INSTR
and then the state machine will start execution from a diff location

---
MOV EXEC permits instructions to be executed from a register
OUT EXEC from the shifter (shifter ?)
both execute in one cycle (MOV/OUT) and then one cycle for EXEC.

OUT EXEC is interesting because it can be used to embed instructions inside
the dataflow, for example in I2C, you can embed STOP alongside normal data.

---
Each state machine has internal registers.
Output Shift Register
TX FIFO -> (PULL) -> MUX(OSR) -> (OUT) pins
and theres also a shifter from the OSR to itself.

PULL remove a 32 bits word from the TX fifo and put it inside the OSR.
OUT shifts data from the OSR to wherever. 1....32 bits at a time
(so I guess you can configure this / choose how many bits ?)
OSR fills with zeroes when you shift data.


if autopull enabled, the state machine automatically refills the OSR
from the FIFO on an OUT instruction, once some shift count threshold is
reached.
shift can be left/right.

example
    1 .program pull_example1
    2 loop:
    3 out pins, 8
    4 public entry_point:
    5 pull
    6 out pins, 8 [1]
    7 out pins, 8 [1]
    8 out pins, 8
    9 jmp loop

autopull is good because
you dont need an instruction to pull at the correct time
can output up to 32 bits on every single clock cycle,
if the FIFO stays topped up

example
```asm
    1 .program pull_example2
    2
    3 loop:
    4 out pins, 8
    5 public entry_point:
    6 jmp loop
```

input shift register is kinda the same thing as OSR. but IN/PUSH.

shift counters track how many bits in total have been shifted out of the
OSR via OUT
and shifted in the
ISR via IN

scratch registers
X and Y used as:

Source/destination for IN/OUT/SET/MOV
Source for branch conditions

example
```asm
    1 .program ws2812_led
    2
    3 public entry_point:
    4 pull
    5 set x, 23 ; Loop over 24 bits
    6 bitloop:
    7 set pins, 1 ; Drive pin high
    8 out y, 1 [5] ; Shift 1 bit out, and write it to y
    9 jmp !y skip ; Skip the extra delay if the bit was 0
    10 nop [5]
    11 skip:
    12 set pins, 0 [5]
    13 jmp x-- bitloop ; Jump if x nonzero, and decrement x
    14 jmp entry_point
```

---
4-word deep FIFOs

one for data transfer from system to state machine (TX), and the
other for state machine to system (RX).
The TX FIFO is written to by system busmasters, such as a processor
or DMA controller,
and the RX FIFO is written to by the state machine.

FIFOs decouple the timing of the PIO state machines and
the system bus, allowing state machines to go for longer periods
without processor intervention.

---
Instruction memory is a 1-write 4-read register file
all four state machines can read
an instruction on the same cycle, without stalling

the use of the four state machines:
Pointing multiple state machines at the same program
Pointing multiple state machines at different programs
Using multiple state machines to run different parts of the same interface,
(e.g. TX and RX side of a UART, or clock/hsync and pixel data on a DPI display)

they cant communicate data.
they can synchronize with each other with IRQ flags.

---
... rest is kinda not too useful for now.
#
