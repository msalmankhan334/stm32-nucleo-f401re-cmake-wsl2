# STM32F401 Nucleo – FreeRTOS demo (CMake, no IDE)

A small FreeRTOS project for the **NUCLEO-F401RE** (STM32F401RET6, Cortex-M4F, 84 MHz),
generated with STM32CubeMX and built with plain **CMake + Ninja + `arm-none-eabi-gcc`**.
It is developed on **WSL2** with VS Code; no STM32CubeIDE is needed.

## What it does

Three FreeRTOS tasks (CMSIS-RTOS v2 API) run on the board:

| Task | Priority | Job |
|---|---|---|
| `BlinkTask` | Low | Toggles the green LED (LD2, PA5) and posts the toggle count to a queue. |
| `defaultTask` | Normal | Waits on the queue and prints `[ tick ms] LED toggle #N (period X ms)` over UART. It is the only task that calls `printf`. |
| `UserButton` | Normal | Sleeps on a semaphore released by the **B1** button interrupt (PC13, EXTI15_10). After debouncing, each press cycles the blink period 500 → 1000 → 100 → 250 → 500 ms. |

`printf` is routed to **USART2** (PA2/PA3, 115200 8N1), which the on-board ST-LINK exposes as a
virtual COM port. HAL time base is **TIM5**, so FreeRTOS owns SysTick.

## Requirements

- NUCLEO-F401RE board (on-board ST-LINK/V2-1), USB cable
- Ubuntu / WSL2 with:

  ```bash
  sudo apt install gcc-arm-none-eabi binutils-arm-none-eabi libnewlib-arm-none-eabi \
                   libstdc++-arm-none-eabi-newlib cmake ninja-build \
                   openocd gdb-multiarch minicom
  ```

- VS Code with the **C/C++**, **CMake Tools** (optional) and **Cortex-Debug** extensions
  (install them in WSL when using the WSL remote).
- On WSL2, pass the ST-LINK through with [usbipd-win](https://github.com/dorssel/usbipd-win)
  (admin PowerShell on Windows):

  ```powershell
  usbipd list
  usbipd bind --busid <BUSID>
  usbipd attach --wsl --busid <BUSID>
  ```

  Re-run `attach` after replugging the board. Detach it again if you want the COM port on Windows.

## Build

```bash
cmake --preset Debug
cmake --build --preset Debug
# clean:   cmake --build --preset Debug --target clean     (or: rm -rf build)
```

Output: `build/Debug/stm324f4.elf`. If you move or copy the project, delete `build/` first –
CMake caches absolute paths.

## Flash

```bash
openocd -f board/st_nucleo_f4.cfg \
        -c "program build/Debug/stm324f4.elf verify reset exit"
```

A successful flash ends with `Verified OK` and `Resetting Target`. The message
`Unable to match requested speed 2000 kHz, using 1800 kHz` is harmless.

## Debug

In VS Code press **F5** and pick *Debug Nucleo-F401RE (OpenOCD)*. It builds, flashes and stops at
`main()`. (Uses `gdb-multiarch` and OpenOCD; see `.vscode/launch.json`.)

The `.vscode/tasks.json` file also provides **Build**, **Clean**, **Rebuild**, **Flash (OpenOCD)**
and **Size** tasks (Ctrl+Shift+B runs Build).

## Serial output

```bash
minicom -D /dev/ttyACM0 -b 115200     # quit: Ctrl+A, then X   (scrollback: Ctrl+A, B)
```

On Windows the same port shows up as *STMicroelectronics STLink Virtual COM Port*.

## Project layout

```
Core/                 Application code (main.c, interrupts, HAL MSP, FreeRTOS config)
Drivers/              STM32F4xx HAL + CMSIS (from STM32Cube FW_F4 V1.28.3)
Middlewares/          FreeRTOS kernel + CMSIS-RTOS v2 wrapper
cmake/                Toolchain file and CubeMX CMake sources
stm324f4.ioc          CubeMX project (open it to change pins, peripherals, RTOS settings)
STM32F401xx_FLASH.ld  Linker script
.vscode/              IntelliSense, build/flash tasks and debug configuration
```

## Regenerating from CubeMX

Open `stm324f4.ioc`, change what you need, and generate the code. Hand-written code lives inside
the `USER CODE BEGIN/END` blocks and is preserved. Requirement for the button: the
*EXTI line[15:10] interrupt* must stay enabled with preemption priority **5 or lower urgency**
(5–15), because its handler calls an RTOS function.

## Licenses

ST HAL/CMSIS drivers and FreeRTOS keep their own license files inside `Drivers/` and
`Middlewares/`. Application code in `Core/` is © STMicroelectronics (CubeMX template) and the
project author; add a `LICENSE` file to state the terms for your own code.
