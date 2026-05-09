# Simplified-AKEDP-Core-for-STM32F

This repository was created from the original AK firmware source code based on the STM32L151, with the goal of migrating to the STM32F103 platform and refactoring the code to improve modularity and maintainability.

The new source code is significantly simplified compared to the original, focusing on core runtime functionality and a lightweight application layer to support users in quickly adopting the event-driven programming (EDP) model.

## Note

Future work of this repository is now redirected to new repository [CIEDPC – Custom Independent Event-Driven Programming Core](https://github.com/1811htsang/CIEDPC-Custom-Independent-Event-Driven-Programming-Core.git) to implement a clean and modular EDP core that can be easily ported to different platforms, including STM32F103. The current repository will be archived after the new repository is fully set up and tested.

## Current Status

As of the latest refactor, the STM32F103 port is organized around `Drivers/EDP` and core runtime stability.

Included in the active profile:

- AKEDP kernel core (`core/inc`, `core`/src`): task scheduler, message system, timer, FSM, TSM (task state machine).
- Core app tasks: `task_if` to support receiving messages from other interface like peripheral and redirecto to `task` & `message` design of the kernel.
- Software watchdog implementation, fatal debug logging, platform define and redirects HAL API to follow AKEDP code style in `Drivers/EDP/sys`.
- Common utilities in `Drivers/EDP/common` like circular buffer, linked list, and other helper functions.

Removed or excluded from the active profile:

- Shell and command-line update flow.
- RF24/Zigbee integration tasks.
- Modbus-related application path.
- UI/screens modules.
- UART, ADC, SPI drivers and related peripheral handling code.
- Debug logging and error handling beyond the core fatal log and watchdog.
- Flash handling and bootloader code.

This keeps the firmware smaller and easier to validate during low-level porting.

Testing is currently focused on ensuring the core runtime can run without errors, and that basic task scheduling and message passing works as expected. Peripheral integration and application-level features can be replaced with auto-generated HAL API from STM32CubeIDE and migrating into AKEDP code style once the core runtime is stable.

## Repository Structure

- `stm32f_porting/`: Active STM32F103C8T6 port.
  - `AK Firmware F103 Custom Porting/`: STM32CubeIDE project
  - `docs/`: Documentation and notes related to the porting process.
- `stm32l_base/`: Original STM32L151 codebase for reference.
- `user-manual.md`: Notes and usage guidance for local development.
- `README.md`: Project overview and current status.

## Build And Development Notes

- Main project type: STM32CubeIDE managed project (GCC + G++).
- Recommended environment: Linux, or Windows with STM32CubeIDE / WSL toolchain.
- When changing generated build files, keep C and C++ compile paths separated.
- If source paths are renamed (for example, legacy `AK_Imported_Driver` -> `EDP`), recheck `.cproject` and generated Debug metadata, also check for headers & sources inclusion in the GCC, G++ settings.

## Quick Start

1. Open `stm32f_porting/AK Firmware F103 Custom Porting` in STM32CubeIDE.
2. Select `Debug` configuration.
3. Build the project and resolve any errors related to include paths or missing files.
4. Check the `Test/sample` to run the basic task scheduler and message passing functionality.

## Compatibility Notes

- This port targets STM32F103 line specifically, if any kits differ from the F103 but still based on the same core, some adjustments may be needed in the HAL layer and peripheral handling is required to make sure the core runtime can run properly.
- It is not a drop-in package for other STM32 families.
- Use `stm32l_base` as the baseline reference when rebasing or comparing behavior to update or enhance feature support.

## Contributing

Contributions are welcome.
For each change, prefer small focused commits with clear scope, and include build/test evidence when possible.
