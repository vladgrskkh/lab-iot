# Understanding the STM32 lab

This guide explains the running-lights project from the beginning: the hardware,
the C program, generated code, and how the project becomes firmware.

## 1. The big picture

Your Mac and the STM32 are two different computers.

- Your Mac runs STM32CubeMX and STM32CubeIDE.
- The STM32 is a small computer designed to run embedded programs.
- You write the program in C on the Mac.
- The development tools translate the C code into machine instructions for the
  STM32's ARM processor.

The complete path looks like this:

```text
CubeMX configuration (.ioc)
          |
          v
Generated initialization code + your C code
          |
          v
Compiler: C -> machine-code pieces (.o)
          |
          v
Linker: pieces -> complete firmware (.elf)
          |
          v
ST-Link writes the firmware into STM32 flash
          |
          v
STM32 executes it and controls physical pins
```

You have completed the process through **building**. Without hardware, the
programming and physical testing stages have not been performed.

## 2. What is an STM32?

An STM32 is a **microcontroller**: a small computer built into a single chip.
Your project targets an **STM32F103C6**, which contains:

- An **ARM Cortex-M3 CPU**, which executes instructions.
- **Flash memory**, which stores the program when power is removed.
- **RAM**, which holds temporary data while the program runs.
- **GPIO ports**, which let the program read and control physical pins.
- A **clock system**, which supplies timing signals.
- Other peripherals, such as timers, serial interfaces, and analog-to-digital
  converters.

Unlike a typical desktop application, the program normally runs directly on
this microcontroller, without macOS or Windows underneath it. This is often
called *bare-metal programming*. In this project, ST's HAL library helps manage
the hardware.

A development board adds components around the chip: power connections,
headers, often a clock crystal, reset circuitry, and LEDs. The microcontroller
is the chip; a Blue Pill is an example of a board built around an STM32.

## 3. Pins and GPIO

A physical chip pin can often serve several functions. It might be usable as a
digital input, a digital output, or part of a peripheral interface.

**GPIO** means *general-purpose input/output*. A pin can be configured as:

- **Input:** the microcontroller reads whether its voltage is low or high.
- **Output:** the microcontroller drives it low or high.
- Another supported mode, such as analog or a peripheral-controlled function.

Pins are grouped into ports. `PA1` means **port A, pin 1**. This project uses
PA1, PA2, and PA3 as outputs.

The proposed LED wiring is:

```text
PA1 --- resistor --- LED --- GND
PA2 --- resistor --- LED --- GND
PA3 --- resistor --- LED --- GND
```

Connect each pin through its own 330-ohm resistor to an LED anode; connect the
LED cathode to ground. When an output is high, current can flow through the
resistor and LED to ground, lighting the LED. When the output is low, the LED
turns off. The resistor limits current; do not connect an LED directly to a GPIO
pin without one.

This behavior depends on the wiring. An LED wired to the supply instead of
ground can be **active-low**, meaning a low output turns it on. Do not assume
that every built-in board LED behaves like the external LEDs above.

### How can software change a voltage?

The microcontroller has hardware **registers**: small storage locations used to
control peripherals. They are accessible at specific memory addresses. Writing
a register changes a hardware setting.

The program calls a HAL function:

```c
HAL_GPIO_WritePin(GPIOA, GPIO_PIN_1, GPIO_PIN_SET);
```

The HAL implementation ultimately writes the appropriate GPIO register. The
GPIO peripheral then drives PA1 high. A line of C has therefore become an
electrical signal on a physical pin.

## 4. What CubeMX generates

CubeMX is a **configuration and code-generation tool**. You describe how you
want to use the microcontroller, and it generates initialization code.

The project configuration is stored in `solution/solution.ioc`. It records
choices such as:

- The target STM32.
- Which pins are GPIO outputs.
- Which pins are reserved for debugging.
- Clock settings.
- The target development environment.

When you generate code, CubeMX uses these settings to create or update project
files. Because PA1 through PA3 are outputs, it generated code in
`MX_GPIO_Init()` to initialize them.

CubeMX does not know you want running lights. It configures the pins, but you
write the logic that turns them on and off in sequence.

### Generated code and user code

Open `solution/Core/Src/main.c`. You will see comments such as:

```c
/* USER CODE BEGIN 0 */

/* USER CODE END 0 */
```

These markers are understood by CubeMX. We placed our additions inside user-code
sections so CubeMX can preserve them when the project is regenerated.

- **CubeMX-managed sections** contain code based on your configuration.
- **USER CODE sections** contain code you write and maintain.

Changes outside user-code sections may be overwritten during regeneration.

## 5. C source code

C is a programming language for describing instructions and data. A human writes
text in `.c` and `.h` files; a compiler translates it into machine code for a
specific processor.

Consider this declaration from the project:

```c
static const uint16_t led_pins[] = {
    GPIO_PIN_1, GPIO_PIN_2, GPIO_PIN_3
};
```

It creates an **array** named `led_pins`. An array stores multiple values in
order:

```text
Index:       0           1           2
Value:   GPIO_PIN_1  GPIO_PIN_2  GPIO_PIN_3
```

C array indexing starts at **zero**:

```c
led_pins[0]  // GPIO_PIN_1
led_pins[1]  // GPIO_PIN_2
led_pins[2]  // GPIO_PIN_3
```

The declaration's parts mean:

- `uint16_t` is an unsigned integer type that is 16 bits wide.
- `led_pins[]` declares an array. The compiler determines its length from the
  three values supplied.
- `const` means the program should not change the values.
- `static`, at file scope, limits the name's visibility to this source file.
- `GPIO_PIN_1`, `GPIO_PIN_2`, and `GPIO_PIN_3` are named constants defined by
  the HAL. They represent pin masks, not physical voltages.
- The semicolon `;` ends the declaration.

### Functions

A function is a named unit of code that can be called. For example:

```c
HAL_Delay(250);
```

This calls `HAL_Delay` with the argument `250`. With the default HAL time base
used here, it requests a delay of approximately 250 milliseconds.

Another function call is:

```c
HAL_GPIO_WritePin(GPIOA, led_pins[i], GPIO_PIN_SET);
```

Its arguments tell HAL:

1. `GPIOA`: use GPIO port A.
2. `led_pins[i]`: select the current pin.
3. `GPIO_PIN_SET`: drive it high.

Using `GPIO_PIN_RESET` instead drives the pin low.

### The loop

The program contains:

```c
for (unsigned int i = 0;
     i < sizeof(led_pins) / sizeof(led_pins[0]);
     ++i)
{
    HAL_GPIO_WritePin(GPIOA, led_pins[i], GPIO_PIN_SET);
    HAL_Delay(250);
    HAL_GPIO_WritePin(GPIOA, led_pins[i], GPIO_PIN_RESET);
}
```

A `for` loop has three parts:

```c
for (initialization; condition; update)
```

In this case:

1. `unsigned int i = 0` creates a counter starting at zero.
2. The condition checks whether `i` is smaller than the array length.
3. `++i` increases `i` by one after each iteration.

This expression calculates the number of elements:

```c
sizeof(led_pins) / sizeof(led_pins[0])
```

`sizeof(led_pins)` gives the size of the whole array in bytes.
`sizeof(led_pins[0])` gives the size of one element. Dividing gives **3**.

The braces `{ ... }` contain the instructions to repeat:

| Counter | Selected pin | What happens |
| --- | --- | --- |
| 0 | PA1 | On for 250 ms, then off |
| 1 | PA2 | On for 250 ms, then off |
| 2 | PA3 | On for 250 ms, then off |

The `for` loop is inside:

```c
while (1)
{
    // Running-lights loop
}
```

In C, `1` is true. Since the condition never becomes false, the `while` loop
repeats indefinitely. Embedded firmware normally keeps running until the device
is reset or powered off.

## 6. The `main()` function

`main()` is the main application function. Startup code runs first, then calls
`main()`.

The generated function has this general structure:

```c
int main(void)
{
    HAL_Init();
    SystemClock_Config();
    MX_GPIO_Init();

    while (1)
    {
        // Our running-lights code
    }
}
```

The important steps are:

1. `HAL_Init()` initializes HAL facilities, including the default time base.
2. `SystemClock_Config()` configures the system clock.
3. `MX_GPIO_Init()` configures the GPIO pins.
4. The infinite loop runs the application.

The order matters: GPIO must be configured before the program controls the
LEDs.

On a desktop, returning from `main()` normally ends the process. There is no
desktop operating system waiting for this firmware to end, so the application
normally stays in its main loop.

## 7. Headers and `#include`

A **source file** usually has a `.c` extension. It often contains function
implementations: the instructions that make functions work.

A **header file**, usually ending in `.h`, commonly contains declarations,
types, constants, and macros needed by other files.

At the top of `main.c` is:

```c
#include "main.h"
```

Before compilation, the **preprocessor** processes directives beginning with
`#`. For `#include`, it effectively inserts the referenced header's contents
into the source being processed.

Through `main.h` and the headers it includes, the compiler learns about things
used in `main.c`, such as:

- What `uint16_t` means.
- What `GPIOA` represents.
- The definition of `GPIO_PIN_SET`.
- The declaration of `HAL_GPIO_WritePin()`.

A simplified function declaration might look like:

```c
void some_function(int value);
```

This tells the compiler the function's name, argument type, and that it returns
no value (`void`). It does not necessarily contain the implementation.

The implementation may live in another `.c` file:

```c
void some_function(int value)
{
    // Instructions go here
}
```

This separation lets multiple source files use the same function without copying
its implementation into each file.

## 8. HAL and supporting code

**HAL** stands for *Hardware Abstraction Layer*. It is ST-provided C code for
working with STM32 peripherals.

Instead of directly writing a GPIO register, you can call:

```c
HAL_GPIO_WritePin(...);
```

The function handles the relevant register operation. HAL makes common
operations easier to read, although it does not make every detail of different
microcontrollers identical.

Your project contains HAL files under `Drivers/STM32F1xx_HAL_Driver`. CubeMX
copied the relevant library files into the project.

Other important pieces include:

- **CMSIS:** ARM and device-specific definitions used for processor and
  peripheral access.
- **Startup code:** prepares the system after reset and eventually calls
  `main()`.
- **Linker script:** describes the target's memory layout to the linker.

## 9. Preprocessing, compiling, linking, and building

These terms are related but describe different stages.

### Preprocessing

The preprocessor handles directives such as:

```c
#include "main.h"
```

It also expands macros such as named pin constants. The result is C code with
those directives processed.

### Compilation

The compiler checks C syntax and types, then translates each source file into an
**object file**, typically ending in `.o`:

```text
main.c                -> main.o
stm32f1xx_hal_gpio.c  -> stm32f1xx_hal_gpio.o
```

An object file contains machine code and information about names it defines or
needs. It may not yet be a complete program.

For example, `main.o` calls `HAL_GPIO_WritePin`, but the implementation is in a
HAL source file compiled separately.

The IDE's compiler targets the STM32's ARM architecture. The resulting firmware
is not a normal Mac application.

### Linking

The **linker** combines object files and resolves references between them:

```text
main.o needs HAL_GPIO_WritePin
                |
                v
HAL GPIO object provides HAL_GPIO_WritePin
```

The linker connects the call to its implementation. It also uses the linker
script to place program code and data in the STM32's flash and RAM address
ranges.

An **undefined reference** error usually means the compiler accepted a function
call, but the linker could not find its implementation.

### Building

**Build** is the overall process coordinating preprocessing, compilation,
assembly, and linking. It may also create additional output formats.

The main output is usually an **ELF** file:

```text
source files + libraries
          |
          v
compiler and assembler
          |
          v
object files
          |
          v
linker + linker script
          |
          v
firmware.elf
```

The ELF contains the firmware and can also contain debugging information that
connects machine instructions back to C source lines.

A successful build tells us the tools could translate and link the project. It
does not prove the wiring is correct or that the program behaves as intended on
a real board.

## 10. Flash, RAM, and startup

The STM32 uses different memory regions for different purposes.

**Flash** is non-volatile: it retains contents after power is removed. Firmware
is programmed into flash.

**RAM** is volatile: its contents are lost when power is removed. The program
uses it for temporary data, including local variables and the stack.

When the microcontroller resets, it follows a predefined startup sequence.
Startup code sets up the execution environment, initializes relevant memory, and
calls `main()`.

The **linker script** tells the linker where the target's memory regions are and
how to place the program's sections. This project includes a flash linker script
for the selected STM32.

## 11. Clocks and `HAL_Delay`

Digital logic needs timing signals. The clock system supplies these signals to
the processor and peripherals.

The generated project currently uses the internal **HSI** oscillator as its
system clock at 8 MHz. The lab handout describes an external **8 MHz crystal**,
called HSE. The configuration reserves the HSE pins, but the generated
`SystemClock_Config()` currently selects HSI.

To match the handout, select HSE as the system clock source in CubeMX's **Clock
Configuration** and regenerate. On a physical board, HSE only works as
configured if the required external clock hardware is present.

HAL normally uses **SysTick**, a processor timer, to provide a regular time
base. `HAL_Delay(250)` waits until approximately 250 milliseconds have elapsed.

This is a **blocking delay**: while waiting, the main loop does not perform
other application work. That is fine for this simple exercise. A more complex
device might use timers or elapsed-time checks so it can do other work while
waiting.

Three LEDs times 250 ms gives approximately **750 ms per full cycle**. There is
also a small amount of execution overhead, so it is not a precision timing
measurement.

## 12. ST-Link and SWD

An **ST-Link** is a programmer and debugger that connects your computer to an
STM32.

**SWD** stands for *Serial Wire Debug*. Its connections include:

- **SWDIO:** debug data.
- **SWCLK:** debug clock.
- **GND:** shared electrical reference.

The programmer also needs the correct target-voltage reference and power
arrangement for the specific hardware.

SWD can be used to:

- Program firmware into flash.
- Reset and run the microcontroller.
- Pause execution.
- Inspect registers and memory.
- Set breakpoints and step through instructions.

A **breakpoint** tells the debugger to pause when execution reaches a chosen
location. In this lab, you could place one on the line that turns an LED on,
then inspect or resume execution.

The CubeMX project reserves PA13 and PA14 for SWD. This prevents their accidental
use for another purpose in the project.

## 13. Describing the result

The implementation satisfies the software portion of variant 1: it generates a
repeating sequence on three GPIO outputs.

For a report, describe the result accurately:

> A project was generated for the STM32F103C6 using STM32CubeMX and built
> successfully in STM32CubeIDE. The program sequentially sets PA1, PA2, and PA3
> high for 250 ms each. GPIO port A is used to control the LEDs, SysTick
> supplies the HAL time base, and the clock system provides timing for the
> processor and peripherals. SWD is reserved for programming and debugging.
> Hardware operation was not tested because a board was unavailable.

To follow the program, open `solution/Core/Src/main.c` and read the execution
order: `HAL_Init()` -> clock setup -> GPIO setup -> the infinite loop -> the
three LED operations.
