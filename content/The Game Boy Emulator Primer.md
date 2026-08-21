---
title: The Game Boy Emulator Primer
draft: false
tags:
  - tutorial
  - primer
  - emulator
  - gameboy
---
### Everything you need to understand *before* you watch The Ultimate Game Boy Talk or open Pan Docs

---

## How to read this document

You have built a CHIP-8 emulator. That means you already own the single most important idea in emulation: **a program is a sequence of numbers, and a machine is a loop that reads those numbers and reacts to them.** Everything here is built on that foundation.

But CHIP-8 taught you that idea in a padded room. The Game Boy will teach it to you outdoors, in weather.

This primer is deliberately *not* a reference. Pan Docs is a reference. It is a beautiful, exhaustive, unforgiving encyclopedia written by people who already understand the machine, for people who already understand the machine. Reading Pan Docs before you have a mental model is like reading a dictionary to learn a language. This document is the language lessons. Pan Docs is the dictionary you'll reach for afterwards.

A few conventions:

- **`$` prefix means hexadecimal.** `$FF40` is hex. This is the convention Pan Docs and the entire Game Boy community uses, so I'll use it too, and you should get comfortable reading it. `0xFF40` means the same thing.
- **`%` prefix means binary.** `%10010001`.
- When I say *"in CHIP-8..."*, I'm building a bridge. Cross it deliberately. The comparison is usually where the real insight lives.
- Boxes labeled **⚙ Emulator implication** tell you what a concept means for the code you will eventually write. Don't write that code yet. Just let it register.
- Boxes labeled **🤔 Why not the obvious way?** explain design decisions that look strange until you know the constraint that caused them.

Read it in order. Part 7 (graphics) will not make sense without Part 5 (memory), and Part 5 will not make sense without Part 2 (architecture foundations). This is a staircase, not a buffet.

---

## Table of Contents

**Part 1**: From CHIP-8 to Real Hardware
**Part 2**: Computer Architecture Foundations
**Part 3**: The Game Boy as a Whole System
**Part 4**: The CPU
**Part 5**: Memory Architecture
**Part 6**: Cartridges and Banking
**Part 7**: The Graphics System
**Part 8**: Timing
**Part 9**: Interrupts
**Part 10**: One Frame of Pokémon, in Slow Motion
**Interlude**: Sound, at a High Level
**Part 11**: Reading Pan Docs: A Survival Guide
**Part: Watching the 33C3 Talk: A Companion Guide
**Appendix A**: A Suggested Build Order
**Appendix B**: Glossary
**Appendix C**: Test ROMs and What They're Actually Testing

---
---

# Part 1: From CHIP-8 to Real Hardware

## 1.1 The uncomfortable truth about CHIP-8

Here is something nobody tells you when you finish your CHIP-8 emulator:

**CHIP-8 is not a computer. It never was.**

CHIP-8 was created in 1977 by Joseph Weisbecker as an *interpreted programming language* (a virtual machine) that ran on top of actual hardware (the COSMAC VIP and Telmac 1800). The real CPU in those machines was an RCA 1802. CHIP-8 programs were interpreted by a program written for the 1802, in the same way that Python bytecode is interpreted by CPython.

So when you wrote a CHIP-8 emulator, you did not emulate a machine. **You wrote an interpreter for a language.** That's why it was pleasant. Languages are designed by humans for humans. Hardware is designed by engineers for factories, under cost pressure, in 1989.

This distinction explains every single thing that is about to get harder.

## 1.2 Why CHIP-8 was easy: a forensic accounting

Let's be precise about what CHIP-8 spared you. Each of these is a gift you are about to have taken away.

**CHIP-8 had no clock.**
Instructions in CHIP-8 don't "take" any particular amount of time. The spec doesn't say how long `6XNN` takes. So in your emulator you probably wrote something like "run 700 instructions per second" or "run N instructions per frame," picked a number that felt good, and it worked. Nothing broke. No game noticed.

That freedom is gone. On the Game Boy, an instruction takes a *specific, exact* number of clock ticks, and other parts of the machine are counting those same ticks and doing things at specific moments. Time is now a shared resource that everything is measuring.

**CHIP-8 had no memory map.**
CHIP-8 has 4KB of memory. All of it behaves identically. Address `$200` and address `$800` are the same kind of thing: a byte you can read and a byte you can write.

That uniformity is gone. On the Game Boy, address `$FF44` is not a byte of memory. It's a *wire to the graphics chip*. Reading it asks the graphics chip a question. Writing to `$FF46` doesn't store a value, it triggers a 160-byte block copy that takes 640 clock cycles. Some addresses are read-only. Some are write-only. Some are read-only *sometimes*, depending on what the graphics chip is doing at that instant.

**CHIP-8 had no interrupts.**
Your CHIP-8 loop was: fetch, decode, execute, repeat. Forever. Nothing ever interrupted it. Nothing ever said "stop what you're doing and go run this other code."

The Game Boy has five interrupt sources that can seize the CPU between instructions, push the program counter onto a stack, and jump to a fixed address. Games depend on this constantly. You cannot run a single commercial Game Boy game without implementing interrupts.

**CHIP-8's display was a framebuffer.**
64×32 pixels. One bit each. You had a 2D array. `DXYN` XORed sprite bytes into it. When you wanted to show it, you drew the whole array. Drawing was an *event*.

The Game Boy has no framebuffer that you can just read. It has a graphics processor that reconstructs the image from *tiles* and *maps* and *sprite descriptors*, **one horizontal line at a time**, continuously, forever, in lockstep with the CPU, and games change the drawing rules *in the middle of a frame* to create effects the hardware was never designed to produce.

**CHIP-8's "hardware" was three things.**
A display, a 16-key keypad, and two timers. That's the whole peripheral set.

The Game Boy has a graphics processor, a 4-channel sound processor, a programmable timer with a selectable frequency, a serial link port, a joypad matrix, a DMA engine, an interrupt controller, and cartridges containing their own memory-mapping hardware.

**CHIP-8 programs were loaded flat.**
`memcpy(&memory[0x200], rom, rom_size)`. Done. The program fit in memory because the spec guaranteed it.

Game Boy games are up to 8 MB. The CPU can address 64 KB. The cartridge contains a chip whose entire job is to lie to the CPU about which part of the game it's currently looking at.

**CHIP-8 had no boot process.**
You set `PC = 0x200` and started. There was no "power-on."

The Game Boy has a 256-byte boot ROM inside the CPU chip itself that runs first, scrolls the Nintendo logo, plays the "ba-ding," verifies the cartridge is legitimate, and then *unmaps itself from memory* and hands control to the game, leaving the CPU registers in a very specific state that some games depend on.

**CHIP-8 had one implementation-defined quirk table.**
You probably hit a few: does `8XY6` shift `VX` or `VY`? Does `FX55` increment `I`? Annoying, but a short list, and games mostly worked either way.

The Game Boy's quirk list is hundreds of items long, and games *depend* on the quirks. Not by accident — deliberately. Programmers discovered them and used them, because a quirk that produces a visual effect for free is worth exploiting when you have 32 KB of ROM and a deadline.

## 1.3 Why the Game Boy is a real computer

Here's the reframe that will save you months:

> **CHIP-8 is a specification. The Game Boy is a *consequence*.**

Every strange thing about the Game Boy exists because of a physical or economic constraint in 1989:

| Design decision | The constraint that caused it |
|---|---|
| 8-bit CPU at 4.19 MHz | Battery life. Four AA batteries had to last ~30 hours. |
| 64 KB address space | 16 address pins on the chip. More pins = more cost. |
| Tile-based graphics | 5,760 bytes for a full-screen bitmap was unaffordable in RAM. Tiles cost ~1/10th as much. |
| 4 shades of grey | The LCD was cheap and monochrome. Color LCDs were expensive and power-hungry. |
| Cartridges with mapper chips | ROM was expensive per byte; games grew past 32 KB; the address bus couldn't grow. |
| Sound made of square waves | Generating waveforms with counters costs almost no silicon. Real samples cost RAM. |
| 8 KB of work RAM | RAM was the single most expensive component per byte. |

Once you internalize "every weirdness has a cause," the Game Boy stops feeling arbitrary and starts feeling *inevitable*. That's the mental model shift this document is built to produce.

## 1.4 The evolution from CHIP-8 to Game Boy

Let me show you the conceptual distance as a series of steps. Each step is a thing you'll need to learn, and each one builds on the last.

```
   CHIP-8                                          GAME BOY
   ──────                                          ────────

   [ Interpreter loop ]                            [ Interpreter loop ]
   one big switch on opcode          ───────►      one big switch on opcode
                                                   ... but now every case
                                                   ALSO reports how many
                                                   clock cycles it consumed
                                                            │
                                                            ▼
   [ Flat 4KB array ]                              [ Address decoder ]
   memory[addr]                      ───────►      read(addr) / write(addr)
                                                   functions that route to
                                                   ROM / RAM / VRAM / I/O
                                                   registers / cartridge chip
                                                            │
                                                            ▼
   [ Draw when DXYN runs ]                         [ A second processor ]
                                     ───────►      that draws continuously,
                                                   line by line, from tile
                                                   data, while the CPU runs
                                                            │
                                                            ▼
   [ Nothing interrupts you ]                      [ Interrupt controller ]
                                     ───────►      that can hijack the CPU
                                                   between instructions
                                                            │
                                                            ▼
   [ Time is whatever you want ]                   [ A shared clock ]
                                     ───────►      that CPU, PPU, timer, and
                                                   APU all count together
                                                            │
                                                            ▼
   [ ROM fits in memory ]                          [ ROM does NOT fit ]
                                     ───────►      cartridge hardware swaps
                                                   chunks in and out
```

Read that diagram again in a week. It will be a table of contents for everything you've learned.

## 1.5 The single most important mindset change

In CHIP-8, your emulator was **a program that runs a program.**

For the Game Boy, your emulator must become **a simulation of several machines that share a clock and a bus.**

The CPU is one machine. The PPU (graphics) is another. The timer is another. The APU (sound) is another. They run *simultaneously* in real hardware. Your emulator will fake simultaneity by interleaving them very finely, running the CPU for one instruction, then telling the PPU "4 ticks passed, catch up," then telling the timer the same, and so on.

That interleaving loop is the heart of a Game Boy emulator. If you understand nothing else from this document, understand this: **your main loop is not "run instructions." Your main loop is "advance time, and let every component react to the time that passed."**

---
---

# Part 2: Computer Architecture Foundations

Pan Docs assumes you know everything in this part. The 33C3 talk assumes it *and* assumes you find it obvious. Let's make it obvious.

I'm going to teach these concepts generically first — the way they'd apply to any 1980s microcomputer — and then in Part 3 we'll snap them onto the Game Boy specifically. Learning the general shape first means the Game Boy will feel like an *instance of a familiar pattern* rather than a pile of trivia.

## 2.1 What a CPU actually is

### Intuition

Imagine an extraordinarily fast, extraordinarily obedient, and extraordinarily stupid office clerk.

The clerk sits at a desk. On the desk are a few small labeled trays where he can hold numbers — that's all the memory he has personally. Along the wall is a vast filing cabinet with numbered drawers. The clerk has a bookmark showing which drawer he's currently reading instructions from.

The clerk's entire life is this loop:

1. Look at the bookmark. Go to that drawer. Read the note in it.
2. Move the bookmark to the next drawer.
3. Do exactly what the note says. No more, no less, no interpretation.
4. Go to step 1.

He does this about four million times a second. He never gets tired, never questions an instruction, never notices when the instructions are nonsense. If a note says "go to drawer 500 and start reading instructions there," he does it, even if drawer 500 contains a picture of a tree.

**You already built this clerk.** That's your CHIP-8 loop. The Game Boy's clerk is the same clerk with more trays, more instructions, and a boss who taps him on the shoulder occasionally.

### Why it exists

The alternative to a CPU is *fixed-function hardware*: a circuit built to do one specific thing. A calculator chip from 1972 can only calculate. A CPU is a circuit that does whatever a list of numbers in memory tells it to do, which means one piece of silicon can be Tetris on Tuesday and Pokémon on Wednesday. **The CPU is the invention of "software."**

### How it works: the three parts

Every CPU, including the Game Boy's, has three functional pieces:

```
          ┌──────────────────────────────────────────┐
          │                  CPU                     │
          │                                          │
          │   ┌──────────────┐   ┌───────────────┐   │
          │   │  REGISTERS   │   │  ALU          │   │
          │   │              │   │ (Arithmetic   │   │
          │   │  A  F        │◄─►│  Logic Unit)  │   │
          │   │  B  C        │   │               │   │
          │   │  D  E        │   │  +  -  AND    │   │
          │   │  H  L        │   │  OR XOR shift │   │
          │   │  SP  PC      │   └───────────────┘   │
          │   └──────────────┘                       │
          │           ▲                              │
          │           │                              │
          │   ┌───────┴───────────────────────────┐  │
          │   │      CONTROL UNIT                 │  │
          │   │  reads opcodes, decides which     │  │
          │   │  wires to switch on, and when     │  │
          │   └───────────────────────────────────┘  │
          └──────────────┬───────────────────────────┘
                         │
                    (to the bus)
```

- **Registers** are the clerk's desk trays. Tiny, extremely fast storage *inside* the CPU. The Game Boy has eight 8-bit ones plus two 16-bit ones. That's it. Ten values. Everything else lives out in memory and must be fetched.
- **The ALU** is the part that can actually compute. Add. Subtract. AND. OR. XOR. Shift. Compare. That's roughly the whole menu on an 8-bit CPU. Notice what's *not* there: no multiply, no divide, no floating point. If a Game Boy game needs to multiply, someone wrote a subroutine that adds in a loop.
- **The control unit** is the decoder. It takes the opcode byte and turns it into a sequence of internal steps: "put register B on the internal bus, tell the ALU to add, store the result in A, update the flags." This is the part that makes an opcode *mean* something.

### ⚙ Emulator implication

In your emulator, registers are variables, the ALU is your arithmetic helper functions, and the control unit is your `switch` statement. You already wrote all three for CHIP-8 without knowing they had names.

---

## 2.2 Buses: the roads between chips

### The problem this solves

The CPU is one chip. Memory is a different chip. They're physically separate pieces of silicon on a circuit board, possibly millimeters apart. How does a number get from one to the other?

Wires. But how many wires, and what do they mean?

### Intuition: the pneumatic tube

Picture an old department store with a pneumatic tube system. To request an item, you write the *shelf number* on one form and drop it in the "request" tube. A moment later, the item arrives in the "delivery" tube. There's also a small switch labeled SEND / RECEIVE that tells the stockroom whether you're fetching or returning something.

That's a bus:

- The **address bus** carries *which* location you want. Request tube.
- The **data bus** carries the *contents*. Delivery tube.
- The **control bus** carries *what you're doing* — read or write. The switch.

```
                       ADDRESS BUS  (16 wires)
        ┌─────┐  ═══════════════════════════════►  ┌──────────┐
        │     │      "I want location $C123"        │          │
        │ CPU │                                     │  MEMORY  │
        │     │  ◄═══════════════════════════════►  │          │
        └─────┘       DATA BUS  (8 wires)           └──────────┘
           │           "here's the byte: $3F"            ▲
           │                                             │
           └──────────────────────────────────────────────┘
                    CONTROL BUS  (read? write?)
```

### Why the sizes matter — and this is the big one

Each wire carries one bit: high or low, 1 or 0. So:

- **16 address wires** → 2¹⁶ = **65,536 distinct addresses**. This is where "64 KB" comes from. It is not a design choice made for elegance. It is *literally the number of pins Sharp put on the chip.*
- **8 data wires** → each transfer moves **one byte**. This is what "8-bit CPU" means. Not that it can't handle bigger numbers — it can, by doing multiple transfers — but that one trip down the tube carries exactly 8 bits.

> 🤔 **Why not the obvious way?**
> Why not 24 address pins and 16 data pins, for 16 MB of address space and faster transfers? Because every pin costs money, board space, and power. In 1989, at a $89 retail price point on four AA batteries, 16 + 8 was the answer. Almost every "why is the Game Boy like this?" question eventually bottoms out in a cost or power decision.

### What would happen if it didn't exist?

Without a shared bus, every chip would need dedicated wires to every other chip — an explosion of connections. A bus is a *shared road*: one set of wires, and everyone takes turns. The cost of sharing is that **only one thing can use the bus at a time**, which is the root cause of a whole family of Game Boy behaviors you'll meet later (VRAM being inaccessible during rendering, DMA locking the bus, etc.).

Hold on to that sentence. It explains more Game Boy weirdness than any other single fact.

### ⚙ Emulator implication

You will never simulate wires. But you *will* simulate the consequence: a `read8(addr)` and `write8(addr, value)` pair that every component funnels through, and the fact that some components can't access memory while others are using it.

---

## 2.3 Address space vs. memory (they are not the same thing)

This is the concept that trips up more beginners than any other, so let's be very explicit.

### The distinction

- **Address space** is the set of *numbers the CPU can say*. The Game Boy's is `$0000`–`$FFFF`. It's an abstraction. A range of names.
- **Memory** is *physical storage chips*. Silicon that holds bits.

They are not the same and they don't have to match up one-to-one. The address space is a set of 65,536 *labeled mailboxes*, and it's up to the system designer to decide what's behind each label.

### Intuition: the hotel with strange rooms

Imagine a hotel with 65,536 room numbers on its keypad. You'd assume 65,536 rooms. But when you walk the halls:

- Rooms 0–32767 are the **library** — you can read the books but you can't write in them. (ROM)
- Rooms 32768–40959 are the **art studio** — but it's locked whenever the painter is working. (VRAM)
- Rooms 49152–57343 are ordinary **guest rooms** — read and write freely. (WRAM)
- Rooms 57344–65023 are a *mirror* of the guest rooms — different door, same room. Walk in through either door, same furniture. (Echo RAM)
- Room 65348 has no furniture at all. It's a **speaking tube to the art department.** Ask it a question, and it tells you which line the painter is currently painting. It's not storage. It's a *conversation.* (The `LY` register at `$FF44`)
- Rooms 65024–65151 are all speaking tubes to various departments. (I/O registers)
- Rooms 65280–65534 are a tiny **notepad by the front desk** — very fast to reach, useful during emergencies. (HRAM)

The CPU cannot tell the difference. It just puts a number on the address bus. Whatever is wired to respond to that number, responds.

### The address decoder

The chip (or circuit) that looks at the address and decides who should answer is called the **address decoder**. It's a glorified `if/else`:

```
                    address on the bus
                           │
                           ▼
              ┌────────────────────────┐
              │    ADDRESS DECODER     │
              │  "who does this        │
              │   address belong to?"  │
              └───┬────┬────┬────┬─────┘
                  │    │    │    │
      ┌───────────┘    │    │    └───────────┐
      ▼                ▼    ▼                ▼
  ┌───────┐      ┌────────┐ ┌───────┐  ┌──────────┐
  │ CART  │      │  VRAM  │ │ WRAM  │  │   I/O    │
  │  ROM  │      │        │ │       │  │ REGISTERS│
  └───────┘      └────────┘ └───────┘  └──────────┘
```

### ⚙ Emulator implication

**This decoder is a function you will write, and it is the single most important function in your emulator.**

```
read8(addr):
    if addr < 0x8000:      return cartridge.read(addr)
    if addr < 0xA000:      return ppu.read_vram(addr)
    if addr < 0xC000:      return cartridge.read_ram(addr)
    ...
```

In CHIP-8 you wrote `memory[addr]`. Delete that instinct. From now on, **every** memory access goes through a routing function. If you take one practical lesson from Part 2, take this one — building your emulator around a proper bus/decoder from day one will save you from a painful rewrite later.

---

## 2.4 Memory mapping and memory-mapped I/O

### The problem this solves

The CPU needs to talk to the graphics chip, the sound chip, the timer, and the joypad. How?

**Option A: special instructions.** Add opcodes like `IN` and `OUT` with a separate "I/O address space." The Intel 8080 and Z80 do this.

**Option B: memory-mapped I/O.** Wire the peripherals to addresses in the *normal* address space. Then `LD A, ($FF44)` — an ordinary memory read — is how you ask the graphics chip a question.

The Game Boy uses Option B almost exclusively. (Its CPU inherits a couple of Z80-ish I/O-flavored opcodes, but they're really just shortcuts into the `$FF00`–`$FFFF` range.)

### Why it was designed this way

Because it costs nothing. You already have an address bus and a decoder. Hanging a peripheral off address `$FF40` requires no new instructions, no new bus, no new silicon in the CPU. Every instruction that can touch memory can now touch hardware. It is *free expressiveness.*

### How it changes your thinking

Here's the mental shift, stated as bluntly as possible:

> **A "register" at `$FF40` is not a byte of storage. It is a control panel.**

When you write `$91` to `$FF40`, you are not storing the number 145. You are flipping seven switches on the graphics processor:

```
   Value written to $FF40 (LCDC):   1  0  0  1  0  0  0  1
                                    │  │  │  │  │  │  │  │
   bit 7  LCD on/off ───────────────┘  │  │  │  │  │  │  │
   bit 6  Window tile map select ──────┘  │  │  │  │  │  │
   bit 5  Window enable ──────────────────┘  │  │  │  │  │
   bit 4  BG/Window tile data select ────────┘  │  │  │  │
   bit 3  BG tile map select ───────────────────┘  │  │  │
   bit 2  Sprite size (8x8 or 8x16) ───────────────┘  │  │
   bit 1  Sprite enable ──────────────────────────────┘  │
   bit 0  BG/Window enable ──────────────────────────────┘
```

Turning off bit 7 physically shuts down the LCD. That's not a data change; that's an *action*.

Three consequences that will bite you if you forget them:

1. **Reads can have side effects.** Reading some registers clears bits.
2. **Writes can trigger events.** Writing to `$FF46` starts a 160-byte DMA transfer.
3. **What you read back is often not what you wrote.** Some bits are read-only (hardware sets them), some are write-only (read back as 1s), some are unused (always read as 1).

### ⚙ Emulator implication

Your I/O region cannot be a plain byte array. Each register needs its own read and write logic. Some of the hardest-to-find emulator bugs are "I stored the written value and returned it on read" when the real hardware masks bits.

---

## 2.5 ROM vs. RAM

### Intuition

- **ROM** = Read-Only Memory = **a printed book.** The words were fixed at the factory. You can read any page instantly. You cannot change a word. Unplug it, come back in twenty years, the words are still there.
- **RAM** = Random Access Memory = **a whiteboard.** Write anything, erase anything, instantly. But it's *volatile*: cut the power and it goes blank.

("Random access" is a historical term meaning "you can jump straight to any location," as opposed to tape, where you have to wind through everything in between. Both ROM and RAM are random access. The name is a fossil.)

### Why both exist

| | ROM | RAM |
|---|---|---|
| Cost per byte (1989) | Very cheap | Expensive |
| Can be written? | No | Yes |
| Survives power off? | Yes | No |
| Good for | Game code, graphics, level data, music | Variables, the stack, current game state |

The economics dictate the design: **put everything you possibly can in ROM, because RAM is precious.** The Game Boy has 8 KB of work RAM and games can have 8,000 KB of ROM. That ratio — 1000:1 — shapes how Game Boy games are written. Level data isn't "loaded into RAM"; it's read directly from ROM as needed. Graphics aren't decompressed into a buffer; tiles are copied from ROM into VRAM in small batches.

### The types you'll meet

- **Mask ROM** — the game cartridge. Contents literally etched into the silicon during manufacturing. Cheapest possible per byte at volume, but a typo costs you a new production run.
- **SRAM** — Static RAM. The Game Boy's internal work RAM, VRAM, OAM, HRAM. Fast, no refresh needed, but each bit costs ~6 transistors, so it's expensive.
- **Battery-backed SRAM** — RAM in a cartridge with a watch battery soldered next to it. This is how Pokémon saves your game. It's RAM, so it's volatile — but the battery keeps power on it forever. When the battery dies, your save dies. Which is why your childhood Pokémon Red eventually forgot everything.

> **A gift for you:** you now understand, at a hardware level, why old Game Boy saves die. That's the kind of understanding this document is trying to produce everywhere.

### ⚙ Emulator implication

ROM = an array you load from file and never write to. But writes *to ROM addresses* are not ignored — on a real cartridge they're intercepted by the mapper chip and used as commands. That's Part 6.

---

## 2.6 Registers, in depth

### Why registers exist

Reading from memory is slow: put the address on the bus, wait for the memory chip, receive the byte. On the Game Boy that's a full machine cycle — 4 clock ticks. Reading from a register is *instant*, because it's inside the CPU.

So registers are the working surface. **Memory is the warehouse; registers are your hands.** You can only hold a couple of things at a time, so the art of assembly programming is minimizing trips to the warehouse.

### The 8-bit / 16-bit pairing trick

CHIP-8 gave you `V0`–`VF`: sixteen 8-bit registers, plus `I` for addresses. Simple and orthogonal.

The Game Boy gives you these:

```
   ┌─────────┬─────────┐
   │    A    │    F    │      A = Accumulator, F = Flags
   ├─────────┼─────────┤
   │    B    │    C    │  ──► can be used together as "BC" (16-bit)
   ├─────────┼─────────┤
   │    D    │    E    │  ──► can be used together as "DE" (16-bit)
   ├─────────┼─────────┤
   │    H    │    L    │  ──► can be used together as "HL" (16-bit)
   ├─────────┴─────────┤
   │        SP         │      Stack Pointer  (always 16-bit)
   ├───────────────────┤
   │        PC         │      Program Counter (always 16-bit)
   └───────────────────┘
```

The pairing is the important idea. **Addresses are 16 bits, but registers are 8 bits.** Rather than adding separate 16-bit registers (expensive), the designers let you glue two 8-bit registers together and treat the pair as one 16-bit value.

So `H` holds the *high* byte and `L` the *low* byte, and `HL` is a 16-bit address:

```
     H = $C0        L = $A7
     ┌────────┐    ┌────────┐
     │  1100  │    │  1010  │
     │  0000  │    │  0111  │
     └────────┘    └────────┘
          └───────┬──────┘
                  ▼
            HL = $C0A7
```

`HL` is roughly the equivalent of CHIP-8's `I` register — a pointer into memory — except you have *three* of them (`BC`, `DE`, `HL`), you can do arithmetic on them, and you can access their halves independently. That flexibility is a big part of why Game Boy code is denser and more efficient than CHIP-8.

### ⚙ Emulator implication

You have a design decision on day one: store eight separate `uint8_t`s and combine on demand, or store four `uint16_t`s and mask/shift for halves. Both work. The first is usually clearer; the second is sometimes faster. Pick one and be consistent — mixing them is a reliable source of bugs.

---

## 2.7 Flags

### The problem this solves

The ALU computes `5 - 5 = 0`. Fine. But the *program* wants to know: "were those two values equal?" The answer is encoded in the fact that the result was zero — but if we just store the result in `A`, we'd have to test it separately.

So the CPU keeps a few extra bits that describe *the last operation's outcome*. Those are **flags**.

### Intuition: the dashboard warning lights

Your car's engine does its work, and a few lights on the dash summarize the state: check engine, low fuel, door ajar. You don't read the engine; you read the lights.

The `F` register is the Game Boy CPU's dashboard. Four lights:

| Flag | Bit | Name | Lights up when... |
|---|---|---|---|
| **Z** | 7 | Zero | the result was exactly 0 |
| **N** | 6 | Subtract | the last op was a subtraction |
| **H** | 5 | Half-carry | a carry happened out of bit 3 (the low nibble overflowed) |
| **C** | 4 | Carry | a carry happened out of bit 7 (the byte overflowed) |

Bits 3–0 of `F` are unused and are **always zero** — you cannot set them, even by writing to `AF`. That's a real hardware behavior a test ROM will check.

### Why each flag exists

**Z (Zero)** — the basis of all comparison. `CP B` is a subtraction that throws away the result and keeps only the flags. If Z is set, `A == B`. Every `if` statement in every Game Boy game is built on this.

**C (Carry)** — 8 bits hold 0–255. `200 + 100 = 300`, which doesn't fit; you get 44 with the carry flag set. That flag is the 9th bit. It's how you do 16-bit and 32-bit arithmetic on an 8-bit machine: add the low bytes, then add the high bytes *plus the carry*. It's also the "borrow" indicator for subtraction, which means it doubles as an unsigned less-than test.

**N and H** — these two exist for exactly one instruction: `DAA` (Decimal Adjust Accumulator). Here's the story, because it's a beautiful example of "why was it designed this way?"

> Games display scores as decimal digits. But binary arithmetic isn't decimal. So programmers use **BCD** (Binary-Coded Decimal): each nibble holds one decimal digit, so `$27` represents twenty-seven, not 39.
>
> Now add `$27 + $15` in binary: you get `$3C`. But in BCD you wanted `$42`. `DAA` fixes this up automatically — but to do it correctly, it needs to know two things: *was the last operation an add or a subtract* (correction goes the opposite direction), and *did the low nibble overflow*. Hence N and H.
>
> That's it. That's why those flags exist. They exist so that Pokémon can show you "Money: ¥3,000" without a division routine.

### ⚙ Emulator implication

Flag handling is where beginners lose weeks. Every arithmetic instruction sets flags in a specific pattern, and a single wrong H flag will make one game desync in a way you'll never trace by eye. Two rules:

1. Write flag logic **once**, in helper functions (`alu_add`, `alu_sub`, `alu_inc`), and call them from every relevant opcode. Never inline flag math per-opcode.
2. Half-carry has a lovely trick: `((a & 0xF) + (b & 0xF)) > 0xF`. For subtraction: `(a & 0xF) < (b & 0xF)`.

---

## 2.8 The stack

### The problem this solves

Your program calls a subroutine. The subroutine finishes. **Where does it return to?**

You need to remember the address. Fine — store it somewhere. But what if the subroutine calls another subroutine? Now you need to remember two addresses. And what if that one calls another? You need arbitrary depth, and you need the *most recent* one first.

That data structure is a **stack**, and it's so fundamental that CPUs implement it in hardware.

### Intuition: the pile of plates

A stack of plates. You put a plate on top (**push**). You take a plate from the top (**pop**). You cannot take one from the middle. Last in, first out.

Or better, for our purposes: **a trail of breadcrumbs.** Every time you go somewhere new, you drop a crumb saying where you came from. To get home, follow the crumbs backwards.

### How it works

The **Stack Pointer (SP)** is a 16-bit register holding the address of the top of the stack. On the Game Boy, the stack **grows downward** — pushing *decreases* SP.

```
  Memory addresses increase downward in this diagram.

           $FFFE  ◄── SP starts here (typical)
                       (grows downward as you push)
           $FFFD
           $FFFC
              ...
           [ free RAM in the middle ]
              ...
           $C000  ◄── your variables start here and grow UP
```

> 🤔 **Why downward?** So that the stack and your variables start at opposite ends of RAM and grow toward each other. That way you don't have to decide in advance how much space each needs — they share the free middle. It's the same trick a Unix process uses with its heap and stack. Elegant, and it fails silently and catastrophically when they collide, which is the classic "stack overflow."

**Push (16-bit):**
```
  SP = SP - 1;  memory[SP] = high_byte
  SP = SP - 1;  memory[SP] = low_byte
```

**Pop (16-bit):**
```
  low_byte  = memory[SP];  SP = SP + 1
  high_byte = memory[SP];  SP = SP + 1
```

**`CALL nn`** is: push PC (which already points to the instruction *after* the CALL), then set PC = nn.
**`RET`** is: pop into PC.

That's the whole mechanism. Subroutines, nested calls, recursion — all of it is that pair of operations.

### CHIP-8 comparison

CHIP-8 had a stack too, but it was a *private* stack: a fixed array of 12–16 entries, invisible to the program, usable only for `CALL`/`RET`. You couldn't push arbitrary values onto it, couldn't read it, couldn't move it.

The Game Boy's stack is **in normal RAM at an address you choose**. That means:
- You can push and pop *any* register pair (`PUSH BC`, `POP HL`) — it's a general-purpose scratch area, and games use it constantly to free up registers.
- You can point SP anywhere. Games sometimes point SP at VRAM and use `POP` as an ultra-fast memory-copy instruction, because `POP` reads two bytes and advances the pointer in one instruction.
- **You can overflow it into your own data**, and nothing will warn you.
- Interrupts push onto it *without asking*, so it must always be valid.

That last point matters more than it sounds. Every interrupt costs 2 bytes of stack. If SP is garbage, an interrupt corrupts memory.

### ⚙ Emulator implication

Push/pop order (high byte first, at the decremented address) must be exactly right or every `RET` lands somewhere insane. Also note `PUSH` takes 16 cycles and `POP` takes 12 — asymmetric, because the push has an extra internal step. Details like that are why cycle-accurate emulators feel fussy.

---

## 2.9 Interrupts (the general idea — details in Part 9)

### The problem this solves

The graphics chip finishes drawing a frame. It needs to tell the CPU: "now is a safe moment to update the screen."

**Option A: polling.** The CPU repeatedly reads a status register in a loop until it changes. This works! It's also a catastrophic waste — the CPU does nothing but ask "are we there yet?" millions of times.

**Option B: interrupts.** The CPU does useful work. When the event happens, hardware *taps the CPU on the shoulder*, the CPU finishes its current instruction, bookmarks where it was, and jumps to a handler routine. When the handler finishes, the CPU returns to exactly where it left off, as if nothing happened.

### Intuition: the phone call

You're reading a book. The phone rings.

1. You finish the *sentence* you're on. (The CPU finishes the current instruction — it never stops mid-instruction.)
2. You put a bookmark in the book. (Push PC onto the stack.)
3. You answer the phone and deal with it. (Jump to the interrupt handler.)
4. You hang up, open the book to the bookmark, and continue. (`RETI` — pop PC.)

You did not have to keep checking the phone. It told you.

There's a subtlety worth noticing: **you must not lose your place.** If answering the phone made you forget which page you were on, interrupts would be useless. That's what the stack push is for, and it's why interrupt handlers must save and restore any registers they modify — the interrupted code has no idea it was interrupted and expects its registers untouched.

### ⚙ Emulator implication

Interrupts are checked **between instructions**, not during. Your CPU step function will end with something like "if interrupts are enabled and one is pending, service it." Getting the *exact* timing of enable/disable right (`EI` takes effect one instruction late; `HALT` has a famous bug) is a Part 9 topic.

---

## 2.10 DMA — Direct Memory Access

### The problem this solves

You need to copy 160 bytes from RAM into the sprite table. Using the CPU:

```
  loop:  LD A, (HL)     ; read a byte
         LD (DE), A     ; write it
         INC HL
         INC DE
         DEC B
         JR NZ, loop
```

That's roughly 5 instructions × 160 iterations ≈ 1,300+ cycles of the CPU doing nothing but shuttling bytes. And it must happen during the brief window when the sprite table is accessible.

### Intuition: the conveyor belt

Instead of carrying boxes from room A to room B yourself, you install a conveyor belt, flip a switch, and go do something else while it runs.

**DMA is a small dedicated circuit whose only skill is copying bytes from one place to another without the CPU's involvement.** You tell it "start here," and it does the rest.

### How the Game Boy's works

Write a byte `XX` to address `$FF46`. That says: *copy 160 bytes from `$XX00` to OAM (`$FE00`–`$FE9F`)*. The transfer runs over the next 160 machine cycles (640 clock ticks) while the CPU continues executing.

**But** — and this is the crucial part, and it's the shared-road principle from §2.2 coming back — the DMA unit is *using the bus*. The CPU can't. During DMA, almost all memory reads by the CPU return garbage.

> 🤔 **So how does any code run during DMA?**
> This is one of my favorite pieces of Game Boy design, and it's the answer to "why does HRAM exist?"
>
> **HRAM** (`$FF80`–`$FFFE`, 127 bytes) sits *inside the CPU chip itself*. Accessing it doesn't use the external bus. So it remains available during DMA.
>
> Therefore every Game Boy game contains a tiny routine, **copied into HRAM at startup**, that looks roughly like: start the DMA, then count down for the right number of cycles, then return. The CPU executes from HRAM, safely, while the conveyor belt runs across a bus it can't touch.
>
> Look at that: an architectural quirk (127 bytes of on-chip RAM) exists to solve a bus-contention problem created by another feature (DMA). Nothing on this machine is arbitrary. **This is exactly the kind of connection Pan Docs will state in one sentence and assume you already understand.**

### ⚙ Emulator implication

Many beginner emulators implement DMA as an instant `memcpy` on write to `$FF46`. That works for the vast majority of games and is a completely reasonable starting point. The accurate version transfers one byte per machine cycle and blocks CPU bus access. Start simple; know that the accurate version exists.

---

## 2.11 Clock cycles, and what "4.19 MHz" actually means

### Intuition: the metronome

Somewhere on the board is a crystal oscillator — a sliver of quartz that vibrates at an extremely precise frequency when you apply voltage. That vibration becomes a square wave: **tick, tick, tick**, 4,194,304 times per second.

Every digital component listens to this metronome. Nothing happens *between* ticks. Digital logic is not continuous; it's a series of frozen frames, and the clock advances the frames.

### Why that specific number?

4,194,304 = 2²². It's a power of two, which makes clean division trivial in hardware (just tap a bit of a counter). Divide by 2²² and you get 1 Hz. Divide by 2¹⁶ and you get 64 Hz. Every timer frequency on the Game Boy is a power-of-two division of this number, which is why they look like 4096 Hz, 16384 Hz, 65536 Hz, 262144 Hz. Those aren't arbitrary — they're just `4194304 >> n`.

### T-cycles vs. M-cycles — learn this now, it's everywhere

Two units, and the community uses both, often without saying which:

- **T-cycle** ("T-state," "clock cycle," "tick") = one oscillation. 4,194,304 per second.
- **M-cycle** ("machine cycle") = **4 T-cycles.** This is the CPU's *actual* smallest useful step — the time it takes to do one bus transaction (one memory read or one memory write).

```
   T:  1   2   3   4   5   6   7   8   9  10  11  12
       ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐
       ┘ └─┘ └─┘ └─┘ └─┘ └─┘ └─┘ └─┘ └─┘ └─┘ └─┘ └─┘ └
       └──── M-cycle 1 ────┘└──── M-cycle 2 ────┘└─ M3 ─
             (one memory
              access)
```

So an instruction like `LD B, (HL)` takes 2 M-cycles = 8 T-cycles: one M-cycle to fetch the opcode, one to read the byte at `HL`.

**Everything the CPU does is a whole number of M-cycles.** You will never see a Game Boy instruction that takes 6 T-cycles. This is a useful sanity check on your opcode timing table: every entry should be divisible by 4.

> ⚠️ **The confusion tax.** Pan Docs mostly says "cycles" meaning T-cycles (or "dots"). Some documents count in M-cycles. The 33C3 talk mixes them. If a number seems off by exactly 4×, this is why. **Pick T-cycles for your emulator's internal counter** — it's the finest granularity, and everything else divides into it cleanly.

### Why this matters so much more than in CHIP-8

In CHIP-8, you invented time. Here, time is *the shared coordinate system* that all components agree on:

```
   T-cycle counter ─────────────────────────────────────────►
        │              │              │              │
        ▼              ▼              ▼              ▼
      CPU            PPU           TIMER            APU
   "I executed     "I drew        "I ticked      "I generated
    an ADD, that    4 more         DIV up          4 samples
    was 4 ticks"    pixels"        by 1"           worth"
```

Every component consumes the same ticks. Getting a component's tick accounting wrong doesn't just make it slow — it makes it *disagree with the others*, and games notice.

### ⚙ Emulator implication

Your main loop, in essence:

```
while (running) {
    cycles = cpu.step();      // execute one instruction, return T-cycles used
    ppu.step(cycles);         // let the PPU catch up
    timer.step(cycles);       // let the timer catch up
    apu.step(cycles);         // let the sound catch up
    cpu.handle_interrupts();
}
```

That's the whole architecture. Simple to state. The rest of this document is about what goes inside each of those `step` functions — and about the games that require the interleaving to be *finer* than one instruction at a time.

---

## 2.12 Peripherals

### The concept

A **peripheral** is any chip or circuit that isn't the CPU or plain memory. It does a specific job, it's wired to the address bus at particular addresses, and it usually runs *independently and continuously* — it doesn't wait for the CPU to ask.

That last part is the mental leap from CHIP-8. Your CHIP-8 "display" only did something when `DXYN` ran. **The Game Boy's PPU is running right now, whether or not the CPU is paying attention.** It's drawing line 42 of the current frame regardless of what your code is doing. The CPU can look, and the CPU can change the rules, but it cannot make the PPU stop and wait.

Same for the timer. Same for the sound channels. They are *co-workers*, not *subroutines*.

### The peripheral pattern

Every peripheral follows the same three-part shape, and once you see the pattern you can learn any of them quickly:

```
  ┌──────────────────────────────────────────────────┐
  │  PERIPHERAL                                      │
  │                                                  │
  │  1. CONTROL registers  — CPU writes these to     │
  │     configure behavior      ("draw sprites: on")  │
  │                                                  │
  │  2. STATUS registers   — CPU reads these to      │
  │     observe state           ("currently line 42") │
  │                                                  │
  │  3. INTERRUPT line     — peripheral raises this  │
  │     to demand attention     ("frame done!")       │
  └──────────────────────────────────────────────────┘
```

The Game Boy's peripherals:

| Peripheral | Job | Registers | Interrupt? |
|---|---|---|---|
| **PPU** | Draws the screen | `$FF40`–`$FF4B` | VBlank, STAT |
| **Timer** | Counts, fires periodically | `$FF04`–`$FF07` | Timer |
| **APU** | Generates sound | `$FF10`–`$FF3F` | No |
| **Joypad** | Reads buttons | `$FF00` | Joypad |
| **Serial** | Link cable | `$FF01`–`$FF02` | Serial |
| **DMA** | Bulk copy to OAM | `$FF46` | No |
| **Cartridge (MBC)** | Bank switching | writes to `$0000`–`$7FFF` | No |

Learn that table's *shape*, not its contents. When you meet a new register in Pan Docs, your first question should be: "is this control, status, or something that triggers an action?"

---

## 2.13 Checkpoint: what you now know

Before moving on, check yourself. You should be able to explain, in your own words:

- Why "64 KB" comes from the number 16.
- The difference between an address space and physical memory.
- Why `$FF44` isn't storage.
- Why the stack grows downward.
- Why interrupts are better than polling.
- Why HRAM exists. *(If you can answer this one, you've genuinely understood something most beginners take months to get.)*
- What an M-cycle is and why every instruction's timing is divisible by 4.

If any of those are shaky, reread that section now. Part 3 assumes all of them.

---
---

# Part 3: The Game Boy as a Whole System

Now let's assemble the pieces. This part is short on detail and long on *shape* — I want you to be able to close your eyes and see the machine.

## 3.1 The bird's-eye view

```
    ┌───────────────────────────────────────────────────────────────┐
    │                    SHARP LR35902 ("DMG-CPU")                  │
    │   One chip containing:                                        │
    │                                                               │
    │   ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐  │
    │   │   SM83   │  │   PPU    │  │   APU    │  │  Timer /     │  │
    │   │   CPU    │  │ graphics │  │  sound   │  │  Joypad /    │  │
    │   │  core    │  │          │  │          │  │  Serial /DMA │  │
    │   └────┬─────┘  └────┬─────┘  └────┬─────┘  └──────┬───────┘  │
    │        │             │             │               │          │
    │   ┌────┴─────┐  ┌────┴─────┐  ┌────┴──────────┐    │          │
    │   │  Boot    │  │   VRAM   │  │  Wave RAM     │    │          │
    │   │  ROM     │  │   8 KB   │  │  16 bytes     │    │          │
    │   │  256 B   │  │          │  └───────────────┘    │          │
    │   └──────────┘  │   OAM    │                       │          │
    │                 │  160 B   │   ┌────────────┐      │          │
    │                 └──────────┘   │   HRAM     │      │          │
    │                                │  127 B     │      │          │
    │                                └────────────┘      │          │
    └───────────────────────────┬───────────────────────┬───────────┘
                                │                       │
              ══════════════════╪═══════════════════════╪═══════════
                        SYSTEM BUS (16 addr + 8 data)
              ══════════════════╪═══════════════════════╪═══════════
                                │                       │
                    ┌───────────┴────────┐   ┌──────────┴──────────┐
                    │   WORK RAM (WRAM)  │   │   CARTRIDGE SLOT    │
                    │       8 KB         │   │                     │
                    │   (external chip)  │   │  ┌───────────────┐  │
                    └────────────────────┘   │  │  Game ROM     │  │
                                             │  │  32KB - 8MB   │  │
        ┌──────────────┐                     │  ├───────────────┤  │
        │  LCD PANEL   │◄────────────────────│  │  MBC chip     │  │
        │  160 x 144   │   (PPU drives it    │  │ (bank switch) │  │
        │  4 shades    │    directly, not    │  ├───────────────┤  │
        └──────────────┘    via the bus)     │  │ Save RAM +    │  │
                                             │  │ battery (opt) │  │
        ┌──────────────┐                     │  └───────────────┘  │
        │  SPEAKER     │◄────── APU          └─────────────────────┘
        └──────────────┘

        ┌──────────────┐
        │ 8 BUTTONS    │──────► Joypad register $FF00
        └──────────────┘
```

Three things to notice immediately:

1. **The "CPU" is not just a CPU.** The Sharp LR35902 is a *system-on-chip*: it contains the CPU core, the graphics processor, the sound processor, the timers, the DMA controller, and several small memories. When Pan Docs or the 33C3 talk says "the CPU," check whether they mean the whole chip or the SM83 core inside it.

2. **VRAM, OAM, and HRAM are inside the chip. WRAM is outside.** This isn't trivia — it's why HRAM survives DMA, and it's part of why VRAM access has special rules.

3. **The LCD is not on the bus.** The PPU drives the display panel directly with its own signals. The CPU never "writes pixels to the screen." The CPU writes *tile data and map data into VRAM*, and the PPU independently turns that into pixels. This is the single biggest conceptual break from CHIP-8, and Part 7 is entirely about it.

## 3.2 The dataflow: how a sprite gets on screen

Let's trace one concrete thing end to end. Suppose Mario needs to move one pixel to the right.

```
   1. GAME CODE (in cartridge ROM, running on the CPU)
      "mario_x = mario_x + 1"
                    │
                    ▼
   2. WRITE TO WRAM
      The game's variable for Mario's X position lives in WRAM.
      CPU: write($C0A3, 57)
                    │
                    ▼
   3. GAME BUILDS A SPRITE TABLE IN WRAM
      Somewhere in WRAM the game maintains a 160-byte
      "shadow OAM" — a copy of what it wants the sprite
      table to look like. It writes Mario's new X there.
                    │
                    ▼
   4. WAIT FOR VBLANK INTERRUPT
      The PPU finishes drawing the frame and raises the
      VBlank interrupt. CPU jumps to $0040.
                    │
                    ▼
   5. START DMA (from the HRAM routine)
      CPU: write($FF46, $C0)
      → DMA engine copies $C000-$C09F into OAM ($FE00-$FE9F)
      → takes 160 M-cycles; CPU spins in HRAM meanwhile
                    │
                    ▼
   6. NEXT FRAME: PPU READS OAM
      For each scanline, the PPU scans OAM for sprites
      that intersect this line, fetches their tile data
      from VRAM, and mixes them with the background.
                    │
                    ▼
   7. PIXELS OUT TO THE LCD
      One pixel per dot, 160 per line, 144 lines,
      ~59.7 times per second.
```

Notice how many *separate systems* had to cooperate: CPU, WRAM, interrupt controller, DMA engine, HRAM, OAM, VRAM, PPU, LCD. In CHIP-8, this was `DXYN`.

Notice also step 4. **Why wait for VBlank?** Because during drawing, OAM is locked — the PPU is using it. Update it mid-frame and you get tearing or dropped sprites. VBlank is the ~1.1 ms window when the PPU isn't drawing and the CPU may safely rearrange the world. Essentially every Game Boy game is structured around that window.

## 3.3 The rhythm of the machine

Here is the heartbeat you should carry in your head:

```
  ONE FRAME = 70,224 T-cycles ≈ 16.74 ms ≈ 59.73 Hz

  ┌────────────────────────────────────────────────────────────┐
  │ Lines 0-143: VISIBLE                                       │
  │   Each line = 456 T-cycles                                 │
  │   PPU is drawing. VRAM/OAM often locked. CPU does          │
  │   background work: AI, physics, music, decompression.      │
  │   (144 lines × 456 = 65,664 T-cycles)                      │
  ├────────────────────────────────────────────────────────────┤
  │ Lines 144-153: VBLANK                                      │
  │   10 lines × 456 = 4,560 T-cycles ≈ 1.09 ms                │
  │   PPU is idle. VRAM and OAM are free.                      │
  │   CPU does: OAM DMA, VRAM updates, scroll changes.         │
  │   THIS IS WHEN THE GAME CHANGES WHAT YOU'LL SEE.           │
  └────────────────────────────────────────────────────────────┘
                            │
                            └──► repeat forever
```

Every Game Boy game's main loop is:

```
  main_loop:
      do all the thinking          ; during visible lines
      wait for VBlank interrupt
      push all the changes to VRAM/OAM   ; during VBlank
      jump to main_loop
```

If the "thinking" takes too long and spills past VBlank, the game misses its window and either glitches or drops a frame. That's why classic games sometimes slow down when there's a lot on screen — the CPU literally ran out of time before the PPU came back around.

## 3.4 The five ways components talk to each other

Worth naming explicitly, because Pan Docs uses all five without introduction:

| Mechanism | Example | Direction |
|---|---|---|
| **CPU reads a register** | `LD A, ($FF44)` → current scanline | Peripheral → CPU |
| **CPU writes a register** | `LD ($FF42), A` → set scroll Y | CPU → Peripheral |
| **Shared memory** | CPU writes VRAM; PPU reads VRAM | Both, asynchronously |
| **Interrupt** | PPU raises VBlank | Peripheral → CPU (urgent) |
| **DMA** | Bulk copy WRAM → OAM | Memory → Memory, no CPU |

That's the whole communication vocabulary of the machine. Everything else is a combination of these five.

---
---

# Part 4: The CPU

## 4.1 What CPU is this, actually?

You'll see three names, and the confusion is real:

- **Sharp LR35902** — the full system-on-chip.
- **SM83** — the CPU core inside it. This is the modern community's preferred name for the instruction set.
- **"a modified Z80"** or **"an 8080 with extras"** — how older documentation describes it. Both are *wrong but useful*.

The honest description: **the SM83 is its own design that borrows the Intel 8080's register layout and adds a subset of the Zilog Z80's extensions, plus a handful of instructions that exist on neither.**

| | Intel 8080 | Zilog Z80 | Sharp SM83 |
|---|---|---|---|
| Registers A,B,C,D,E,H,L | ✅ | ✅ | ✅ |
| Shadow register set (`AF'`, `BC'`...) | ❌ | ✅ | ❌ |
| Index registers `IX`, `IY` | ❌ | ✅ | ❌ |
| `CB`-prefixed bit instructions | ❌ | ✅ | ✅ |
| Block instructions (`LDIR`) | ❌ | ✅ | ❌ |
| Parity flag | ✅ | ✅ | ❌ (only Z, N, H, C) |
| `LDH` / `$FF00` zero-page ops | ❌ | ❌ | ✅ (unique) |
| `LD (HL+), A` auto-increment | ❌ | ❌ | ✅ (unique) |
| `STOP`, `SWAP` | ❌ | ❌ | ✅ (unique) |

**Practical warning:** never copy a Z80 opcode table into a Game Boy emulator. Several opcodes differ in meaning, timing, or flag behavior. Use a Game Boy–specific table. (The one at `izik1.github.io/gbops` is the community standard and is checked against real hardware.)

## 4.2 The register file, revisited

```
   16-bit view          8-bit view         Notes
   ───────────          ──────────         ─────
      AF        =       A   |   F          F is FLAGS, low 4 bits always 0
      BC        =       B   |   C          general purpose
      DE        =       D   |   E          general purpose
      HL        =       H   |   L          the "pointer" pair, most privileged
      SP                (16-bit only)      stack pointer
      PC                (16-bit only)      program counter
```

**Why `A` is special:** almost all arithmetic goes through `A`. `ADD A, B` — the destination is always `A`. There is no `ADD B, C`. This is called an *accumulator architecture*, and it exists because it makes opcodes smaller: you don't need bits to encode the destination when there's only one possible destination. In a machine where every byte of ROM costs money, that's a real win.

**Why `HL` is special:** `HL` is the default memory pointer. `(HL)` — meaning "the byte at the address in HL" — is available almost everywhere a register is. In the opcode table, `(HL)` occupies the slot where a 7th register would be, so `LD B, (HL)` sits right next to `LD B, A`. It behaves like a register that happens to live in memory.

Compare with CHIP-8: you had `I` as your only pointer, and only a few instructions used it. Here, pointer-based access is pervasive, which is why Game Boy code manipulates data structures so much more naturally.

**Post-increment addressing** is one of the SM83's genuinely nice inventions:

```
  LD A, (HL+)    ; read the byte at HL into A, THEN increment HL
  LD (HL-), A    ; write A to the byte at HL, THEN decrement HL
```

One instruction, two operations. Copying a block of memory becomes:

```
  copy_loop:
      LD A, (HL+)     ; read + advance source
      LD (DE), A      ; write
      INC DE
      DEC BC
      LD A, B
      OR C            ; is BC zero?
      JR NZ, copy_loop
```

You will see this exact loop in almost every Game Boy game. Recognize it on sight.

## 4.3 Flags in practice

We covered *why* they exist in §2.7. Here's *how you'll use them*, which is what actually matters when debugging.

```
   F register:   bit 7   6   5   4   3   2   1   0
                 ┌───┬───┬───┬───┬───┬───┬───┬───┐
                 │ Z │ N │ H │ C │ 0 │ 0 │ 0 │ 0 │
                 └───┴───┴───┴───┴───┴───┴───┴───┘
```

The compare-and-branch idiom, which is every conditional in every game:

```
  LD A, (player_health)
  CP 0                    ; compute A - 0, discard result, set flags
  JR Z, player_died       ; jump if the Z flag is set
```

`CP` = "compare" = subtract without storing. The result goes in the bin; the flags are the product.

**Conditional instructions** come in four flavors, and the naming is worth internalizing:

| Condition | Meaning | Typical use |
|---|---|---|
| `Z` | Zero flag set | values were equal |
| `NZ` | Zero flag clear | values differ |
| `C` | Carry flag set | unsigned less-than (after `CP`) |
| `NC` | Carry flag clear | unsigned greater-or-equal |

Note that `C` is doing double duty as both a register name and a condition name. `JR C, label` means "jump if carry"; `LD A, C` means "load register C." Context disambiguates. This trips up everyone once.

**The trap that will cost you a weekend:** conditional jumps and calls take *different cycle counts depending on whether they're taken.* `JR NZ, e` is 12 T-cycles if taken, 8 if not. `CALL Z, nn` is 24 taken, 12 not taken. Your opcode timing table needs two columns for these instructions. If you use one number, timing drifts and eventually something desyncs.

## 4.4 The opcode table has a structure (and this will save you)

256 opcodes looks like 256 things to memorize. It isn't. **The table is arranged in a grid, and the grid encodes meaning in the bit pattern.**

Take the `LD r, r'` block — opcodes `$40`–`$7F`. The pattern is:

```
        01  d d d   s s s
        ▲   ▲       ▲
        │   │       └── source register (3 bits)
        │   └────────── destination register (3 bits)
        └────────────── "this is a register-to-register load"

   Register encoding (3 bits):
     000 = B     001 = C     010 = D     011 = E
     100 = H     101 = L     110 = (HL)  111 = A
```

So `$47` = `%01 000 111` = `LD B, A`. And `$78` = `%01 111 000` = `LD A, B`. You can *decode these in your head* once you see the pattern.

The one exception: `%01 110 110` would be `LD (HL), (HL)`, which is meaningless. So that slot — opcode `$76` — was repurposed for **`HALT`**. A famous piece of trivia that's actually a nice illustration of how ISA designers use every scrap of encoding space.

The same regularity appears in the arithmetic block (`$80`–`$BF`):

```
        10  o o o   s s s
            ▲       ▲
            │       └── source register
            └────────── operation:
                        000 = ADD   001 = ADC
                        010 = SUB   011 = SBC
                        100 = AND   101 = XOR
                        110 = OR    111 = CP
```

Here's the visual map of the whole table:

```
        ═══════════════════════════════════════════════════
        $00-$3F   Misc: loads of immediates, INC/DEC,
                  16-bit ops, jumps, rotates on A
        ───────────────────────────────────────────────────
        $40-$7F   LD r, r'      (highly regular grid)
                  ...except $76 = HALT
        ───────────────────────────────────────────────────
        $80-$BF   ALU A, r      (highly regular grid)
                  ADD ADC SUB SBC AND XOR OR CP
        ───────────────────────────────────────────────────
        $C0-$FF   Control flow: RET, POP, JP, CALL, PUSH,
                  RST, plus the odds and ends, plus $CB
        ═══════════════════════════════════════════════════
```

**The `$CB` prefix.** 256 opcodes weren't enough. So opcode `$CB` means "the *next* byte is from a second, different table." That second table is 256 more instructions, all bit manipulation:

| CB range | Instruction | What it does |
|---|---|---|
| `$00`–`$3F` | `RLC`, `RRC`, `RL`, `RR`, `SLA`, `SRA`, `SWAP`, `SRL` | rotates and shifts |
| `$40`–`$7F` | `BIT n, r` | test bit n, set Z flag |
| `$80`–`$BF` | `RES n, r` | clear bit n |
| `$C0`–`$FF` | `SET n, r` | set bit n |

`BIT`/`RES`/`SET` encode as `%oo bbb rrr` — operation, bit number, register. Perfectly regular. This is why `$CB` instructions are the *easiest* 256 opcodes to implement: write eight shift/rotate functions and three bit functions, then loop.

> ⚙ **Emulator implication:** resist the urge to hand-write 512 `case` statements. Decode the bit patterns. A well-structured decoder is maybe 300 lines instead of 3,000, and it has far fewer typos — and typo-driven bugs in opcode tables are *miserable* to find.

## 4.5 Instruction categories

Everything the SM83 can do falls into six buckets:

**1. Load / store (the majority).** Move bytes between registers, memory, and immediates. `LD A, B`. `LD (HL), $42`. `LD A, ($FF44)`. Nothing computes; things move.

**2. Arithmetic / logic.** `ADD`, `ADC`, `SUB`, `SBC`, `AND`, `OR`, `XOR`, `CP`, `INC`, `DEC`. Plus 16-bit `ADD HL, rr`. **No multiply. No divide.** If a game needs `x * 12`, someone wrote shifts and adds.

**3. Bit operations.** Rotates, shifts, `BIT`/`SET`/`RES`, `SWAP` (exchange nibbles). Heavily used, because on a machine this small you pack multiple flags into one byte constantly.

**4. Jumps and calls.** `JP` (absolute), `JR` (relative, ±127 bytes, one byte shorter and one cycle faster), `CALL`, `RET`, `RETI`, and `RST n` — a one-byte call to one of eight fixed addresses (`$00`, `$08`, `$10`... `$38`). `RST` exists because a one-byte call to a common routine saves two bytes every time you use it. In a 32 KB game, that adds up.

**5. Stack.** `PUSH rr`, `POP rr`, plus `ADD SP, e8` and `LD HL, SP+e8` for stack-relative addressing.

**6. CPU control.** `NOP`, `HALT`, `STOP`, `DI`, `EI`, `DAA`, `CPL`, `SCF`, `CCF`.

Compare to CHIP-8's 35 opcodes: same *categories*, roughly, but CHIP-8 had `DXYN` (draw) and `FX0A` (wait for key) as *instructions*. On the Game Boy, drawing and input aren't instructions — they're peripherals you talk to through memory. **The instruction set got more general so the machine could get more capable.**

## 4.6 Fetch–decode–execute, and why it's not what you think

Your CHIP-8 loop was:

```
   opcode = (memory[PC] << 8) | memory[PC+1]
   PC += 2
   switch (opcode & 0xF000) { ... }
```

Clean phases. Fetch, then decode, then execute.

Real hardware doesn't work that way, and understanding this will demystify half the "cycle accuracy" discourse.

**The SM83 overlaps the fetch of the next instruction with the last cycle of the current one.**

```
   Naive model:
   ┌────────┬────────┬────────┐┌────────┬────────┬────────┐
   │ FETCH  │ DECODE │ EXEC   ││ FETCH  │ DECODE │ EXEC   │
   └────────┴────────┴────────┘└────────┴────────┴────────┘
        instruction 1               instruction 2

   Actual hardware:
   ┌────────┬────────┬─────────────┐
   │ FETCH  │ DECODE │   EXEC      │
   └────────┴────────┴──────┬──────┘
                            │ ┌────────┬────────┬─────────┐
                            └►│ FETCH  │ DECODE │  EXEC   │
                              └────────┴────────┴─────────┘
                              (the next fetch happens DURING
                               the last cycle of the previous
                               instruction)
```

This is a one-stage **pipeline**, and it's why instruction timings look "off by one" sometimes, and why the exact cycle on which a memory access happens matters.

**Do you need to model this?** Not at first. A perfectly good first emulator treats each instruction as atomic: execute it entirely, then report "that took N cycles." Ninety-plus percent of games work fine. But knowing the pipeline exists explains:

- Why `HALT` has a bug (the fetch and the interrupt check collide).
- Why writing to a PPU register "one cycle too late" changes what gets drawn.
- Why the community obsesses over M-cycle-accurate emulation.

**The important refinement to your mental model:** instead of "execute, then advance time by N," accurate emulators do "advance time in M-cycle steps, doing one bus access per step." Like this:

```
   LD A, (nn)     ; 16 T-cycles = 4 M-cycles

   M-cycle 1:  read opcode byte at PC          PC++
   M-cycle 2:  read low byte of nn at PC       PC++
   M-cycle 3:  read high byte of nn at PC      PC++
   M-cycle 4:  read the byte at address nn  →  A
```

Every M-cycle is exactly one bus transaction. Once you see instructions this way, their timings stop being a lookup table you memorize and start being *derivable*: count the memory accesses, multiply by 4, add internal-operation cycles.

## 4.7 Instruction timing, concretely

| Instruction | T-cycles | Why |
|---|---|---|
| `NOP` | 4 | 1 fetch |
| `LD B, C` | 4 | 1 fetch, register-to-register is free |
| `LD B, n` | 8 | fetch + read immediate |
| `LD B, (HL)` | 8 | fetch + memory read |
| `LD (HL), n` | 12 | fetch + read immediate + memory write |
| `LD A, (nn)` | 16 | fetch + 2 address bytes + read |
| `INC BC` | 8 | fetch + an internal cycle for 16-bit math |
| `PUSH BC` | 16 | fetch + internal + 2 writes |
| `POP BC` | 12 | fetch + 2 reads |
| `JR e` | 12 | fetch + offset + internal (PC recalculation) |
| `JR NZ, e` | 12 / 8 | taken / not taken |
| `CALL nn` | 24 | fetch + 2 addr + internal + 2 pushes |
| `RET` | 16 | fetch + 2 pops + internal |
| `RST n` | 16 | fetch + internal + 2 pushes |

Look at the pattern: **cycles ≈ 4 × (number of memory accesses + internal steps)**. `PUSH` costs more than `POP` because of an extra internal cycle for the SP decrement. These aren't arbitrary numbers to memorize — they're consequences of what the hardware physically has to do.

## 4.8 HALT and STOP

**`HALT`** stops the CPU until an interrupt occurs. Why? **Battery life.** If the game is waiting for VBlank anyway, a `HALT` lets the CPU idle instead of burning through a spin loop. On four AA batteries, this is not a micro-optimization.

**The HALT bug.** If `HALT` executes while `IME` (the master interrupt enable) is off *and* an interrupt is already pending, the CPU doesn't halt — and the byte after `HALT` gets read twice. The PC fails to increment.

```
   Normal:      HALT     ; sleeps
                INC A    ; runs once when woken

   Bug case:    HALT     ; doesn't sleep; PC doesn't advance
                INC A    ; executes TWICE
```

This is not a rumor — it's documented hardware behavior, and a few games depend on it. It's also a perfect example of why "quirks" matter: it's a *bug in the silicon* that became part of the platform's definition.

**`STOP`** puts the machine into very deep sleep until a button is pressed. It's also the mechanism for switching CPU speed on the Game Boy Color. It's poorly documented, weirdly implemented (it's a two-byte instruction whose second byte is ignored), and rarely used. Implement a stub and move on.

## 4.9 What's different from CHIP-8: a summary table

| | CHIP-8 | Game Boy (SM83) |
|---|---|---|
| Opcode size | Always 2 bytes | 1–3 bytes (variable!) |
| Opcode count | 35 | ~500 (256 + 256 CB, minus gaps) |
| Registers | 16× 8-bit + I | 8× 8-bit (pairable) + SP + PC |
| Flags | `VF` used ad hoc | Dedicated `F` register, 4 flags |
| Timing | Undefined | Exact, 4–24 T-cycles per instruction |
| Memory access | Uniform array | Routed through a decoder |
| Stack | Private, fixed-size | In RAM, general-purpose |
| Interrupts | None | 5 sources |
| Illegal opcodes | N/A | 11 opcodes lock up the CPU |

**Variable-length instructions** deserve a note. In CHIP-8, `PC += 2` always. On the Game Boy, `NOP` is 1 byte, `LD B, n` is 2, `LD BC, nn` is 3. The number of bytes to advance depends on the opcode you just decoded. This has a consequence: **you cannot disassemble Game Boy code backwards or from an arbitrary offset.** You must start at a known instruction boundary. Data and code are interleaved in ROM and there's no way to tell them apart by inspection. (This is why disassembling old games is genuinely hard work.)

---
---

# Part 5: Memory Architecture

## 5.1 The map

Here it is. Print it out. Tape it to your monitor. You will look at it a thousand times.

```
  $FFFF ┌──────────────────────────────────┐
        │ IE — Interrupt Enable register   │  1 byte
  $FFFE ├──────────────────────────────────┤
        │                                  │
        │   HRAM — High RAM                │  127 bytes
        │   (inside the CPU chip;          │
        │    survives DMA)                 │
  $FF80 ├──────────────────────────────────┤
        │   I/O REGISTERS                  │  128 bytes
        │   joypad, serial, timer,         │
        │   sound, PPU control             │
  $FF00 ├──────────────────────────────────┤
        │   ▓▓▓ PROHIBITED ▓▓▓             │  96 bytes
        │   (weird behavior; don't touch)  │
  $FEA0 ├──────────────────────────────────┤
        │   OAM — Object Attribute Memory  │  160 bytes
        │   40 sprites × 4 bytes           │
  $FE00 ├──────────────────────────────────┤
        │   ECHO RAM                       │  7,680 bytes
        │   (a mirror of $C000-$DDFF)      │
  $E000 ├──────────────────────────────────┤
        │   WRAM — Work RAM                │  8 KB
        │   variables, stack, shadow OAM   │
  $C000 ├──────────────────────────────────┤
        │   EXTERNAL RAM (cartridge)       │  8 KB
        │   save data, extra scratch       │
        │   — often absent, often banked   │
  $A000 ├──────────────────────────────────┤
        │   VRAM — Video RAM               │  8 KB
        │   tile data + tile maps          │
  $8000 ├──────────────────────────────────┤
        │                                  │
        │   ROM BANK 01-NN  (switchable)   │  16 KB
        │   whichever bank the MBC selects │
        │                                  │
  $4000 ├──────────────────────────────────┤
        │                                  │
        │   ROM BANK 00  (fixed)           │  16 KB
        │   always present; contains the   │
        │   interrupt vectors, the header, │
        │   and the bank-switching code    │
  $0000 └──────────────────────────────────┘
```

Now let's go region by region and ask, for each: **why does this exist?**

## 5.2 `$0000`–`$3FFF` — ROM Bank 00

**What:** The first 16 KB of the cartridge. Always mapped, never switched.

**Why fixed?** Because if you could switch away from *all* of ROM, the bank-switching code itself would vanish mid-execution. Imagine sawing off the branch you're sitting on. Something must always be present, so bank 00 is nailed down.

**What lives here:**

```
  $0000-$00FF   Interrupt vectors and RST targets
                  $00,$08,$10,$18,$20,$28,$30,$38  — RST targets
                  $40 VBlank  $48 STAT  $50 Timer
                  $58 Serial  $60 Joypad
  $0100-$0103   Entry point (usually: NOP; JP $0150)
  $0104-$0133   The Nintendo logo bitmap (see below)
  $0134-$0143   Game title, in ASCII
  $0144-$014F   Cartridge type, ROM size, RAM size, region,
                  version, header checksum, global checksum
  $0150-        The game actually starts here
```

**The Nintendo logo is the best story in Game Boy hardware.** The boot ROM scrolls that logo down the screen — and then *compares the cartridge's copy of it against a copy stored in the boot ROM.* If they don't match byte for byte, the console locks up and won't boot.

Why? **Trademark law as copy protection.** To make a working Game Boy cartridge you had to reproduce Nintendo's logo, which meant reproducing their trademark, which meant Nintendo could sue unlicensed publishers. It's not a technical protection at all — it's a *legal* protection enforced by silicon. Ingenious, and it mostly worked.

> ⚙ **Emulator implication:** if you implement the boot ROM, you must implement the logo check, or you must load a real boot ROM dump. Most emulators skip the boot ROM entirely and instead initialize the registers to their documented post-boot values (`AF=$01B0, BC=$0013, DE=$00D8, HL=$014D, SP=$FFFE, PC=$0100`). That's a perfectly good shortcut, and it's what you should do first.

## 5.3 `$4000`–`$7FFF` — Switchable ROM Bank

**What:** A 16 KB window onto *any other* bank of the cartridge ROM.

**Why:** Because 32 KB of address space isn't enough for a real game, and you can't add address pins to a chip that already exists. This is all of Part 6.

## 5.4 `$8000`–`$9FFF` — VRAM

**What:** 8 KB of RAM that the PPU reads to build the picture.

**Why separate from WRAM?** Because the PPU needs to read it *constantly* — several bytes per scanline — and if it shared a bus with the CPU's general memory, they'd collide continuously. Giving graphics its own memory with its own port lets the PPU work without permanently starving the CPU.

**The layout (details in Part 7):**

```
  $9FFF ┌──────────────────────────┐
        │  Tile Map 1              │  32×32 bytes = 1024
  $9C00 ├──────────────────────────┤
        │  Tile Map 0              │  32×32 bytes = 1024
  $9800 ├──────────────────────────┤
        │  Tile data block 2       │  128 tiles
  $9000 ├──────────────────────────┤
        │  Tile data block 1       │  128 tiles
  $8800 ├──────────────────────────┤
        │  Tile data block 0       │  128 tiles
  $8000 └──────────────────────────┘
```

**The catch:** during PPU **Mode 3** (actively drawing a line), VRAM is locked. CPU reads return `$FF`; writes are dropped. This is the bus-sharing principle from §2.2 in its most consequential form, and it's *the* reason games do graphics updates during VBlank.

> ⚙ **Emulator implication:** implementing VRAM locking is optional at first — most games never read VRAM at a bad time, because they were written correctly. But a few games rely on the `$FF` return, and the dmg-acid2 test ROM checks related behavior. Add it once the basics work.

## 5.5 `$A000`–`$BFFF` — External RAM

**What:** RAM *on the cartridge*, if the cartridge has any.

**Why on the cartridge?** Because most games don't need it, and putting it on the console would mean every buyer pays for it. Putting it on the cartridge means only games that need it pay for it. **This is the single most important economic principle in cartridge-based hardware: push optional cost onto the cartridge.** The same logic explains mapper chips, save batteries, real-time clocks, and even extra sound hardware on some systems.

**Why it's usually disabled by default:** on real hardware, when you power the console down, the voltage doesn't drop instantly — for a few milliseconds, signals are unstable and the CPU can emit garbage writes. If save RAM were always writable, that garbage could corrupt your save. So MBCs require you to *deliberately enable* RAM by writing a magic value (`$0A`) to a control region, and games disable it again immediately after saving. It's a hardware-level safety catch.

## 5.6 `$C000`–`$DFFF` — WRAM

**What:** 8 KB of general-purpose work RAM.

**Why so little?** RAM was the most expensive component per byte. 8 KB was what the price point allowed.

**What it holds:** every variable the game has. Player position, enemy states, inventory, the shadow OAM buffer, decompression scratch space, and **the stack** (typically starting at `$FFFE` and growing down through HRAM into... well, into whatever's there, which on a real Game Boy is HRAM then the I/O area — so in practice games keep the stack shallow).

To calibrate how tight this is: Pokémon Red tracks 151 species, a 6-Pokémon party with individual stats, a 20-slot inventory, a 12-box PC storage system, map state, NPC state, and battle state — **in 8 KB.** That's less memory than a single modern JPEG thumbnail.

## 5.7 `$E000`–`$FDFF` — Echo RAM

**What:** A mirror. Reading `$E000` gives you the same byte as `$C000`. Writing `$E123` writes to `$C123`.

**Why does this exist?** *It wasn't designed.* It's a side effect of cheap address decoding.

Here's the actual mechanism. To decode "is this address in WRAM?", proper logic would check the full 16-bit address range `$C000`–`$DFFF`. But that costs gates. Instead, the hardware essentially checks a few high bits and then **ignores address bit 13** when indexing the RAM chip. The result: two different addresses land on the same physical byte.

```
   $C123  =  1100 0001 0010 0011
   $E123  =  1110 0001 0010 0011
                ▲
                └── this bit is ignored by the WRAM chip
```

Nintendo's documentation told developers not to use this area. A handful of games used it anyway (sometimes by accident, via a pointer bug that happened to work).

**This is your first encounter with a very Game Boy idea:** a behavior that exists because of what was *cheapest to build*, not what was *intended*. Pan Docs is full of these. When something seems arbitrary, ask "what would the lazy circuit do?" — it's usually the answer.

> ⚙ **Emulator implication:** two lines of code. Map `$E000`–`$FDFF` to `WRAM[addr - 0xE000]`. Don't skip it; a few games really do read from there.

## 5.8 `$FE00`–`$FE9F` — OAM

**What:** Object Attribute Memory. 160 bytes describing 40 sprites, 4 bytes each.

```
   One OAM entry (4 bytes):
   ┌──────────┬──────────┬──────────┬─────────────────┐
   │  Byte 0  │  Byte 1  │  Byte 2  │     Byte 3      │
   │  Y pos   │  X pos   │  Tile #  │     Flags       │
   └──────────┴──────────┴──────────┴─────────────────┘
                                       │
              ┌────────────────────────┘
              ▼
    bit 7  priority  (0 = above BG, 1 = behind BG colors 1-3)
    bit 6  Y flip
    bit 5  X flip
    bit 4  palette   (OBP0 or OBP1)
    bits 3-0  unused on DMG (used on Game Boy Color)
```

**Why is it separate from VRAM?** Because the PPU accesses it on a *different schedule*. At the start of every scanline, the PPU scans all 40 entries to find which sprites appear on this line (**Mode 2**, OAM scan). During drawing (Mode 3) it fetches tile data from VRAM. Separate memories with separate access windows means the PPU can be doing one while preparing the other.

**Why 40 sprites, and only 10 per scanline?** Silicon. The PPU has physical registers for exactly 10 sprites' worth of line data. It can hold 40 *definitions*, but it can only *composite* 10 onto any one line. Exceed that and the extras simply vanish — the famous Game Boy sprite flicker, which games worked around by rotating sprite priority every frame so different sprites disappeared each time. Your eye averages it into "flickering" rather than "missing."

**The Y coordinate is offset by 16 and X by 8.** A sprite at Y=16, X=8 appears at the top-left corner of the screen. Why? So sprites can slide *off* the edges. A sprite at Y=0 is entirely above the screen; Y=8 is half-visible. Without the offset you couldn't have sprites partially entering from the top or left, because coordinates are unsigned. **This is a recurring hardware-design pattern: use a biased coordinate system to get negative positions out of unsigned numbers.**

## 5.9 `$FEA0`–`$FEFF` — The prohibited area

**What:** 96 bytes that aren't really anything.

Behavior depends on the model and the PPU's current mode. On the DMG it usually returns `$00`. On some hardware, reading here during OAM scan triggers the "OAM corruption bug," which actually scrambles OAM contents.

**Why it exists:** the address decoder carves the map into power-of-two chunks. OAM needs 160 bytes; the chunk allocated is 256. The leftover 96 bytes are unassigned, and unassigned means "whatever the floating bus happens to do."

> ⚙ **Emulator implication:** return `$00`, ignore writes, move on. Note it exists so that when Pan Docs mentions the OAM corruption bug, you're not baffled.

## 5.10 `$FF00`–`$FF7F` — I/O Registers

**What:** 128 control-panel bytes. Not memory. Wires.

```
  $FF00  P1/JOYP     Joypad
  $FF01  SB          Serial transfer data
  $FF02  SC          Serial transfer control
  ──────────────────────────────────────────────
  $FF04  DIV         Divider register (free-running counter)
  $FF05  TIMA        Timer counter
  $FF06  TMA         Timer modulo (reload value)
  $FF07  TAC         Timer control
  ──────────────────────────────────────────────
  $FF0F  IF          Interrupt Flag (which are pending)
  ──────────────────────────────────────────────
  $FF10-$FF26        Sound channels 1-4 + master control
  $FF30-$FF3F        Wave pattern RAM (channel 3 waveform)
  ──────────────────────────────────────────────
  $FF40  LCDC        LCD Control          ◄── the big one
  $FF41  STAT        LCD Status
  $FF42  SCY         Background scroll Y
  $FF43  SCX         Background scroll X
  $FF44  LY          Current scanline     ◄── read-only
  $FF45  LYC         Scanline compare
  $FF46  DMA         Start OAM DMA        ◄── writing = action
  $FF47  BGP         Background palette
  $FF48  OBP0        Sprite palette 0
  $FF49  OBP1        Sprite palette 1
  $FF4A  WY          Window Y position
  $FF4B  WX          Window X position
  ──────────────────────────────────────────────
  $FF50  Boot ROM disable  ◄── write once, boot ROM vanishes forever
```

**The `$FF50` register deserves a moment.** The last thing the boot ROM does is write to `$FF50`. That write **unmaps the boot ROM from `$0000`–`$00FF`**, revealing the cartridge's interrupt vectors underneath. The boot ROM erases itself from the address space and then execution falls into the game at `$0100`. It's a one-way door: you cannot map it back without a power cycle.

Beautiful design. The boot ROM occupies address space only while it needs it, then gives it back.

**Why is the I/O area at the very top of memory?** Two reasons, both about efficiency:

1. `LDH A, (n)` is a one-byte-shorter, one-cycle-faster instruction that reads from `$FF00 + n`. Putting the registers you touch most often in the highest page makes accessing them cheap. Games hammer these registers every frame, so those saved bytes and cycles are significant.
2. It's the natural leftover space after ROM, VRAM, and RAM have claimed the big contiguous blocks.

## 5.11 `$FF80`–`$FFFE` — HRAM

**What:** 127 bytes of RAM inside the CPU chip.

**Why:** Three reasons, all good:

1. **It survives DMA** (§2.10). This alone justifies it.
2. **It's fast to address.** `LDH` reaches it in fewer bytes and fewer cycles than normal RAM.
3. **Some games put the stack here.** `SP = $FFFE` is the conventional initialization, meaning the first pushes land in HRAM.

127 bytes sounds like nothing. It's enough for the DMA routine, a few hot variables, and a shallow stack. On this machine that's a meaningful amount of real estate.

## 5.12 `$FFFF` — IE

One byte, all by itself at the very top. The **Interrupt Enable** register: five bits, one per interrupt source.

**Why is it isolated up here instead of next to `IF` at `$FF0F`?** Because `$FF0F` is in the I/O block and `$FFFF` is... the last address. Honestly, this is one where the answer is mostly "the address decoder had a spare slot." Its neighbor `IF` sits down in I/O space. You'll just have to remember they're separated. Everyone finds this annoying.

## 5.13 The mental model to keep

Don't memorize the map as a list. Memorize it as **a story about ownership**:

```
   Lower half   ($0000-$7FFF)  ── belongs to the CARTRIDGE
   Next eighth  ($8000-$9FFF)  ── belongs to the PPU
   Next eighth  ($A000-$BFFF)  ── belongs to the CARTRIDGE again
   Next quarter ($C000-$DFFF)  ── belongs to the CPU (WRAM)
   Echo         ($E000-$FDFF)  ── accident
   Top page     ($FE00-$FFFF)  ── belongs to the HARDWARE
                                  (sprites, controls, fast RAM)
```

Six regions. Four owners. That's the map.

---
---

# Part 6: Cartridges and Banking

## 6.1 The problem, stated plainly

The CPU has 16 address pins. 2¹⁶ = 65,536. And of those 65,536 addresses, only **32,768** are allocated to cartridge ROM (`$0000`–`$7FFF`).

Pokémon Red is **1 MB**. That's 1,048,576 bytes. It is *32 times larger* than the address space available to it.

How does a CPU that can only say numbers up to 65,535 read a program with a million bytes?

## 6.2 Intuition: the window and the warehouse

Imagine a warehouse with a million boxes. You're in an office next door. Between the office and the warehouse there's a small window through which exactly 16,384 boxes are visible at a time.

You can't see the whole warehouse. But there's a lever next to the window. Pull it, and the warehouse floor **slides** so that a *different* 16,384 boxes are in view. The window never moves. The contents behind it change.

```
   THE WAREHOUSE (cartridge ROM, 1 MB = 64 banks of 16 KB)

   ┌──────┬──────┬──────┬──────┬──────┬─── ... ───┬──────┐
   │ Bank │ Bank │ Bank │ Bank │ Bank │           │ Bank │
   │  00  │  01  │  02  │  03  │  04  │           │  63  │
   └──────┴──────┴──────┴──────┴──────┴─── ... ───┴──────┘
      │        ▲
      │        │  ┌──────────────────────────┐
      │        └──│  the lever selects which  │
      │           │  bank appears in the      │
      │           │  window                   │
      │           └──────────────────────────┘
      │                    │
      ▼                    ▼
   ┌──────────────────────────────────┐
   │  CPU's view of $0000-$7FFF        │
   │                                   │
   │  $0000-$3FFF: always Bank 00      │  ◄── the fixed pane
   │  $4000-$7FFF: whichever bank      │  ◄── the sliding pane
   │                the lever selects   │
   └──────────────────────────────────┘
```

The thing that implements the lever is a chip inside the cartridge called an **MBC** — **Memory Bank Controller**.

## 6.3 How do you pull the lever?

Here's the part that surprises everyone. **You write to ROM.**

```
   LD A, $05
   LD ($2000), A     ; "switch to ROM bank 5"
```

But `$2000` is in ROM. ROM is read-only. What?

The resolution: **the MBC chip sits between the CPU and the ROM chip, and it watches the bus.** When it sees a *write* to an address in the `$0000`–`$7FFF` range, it knows that can't be a real memory write (ROM can't be written), so it interprets it as a **command to itself.**

```
                      write to $2000, value 5
                                │
                                ▼
   ┌─────┐    ┌──────────────────────────────┐    ┌──────────┐
   │ CPU │───►│         MBC CHIP             │───►│ ROM CHIP │
   └─────┘    │                              │    │          │
              │  "Is this a write to         │    │ 1 MB of  │
              │   $0000-$7FFF? Then it's     │    │ program  │
              │   a command for me, not      │    │ data     │
              │   data for the ROM."         │    │          │
              │                              │    └──────────┘
              │  bank_register = 5           │
              │                              │
              │  On subsequent READS from    │
              │  $4000-$7FFF, I output       │
              │  extra address bits so the   │
              │  ROM chip sees address       │
              │  (5 * 16384) + offset        │
              └──────────────────────────────┘
```

**The MBC is an address translator.** It takes the CPU's 16-bit address and, using its internal bank register, produces a wider address (up to 23 bits) for the actual ROM chip.

```
   CPU says:      $4A37   (16 bits)
   Bank register: 5

   MBC computes:  (5 - 1) * $4000 + $4A37   ... conceptually:
                  physical = (bank * $4000) + (addr - $4000)
                  physical = (5 * 16384) + 2615
                  physical = $016A37   (23 bits)
                             ▲
                             └── the ROM chip's actual address
```

> 🤔 **Why not the obvious way?**
> Why not just make a CPU with more address pins? Because the CPU was designed and fabricated before anyone knew games would get this big — and once millions of consoles are in people's hands, you cannot change the CPU. But you *can* change the cartridge, because a new cartridge ships with every new game. **The Game Boy's expandability lives in the cartridge, not the console.** This is why later Game Boy games have features the original hardware couldn't have supported: real-time clocks, rumble motors, accelerometers, even extra RAM. All of it rides in on the cart.

## 6.4 Why is bank 0 fixed?

Now you can answer this yourself, but let's make it explicit.

Suppose *all* of `$0000`–`$7FFF` were switchable. Your code, executing at address `$1234` in bank 3, writes to the bank register to switch to bank 7. On the very next cycle, the CPU fetches the instruction at `$1235`... which is now a completely different byte, from bank 7. Your program has teleported into unrelated data mid-function.

Fixing bank 0 gives you a **safe island**. The bank-switching routine lives in bank 0, so it's always present regardless of which bank is selected. Code in bank 0 can safely switch banks and then jump into the newly-mapped bank.

This creates the standard architecture of a Game Boy game:

```
   BANK 00 (always present)              BANKED (00-NN)
   ─────────────────────────             ──────────────
   • interrupt vectors                   • level data
   • header                              • graphics tiles
   • bank switching helpers              • music
   • the main loop                       • text/dialogue
   • frequently-used subroutines         • per-area code
   • "far call" trampoline
```

That last item is worth explaining. A **far call** (also called a "banked call" or "trampoline") is a routine in bank 0 that: saves the current bank, switches to a target bank, calls a function there, switches back, and returns. It's the Game Boy equivalent of a dynamic library call, hand-written in assembly, and it exists in every large game.

## 6.5 The MBC family

| MBC | Max ROM | Max RAM | Extra features | Used by |
|---|---|---|---|---|
| **None** | 32 KB | 8 KB | — | Tetris, Dr. Mario |
| **MBC1** | 2 MB | 32 KB | — | Super Mario Land, most early games |
| **MBC2** | 256 KB | 512×4 bits | built-in RAM | Kirby's Dream Land |
| **MBC3** | 2 MB | 32 KB | **real-time clock** | Pokémon Gold/Silver |
| **MBC5** | 8 MB | 128 KB | rumble, GBC-safe | Pokémon Crystal, later GBC games |

**MBC3's real-time clock** is a lovely example of pushing features into the cartridge. Pokémon Gold has day/night cycles and berries that grow overnight. That required knowing real-world time — which the Game Boy has no concept of. So Nintendo put a quartz clock chip and a battery *in the cartridge*, and the game reads the time through the MBC's register interface. The console learned to tell time by buying a watch.

**MBC5 exists because MBC1/3 had a subtle bug** with rapid bank switching that mattered when the Game Boy Color ran at double speed. It also cleanly supports 8 MB and adds a rumble motor line. If you're picking one mapper to implement really well, MBC5 is the simplest and most regular.

## 6.6 MBC1 in detail (because it's the weird one)

MBC1 is the most common early mapper and it has a genuinely confusing design. Understanding it now means Pan Docs' MBC1 page won't ambush you.

**The register regions** (all triggered by *writes*):

```
  $0000-$1FFF   RAM Enable      write $0A to enable, anything else to disable
  $2000-$3FFF   ROM Bank Number (lower 5 bits)
  $4000-$5FFF   RAM Bank Number  OR  upper 2 bits of ROM bank
  $6000-$7FFF   Banking Mode Select (0 or 1)
```

Three quirks, and each one has a *reason*:

**Quirk 1: bank 0 becomes bank 1.** Writing `0` to `$2000` selects bank *1*, not bank 0. Why? Because bank 0 is already permanently at `$0000`–`$3FFF`, so selecting it in the switchable window would be pointless duplication. The hardware just forces the low bits to be non-zero. Consequence: banks `$00`, `$20`, `$40`, `$60` are unreachable in the switchable window.

**Quirk 2: the 2-bit register does double duty.** The register at `$4000`–`$5FFF` is only 2 bits. In small cartridges it means "RAM bank." In large cartridges it means "the top 2 bits of the ROM bank number." Why overload it? **Pin count and gate count.** Adding a dedicated register costs silicon; reusing one is free. The mode register at `$6000` decides which meaning applies.

**Quirk 3: mode 1 changes what's at `$0000`.** In banking mode 1, the upper-bit register also affects the *fixed* region `$0000`–`$3FFF`, letting it show bank `$20`, `$40`, or `$60`. This is how a few 1 MB games access all their data. Almost nothing uses it, but the MBC1 test ROMs check it.

Look at how much complexity comes from "we wanted to save two register bits." That is the Game Boy in a sentence.

## 6.7 The cartridge header tells you everything

At `$0147`, one byte says what kind of cartridge this is:

```
  $00  ROM ONLY               $13  MBC3+RAM+BATTERY
  $01  MBC1                   $19  MBC5
  $02  MBC1+RAM               $1A  MBC5+RAM
  $03  MBC1+RAM+BATTERY       $1B  MBC5+RAM+BATTERY
  $0F  MBC3+TIMER+BATTERY     $1C  MBC5+RUMBLE
  ...
```

At `$0148`: ROM size (a code, where the number of banks is `2 << value`).
At `$0149`: RAM size.

> ⚙ **Emulator implication:** your ROM loader reads these three bytes and instantiates the right mapper object. A clean design has a `Cartridge` interface with `read(addr)` and `write(addr, value)`, and one implementation per MBC. Then your bus code doesn't care which mapper is present — it just calls the interface. **Start with "no MBC" (Tetris works) and add MBC1, then MBC3, then MBC5.**

## 6.8 Bank switching in your emulator

The naïve version is genuinely trivial:

```
  read(addr):
      if addr < 0x4000:  return rom[addr]
      if addr < 0x8000:  return rom[(current_bank * 0x4000) + (addr - 0x4000)]

  write(addr, val):
      if addr in 0x2000..0x3FFF:
          current_bank = val & 0x1F
          if current_bank == 0: current_bank = 1
```

That's the whole idea. The complexity in real mappers is in the edge cases — bank masking for undersized ROMs, mode registers, RAM enable — not in the concept. **Don't let the concept intimidate you; it's a multiply and an add.**

---
---

# Part 7: The Graphics System

This is the longest part, and it should be. Graphics is where the Game Boy is *least* like CHIP-8 and where most beginners get lost. We're going to build up slowly.

## 7.1 First, unlearn the framebuffer

In CHIP-8, you had this:

```
   display[64][32]     // a 2D array of pixels
   DXYN → XOR bytes into display
   render → draw the whole array
```

The screen was a **framebuffer**: an array where each element *is* a pixel. Simple, direct, and — here's the problem — **expensive.**

Let's do the arithmetic for the Game Boy.

```
   Screen: 160 × 144 pixels = 23,040 pixels
   Colors: 4 shades = 2 bits per pixel
   Framebuffer size: 23,040 × 2 bits = 46,080 bits = 5,760 bytes
```

5,760 bytes. The Game Boy has 8,192 bytes of VRAM. So a single framebuffer would consume **70% of all video memory**, leaving nothing for a second buffer, no room for sprite graphics, no room for anything.

And that's just storage. Now consider *bandwidth*: to change the screen, the CPU has to write 5,760 bytes. At roughly 8 cycles per byte write, that's ~46,000 cycles — but a whole frame is only 70,224 cycles. **Redrawing the screen once would consume two-thirds of the CPU's entire time budget.** There would be no cycles left for the game.

Framebuffers are simply not affordable on this hardware. So the Game Boy uses a completely different approach.

## 7.2 The big idea: tiles

### Intuition: the mosaic and the rubber stamps

You want to decorate a wall. Two approaches:

**Approach A (framebuffer):** paint every square inch individually. Total freedom. Enormous effort.

**Approach B (tiles):** make a set of 8×8 ceramic tiles — a brick tile, a grass tile, a cloud tile, a letter-A tile. Then describe the wall as a *grid of tile numbers*. "Row 1: tiles 3, 3, 3, 7, 7, 3..." To build the wall, you look up each number and stamp the corresponding tile.

Approach B is *dramatically* cheaper, and here's why it works for games: **game screens are enormously repetitive.** A brick wall is the same brick 200 times. A grass field is the same grass tile everywhere. Text is 26 letters reused constantly. Why store 200 copies of a brick when you can store one brick and 200 pointers to it?

```
   FRAMEBUFFER approach:
   ┌─────────────────────────────────────┐
   │ every one of 23,040 pixels stored   │  5,760 bytes
   │ individually                        │
   └─────────────────────────────────────┘

   TILE approach:
   ┌──────────────────┐   ┌─────────────────────────┐
   │ TILE DATA        │   │ TILE MAP                │
   │ 8x8 patterns     │ + │ which tile goes where   │
   │ 16 bytes each    │   │ 1 byte per grid cell    │
   │ up to 384 tiles  │   │ 32x32 grid = 1024 bytes │
   └──────────────────┘   └─────────────────────────┘
        6,144 bytes             1,024 bytes
        (but shared!)
```

The screen is 160×144 = **20 tiles wide by 18 tiles tall** = 360 visible cells. Each cell is one byte. **360 bytes describes an entire screen.** Compare that to 5,760. A 16× reduction — and crucially, changing what's on screen means writing 360 bytes instead of 5,760.

**This is the fundamental trade of 2D console graphics, and every console from the NES to the SNES to the Genesis makes it.** You give up per-pixel freedom and you get a machine that can actually afford to draw.

### What would happen if it didn't exist?

The Game Boy would need ~4× the VRAM (cost), the CPU would spend most of its time copying pixels (no gameplay), and battery life would suffer. It would have been a worse product at a higher price. Tiles aren't a limitation — they're what made the machine possible.

## 7.3 What a tile actually looks like in memory

A tile is 8×8 pixels, 2 bits each = 128 bits = **16 bytes**.

But the bit layout is not what you'd guess. The Game Boy uses **planar** (bitplane) format: the two bits of each pixel are stored in two *separate* bytes.

```
   One row of 8 pixels needs 2 bytes:

   Byte 0 (low bits):   0 1 1 1 1 1 1 0        = $7E
   Byte 1 (high bits):  0 0 0 0 0 0 0 0        = $00
                        │ │ │ │ │ │ │ │
                        ▼ ▼ ▼ ▼ ▼ ▼ ▼ ▼
   Pixel color IDs:     0 1 1 1 1 1 1 0

   Combine: pixel N's color = (bit N of byte1)<<1 | (bit N of byte0)
```

A full tile — the classic smiley from Pan Docs, roughly:

```
   Bytes                Binary (lo/hi)              Rendered
   ─────                ──────────────              ────────
   $3C $7E    00111100 / 01111110                   .11223 3 2 1
   $42 $42    01000010 / 01000010                   .3....  ..3.
   $42 $42    01000010 / 01000010                   .3......3.
   $42 $42    01000010 / 01000010                   .3......3.
   $7E $5E    01111110 / 01011110                   .3333333.
   $7E $0A    01111110 / 00001010                   .1111111.
   $7C $56    01111100 / 01010110                   .222222..
   $38 $7C    00111000 / 01111100                   ..3333...
```

(Don't worry about reproducing that exactly — the point is the *shape* of the encoding.)

> 🤔 **Why planar? Why not just pack 4 pixels per byte?**
> Because of how the PPU works internally. It processes a whole row of 8 pixels at once by loading both bytes into **shift registers** and shifting one bit out of each per clock. Two shift registers, one bit each, combine into a 2-bit color — one pixel per clock tick, with no arithmetic, no masking, no shifting by variable amounts. It's the layout that makes the *hardware* simplest, at the cost of making the *software* slightly weirder. Given the choice, 1980s hardware always optimizes for the silicon.

> ⚙ **Emulator implication:** your tile decoder is a small function. Given a tile index and a row, fetch two bytes, and for each of 8 pixels extract the corresponding bit from each and combine. Write it once, test it against a known tile, and never think about it again.

## 7.4 Where tiles live: the addressing modes

VRAM's tile data area is `$8000`–`$97FF` — 6,144 bytes = **384 tiles**. It's divided into three blocks of 128:

```
   $97FF ┌──────────────────────────┐
         │  Block 2                 │  ── reachable ONLY in $8800 mode,
   $9000 │  ($9000-$97FF)           │     as signed indices 0..127
         ├──────────────────────────┤
         │  Block 1                 │  ── reachable in BOTH modes:
   $8800 │  ($8800-$8FFF)           │     unsigned 128-255, or signed -128..-1
         ├──────────────────────────┤
         │  Block 0                 │  ── reachable ONLY in $8000 mode,
   $8000 │  ($8000-$87FF)           │     as unsigned 0..127
         └──────────────────────────┘     (sprites ALWAYS use this mode)
```

Each block holds 128 tiles (128 × 16 bytes = 2,048 bytes).

Now the part that confuses everyone. **LCDC bit 4 selects between two addressing modes for background and window tiles:**

| LCDC bit 4 | Base address | Index type | Range covered |
|---|---|---|---|
| **1** | `$8000` | **unsigned** 0–255 | `$8000`–`$8FFF` (blocks 0 and 1) |
| **0** | `$9000` | **signed** −128 to +127 | `$8800`–`$97FF` (blocks 1 and 2) |

In signed mode, tile index `$00` means `$9000`, and tile index `$FF` (= −1) means `$9000 − 16 = $8FF0`.

**Sprites always use the `$8000` unsigned mode**, regardless of LCDC bit 4.

> 🤔 **Why on earth would you do this?**
> Because of block 1, the shared middle. Look at the diagram again: block 1 is reachable by *both* modes. That means you can put your most-used tiles there and access them from either addressing mode.
>
> More importantly: it lets the 384 tiles be *partitioned*. Sprites use the low blocks; backgrounds use the high blocks; the middle is shared. Without two modes, background tiles and sprite tiles would compete for the same 256 indices. With two modes, you effectively get 256 tiles for backgrounds *and* 256 for sprites out of a pool of 384. It's an overlapping-window trick to make a small memory feel bigger.
>
> The signed indexing specifically is a hardware convenience: the PPU computes `$9000 + (signed_index × 16)`, and sign extension is nearly free in hardware.

> ⚙ **Emulator implication:** this one line of code is the source of a classic bug where "the game runs but the graphics are garbage." If your background looks like scrambled noise, check your LCDC bit 4 handling first.

## 7.5 Tile maps: the blueprint

A **tile map** is a 32×32 grid of bytes. Each byte is a tile index.

```
   $9FFF ┌──────────────────────────┐
         │  Tile Map 1  (32 × 32)   │  1024 bytes
   $9C00 ├──────────────────────────┤
         │  Tile Map 0  (32 × 32)   │  1024 bytes
   $9800 └──────────────────────────┘
```

Two maps. LCDC bit 3 selects which one the background uses; LCDC bit 6 selects which one the window uses.

**Why 32×32 when the screen is only 20×18?**

Because the map is the *world* and the screen is a *window onto it*.

```
                THE FULL TILE MAP: 32 x 32 tiles = 256 x 256 pixels

         ┌────────────────────────────────────────────────┐
         │                                                │
         │       ┌───────────────────┐                    │
         │       │                   │                    │
         │       │   VISIBLE SCREEN  │  ◄── 20 x 18 tiles │
         │       │   160 x 144 px    │      (160x144 px)  │
         │       │                   │                    │
         │       └───────────────────┘                    │
         │        ▲                                       │
         │        └── position controlled by SCX, SCY     │
         │                                                │
         └────────────────────────────────────────────────┘
                        256 x 256 pixels
```

The extra map area is *off-screen space you can prepare in advance*. This is the foundation of scrolling.

**Why two maps?** So you can build a new screen in the hidden one while the visible one is displayed, then flip. It's double-buffering at the map level — cheap, because flipping is one bit in LCDC.

## 7.6 Scrolling

### The intuition

Scrolling is not "move all the pixels." Scrolling is "**move the window**."

`SCY` (`$FF42`) and `SCX` (`$FF43`) are the coordinates of the screen's top-left corner within the 256×256 map. Increase `SCX` by 1 and everything appears to shift left by one pixel — but nothing in VRAM changed. **One byte write scrolls the entire screen.**

Contrast with CHIP-8, where "scrolling" would mean rewriting your whole display array. Here it's free.

### The wraparound is the magic part

The map **wraps**. If `SCX = 250`, the right side of the screen shows columns 250–255 and then wraps around to columns 0–13.

```
   SCX = 250:

   Map columns:  ... 248 249 250 251 252 253 254 255 │ 0  1  2  3 ...
                                 └──────────────────────────────┘
                                    what's on screen (wraps!)
```

This makes **infinite scrolling** possible with a finite map. As you scroll right, tiles scroll off the left edge — and those map cells are now off-screen and free. The game writes *new* tiles into that off-screen column just before it wraps into view. You're always drawing one column ahead of the player.

```
   Frame 1:            Frame 30:           Frame 60:
   ┌────────┐          ┌────────┐          ┌────────┐
   │▓▓▓▓░░░░│          │▓▓▓░░░░▒│          │▓▓░░░░▒▒│
   │ visible│          │ visible│          │ visible│
   └────────┘          └────────┘          └────────┘
        ▲                    ▲                   ▲
        │                    │                   │
   game writes new      still writing      the new tiles
   tiles just off       ahead of the       have scrolled
   the right edge       viewport            into view
```

That's Super Mario Land. That's every side-scroller on the system. **The scroll registers do the movement; the CPU only has to fill in the newly-exposed edge.**

### Mid-frame scroll changes: the raster trick

Here's where things get delicious, and where "cycle accuracy" starts to matter.

The PPU reads `SCX` and `SCY` **while drawing each scanline**. So if you change `SCX` *between* scanlines, different lines scroll by different amounts.

```
   Normal (SCX constant):        Changing SCX per line:

   ████████████████              ████████████████
   ████████████████                ████████████████
   ████████████████              ████████████████
   ████████████████                  ████████████████
   ████████████████                ████████████████
   ████████████████              ████████████████

   (flat)                        (wavy! water effect)
```

Games do this constantly:
- **Wavy water / heat shimmer:** modulate `SCX` with a sine table per scanline.
- **Split screens:** set `SCX = 0` for the top of the screen (a status bar) and `SCX = camera_x` for the rest.
- **Parallax:** background layers scrolling at different speeds by changing `SCX` at layer boundaries.

How does the game know when a scanline ends? **The STAT interrupt** (Part 9). The PPU can fire an interrupt at the start of HBlank or when `LY` hits a specific value. The game's handler writes the new `SCX` in the few dozen cycles before drawing resumes.

> **This is the key insight for why Game Boy emulation is hard.** If your emulator renders the whole frame at once at the end (a "scanline-inaccurate" or "frame-based" renderer), all of these effects break, because you only ever see the *final* value of `SCX`. **You must render scanline by scanline, reading the registers as they are at that moment.** That single requirement is what pushes you from a frame-based to a scanline-based architecture, and it's the first real step up in emulator complexity.

## 7.7 The window

### What problem does this solve?

Games need a **status bar** — score, health, lives — that stays fixed while the world scrolls behind it.

You could do this with sprites, but you only get 40, and 10 per line. A status bar would consume them all.

So the Game Boy adds a **second background layer**, called the **window**, that does *not* scroll. It's drawn on top of the background, and once it starts on a scanline, it takes over completely for the rest of that line.

```
   ┌──────────────────────────────┐
   │                              │
   │      background (scrolls)    │
   │      ░░░░░░░░░░░░░░░░░░      │
   │      ░░░░░░░░░░░░░░░░░░      │
   │                              │
   ├──────────────────────────────┤  ◄── WY = this line
   │  SCORE 12300   LIVES 3       │  ◄── the window (fixed)
   └──────────────────────────────┘
```

### How it works

- `WY` (`$FF4A`): the screen Y coordinate where the window begins.
- `WX` (`$FF4B`): the screen X coordinate **plus 7**.

**`WX = 7` means the window starts at screen X = 0.**

> 🤔 **Why the +7 offset?** Because of the pixel FIFO's internal pipeline — the window fetcher needs a few pixels of lead time, and the offset accounts for it. `WX` values 0–6 produce hardware-specific edge-case behavior that a couple of games actually rely on. It's a leaky abstraction: an internal implementation detail poking out into the programmer-visible interface. Pan Docs will mention this without explaining why; now you know.

**The window's internal line counter is its own thing.** The window doesn't use `LY` to decide which of its rows to draw. It has a private counter that only increments on lines where the window was actually visible. This means if you disable the window mid-frame and re-enable it, the window "remembers" where it was. This trips up a lot of emulator authors and is a specific check in the dmg-acid2 test ROM.

**The window can also be moved mid-frame** — the same STAT-interrupt trick as scrolling. Some games slide a text box up from the bottom by decreasing `WY` a little each frame; some create diagonal wipes by changing `WX` per scanline.

## 7.8 Sprites (Objects)

### The problem

The background is a grid. Tiles snap to 8×8 boundaries. But Mario needs to be at pixel X=53, not tile X=6. **Moving objects need pixel-level positioning, independent of the grid.**

### The solution

A separate system: **objects** (Nintendo's term; everyone says "sprites"). 40 of them, each described by 4 bytes in OAM (§5.8), each positioned at arbitrary pixel coordinates and drawn *over* the background.

```
              BACKGROUND (tile grid)          SPRITES (free-floating)
        ┌───┬───┬───┬───┬───┬───┐
        │▓▓▓│▓▓▓│▓▓▓│▓▓▓│▓▓▓│▓▓▓│               ╔═══╗
        ├───┼───┼───┼───┼───┼───┤               ║ ☺ ║  ◄── at any pixel
        │▓▓▓│▓▓▓│▓▓▓│▓▓▓│▓▓▓│▓▓▓│               ╚═══╝      X, Y
        ├───┼───┼───┼───┼───┼───┤
        │███│███│███│███│███│███│    ═══►    composited together
        └───┴───┴───┴───┴───┴───┘             on the fly, per pixel
         locked to an 8x8 grid                by the PPU
```

### The rules and their reasons

**8×8 or 8×16.** LCDC bit 2 selects the size, globally, for all sprites. In 8×16 mode, the tile index's low bit is ignored — sprites always use an even/odd tile pair. Why? Because a 16-pixel-tall sprite is just two stacked 8×8 tiles, and forcing them adjacent means the PPU can compute the second tile's address by adding 1 rather than needing a second index. Cheaper silicon.

**Why 8×16 exists at all:** most game characters are taller than they are wide. One 8×16 sprite instead of two 8×8 sprites halves your OAM consumption for human-shaped things.

**Color 0 is transparent.** For sprites only — not for backgrounds. This is why you can have non-rectangular characters. Background color 0 is a real color (usually white); sprite color 0 means "show what's behind."

**Consequence:** sprites only get **3 usable colors**, not 4. That's why Game Boy characters often look flatter than backgrounds.

**Two sprite palettes.** `OBP0` and `OBP1` at `$FF48`/`$FF49`. Each OAM entry picks one via a flag bit. Why two? So you can have, say, a light-colored enemy and a dark-colored enemy using the same tile data with different shading. Palette swapping is the cheapest form of graphical variety ever invented, and every 8-bit game abuses it.

**X and Y flip.** Bits in the OAM flags. Draw a character facing left, flip it for facing right. **You just halved your tile memory for every character.** In a machine this constrained, that's enormous.

**The 10-sprites-per-line limit.** The PPU has hardware registers for exactly 10 sprites per scanline. It scans OAM in index order (0 to 39) during Mode 2 and keeps the first 10 that intersect the current line. **The rest are simply not drawn.**

```
   Scanline with 14 sprites intersecting it:

   OAM index:  0  1  2  3  4  5  6  7  8  9 │ 10 11 12 13
               ✓  ✓  ✓  ✓  ✓  ✓  ✓  ✓  ✓  ✓ │  ✗  ✗  ✗  ✗
               └────── drawn ──────────────┘   └─ dropped ─┘
```

Games work around this by **rotating OAM order each frame**, so the dropped sprites change every frame. The result: flicker instead of invisibility. Your visual system merges the frames into "a flickering sprite," which reads as *present but glitchy* rather than *absent*. Given the hardware constraint, it's the best possible failure mode.

**Priority (the DMG rule).** When two sprites overlap:
1. The one with the **smaller X coordinate** wins.
2. If X is equal, the one with the **lower OAM index** wins.

Rule 1 is a DMG-only behavior — the Game Boy Color drops it and uses OAM index only. Why did the DMG do it by X? Because the DMG's sprite compositing hardware sorts by X as part of its scanline pipeline. It's an implementation detail that became a visible rule.

> ⚙ **Emulator implication:** get this backwards and sprites will render in the wrong order in overlapping situations — subtle, occasional, and infuriating to debug. It's explicitly tested by dmg-acid2.

**BG-over-sprite priority.** OAM flag bit 7. If set, the sprite is drawn *behind* background colors 1, 2, and 3 — but still in front of color 0. This lets a character walk behind a bush: the bush's non-zero pixels cover the sprite, the bush's color-0 pixels don't. It's a per-pixel depth test with a one-bit depth buffer.

## 7.9 Palettes

Four shades. But `BGP`, `OBP0`, `OBP1` are indirection tables:

```
   BGP ($FF47) = %11 10 01 00
                  │  │  │  └── color ID 0 → shade 0 (white)
                  │  │  └───── color ID 1 → shade 1 (light grey)
                  │  └──────── color ID 2 → shade 2 (dark grey)
                  └─────────── color ID 3 → shade 3 (black)

   Shades:  0 = white   1 = light grey   2 = dark grey   3 = black
```

**Why indirection instead of using color IDs directly?** Because you can change the mapping *instantly*, without touching a single tile.

- **Fade to black:** write four palette values over four frames. `%11100100` → `%11111001` → `%11111110` → `%11111111`. The whole screen fades. Cost: 4 byte writes.
- **Flash on damage:** invert the palette for 2 frames.
- **Reuse tiles:** the same tile drawn with two different palettes looks like two different objects.

Doing a fade with a framebuffer would mean rewriting every pixel. Here it's one byte. **This is the payoff of indirection, and it's a pattern you'll see throughout the machine: never store what you can look up.**

## 7.10 The PPU's modes and the scanline

Now we assemble it. The PPU cycles through four modes, and its position in that cycle determines what memory the CPU can touch.

```
   ONE SCANLINE = 456 T-cycles ("dots")

   ┌──────────┬────────────────────┬──────────────────────┐
   │ Mode 2   │      Mode 3        │       Mode 0         │
   │ OAM Scan │   Drawing          │       HBlank         │
   │ 80 dots  │   172-289 dots     │     87-204 dots      │
   ├──────────┼────────────────────┼──────────────────────┤
   │ OAM      │ OAM: LOCKED        │ everything           │
   │ LOCKED   │ VRAM: LOCKED       │ ACCESSIBLE           │
   │ VRAM ok  │                    │                      │
   └──────────┴────────────────────┴──────────────────────┘
        └──── Mode 3 + Mode 0 always total 376 dots ────┘

   After line 143:

   ┌────────────────────────────────────────────────────────┐
   │  Mode 1: VBLANK — lines 144-153, 4560 dots total       │
   │  Everything accessible. PPU idle. The golden window.   │
   └────────────────────────────────────────────────────────┘
```

**Mode 2 — OAM Scan (80 dots).** The PPU reads all 40 OAM entries and selects up to 10 whose Y range includes this line. 40 entries in 80 dots = 2 dots per entry. OAM is locked because the PPU is reading it.

**Mode 3 — Drawing (172–289 dots).** The PPU emits 160 pixels. Both VRAM and OAM are locked.

**Why is the length variable?** Because of **penalties**:
- `SCX & 7` costs up to 5 extra dots at the start of the line (the fetcher has to discard partial pixels for fine scrolling).
- Each sprite on the line costs roughly 6–11 extra dots.
- The window starting mid-line costs ~6 dots.

So a line with 10 sprites and awkward scrolling takes meaningfully longer to draw than an empty line. And since Mode 3 + Mode 0 is a fixed 376 dots, **a longer Mode 3 means a shorter HBlank** — which means less time for the CPU to do its mid-frame register writes. Games that push the sprite limit have less HBlank time to work with. Everything is connected.

**Mode 0 — HBlank (87–204 dots).** The PPU rests. Whatever's left of the 376. **This is when a game can safely write to VRAM mid-frame.**

**Mode 1 — VBlank (4,560 dots, ~1.09 ms).** Ten fake scanlines (144–153) where nothing is drawn. Everything is accessible.

> 🤔 **Why does VBlank exist at all? Why not just start the next frame immediately?**
> On a CRT, VBlank is the time the electron beam takes to fly from the bottom-right back to the top-left. It's physically necessary.
>
> The Game Boy has an LCD, which has no electron beam. **VBlank is preserved anyway** — partly because the LCD controller design descends from CRT conventions, and partly because it's *genuinely useful*: it's the CPU's guaranteed window to rearrange video memory. It's a vestigial organ that turned out to be load-bearing.

**Status registers:**

- `LY` (`$FF44`) — the current line, 0–153. Read-only. Games poll it, and hardware compares it.
- `LYC` (`$FF45`) — compare value. When `LY == LYC`, a STAT flag sets and can fire an interrupt.
- `STAT` (`$FF41`) — bits 0–1 hold the current mode; bit 2 is the LY==LYC flag; bits 3–6 are interrupt enables for each mode.

`LYC` is how games trigger effects at an exact scanline. "Fire an interrupt at line 112" → set `LYC = 112`, enable STAT LYC interrupt, and your handler changes `SCX` for the status bar split. This is the mechanism behind almost every fancy Game Boy visual effect.

## 7.11 The rendering pipeline: FIFO and fetcher

This is the deep end. **You do not need this for your first emulator.** Read it for the mental model; implement it later, if ever.

The PPU doesn't render a scanline all at once. It produces **one pixel per dot**, using two cooperating machines:

```
   ┌────────────────────────────────────────────────────────┐
   │                                                         │
   │   FETCHER                          FIFO                 │
   │   (produces 8 pixels               (holds pixels,       │
   │    at a time, in 6 steps)           pops 1 per dot)     │
   │                                                         │
   │   1. Get tile number     ──────►   ┌─┬─┬─┬─┬─┬─┬─┬─┐   │
   │      from the tile map             │ │ │ │ │ │ │ │ │──►│──► LCD
   │   2. Get tile data low             └─┴─┴─┴─┴─┴─┴─┴─┘   │    1 px
   │   3. Get tile data high             BG FIFO (8 deep)    │    per dot
   │   4. Push 8 pixels to FIFO                              │
   │                                    ┌─┬─┬─┬─┬─┬─┬─┬─┐   │
   │   (a separate sprite fetcher       │ │ │ │ │ │ │ │ │   │
   │    fills the sprite FIFO when      └─┴─┴─┴─┴─┴─┴─┴─┘   │
   │    a sprite's X is reached)         Sprite FIFO         │
   └────────────────────────────────────────────────────────┘
```

The two FIFOs are mixed per pixel: if the sprite pixel is color 0, or BG priority wins, the background pixel is used; otherwise the sprite pixel is.

**Why does this design exist?** Because it lets the PPU work at a *steady one-pixel-per-dot rate* while fetching in bursts of 8. The FIFO is a buffer that smooths out the mismatch between "fetch 8 at a time" and "output 1 at a time." It's the same reason a factory has a small stock of parts between stations.

**Why should you care?** Because it explains the timing penalties in §7.10:
- When a sprite is encountered, the BG fetcher **pauses** while the sprite is fetched. That pause is the per-sprite dot penalty.
- Fine scrolling (`SCX & 7`) means the first fetched tile has pixels that must be discarded, so the FIFO starts partially drained. That's the SCX penalty.
- The window switch requires **flushing the BG FIFO** and restarting the fetcher. That's the window penalty.

Every "weird timing number" in Pan Docs' PPU section is a consequence of this pipeline. Once you can see the FIFO, the numbers stop being magic.

> ⚙ **Emulator implication and a strong recommendation:** implement a **scanline renderer** first. At the end of Mode 3 for each line, read the registers as they currently are, and draw all 160 pixels of that line in one go. This gets you accurate scroll effects, accurate window behavior, and correct sprite priority — it handles the overwhelming majority of games. Only move to a dot-accurate FIFO if you want to run the handful of games and demos that change registers *mid-scanline*. Going straight to FIFO as a beginner is how people burn out.

## 7.12 Putting the layers together

```
   Final pixel at screen position (x, y):

   ┌─────────────────────────────────────────────────────────┐
   │  Step 1: BACKGROUND                                     │
   │    map_x = (x + SCX) / 8,  map_y = (y + SCY) / 8        │
   │    tile = tilemap[map_y % 32][map_x % 32]               │
   │    bg_color_id = pixel from that tile                   │
   ├─────────────────────────────────────────────────────────┤
   │  Step 2: WINDOW (if enabled and x >= WX-7 and y >= WY)  │
   │    replaces the background entirely from here on        │
   ├─────────────────────────────────────────────────────────┤
   │  Step 3: SPRITES                                        │
   │    find highest-priority sprite covering x              │
   │    if its color_id != 0:                                │
   │       if sprite has BG-priority flag AND bg_color_id!=0 │
   │            → keep background                            │
   │       else → use sprite pixel                           │
   ├─────────────────────────────────────────────────────────┤
   │  Step 4: PALETTE                                        │
   │    shade = palette[color_id]                            │
   └─────────────────────────────────────────────────────────┘
                            │
                            ▼
                    one of 4 grey shades
```

That's the whole graphics system. Read that block until it feels obvious, because everything in Pan Docs' rendering section is a footnote to it.

## 7.13 CHIP-8 vs. Game Boy graphics, side by side

| | CHIP-8 | Game Boy |
|---|---|---|
| Model | Framebuffer | Tiles + maps + sprites |
| Resolution | 64×32 | 160×144 |
| Colors | 2 (on/off) | 4 shades, via palettes |
| Drawing | CPU instruction (`DXYN`) | Independent processor, continuous |
| When drawn | When the game says | Always, ~59.7 times/sec |
| Scrolling | Rewrite everything | One register write |
| Sprites | XOR'd into the buffer | Composited by hardware, 40 max |
| Transparency | XOR (no real transparency) | Color 0 is transparent |
| Timing | Irrelevant | Determines what's legal to access |
| Layers | One | Background + Window + Sprites |

The single biggest shift: **in CHIP-8 the CPU draws; on the Game Boy the CPU describes and the PPU draws.** Your emulator's PPU is not a function you call. It's a state machine that runs on every cycle.

---
---

# Part 8: Timing

## 8.1 Why CHIP-8 let you off the hook

In CHIP-8, nothing measured time except the two timers, and those only counted down at 60 Hz. There was no relationship between "how many instructions I've run" and "what the display is doing," because the display only changed when you told it to.

The Game Boy has **one clock and four consumers**, and they all have to agree.

```
                     4,194,304 T-cycles per second
    ┌──────────────────────────┬──────────────┬──────────────┐
    │                          │              │              │
    ▼                          ▼              ▼              ▼
  ┌─────┐                   ┌─────┐        ┌───────┐     ┌─────┐
  │ CPU │                   │ PPU │        │ TIMER │     │ APU │
  │     │                   │     │        │       │     │     │
  │ ÷4  │                   │ ÷1  │        │ ÷16   │     │ ÷2  │
  │(M-cy)│                  │(dots)│       │ ÷64   │     │     │
  └─────┘                   └─────┘        │ ÷256  │     └─────┘
                                           │ ÷1024 │
                                           └───────┘
```

They don't just need to be *individually* correct. They need to be correct **relative to each other**. If your PPU runs 1% fast relative to your CPU, a game that writes `SCX` in an HBlank handler will sometimes write it a scanline late, and you'll get a one-line graphical tear that appears randomly and is nearly impossible to trace.

## 8.2 The hierarchy of time

```
   1 T-cycle    = the fundamental tick.  1 / 4,194,304 second ≈ 238 ns
                  Also called a "dot" when talking about the PPU.

   1 M-cycle    = 4 T-cycles.  One memory access. The CPU's real step.

   1 scanline   = 456 T-cycles = 114 M-cycles.

   1 frame      = 154 scanlines = 70,224 T-cycles
                  (144 visible + 10 VBlank)

   1 second     ≈ 59.727 frames
```

Let's verify: 456 × 154 = 70,224. And 4,194,304 / 70,224 = **59.727 Hz**.

Note that it is *not* 60 Hz. It's 59.727. Over a minute, a naïve 60 Hz emulator drifts by about 16 frames' worth of time — which is audible as pitch drift in music and visible as slightly-too-fast gameplay. Use the real number.

**Everything is a power-of-two division of 4,194,304**, which is why these numbers work out so cleanly in hardware. 456 = 8 × 57, not a power of two, but the PPU's line counter is just a counter — it doesn't need to divide evenly.

## 8.3 The timer peripheral

Four registers, and they're a great small example of how a peripheral works.

```
  $FF04  DIV   Divider    — increments at 16,384 Hz, always, unstoppable
  $FF05  TIMA  Counter    — increments at a selectable rate
  $FF06  TMA   Modulo     — the value TIMA reloads to on overflow
  $FF07  TAC   Control    — enable bit + rate selection
```

**`DIV`** is a free-running counter. It increments every 256 T-cycles and you cannot stop it. Writing *any* value to `DIV` resets it to 0.

**Why does DIV exist?** Two uses:
1. **Randomness.** The Game Boy has no random number generator. Reading `DIV` at an unpredictable moment (like when the player presses a button) gives you an unpredictable byte. Nearly every Game Boy game seeds its RNG from `DIV`. Tetris's piece selection is downstream of this register.
2. **It's the source for everything else.** `TIMA` and the sound channels are driven by taps off the same internal counter that `DIV` exposes. Which produces a lovely quirk: **writing to `DIV` (resetting it) can cause `TIMA` to tick**, because a bit in the counter transitions from 1 to 0. Games have been known to rely on this. It's tested by Blargg's and Mooneye's timer tests.

**`TIMA`** counts up at one of four rates set by `TAC`:

| TAC bits 1–0 | Frequency | Every N T-cycles |
|---|---|---|
| 00 | 4,096 Hz | 1024 |
| 01 | 262,144 Hz | 16 |
| 10 | 65,536 Hz | 64 |
| 11 | 16,384 Hz | 256 |

(Note the ordering — `00` is the *slowest*. Not sorted. Everyone misreads this table once.)

When `TIMA` overflows past 255, it reloads from `TMA` and fires the **Timer interrupt**.

**What's this for?** A programmable heartbeat independent of the display. Music drivers are the classic use: you want to advance the song at a musically meaningful tempo, not at 59.7 Hz. Set `TMA` so the timer fires at, say, 256 Hz, and your music routine runs at a stable rate regardless of frame rate.

**The TIMA overflow quirk** (which you'll meet in test ROMs): when `TIMA` overflows, the reload from `TMA` doesn't happen instantly. There's a 4-cycle window where `TIMA` reads as `$00` before the reload takes effect, and writing to `TIMA` during that window cancels the reload. This is exactly the kind of thing that makes people write cycle-accurate emulators.

## 8.4 The main loop: three architectures

Your emulator's timing architecture is the most important structural decision you'll make. Here are the three common approaches, in order of increasing accuracy and difficulty.

### Architecture 1: Instruction-stepped ("catch-up")

```
   while (true) {
       cycles = cpu.step();          // run one whole instruction
       ppu.step(cycles);             // advance PPU by that many dots
       timer.step(cycles);
       apu.step(cycles);
       cpu.check_interrupts();
   }
```

**Pros:** simple, fast, and runs the large majority of commercial games correctly.
**Cons:** components only sync at instruction boundaries. A `CALL` takes 24 cycles, so the PPU can jump 24 dots at once — potentially crossing a mode boundary in the middle. If the game wrote a register "during" those 24 cycles, the ordering is wrong.

**This is where you should start.** Absolutely, unambiguously. Get a game booting before you worry about anything finer.

### Architecture 2: M-cycle stepped

```
   while (true) {
       cpu.step_one_m_cycle();       // one bus access worth of work
       ppu.step(4);
       timer.step(4);
       apu.step(4);
   }
```

Here the CPU is a state machine that does one memory access per call. Every component advances 4 T-cycles at a time, in lockstep. Register writes land at the correct M-cycle.

**Pros:** handles nearly everything, including most timing tests.
**Cons:** your CPU must be restructured as a state machine, which is a substantial rewrite. This is why people say "design for it from the start" — but honestly, understanding the machine first is more valuable than premature architecture.

### Architecture 3: T-cycle stepped

Everything advances one T-cycle at a time. The PPU's FIFO is modeled dot by dot. This is what emulators like SameBoy and BGB do.

**Pros:** runs everything, including hardware test demos.
**Cons:** slow, complex, and requires knowing the hardware in extraordinary detail.

### My honest recommendation

```
   Stage 1: Instruction-stepped, scanline PPU renderer
            → boots games, plays Tetris, Mario, Pokémon
            → this is a genuine achievement, be happy here

   Stage 2: Add proper VRAM/OAM locking, accurate DMA,
            timer quirks
            → passes most of Blargg's tests

   Stage 3: M-cycle stepping, FIFO renderer
            → passes Mooneye, runs demos
            → only if you're enjoying it
```

Most people who fail at writing a Game Boy emulator fail because they tried to start at stage 3.

## 8.5 Synchronizing to real time

Your emulator runs as fast as your host CPU allows — which is thousands of times too fast. You need to throttle.

**Naïve approach:** after every frame, sleep until 16.74 ms have elapsed. Works, but `sleep()` granularity on most operating systems is coarse (1–15 ms), so you'll get stuttering.

**Better approach: sync to audio.** Your APU generates samples at a fixed rate (say 44,100 Hz). Feed them into an audio buffer. When the buffer is full, the audio device *blocks* until it has room. **The sound card becomes your clock.** This is elegant because audio hardware has a genuinely accurate crystal, and because audio glitches are far more noticeable to humans than a dropped video frame — so syncing to audio optimizes for the sense that cares most.

Most good emulators do this. It's worth knowing about early, even if you implement audio last.

## 8.6 Why timing is *the* difficulty of Game Boy emulation

Let me state the core problem as directly as I can.

**In real hardware, everything is simultaneous. In your emulator, everything is sequential.**

Real hardware:
```
   T-cycle 1000:  CPU is mid-instruction  AND  PPU draws pixel 47
                  AND timer's internal counter increments
                  AND channel 1 advances its duty step
                            ── all at the same instant ──
```

Your emulator:
```
   run CPU for 8 cycles  →  then run PPU for 8 cycles  →
   then timer for 8  →  then APU for 8
                            ── strictly one after another ──
```

You're approximating parallelism with fine-grained interleaving. The finer the interleaving, the better the approximation, and the slower and more complex the emulator. **That trade-off is the entire field of emulator accuracy.**

Games that "just work" in a coarse emulator are games that only interact with hardware at safe boundaries (during VBlank, with plenty of slack). Games that break are games whose programmers discovered that if you write `SCX` exactly 43 cycles into HBlank you get a cool effect — and shipped it.

---
---

# Part 9: Interrupts

## 9.1 First principles: what problem are we solving?

Your program needs to react to events it didn't cause and can't predict:
- The PPU finished a frame.
- The timer overflowed.
- A button was pressed.

**Approach A: polling.**

```
   wait_for_vblank:
       LD A, ($FF44)     ; read LY
       CP 144
       JR NZ, wait_for_vblank
```

This works. Games actually do it sometimes. But it burns the CPU entirely — you're spending 100% of your cycles asking a question. And if you're doing something *else* important, you can't poll at the same time.

**Approach B: interrupts.** The hardware notifies you.

## 9.2 The mechanism, step by step

```
   ┌─────────────────────────────────────────────────────────┐
   │  1. An event occurs (PPU enters VBlank)                 │
   │                                                          │
   │  2. Hardware sets the corresponding bit in IF ($FF0F)   │
   │     "an interrupt is REQUESTED"                          │
   │                                                          │
   │  3. Between instructions, the CPU checks:                │
   │        IME == 1?                (master switch on)       │
   │        AND (IE & IF) != 0?      (this one is enabled     │
   │                                  AND requested)          │
   │                                                          │
   │  4. If yes:                                              │
   │        a. IME = 0        (disable further interrupts)    │
   │        b. clear the IF bit for this interrupt            │
   │        c. push PC onto the stack     (2 bytes)           │
   │        d. PC = the interrupt's vector address            │
   │        e. this whole process takes 20 T-cycles           │
   │                                                          │
   │  5. Handler runs. Ends with RETI:                        │
   │        pop PC from stack, and set IME = 1                │
   └─────────────────────────────────────────────────────────┘
```

## 9.3 The three registers

**`IME`** — Interrupt Master Enable. A single internal flag, **not memory-mapped**. You cannot read it. `EI` sets it, `DI` clears it, `RETI` sets it, and dispatching an interrupt clears it.

**`IE`** (`$FFFF`) — Interrupt Enable. Five bits: which interrupts you *care about*.

**`IF`** (`$FF0F`) — Interrupt Flag. Five bits: which interrupts are *currently requested*.

```
   Bit:      4        3       2       1        0
          ┌───────┬────────┬───────┬───────┬────────┐
   IE     │Joypad │ Serial │ Timer │ STAT  │ VBlank │   $FFFF
          ├───────┼────────┼───────┼───────┼────────┤
   IF     │Joypad │ Serial │ Timer │ STAT  │ VBlank │   $FF0F
          └───────┴────────┴───────┴───────┴────────┘
   Vector:  $0060   $0058    $0050   $0048    $0040

   Priority:  lowest ◄─────────────────────► highest
```

**Why three levels of gating?** Each serves a distinct purpose:
- `IF` says *what happened*. It's set by hardware, regardless of whether you care.
- `IE` says *what you're interested in*. A game that doesn't use the timer just leaves that bit clear.
- `IME` is a **global off switch** for critical sections. When you're updating a data structure that an interrupt handler also touches, you `DI`, do the update, and `EI`. Without it, an interrupt could fire halfway through and see a half-updated structure. It's a mutex, in hardware, in one bit.

**Priority:** if multiple interrupts are pending simultaneously, the lowest-numbered bit wins. VBlank always beats Joypad. Makes sense — VBlank is time-critical and Joypad isn't.

## 9.4 Why the vectors are 8 bytes apart

```
   $0040  VBlank
   $0048  STAT
   $0050  Timer
   $0058  Serial
   $0060  Joypad
```

Eight bytes each. You can't fit a handler in 8 bytes. So what goes there?

```
   $0040:  JP vblank_handler      ; 3 bytes
           (5 bytes wasted)
```

A jump. The vector table is a table of **trampolines**. Eight bytes is enough for a `JP nn` with room to spare, and 8 is a convenient power of two for address decoding (the vector address is `$40 + (interrupt_number × 8)`, which is a shift and an add).

Note these vectors sit in **ROM bank 0**, at fixed addresses, which is another reason bank 0 must always be mapped.

## 9.5 The five interrupts, and what games use them for

**VBlank (`$0040`)** — fires when the PPU enters line 144. Used by: **everything**. This is the main game heartbeat. The handler does OAM DMA, updates VRAM, reads input, advances music. Then the main loop resumes its thinking.

**STAT / LCD (`$0048`)** — fires on a configurable PPU condition. Four selectable sources (STAT bits 3–6):
- Mode 0 (HBlank) entered
- Mode 1 (VBlank) entered
- Mode 2 (OAM scan) entered
- `LY == LYC`

This is the **special effects interrupt**. Every mid-frame trick uses it:

```
   Example: a fixed status bar with a scrolling world

   Setup:  LYC = 128
           enable STAT LYC interrupt

   At line 128:
     STAT interrupt fires
     handler: LD A, 0
              LD ($FF43), A     ; SCX = 0, stop scrolling
              RETI

   During VBlank:
     restore SCX = camera_x for the next frame
```

Result: lines 0–127 scroll with the world, lines 128–143 are a fixed HUD. Two register writes per frame.

**Timer (`$0050`)** — when `TIMA` overflows. Used for music tempo, and for anything that needs a rate other than 59.7 Hz.

**Serial (`$0058`)** — link cable transfer complete. Pokémon trading, Tetris versus.

**Joypad (`$0060`)** — a button went from high to low. Rarely used for actual input reading (games poll `$FF00` during VBlank instead, because polling gives you all buttons at once). Its real purpose is **waking the CPU from `STOP` mode** — the power-saving sleep. It's a wake-up line more than an input mechanism.

## 9.6 Input: how `$FF00` actually works

This deserves its own section because it's genuinely strange the first time you see it.

There are 8 buttons but the joypad register is one byte with only 4 usable input bits. How?

**Matrix scanning.** You *select* which group of 4 buttons to read, then read them.

```
   $FF00 (P1/JOYP)

   bit 7  unused (reads 1)
   bit 6  unused (reads 1)
   bit 5  ← WRITE 0 here to select the ACTION buttons
   bit 4  ← WRITE 0 here to select the DIRECTION buttons
   bit 3  → READ: Start   / Down
   bit 2  → READ: Select  / Up
   bit 1  → READ: B       / Left
   bit 0  → READ: A       / Right
```

**Note the inversion: 0 means pressed, 1 means not pressed.** This is because the buttons are wired to ground — pressing one pulls the line low. Active-low logic is standard in hardware for exactly this reason, and it catches every beginner.

The reading procedure:

```
   ; read direction pad
   LD A, %00100000        ; bit 5 = 1 (deselect actions),
   LD ($FF00), A          ; bit 4 = 0 (select directions)
   LD A, ($FF00)          ; read... twice, actually
   LD A, ($FF00)          ; (hardware needs settling time)
   AND $0F                ; keep the low 4 bits
   ; A now has direction states, active-low

   ; then repeat with %00010000 for the action buttons
```

**Why the double read?** Real hardware has capacitance on those lines; the signal takes a moment to settle after you change the select bits. Games read twice (sometimes six times) to let it stabilize. Your emulator doesn't need to simulate this, but you should recognize the pattern when you see it in disassemblies.

**Why a matrix at all?** To save pins. Eight buttons on eight pins costs eight pins. A 2×4 matrix costs 2 select + 4 read = 6 pins. On a chip where every pin costs money, saving two pins is worth adding a scan procedure. Same reasoning as everything else on this machine.

> ⚙ **Emulator implication:** keep your button states in a byte, and implement `read($FF00)` as: check which select bits are low, return the corresponding nibble inverted, with unused bits set to 1. Getting the inversion backwards means your character walks constantly and no button does anything — a very recognizable bug.

## 9.7 The subtleties that break emulators

**`EI` is delayed by one instruction.**

```
   EI
   DI       ; ← IME is still 0 here!
```

`EI` sets a "will enable after the next instruction" latch. The idiom `EI; RETI` or the classic:

```
   EI
   HALT
```

...relies on this. The delay exists because of the CPU's pipelining: the decision about whether to service an interrupt has already been made by the time `EI` finishes executing.

**`RETI` is immediate**, unlike `EI`. It's `RET` + instant `IME = 1`.

**The HALT bug** (revisited from §4.8). If `HALT` runs with `IME = 0` and `(IE & IF) != 0`, the CPU doesn't halt and the next byte is fetched twice. Real games — including some commercial ones — hit this. It's not obscure trivia; it's on Blargg's test suite.

**Interrupt dispatch takes 20 T-cycles** and involves 2 stack pushes. Those pushes go through the normal bus. If your emulator dispatches interrupts "for free," your timing drifts.

**An interrupt can wake a `HALT` even with `IME = 0`.** If `IME` is off but a bit in `IE & IF` becomes set, the CPU resumes from `HALT` — it just doesn't jump to the vector. It continues with the next instruction. This is used deliberately: `HALT` with `IME = 0` is "sleep until something happens, then handle it myself."

## 9.8 Interrupts in your emulator

```
   check_interrupts():
       pending = IE & IF & 0x1F
       if pending == 0: return 0

       if halted:
           halted = false          # wake up regardless of IME

       if not IME: return 0

       IME = false
       for bit in 0..4:
           if pending & (1 << bit):
               IF &= ~(1 << bit)
               push(PC)
               PC = 0x40 + (bit * 8)
               return 20            # T-cycles consumed
```

Call this after each instruction. That's the whole thing — maybe 15 lines. **Interrupts are conceptually large and implementationally tiny**, which is a nice change from most of this document.

---
---

# Part 10: One Frame of Pokémon, in Slow Motion

Time to put everything together. We'll observe ~70,224 T-cycles of a real machine, with all systems visible at once.

**The scene:** Red is standing in Pallet Town. The player is holding Right. Music is playing.

---

## T-cycle 0 — VBlank has just begun (LY = 144)

```
  ┌────────────────────────────────────────────────────────────┐
  │ PPU     │ Just finished line 143. Entering Mode 1 (VBlank).│
  │         │ Sets IF bit 0 (VBlank requested).                │
  │         │ VRAM and OAM now UNLOCKED.                        │
  ├─────────┼──────────────────────────────────────────────────┤
  │ CPU     │ Was executing map-loading code in ROM bank 3.     │
  │         │ Finishes its current instruction.                 │
  ├─────────┼──────────────────────────────────────────────────┤
  │ TIMER   │ DIV has been ticking all frame. TIMA at 0x8F.     │
  ├─────────┼──────────────────────────────────────────────────┤
  │ APU     │ Channel 1 mid-note, sweeping frequency.           │
  └─────────┴──────────────────────────────────────────────────┘
```

## T-cycle ~4 — Interrupt dispatch

The CPU checks: `IME` is 1, `IE & IF` has bit 0 set. Dispatch.

```
   IME = 0
   IF bit 0 cleared
   PUSH PC      ; SP: $DFF0 → $DFEE, two bytes written to WRAM
   PC = $0040
```

**20 T-cycles consumed.** Memory accesses: 2 writes to WRAM.

At `$0040` sits `JP VBlankHandler`. The CPU jumps into bank 0.

## T-cycles ~28–120 — Save state, start DMA

The handler first pushes every register it will clobber:

```
   PUSH AF      ; 16 cycles, 2 WRAM writes
   PUSH BC      ; 16
   PUSH DE      ; 16
   PUSH HL      ; 16
```

**Why?** The interrupted code is mid-computation and expects its registers untouched. This is the phone-call analogy from §2.9 — you must remember your page.

Then it calls the DMA routine that lives in **HRAM**:

```
   CALL hDMARoutine        ; jumps to $FF80

   ; In HRAM at $FF80:
   hDMARoutine:
       LD A, $C3           ; shadow OAM is at $C300
       LD ($FF46), A       ; ◄── START DMA
       LD A, 40
   .wait:
       DEC A
       JR NZ, .wait        ; spin for ~160 M-cycles
       RET
```

## T-cycles ~120–760 — DMA runs

```
  ┌────────────────────────────────────────────────────────────┐
  │ DMA     │ Copying $C300-$C39F → $FE00-$FE9F                │
  │         │ One byte per M-cycle, 160 M-cycles = 640 T.      │
  │         │ ★ THE BUS IS BUSY. External memory unavailable.  │
  ├─────────┼──────────────────────────────────────────────────┤
  │ CPU     │ Executing the wait loop FROM HRAM.                │
  │         │ HRAM is on-chip — no external bus needed.        │
  │         │ This is the only safe place to be right now.     │
  ├─────────┼──────────────────────────────────────────────────┤
  │ PPU     │ Still VBlank. Line ~145. Doing nothing.          │
  ├─────────┼──────────────────────────────────────────────────┤
  │ TIMER   │ Still ticking. Doesn't care about the bus.       │
  └─────────┴──────────────────────────────────────────────────┘
```

The 40 sprite entries — Red's four sprite tiles, the NPCs, the grass-rustle effect — are now in OAM, ready for the PPU to consume next frame.

## T-cycles ~760–900 — Read input

```
   LD A, %00100000
   LD ($FF00), A          ; select direction pad
   LD A, ($FF00)          ; read (unstable)
   LD A, ($FF00)          ; read again (settled)
   CPL                    ; invert — active-low becomes active-high
   AND $0F
   LD (hJoypadState), A   ; store in HRAM
```

Right is held, so bit 0 was 0 in the raw read; after `CPL` it's 1. The game now knows: **Right pressed.**

Note this is *polling*, during VBlank, not using the Joypad interrupt. That's how essentially every game does it.

## T-cycles ~900–2,500 — Push graphics updates

Red walked one step, so a new column of the map has scrolled into view. The game copies fresh tile map bytes into VRAM:

```
   ; copy 18 tile indices into the off-screen column
   LD HL, wTileMapBuffer
   LD DE, $9800 + offset
   LD B, 18
   .loop:
       LD A, (HL+)
       LD (DE), A
       ; advance DE by 32 (next map row)
       ...
       DEC B
       JR NZ, .loop
```

**This is only legal right now.** During Mode 3, that `LD (DE), A` would be silently discarded. The whole architecture of the game exists to get these writes into this window.

Also updated: `SCX` and `SCY` for the new camera position, and the palette if a fade is in progress.

## T-cycles ~2,500–3,800 — Advance the music

The sound driver reads the next few bytes of the song, and writes to the APU registers:

```
   LD A, freq_low
   LD ($FF13), A         ; channel 1 frequency low
   LD A, %10000111
   LD ($FF14), A         ; trigger + frequency high
```

Notice: writing to `$FF14` with bit 7 set doesn't store a value — it **triggers** the channel to restart its envelope and begin the note. Another register that's an action, not storage.

## T-cycles ~3,800–4,560 — Finish VBlank

```
   POP HL
   POP DE
   POP BC
   POP AF
   RETI              ; pop PC, IME = 1
```

The CPU returns to exactly where it was in the map-loading code, with every register restored. **The interrupted code has no idea any of that happened.**

## T-cycle 4,560 — LY = 0. The frame begins.

```
  ┌────────────────────────────────────────────────────────────┐
  │ PPU     │ Mode 2 (OAM scan), line 0.                       │
  │         │ Reading all 40 OAM entries, looking for sprites  │
  │         │ that intersect line 0. Finds 0 (Red is at y=72). │
  │         │ ★ OAM LOCKED for 80 dots.                        │
  ├─────────┼──────────────────────────────────────────────────┤
  │ CPU     │ Back in bank 3, doing map logic. It does NOT     │
  │         │ touch VRAM or OAM now — the programmer knew.     │
  └─────────┴──────────────────────────────────────────────────┘
```

## T-cycles 4,640–4,812 — Mode 3, line 0 is drawn

```
   dot 0:   fetcher gets tile number from tilemap at
            ($9800 + ((0+SCY)/8)*32 + (SCX/8))
   dot 2:   fetcher gets tile data low byte from VRAM
   dot 4:   fetcher gets tile data high byte
   dot 6:   8 pixels pushed into the BG FIFO
   dot 7:   first pixel popped, palette applied, sent to LCD
   ...
   dot 172: 160th pixel sent. Line complete.

   ★ VRAM AND OAM BOTH LOCKED this entire time.
```

`SCX` was 3, so `SCX & 7 = 3` → 3 extra dots of penalty at the start. No sprites on this line, no window. Mode 3 takes 172 + 3 = 175 dots.

## T-cycles 4,815–5,016 — Mode 0, HBlank

201 dots of nothing. The CPU is free. Pokémon doesn't do much here, but a game with a fancier effect would have a STAT HBlank interrupt firing right now.

## ... 143 more lines ...

The interesting ones:

**Line 72** — Red's sprite. During Mode 2, the PPU finds 4 OAM entries whose Y range covers line 72. During Mode 3, when the fetcher reaches X = Red's position, it pauses the background fetch, fetches Red's tile row from VRAM, and pushes it into the sprite FIFO. **Mode 3 grows by ~6 dots per sprite**, so this line takes longer than line 0, and HBlank is correspondingly shorter.

**Line 112 (if a text box is open)** — `LYC = 112`, STAT interrupt fires. The handler sets `WY` so the window (the text box) appears. From here down, the window overrides the background entirely.

**Lines 0–143 generally** — the CPU is doing everything that isn't graphics: map collision, NPC movement scripts, wild-encounter checks against `DIV`-seeded RNG, decompressing the next map's tiles into a WRAM buffer, running the battle state machine.

## T-cycle 70,224 — LY = 144 again

VBlank fires. We're back where we started, 16.74 milliseconds later, and the whole cycle repeats 59.7 times a second.

---

## What to take from this

Look at the *shape* of that frame:

```
   ┌──────────────────────────────────────────────────────────┐
   │  6.5% of the frame: VBlank                                │
   │  ██ all the graphics changes happen here                  │
   ├──────────────────────────────────────────────────────────┤
   │  93.5% of the frame: visible lines                        │
   │  ░░ CPU thinks; PPU draws; they mostly stay out of each   │
   │     other's way, coordinated by interrupts                │
   └──────────────────────────────────────────────────────────┘
```

**The Game Boy is a machine built around a 1.09-millisecond window.** Everything a game wants to *show* has to fit through it. Everything a game wants to *compute* has to happen around it. Interrupts are the coordination mechanism. Timing is what makes the coordination possible.

If you understand that paragraph, you understand the Game Boy.

---
---

# Interlude: Sound, at a High Level

You asked for sound at a high level, and high level is genuinely the right altitude here. The APU is the part of the Game Boy you should implement **last**, and understanding its shape is enough for now.

## The problem

You want music and sound effects. You have almost no RAM and a CPU that's already busy.

**The expensive approach:** store recorded audio samples and play them back. A single second of even terrible-quality audio is thousands of bytes. Impossible.

**The cheap approach:** don't store sound — **generate** it. A square wave is just a counter flipping a bit. A noise generator is a shift register. These cost almost nothing in silicon and zero bytes of storage. You describe a note with a few register writes, and the hardware makes the sound continuously until you tell it otherwise.

This is why 8-bit music sounds the way it does. **The aesthetic is a direct consequence of the cost of memory.**

## The four channels

```
   ┌────────────────────────────────────────────────────────────┐
   │ CH1  Square wave + frequency SWEEP                          │
   │      ┌──┐  ┌──┐  ┌──┐     Melody, and "pew" laser sounds    │
   │      ┘  └──┘  └──┘  └──   (the sweep slides the pitch)      │
   ├────────────────────────────────────────────────────────────┤
   │ CH2  Square wave (no sweep)                                 │
   │      Harmony, counter-melody, second voice                  │
   ├────────────────────────────────────────────────────────────┤
   │ CH3  Wave channel — plays a 32-sample waveform from a       │
   │      16-byte RAM buffer at $FF30-$FF3F                      │
   │      Bass lines, and occasionally crude speech              │
   ├────────────────────────────────────────────────────────────┤
   │ CH4  Noise — a linear-feedback shift register               │
   │      ░▒▓░▓▒░░▓▒  Drums, explosions, footsteps               │
   └────────────────────────────────────────────────────────────┘
```

Four voices. That's the entire orchestra. Every piece of Game Boy music you've ever heard — the Tetris theme, Pokémon's route music, the Zelda overworld — is three tones and a noise generator.

## The three modifiers

Each channel has a small set of automatic modulators, which exist so the CPU doesn't have to update registers constantly:

| Modifier | What it does | Why it exists |
|---|---|---|
| **Length counter** | Auto-stops the note after N ticks | so the CPU doesn't have to time note-offs |
| **Volume envelope** | Ramps volume up or down over time | gives notes attack and decay — the difference between a "beep" and a "pluck" |
| **Frequency sweep** (CH1 only) | Slides pitch up or down | sirens, lasers, power-up sounds, all free |

**The duty cycle** on the square channels is worth one sentence: you can select 12.5%, 25%, 50%, or 75% "on" time. This changes the timbre substantially — 50% is a hollow, clarinet-like tone; 12.5% is thin and nasal. Four timbres per channel, free.

## The frame sequencer

All these modulators are clocked by a shared 512 Hz counter called the **frame sequencer** (unrelated to video frames, confusingly). It runs an 8-step cycle:

```
   Step:    0    1    2    3    4    5    6    7
   Length:  ✓         ✓         ✓         ✓        (256 Hz)
   Sweep:        ✓                   ✓             (128 Hz)
   Envelope:                                  ✓    (64 Hz)
```

The frame sequencer is itself driven by a bit of the `DIV` counter — which means **writing to `DIV` can affect audio timing.** Everything on this machine is connected to everything else.

## What this means for your emulator

```
   Stage 1:  Skip audio entirely. Stub the registers. Games run fine.
   Stage 2:  Implement channels 1, 2, 4 (square + noise). This is
             most of the recognizable sound.
   Stage 3:  Channel 3 (wave), then the accuracy details —
             trigger behavior, obscure length-counter edge cases.
```

The mechanical part: each channel is a small counter-based generator you advance by N T-cycles. You mix the four outputs, downsample to your host's audio rate (44,100 Hz), and push into a buffer. §8.5 explains why that buffer can double as your frame-rate clock.

**Do not implement audio until video and input work.** It is the single most common place where people get bogged down and abandon the project.

---
---

# Part 11: Reading Pan Docs — A Survival Guide

## 11.1 What Pan Docs actually is

Pan Docs is a **reference**, assembled over 25+ years by many people, refined by hardware testing. It is:

- Extremely accurate.
- Extremely complete.
- Organized by *hardware component*, not by *learning order*.
- Written by experts, for people who already have the mental model.

**It is not a tutorial and was never trying to be.** Reading it front-to-back is a mistake, and feeling lost while doing so is not a reflection of your ability.

## 11.2 The right way to use it

**Rule 1: Look things up. Don't read through.**
Treat it like `man` pages. You're implementing sprite rendering → read the OAM page. You hit a flag question → read the CPU instruction page. Targeted reading, with a purpose.

**Rule 2: Read a section three times.**
- First pass: skim for structure. What are the pieces?
- Second pass: understand the mechanism.
- Third pass: only when implementing, for the exact bit-level detail.

**Rule 3: When you hit something incomprehensible, note it and move on.**
Pan Docs sections have dependencies that aren't marked. If the PPU timing page is opaque, it might be because you haven't read the PPU modes page. Come back.

**Rule 4: The "obscure behavior" sections are not for you yet.**
Pan Docs documents hardware quirks that maybe three games in existence rely on. These are there for completeness. Skipping them is correct until a specific test ROM fails.

## 11.3 Section-by-section briefing

### "Memory Map"

**Assumes you know:** address space vs. memory (§2.3), memory-mapped I/O (§2.4), that regions have owners.

**How to approach:** this is the first page you should read, and you should already understand it from Part 5. It's a table. Bookmark it.

**Common misunderstanding:** thinking the map describes 64 KB of RAM. It describes 64 KB of *addresses*, most of which aren't RAM at all.

### "The Cartridge Header"

**Assumes:** you know what a checksum is and that the boot ROM validates the cartridge.

**How to approach:** read it once when you write your ROM loader. You need three bytes: `$0147` (type), `$0148` (ROM size), `$0149` (RAM size).

**Common misunderstanding:** trying to implement the global checksum at `$014E`. The real hardware **doesn't check it**. Only the header checksum at `$014D` matters, and only if you emulate the boot ROM.

### "MBCs"

**Assumes:** you understand bank switching conceptually and know that writes to ROM are commands.

**How to approach:** Part 6 gave you the model. Read only the MBC you're currently implementing. **Start with "No MBC," then MBC1.**

**Common misunderstanding:** MBC1's mode register is the single most confusing thing on the page. If it doesn't click, implement mode 0 only — almost every MBC1 game works with just that.

### "I/O Registers"

**Assumes:** memory-mapped I/O, and that reads and writes can have side effects.

**How to approach:** don't read the whole list. Read the entry for the specific register you're working on.

**Common misunderstanding:** implementing I/O as a byte array. Some bits are read-only, some are write-only, some always read as 1. If you store-and-return, you will pass no test ROMs.

### "LCDC" / "STAT" / "Rendering"

**Assumes:** tiles, tile maps, scrolling, sprites, PPU modes, dot timing — everything in Part 7.

**How to approach:** this is the densest area. Take it in this order:
1. LCDC (what the bits control)
2. Tile data + tile maps
3. Palettes
4. OAM / sprite attributes
5. STAT and PPU modes
6. Rendering timing (Mode 3 penalties) — **last, and only if needed**

**Common misunderstandings:**
- Confusing the two tile *data* addressing modes with the two tile *map* selections. They're independent settings: LCDC bit 4 picks tile data addressing, bit 3 picks the BG map, bit 6 picks the window map.
- Forgetting sprites always use `$8000` unsigned mode.
- Forgetting the window's internal line counter is separate from `LY`.
- Forgetting the sprite Y/X offsets (16 and 8).

### "Pixel FIFO"

**Assumes:** a great deal, including hardware pipelining and shift registers.

**How to approach:** **read it for understanding, not for implementation.** It explains *why* Mode 3 has variable length. You can build an excellent emulator with a scanline renderer and never implement a FIFO.

**Common misunderstanding:** believing you need this to start. You don't. Many well-regarded emulators use scanline rendering.

### "Interrupts"

**Assumes:** the stack, `IME` vs `IE` vs `IF`, and that interrupts fire between instructions.

**How to approach:** Part 9 covered the model. Pan Docs adds exact timing and the edge cases.

**Common misunderstandings:**
- Mixing up `IE` (`$FFFF`) and `IF` (`$FF0F`). They're separated in the map for no good reason and everyone confuses them.
- Forgetting `EI`'s one-instruction delay.
- Not implementing the HALT bug, then failing tests mysteriously.

### "Timer and Divider Registers"

**Assumes:** clock division, and that `DIV` and `TIMA` share an internal counter.

**How to approach:** implement the basic version first (a counter that increments at the `TAC` rate and fires an interrupt). Add the quirks — the internal-counter model, the TIMA reload delay — when Blargg's timer tests fail.

**Common misunderstanding:** treating `DIV` and `TIMA` as unrelated. They're taps off the same 16-bit counter, which is why writing `DIV` can tick `TIMA`.

### "Audio"

**Assumes:** basic signal concepts — frequency, duty cycle, envelopes, LFSRs.

**How to approach:** last. Genuinely last. The Interlude above is enough context to come back to this whenever you're ready.

### "OAM DMA Transfer"

**Assumes:** DMA, bus contention, and why HRAM exists.

**How to approach:** you understand this from §2.10. Implement the instant version first.

**Common misunderstanding:** not understanding *why* the DMA routine must live in HRAM, and therefore not understanding what the bus-conflict rules are protecting against.

## 11.4 Vocabulary Pan Docs uses without defining

| Term | Meaning |
|---|---|
| **dot** | one T-cycle, from the PPU's point of view |
| **object / OBJ** | a sprite (Nintendo's official term) |
| **DMG** | the original Game Boy (Dot Matrix Game) |
| **CGB / GBC** | Game Boy Color |
| **SGB** | Super Game Boy (the SNES adapter) |
| **MGB** | Game Boy Pocket |
| **AGB / GBA** | Game Boy Advance |
| **scanline** | one horizontal row of the display |
| **VBlank / HBlank** | the idle periods after a frame / after a line |
| **LY** | the current scanline number |
| **mode** | which of the PPU's four phases is active |
| **STAT** | the LCD status register, and by extension its interrupt |
| **fetcher** | the PPU unit that reads tile data |
| **FIFO** | the pixel queue between fetcher and screen |
| **MBC** | Memory Bank Controller (the cartridge mapper chip) |
| **bank** | a 16 KB chunk of cartridge ROM (or 8 KB of cart RAM) |
| **latch** | to snapshot a value at a specific moment |
| **prefixed instruction** | a `$CB`-prefixed opcode |

---
---

# Part 12: Watching the 33C3 Talk — A Companion Guide

## 12.1 What the talk is and why it's hard

*The Ultimate Game Boy Talk* by Michael Steil (Chaos Communication Congress, 2016) is roughly an hour of extremely dense, extremely good material. It's hard for a specific reason:

**It is a *summary* talk, not a *teaching* talk.** It was delivered to a room full of hackers and reverse engineers. Steil moves fast because he assumes his audience already knows what a bus is, what a tile is, and what an interrupt does. He's not explaining the Game Boy from scratch — he's showing you the *interesting* parts, quickly, with the boring parts assumed.

**With this primer completed, you now have the assumed background.** The talk should go from "incomprehensible" to "dense but followable."

## 12.2 How to watch it

1. **Watch it once at normal speed without pausing.** Don't try to absorb everything. Let it wash over you. Notice which parts you now recognize.
2. **Watch it again with the slides open**, pausing freely.
3. **Watch specific sections again** while implementing that part of your emulator. The graphics segment in particular will mean something completely different after you've written a tile decoder.

Steil's slides are available separately and are worth having open. He also packs a lot into single slides — pausing on them is often more valuable than the spoken words.

## 12.3 The major areas, and what you need for each

The talk moves roughly through the following territory. For each, here's what background you now have and what to focus on.

### Hardware overview and the CPU

**You should already understand:** that the LR35902 is a system-on-chip containing the CPU core plus peripherals (§3.1); the register file (§4.2); why "it's a Z80" is a useful lie (§4.1).

**Hidden assumption:** that you know what the 8080 and Z80 *are* and why comparing to them is meaningful. They were the dominant 8-bit CPUs of the era; Steil's audience grew up on them. You don't need to know them — just know that the SM83 borrowed from both and invented a few things.

**Focus on:** the opcode table visualization. Steil shows the instruction set as a colored grid, and it's the single best illustration of §4.4's point that the table has *structure*. This slide alone is worth the watch.

**Don't worry about:** the exhaustive comparison of which specific Z80 instructions are missing. It's trivia unless you're writing a disassembler.

### Memory map

**You should already understand:** all of Part 5.

**Hidden assumption:** that memory-mapped I/O needs no explanation.

**Focus on:** how he frames the map as *regions with owners*, which is the same framing as §5.13. If your model matches his, you're in good shape.

### Graphics — the largest and best section

**You should already understand:** all of Part 7. Tiles, maps, the two addressing modes, scrolling, the window, sprites, the 10-per-line limit, priority, palettes, PPU modes.

**Hidden assumptions, which this primer has now covered:**
- That you know *why* tiles exist (the framebuffer arithmetic in §7.1). Steil states that the Game Boy is tile-based and moves on. He doesn't do the cost calculation. Without it, "tiles" seems like an arbitrary choice rather than an economic necessity.
- That you understand the 256×256 map with a 160×144 viewport, and that scrolling moves the viewport. This is stated quickly and is fundamental to everything after it.
- That sprite coordinates are offset. He mentions it in passing.

**Focus on:**
- The visual demonstrations of scrolling and wraparound. Seeing it animated is worth more than my ASCII diagrams.
- The section on how sprites are composited and the per-scanline limit.
- Any discussion of the pixel pipeline. Steil's explanation of the FIFO/fetcher is one of the clearest available.

**This section will reward re-watching more than any other.**

### Sound

**You should already understand:** the Interlude — four channels, the modifiers, why generation beats sampling.

**Hidden assumption:** basic audio-synthesis vocabulary (duty cycle, envelope, LFSR). If any of those are unfamiliar, the Interlude's table covers what they do; you don't need the DSP theory.

**Focus on:** the high-level architecture and the demonstrations of what the channels sound like.

**Don't worry about:** register-level detail. Come back when you implement audio.

### Joypad, timers, interrupts

**You should already understand:** §9.6 (the matrix and active-low logic), §8.3 (the timer), all of Part 9.

**Focus on:** the joypad matrix explanation, which is genuinely non-obvious and which Steil covers well.

### The boot process and the Nintendo logo

**You should already understand:** §5.2 — the logo comparison, the trademark-as-copy-protection story, and the `$FF50` self-unmapping trick.

**This is one of the most entertaining parts of the talk** and you now have the full context to enjoy it. Watch for the discussion of how the logo data is *also* used as graphics data during the boot animation — a genuinely clever piece of dual-purposing.

### Cartridges and MBCs

**You should already understand:** all of Part 6.

**Hidden assumption:** that you understand address translation — that the mapper takes a 16-bit address and produces a wider one.

**Focus on:** the variety of what publishers put in cartridges over the years. It reinforces §6.3's point that the Game Boy's expandability lived in the cart.

### Game Boy Color and Super Game Boy

**You should already understand:** nothing specific — this is new material.

**How to approach:** **you can safely skip this on a first watch.** It's about extensions to the base system. Come back after your DMG emulator works. Just note that the GBC adds: double CPU speed, more VRAM banks, more WRAM banks, real color palettes, and a general-purpose DMA. It's the same machine with more of everything.

### Demos, tricks, and hardware abuse

**You should already understand:** why mid-frame register writes work (§7.6), why STAT interrupts enable them (§9.5), and what "cycle accuracy" is protecting (§8.6).

**This is the payoff section**, and it's where the talk is most fun. Steil shows demos doing things the hardware was never designed to do — more colors than exist, effects that require register writes at exact cycles.

**Focus on:** understanding *why* each trick works in terms of what you now know. Every trick is an exploitation of something in this primer: the PPU reading registers mid-scanline, the FIFO's timing, the palette indirection, the sprite-per-line limit.

**Do not try to support these demos in your emulator.** They're the hardest possible test cases. Appreciate them, and let them motivate you later.

## 12.4 The single biggest mental shift the talk assumes

Steil's whole framing assumes you've internalized this:

> **The Game Boy is not a CPU that draws pictures. It's several processors sharing a bus and a clock, and software's job is to coordinate them.**

If you finish this primer with only that, the talk will land. Everything he shows — the timing tricks, the demos, the quirks — is a consequence of *coordination between independent components*.

---
---

# Appendix A: A Suggested Build Order

You asked to understand the hardware before implementing, and this primer honored that. But here's the road ahead, so you know where you're going.

```
   ┌─────────────────────────────────────────────────────────┐
   │ MILESTONE 1: The bus and the CPU skeleton               │
   │  • ROM loader (no MBC — Tetris is 32 KB)                │
   │  • read8/write8 with proper region routing              │
   │  • register file, flags                                 │
   │  • a handful of opcodes: NOP, LD, JP                    │
   │  • Test: Blargg cpu_instrs #6 (LD r,r)                  │
   ├─────────────────────────────────────────────────────────┤
   │ MILESTONE 2: The full instruction set                    │
   │  • all 256 + 256 CB opcodes                             │
   │  • correct cycle counts (two columns for conditionals)   │
   │  • interrupts                                            │
   │  • Test: ALL of Blargg's cpu_instrs. Do not proceed      │
   │    until all 11 pass. Seriously.                        │
   ├─────────────────────────────────────────────────────────┤
   │ MILESTONE 3: Video                                       │
   │  • PPU mode state machine, LY, STAT                      │
   │  • scanline renderer: background only                    │
   │  • then: window                                          │
   │  • then: sprites, with correct priority                  │
   │  • Test: Tetris title screen. Then dmg-acid2.            │
   ├─────────────────────────────────────────────────────────┤
   │ MILESTONE 4: Playable                                    │
   │  • joypad ($FF00 matrix, active-low)                     │
   │  • OAM DMA (instant version is fine)                     │
   │  • timer                                                 │
   │  • frame pacing                                          │
   │  ★ YOU CAN NOW PLAY TETRIS. This is the moment.         │
   ├─────────────────────────────────────────────────────────┤
   │ MILESTONE 5: Real games                                  │
   │  • MBC1, then MBC3, then MBC5                            │
   │  • battery-backed save files                             │
   │  ★ Pokémon, Zelda, Mario Land all run.                  │
   ├─────────────────────────────────────────────────────────┤
   │ MILESTONE 6: Accuracy                                    │
   │  • VRAM/OAM access blocking                              │
   │  • accurate DMA timing                                   │
   │  • timer edge cases, HALT bug                            │
   │  • Test: Blargg's other suites, Mooneye                  │
   ├─────────────────────────────────────────────────────────┤
   │ MILESTONE 7: Audio                                       │
   │  • channels 1, 2, 4, then 3                              │
   │  • sync to the audio buffer                              │
   ├─────────────────────────────────────────────────────────┤
   │ MILESTONE 8: If you're still having fun                  │
   │  • M-cycle stepping, pixel FIFO                          │
   │  • Game Boy Color                                        │
   │  • a debugger (honestly, build this at milestone 2)      │
   └─────────────────────────────────────────────────────────┘
```

**Two pieces of advice that matter more than they sound:**

**Build a debugger early.** Step, breakpoint, disassemble, dump memory, view VRAM as tiles. You will spend more time debugging than writing, and a debugger turns "the screen is black" from an unsolvable mystery into a ten-minute investigation. Build it at milestone 2, not milestone 8.

**Get a known-good log.** Emulators like BGB can produce a trace of CPU state per instruction. Run the same ROM in yours, diff the logs, and the first divergent line is your bug. This technique will find in five minutes what eyeballing finds in five days.

---

# Appendix B: Glossary

| Term | Definition |
|---|---|
| **APU** | Audio Processing Unit — the sound hardware |
| **Bank** | A 16 KB chunk of cartridge ROM (or 8 KB of cart RAM) |
| **Bus** | Shared wires connecting chips: address, data, control |
| **BGP / OBP0 / OBP1** | Background and sprite palette registers |
| **Bus contention** | Two components wanting the bus at once; one must wait |
| **CB prefix** | Opcode `$CB`, meaning "next byte is from the second table" |
| **DMA** | Direct Memory Access — hardware-driven bulk copy |
| **DIV** | The free-running divider register at `$FF04` |
| **DMG** | Dot Matrix Game — the original Game Boy |
| **Dot** | One T-cycle, in PPU terminology |
| **Echo RAM** | The accidental mirror of WRAM at `$E000`–`$FDFF` |
| **FIFO** | First-In-First-Out queue; the PPU's pixel buffer |
| **HBlank** | The idle period after each scanline (PPU Mode 0) |
| **HRAM** | High RAM, `$FF80`–`$FFFE`, inside the CPU chip |
| **IE / IF / IME** | Interrupt Enable / Flag / Master Enable |
| **LCDC** | LCD Control register, `$FF40` |
| **LY / LYC** | Current scanline / scanline compare value |
| **M-cycle** | Machine cycle = 4 T-cycles = one bus access |
| **MBC** | Memory Bank Controller — the cartridge mapper chip |
| **Mode 0/1/2/3** | HBlank / VBlank / OAM scan / Drawing |
| **OAM** | Object Attribute Memory — the 40-sprite table |
| **Object / OBJ** | Nintendo's word for a sprite |
| **PPU** | Picture Processing Unit — the graphics hardware |
| **SCX / SCY** | Background scroll registers |
| **SM83** | The CPU core inside the LR35902 |
| **STAT** | LCD status register, `$FF41`, and its interrupt |
| **T-cycle** | One clock tick; 1/4,194,304 second |
| **Tile** | An 8×8 pixel pattern, 16 bytes, 2 bits per pixel |
| **Tile map** | A 32×32 grid of tile indices |
| **VBlank** | The idle period after each frame (PPU Mode 1) |
| **VRAM** | Video RAM, `$8000`–`$9FFF` |
| **Window** | The second, non-scrolling background layer |
| **WRAM** | Work RAM, `$C000`–`$DFFF` |
| **WX / WY** | Window position registers |

---

# Appendix C: Test ROMs and What They're Actually Testing

Test ROMs are your regression suite, and they'll tell you things no game will.

| Suite | What it checks | When to use it |
|---|---|---|
| **Blargg `cpu_instrs`** | All instruction behavior and flags | Milestone 2. Non-negotiable. |
| **Blargg `instr_timing`** | Cycle counts per instruction | After cpu_instrs passes |
| **Blargg `mem_timing`** | *When* within an instruction memory is accessed | Milestone 6 |
| **Blargg `halt_bug`** | The HALT bug specifically | Milestone 6 |
| **Blargg `dmg_sound`** | APU behavior | Milestone 7 |
| **dmg-acid2** | PPU rendering correctness — renders a face; wrong output shows visibly which feature is broken | Milestone 3 |
| **Mooneye Test Suite** | Extremely precise timing, MBC behavior, PPU edge cases | Milestone 6+ |

**dmg-acid2 deserves special mention.** It renders a cartoon face, and each facial feature tests a different behavior — sprite priority, the window line counter, the 10-sprite limit, palette handling. When something's wrong, the face is visibly deformed in a specific way, and the accompanying documentation tells you which bug causes which deformity. It's the friendliest debugging experience in the whole ecosystem.

**A note on how to fail well:** when a test ROM fails, resist the urge to guess. Use your debugger to find the exact instruction where behavior diverges from a reference emulator's log. Guessing at flag logic is how people spend three weeks on a one-line bug.

---

# A Closing Note

You now have something that most people building their first Game Boy emulator don't: **a model of the machine as a system of cooperating parts, and an understanding of why each part is shaped the way it is.**

The specific facts in this document — that `$FF44` is `LY`, that a tile is 16 bytes, that a frame is 70,224 cycles — you'll look up a hundred times and eventually memorize by accident. Those aren't the valuable part.

The valuable part is the *reflex*: when you encounter something strange, asking **"what constraint caused this?"** rather than "why is this so arbitrary?" That reflex will carry you through Pan Docs, through the 33C3 talk, and through every confusing evening of staring at a black screen wondering why your emulator won't boot.

The Game Boy is a small machine, and it is *knowable*. Twelve people designed it. You can hold all of it in your head at once, which is a rare and wonderful thing in computing — and it's precisely why so many people find that emulating it changes how they understand computers in general.

Go watch the talk. It'll make sense now.
