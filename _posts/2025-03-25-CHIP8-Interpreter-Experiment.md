## An adventure in emulating with CHIP-8
### Special Thanks
This interpreter was built with some references to these sources:

-[Austin Morlan's post](https://austinmorlan.com/posts/chip8_emulator/)

-[Tobias' guide](https://tobiasvl.github.io/blog/write-a-chip-8-emulator/)

-[Cowgod's CHIP8 documentation](http://devernay.free.fr/hacks/chip8/C8TECH10.HTM)

-r/EmuDev subreddit for all additionnal info about input timing and clock cycles.

---

### Introduction

With my interest for computer architecture, I've wanted to do a small project that would reinforce my knowledge of CPU innerworkings.
I had the idea to get into making an emulator while playing Final Fantasy VI on Snes9x for 3DS. I was always fascinated by how they could read the actual game ROM and perfectly translate it to an experience on modern hardware.
From the r/EmuDev subreddit, it seemed like Chip-8 was the most beginner-friendly, while still having the challenging parts of bitwise operations, clock cycles and making it all fit within the terminal for my concept.
It isn't really an emulator per se, more an interpreter of the language, since Chip-8 isn't a real physical console/computer. It was always "emulated" while used. 
It was supposed to be a couple hundred lines of code and do-able in a couple days so it seemed perfect for my busy schedule. So about a month ago from this post I started building it.

---

### Building...

Aiming for simplicity and minimalism, the goal was to make my Chip-8 emulator work entirely within the Linux terminal. 
I had the idea to make it cross-platform with Windows, but that proved to be a small pain and wasn't worth it for the scale of this program as a personal project. 
The C language was chosen since I'm currently used to coding embedded systems, some OS stuff (threads, clocks, files, etc.) on Linux for my class and I've used it a lot in the past while learning
the fundamentals of programming. It seemed perfect to code an emulator with a low-level language to really understand the architecture.

The recommended way to begin is to be able to show the IBM Logo using the .ch8 ROM. Only a couple instructions are needed such as moving variables in registers and the draw instruction for our display "memory".
The ROM loading and display also have to be implemented. The struct I used, based on Austin's post, contained all registers and memory needed for the fake CPU:

#### The Chip-8 structure: 

```c
#define START_ADDRESS 0x200  // offset for ram
#define FONTSET_ADDRESS 0x50 // reserved mem space

uint8_t fontset[80] = {
    0xF0, 0x90, 0x90, 0x90, 0xF0, // 0
    0x20, 0x60, 0x20, 0x20, 0x70, // 1
    0xF0, 0x10, 0xF0, 0x80, 0xF0, // 2
    0xF0, 0x10, 0xF0, 0x10, 0xF0, // 3
    0x90, 0x90, 0xF0, 0x10, 0x10, // 4
    0xF0, 0x80, 0xF0, 0x10, 0xF0, // 5
    0xF0, 0x80, 0xF0, 0x90, 0xF0, // 6
    0xF0, 0x10, 0x20, 0x40, 0x40, // 7
    0xF0, 0x90, 0xF0, 0x90, 0xF0, // 8
    0xF0, 0x90, 0xF0, 0x10, 0xF0, // 9
    0xF0, 0x90, 0xF0, 0x90, 0x90, // A
    0xE0, 0x90, 0xE0, 0x90, 0xE0, // B
    0xF0, 0x80, 0x80, 0x80, 0xF0, // C
    0xE0, 0x90, 0x90, 0x90, 0xE0, // D
    0xF0, 0x80, 0xF0, 0x80, 0xF0, // E
    0xF0, 0x80, 0xF0, 0x80, 0x80  // F

};

/**
 * struct to emulate a Chip8 computer.
 */
typedef struct chip_8_ {
  // 4 kb of mem
  uint8_t mem[4096];
  // 16 8 bit regs
  uint8_t Vregs[16];   // args V0 to VF
  uint16_t Ireg;       // 16 bits to store mem addresses
  uint8_t delay_timer; // 8 bits regs for timers
  uint8_t sound_timer;

  uint16_t PC; // initialize at 0x200 in constructor, program counter

  // The stack is an array of 16 16-bit values, used to store the address that
  // the interpreter
  //...should return to when finished with a subroutine.
  // Chip-8 allows for up to 16 levels of nested subroutines.
  uint8_t SP;         // stack pointer
  uint16_t stack[16]; // stack memory for instruct

  uint8_t keypad[16]; // keypad size (16 key hexadecimal)
  // Array for display, 2d grid flattened in 1d
  uint32_t video[64 * 32]; // 64 columns (x) and 32 rows(y)

  uint16_t opcode;
  uint8_t keyboard[16];

} Chip8;

```

#### Drawing on screen

My idea for drawing on the terminal was to render each pixel (monochromatic) as a block character. Sure this isn't ideal since there arent perfect squares I can use, but it was good enough
to be able to see something on screen and understand it. The only noticeable visual bugs come from the tetris blocks when rotated. I chose to draw the pixels one by one, with a line skip when needed. Printing full lines could be
a possible optimization, but I haven't noticed a big performance loss using this method. I'm also using `sytem("clear")` in the `clear_console()` func. 

#### Draw instruction :
```c
// INSTRUCTION DXYN
void DRAW(struct chip_8_ *chip8, uint16_t opcode) {
  const uint8_t Vx = (opcode & 0x0F00u) >> 8u;
  const uint8_t Vy = (opcode & 0x00F0u) >> 4u;

  const uint8_t x = chip8->Vregs[Vx] % 64; // Wrap around if x > 63
  const uint8_t y = chip8->Vregs[Vy] % 32; // Wrap around if y > 31

  const uint8_t n = opcode & 0x000Fu;
  chip8->Vregs[0xFu] = 0;

  for (int i = 0; i < n; i++) {
    const uint8_t data = chip8->mem[chip8->Ireg + i];
    for (int j = 0; j < 8; j++) {
      if (data & (0x80u >> j)) {
        const int screenX = (x + j) % 64;
        const int screenY = (y + i) % 32;

        const int pixelIndex = screenY * 64 + screenX;
        if (chip8->video[pixelIndex] == 1) {
          // If the pixel was already set, a collision occurred
          chip8->Vregs[0xFu] = 1;
        }
        chip8->video[pixelIndex] ^= 1;
      }
    }
  }

  // chip8->Vregs[15] = 1; if collision
}
```

#### Printing on terminal :

```c
void draw_console(
    const struct chip_8_
        *chip8) { 
  clear_console();
  for (unsigned int i = 0; i < 64 * 32; ++i) {
    // Print a block character for set pixels, otherwise a space
    if (chip8->video[i] == 1) {
      if (strcmp(OS, "Windows") == 0) { //leftover Windows implementation
        printf("#");
      } else {
        printf("█");
      }
      // other char options: █ ■ ▮ ▓
    } else {
      printf(" ");
    }

    // Print a newline after every 64 pixels (end of a row)
    if ((i + 1) % 64 == 0) {
      printf("\n");
    }
  }
  fflush(stdout); //fix visual bugs
}
```

#### Interpreting instructions

The way I chose to do this was with a simple (long) switch/case. We take the current opcode, check the first 4 bits and compare them to values. 
If multiple instructions have the same value for the first hexadecimal number, we simply check the next one (or any relevant one).

```c
void exec_instruction(struct chip_8_ *chip8, const uint16_t opcode) {
  switch (opcode >> 12 & 0xFu) {
  case 0x0: {
    if (opcode == 0x00E0u) {
      clear_screen(chip8);
    } else if (opcode == 0x00EEu) {
      // RET
      // The interpreter sets the program counter to the address at the top of
      // the stack, then subtracts 1 from the stack pointer.
      --chip8->SP;
      chip8->PC = chip8->stack[chip8->SP];
    } else {
      // SYS addr, ignored by modern chip8 computer
    }
    break;
  }
  case 0x1: {
    // 1NNN
    jump(chip8, opcode);
    break;
  }
//rest of instructions omitted, see source code
```

#### Clock cycles and timers (a fun part of emulation)

A typical CPU cycle fetches the current opcode, increments the program counter to the next and then executes the fetched opcode.
The function `chip8_clock_cycle()` called in our main was used to do this.

```c
void chip8_clock_cycle(struct chip_8_ *chip8) {
  // Fetch opcode (16 bits) using program counter
  const uint16_t opcode =
      chip8->mem[chip8->PC] << 8 | chip8->mem[chip8->PC + 1];
  chip8->PC += 2;
  // Exec the code
  exec_instruction(chip8, opcode);
}
```
Since the display historically ran at around 60hz and the CPU at 500hz (still depends on who you ask), we are aiming for about 8-9 instructions per frame.
For my solution, I chose to use CPU clock timers in `<time.h>`. Each time a frame or instruction is executed, we start the corresponding clock. With some clever time differences and approximations, we can get the following loops.
A difference of 2000 microseconds is about 500Hz and a difference of 16667 (16666.66...) microsecondes is about 500Hz.

```c
// timers used
clock_t frame_init = clock();
clock_t frame_end = clock();
clock_t cycle_init = clock();
clock_t cycle_end = clock();
```
Using the above: 

```c
 if (cycle_end - cycle_init > 2000) {
      // Check if input is available
      chip8_clock_cycle(chip8);
      cycle_init = clock();
      // Clear the keyboard state for the next cycle
    }
    cycle_end = clock();
    frame_end = clock();

    // approx 1 microsecond per clock unit = 60hz here
    if (frame_end - frame_init > 16667) {

      if (chip8->delay_timer > 0) {
        --chip8->delay_timer;
      }

      // Decrement the sound timer if it's been set
      if (chip8->sound_timer > 0) {
        --chip8->sound_timer;
      }
      // sound play, seems to work
      if (chip8->sound_timer > 0) {
        if (strcmp(OS, "Linux") == 0) {
          FILE *fd = fopen("/dev/tty", "w");
          if (fd != NULL) {
            fwrite("\a", sizeof(char), 1, fd); //"prints" a motherboard beep
            // Needs the terminal to have sound alarms enabled. A flash alarm
            // will not work.
            fclose(fd);
          }
        } else if (strcmp(OS, "Windows") == 0) { //leftover code
          // windows Beep()sound func
        } else
          printf("\a");
      }
      frame_end = clock();
      frame_init = clock();
      memset(chip8->keyboard, 0, sizeof(chip8->keyboard));
      draw_console(chip8);
    }

    // needs to be every 60 s (adjust whole while loop with clock()) ?
  }
```


The sound and delay timers are also modified every 60hz, so I chose to put them in the same clock as the frame draw. 
Sound is handled by "printing" an alarm character to the terminal. Depending on the terminal used and its settings, it could also be a flash or nothing. 
If alarm sounds are enabled, it will emit a hardware (I assume motherboard in most cases) beep.

---

### Terminal black magic

At this step, I was really happy with my results (and annoyed by what was left to do). I had to make sure to get an input in the terminal, NOT print the character to the screen and make sure I didn't need to make the user press ENTER everytime a char was input.
I found out about the `<termios.h>` and `<ncurses.h>` libraries.

The terminal is set to non-canonical mode and then it gets every character that is input. It then compares it with a direct value using switch case and sets the corresponding case in the input array to 1.

#### How are inputs managed? 

This step was hard to do since there are no official ways of handling inputs. Everyone just seems to be doing it to make it work. I had to read a lot of
confusing stuff about canonical and non-canonical terminal modes and functions I'd never used. It's pretty janky and at this point I wondered if using SDL would
be simpler (..it would). I still pushed through.

I chose to map the keyboard the way most interpreters do: keys 1 to 4 horizontally, 1 to z vertically.
A pressed state would be indicated by a value of 1, not pressed 0.
With every frame, the states are reset.

chip8 keyboard:
    1 2 3 C
    4 5 6 D
    7 8 9 E
    A 0 B F

Mapped to our QWERTY keyboard:
 1 2 3 4
  Q W E R
   A S D F
    Z X C V

---

### The test ROMs

The CHIP-8 Test Suite available [here](https://github.com/Timendus/chip8-test-suite) provides a series of basic functionality tests. Our interpreter passes all the basic functionality tests and the quirks tests. Tetris and other game ROMs are also playable at decent speed.

---

### The next step!

Having learned the basic framework for emulator development, I want to try making a C++ 6502 emulator leading to a NES Emulator. There's also the possiblity of a 6510 emulator to eventually simulate the Commodore 64 environment.

### Thanks for reading!

