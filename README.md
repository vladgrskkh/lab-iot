# Lab 1 — Variant 1: running lights

Target: STM32F103C6T6A, STM32Cube HAL. Three LEDs turn on one at a time in a repeating sequence.

## CubeMX configuration

1. Create a project for **STM32F103C6T6A**. Enable RCC > High Speed Clock (HSE) > Crystal/Ceramic Resonator, and set the HSE input to 8 MHz in Clock Configuration, as specified by the lab. Keep a valid clock configuration; CubeMX reports clock errors in red.
2. Under SYS, select Debug > Serial Wire. This reserves PA13 and PA14 for SWD.
3. Set PA1, PA2 and PA3 to GPIO_Output. In GPIO settings set each to Output Push Pull, No pull-up and no pull-down, Low speed, and initial output level Low.
4. In Project Manager select STM32CubeIDE as the toolchain and generate the project. Open it in STM32CubeIDE. Add the code below to the indicated sections of Core/Src/main.c. Keep the generated initialization calls, including MX_GPIO_Init().

## Code

Add inside `/* USER CODE BEGIN 0 */`:

```c
static const uint16_t led_pins[] = {
    GPIO_PIN_1, GPIO_PIN_2, GPIO_PIN_3
};
```

Replace the body of the generated `while (1)` loop, keeping its USER CODE markers, with:

```c
  /* USER CODE BEGIN WHILE */
  while (1)
  {
    for (unsigned int i = 0; i < sizeof(led_pins) / sizeof(led_pins[0]); ++i)
    {
      HAL_GPIO_WritePin(GPIOA, led_pins[i], GPIO_PIN_SET);
      HAL_Delay(250);
      HAL_GPIO_WritePin(GPIOA, led_pins[i], GPIO_PIN_RESET);
    }
    /* USER CODE END WHILE */

    /* USER CODE BEGIN 3 */
  }
  /* USER CODE END 3 */
```

The generated `main.c` already includes `main.h`, which includes the HAL definitions. Put all custom code within USER CODE sections so it survives code regeneration.

## Connections

For each of PA1–PA3: GPIO pin → 330 Ω resistor → LED anode (long leg); LED cathode (short leg) → GND. Each LED needs its own resistor. This arrangement turns the LED on when the pin is high. Use 3.3 V-compatible LEDs and do not connect an LED directly to a GPIO without a current-limiting resistor. The three GPIO pins must share ground with the board.

For actual debugging, connect the ST-Link SWDIO to PA13/SWDIO, SWCLK to PA14/SWCLK, and GND to GND; connect the board's 3.3 V reference/power according to the specific programmer and board documentation. Do not apply 5 V to a 3.3 V pin. The lab assumes an 8 MHz external crystal; hardware must actually have it for the HSE configuration to run.

## How it works

CubeMX generates `MX_GPIO_Init()` to configure PA1–PA3 as push-pull outputs. The `led_pins` array holds their bit masks. The loop sets one pin high, waits 250 ms, then clears it before moving to the next pin. The sequence repeats every 750 ms. Only one LED is on at a time, assuming the wiring above. `HAL_Delay()` uses the HAL time base (normally SysTick); it is a blocking delay, which is appropriate for this simple task.

Hardware modules: GPIO port A controls the LEDs; SysTick supplies the HAL millisecond time base; RCC configures the clock; SWD allows programming and debugging. GPIO is used because only digital on/off control is needed.

## Verification

Build the project in STM32CubeIDE. With hardware, flash it and check that the LEDs light PA1 → PA2 → PA3, each for approximately 250 ms. Without hardware this verifies compilation only; physical wiring, clock startup and timing remain untested. In Debug, breakpoints on the write calls can be used to step through the sequence.

## Generated project notes

The provided project uses the internal 8 MHz HSI as its system clock. HSE pins are reserved, but the generated clock setup does not select HSE; set the system clock source to HSE in Clock Configuration and regenerate if your instructor requires the external crystal setup from the handout. PA0 is configured as analog and is not used. The program has been added to the generated main.c.
