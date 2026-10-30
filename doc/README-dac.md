# DAC12 (with ADC Loopback)

## Overview

This Software Building Block (SBB) demonstrates use of the Renesas DAC together with an ADC to generate an analog output and read that output back on an ADC input. This document covers only:

- FPB-RA4E2
- FPB-RA6E2
- FPB-RA8E1

The reference application accepts a DAC value from **1 to 4095**, writes the value to the DAC, starts conversion, waits for the DAC output to stabilize, starts an ADC scan, and prints the resulting ADC value to the serial terminal.

A jumper wire connects the DAC output to ADC channel 0 on each of the supported reference boards.

## Use case demonstrated by this SBB

The SBB demonstrates a DAC-to-ADC analog loopback use case:

1. Open the DAC and ADC peripherals.
2. Configure the ADC scan.
3. Calibrate the ADC where required by the selected ADC implementation.
4. Accept a DAC value from the reference application's terminal interface.
5. Write and start the DAC conversion.
6. Allow the DAC output voltage to stabilize.
7. Start the ADC scan.
8. Read the ADC result after scan completion.
9. Print the ADC value to the terminal.

This provides a functional reference for generating an analog value with the DAC and observing it through an ADC input.

## Hardware setup

The supplied documentation requires:

- 1 x supported Renesas RA board
- 1 x appropriate USB cable for programming and debugging
- 1 x jumper wire

Connect the board's USB debug port to the host PC and make the DAC-to-ADC loopback connection for the selected board.

### FPB-RA4E2

Connect:

- **P014 (J4:21), DAC Output** to **P000 (J4:32), ADC Channel 0**

### FPB-RA6E2

Connect:

- **P014 (J4:21), DAC Output** to **P000 (J4:32), ADC Channel 0**

### FPB-RA8E1

Before using the on-board debug functionality:

- Open jumper **J6**.
- Open jumper **J9**.

Connect:

- **P014 (J1:34), DAC Output** to **P004 (J1:20), ADC Channel 0**

## SBB core code

The reusable SBB layer is implemented by:

- `dac_ep.c`
- `dac_ep.h`

The core provides the peripheral-facing operations needed by the example, including:

- DAC open and close.
- ADC open and close.
- ADC scan configuration.
- Optional ADC calibration when enabled by the selected ADC implementation.
- DAC write, start, stabilization delay, ADC scan start, and DAC stop sequencing.
- Reading an ADC conversion result.
- Handling ADC scan-complete callbacks.
- Recording ADC completion and error state for the application.
- Compile-time abstraction between the supported DAC and DAC_B APIs.
- Compile-time abstraction between the supported ADC and ADC_B APIs.

## Reusable SBB code versus demo/reference application

### Reusable SBB code

`dac_ep.c` and `dac_ep.h` are the reusable peripheral-facing layer. They contain the DAC/ADC wrapper functions, driver-selection macros, conversion sequencing, ADC callback handling, and the shared result/status variables.

### Demo/reference application code

`dac_example_app.c` and `dac_example_app.h` implement the terminal-driven demonstration. The reference application is responsible for:

- Initializing the UART console.
- Displaying example and FSP version information.
- Initializing the SBB's DAC and ADC resources.
- Prompting the user for a DAC value.
- Reading and validating terminal input.
- Calling the DAC/ADC operation.
- Printing the ADC result after conversion completes.
- Printing errors, deinitializing resources, and entering a breakpoint on failure.

Its application entry point is:

```c
void dac_example_app_run(void);
```

The source identifies this function as the single entry point called by `hal_entry.c`.

The terminal UI, input parsing, prompts, and continuous foreground loop are reference-application behavior rather than reusable DAC/ADC driver operations.

## Public functions to look at first

### `dac_open()`

```c
fsp_err_t dac_open(void);
```

Opens the generated DAC instance exposed through `g_dac_ctrl` and `g_dac_cfg`.

### `adc_open()`

```c
fsp_err_t adc_open(void);
```

Opens the generated ADC instance exposed through `g_adc_ctrl` and `g_adc_cfg`.

### `adc_scan_cfg()`

```c
fsp_err_t adc_scan_cfg(void);
```

Configures ADC scanning. With ADC_B it uses `g_adc_scan_cfg`; otherwise it uses `g_adc_channel_cfg`.

### `dac_adc_operations()`

```c
fsp_err_t dac_adc_operations(int32_t * input);
```

This is the main SBB operation to study. It writes the requested DAC value, starts the DAC, applies the configured stabilization delay, starts the ADC scan, and stops the DAC output for the next conversion cycle.

### `adc_read()`

```c
fsp_err_t adc_read(adc_ctrl_t * p_ctrl,
                   adc_channel_t const reg_id,
                   uint16_t * const p_data);
```

Reads a conversion result from the selected ADC channel.

### `adc_callback()`

```c
void adc_callback(adc_callback_args_t * p_args);
```

Handles ADC events. On `ADC_EVENT_SCAN_COMPLETE`, it reads `ADC_CHANNEL` into `g_adc_data` and sets either the completion or error flag. When ADC calibration support is compiled in, it also handles `ADC_EVENT_CALIBRATION_COMPLETE`.

### `adc_calibrate()`

```c
fsp_err_t adc_calibrate(void);
```

This API exists when `BOARD_RA2A1_EK` or `BSP_PERIPHERAL_ADC_B_PRESENT` is defined. It initiates calibration and waits until the callback reports calibration completion.

### Cleanup functions

```c
void dac_deinit(void);
void adc_deinit(void);
```

These close the DAC and ADC instances when they are open.

## Reference configuration assumptions

### DAC input range

The SBB headers define:

```c
#define DAC_MIN_VAL (1)
#define DAC_MAX_VAL (4095)
```

The reference application therefore accepts values from **1 through 4095**. The original documentation explicitly excludes zero because the ADC output will not be zero due to an offset-correction value.

### ADC channel

For FPB-RA4E2, FPB-RA6E2, and FPB-RA8E1, none of the special board selections in `dac_ep.h` apply. The SBB therefore selects:

```c
#define ADC_CHANNEL (ADC_CHANNEL_0)
```

This agrees with the ADC channel 0 hardware connections specified for all three boards in the supplied README.

### DAC/ADC implementation selection

The SBB selects the FSP driver API at compile time:

- `BSP_PERIPHERAL_DAC_B_PRESENT` selects the DAC_B API; otherwise the DAC API is used.
- `BSP_PERIPHERAL_ADC_B_PRESENT` selects the ADC_B API; otherwise the ADC API is used.

The generated project is therefore expected to provide the control and configuration objects used by these wrappers.

### Stabilization delay

The SBB defines:

```c
#define STABILIZATION_DELAY (1)
```

and calls:

```c
R_BSP_SoftwareDelay(STABILIZATION_DELAY,
                    BSP_DELAY_UNITS_MILLISECONDS);
```

between starting the DAC and starting the ADC scan.

### Terminal input

The input buffer is 16 bytes. The original README states that terminal input should not exceed 15 bytes and that operation is not guaranteed for non-integer input. It also notes that only the integer part of floating-point input is processed.

## What must change when adapting the SBB?

### DAC and ADC pins

The DAC output to ADC input connection is board-specific. If the SBB is moved to different pins or another board, update the generated pin/peripheral configuration and connect the selected DAC output to the intended ADC input.

For the documented reference boards, retain the connections listed in the Hardware setup section.

### ADC channel

The reusable source reads the channel selected by `ADC_CHANNEL`. The three documented boards use `ADC_CHANNEL_0`. If a different ADC input is selected in another design, `ADC_CHANNEL` and the generated ADC scan configuration must agree with that selection.

### DAC and ADC peripheral instances

The implementation expects generated objects named:

- `g_dac_ctrl`
- `g_dac_cfg`
- `g_adc_ctrl`
- `g_adc_cfg`
- `g_adc_scan_cfg` when ADC_B is selected, or `g_adc_channel_cfg` for the other ADC path

These must correspond to the target project's generated FSP configuration.

### ADC calibration

Calibration is conditional. The application calls `adc_calibrate()` when `BSP_PERIPHERAL_ADC_B_PRESENT` is true for the target. That path waits for `ADC_EVENT_CALIBRATION_COMPLETE`. If ADC_B is used, preserve the required generated callback configuration so calibration completion can be detected.

### Timing

The SBB currently uses a 1 ms software stabilization delay between starting the DAC and starting the ADC scan. If the target device, electrical load, or required settling behavior differs, validate the appropriate conversion/settling timing for that target rather than assuming the reference value is universally applicable.

### Clocks

The supplied source and README do not state the DAC or ADC peripheral clock configuration for FPB-RA4E2, FPB-RA6E2, or FPB-RA8E1. If application clock settings are changed, the resulting DAC and ADC configuration must be validated against the target MCU configuration.

### Memory and task priorities

The supplied files do not specify memory-footprint requirements, an RTOS, stack or heap sizing, or task priorities. The reference application runs as a continuous foreground loop and reacts to shared flags set by the ADC callback. Any RTOS scheduling or memory requirements need to be established by the integrating application.

### Console middleware

The demo uses `console/uart_console.h` and standard I/O. This terminal interface is reference-application behavior. An application that reuses only the DAC/ADC SBB core does not need to preserve the interactive console unless required by its own design.

## Dependencies

### FSP DAC driver

Depending on device capabilities, the SBB maps its DAC operations to either the DAC or DAC_B FSP implementation. The operations used are:

- Open
- Write
- Start
- Stop
- Close

### FSP ADC driver

Depending on device capabilities, the SBB maps its ADC operations to either ADC or ADC_B. Its operations include:

- Open
- Scan configuration
- Calibration when compiled for the target
- Scan start or scan-group start
- Read
- Close

### FSP/BSP services

The supplied code also depends on:

- `hal_data.h` for generated configuration and FSP types.
- `R_BSP_SoftwareDelay()` for DAC output stabilization.
- `R_FSP_VersionGet()` in the reference application.
- ADC callback events for scan completion and, where applicable, calibration completion.

### Console and C runtime

The reference application depends on:

- `console/uart_console.h`
- `uart_console_init()`
- `uart_console_has_char()`
- `uart_console_flush()`
- `printf()`
- `fgets()`
- `atoi()`
- `strcspn()`

### Hardware resources

The reference implementation requires:

- A DAC peripheral/output.
- An ADC peripheral/input.
- A jumper connecting the DAC output to ADC channel 0.
- A USB/debug connection.
- A serial console for the supplied demonstration.

## Serial terminal configuration

The original README specifies:

| Setting | Value |
|---|---|
| Port | Port provided by J-Link On-Board |
| Speed | 115200 |
| Data | 8 bit |
| Parity | None |
| Stop bits | 1 bit |
| Flow control | None |

For Tera Term, it additionally specifies:

- Receive newline: **CR**
- Transmit newline: **CR**
- **Local echo** enabled

## Demonstrated behavior and performance

### Demonstrated behavior

The supplied SBB/reference application demonstrates that:

- A user-selected value from 1 to 4095 can be written to the DAC.
- The DAC conversion can be started and stopped through the selected FSP DAC implementation.
- An ADC scan can be configured and initiated.
- The DAC output can be physically routed back to ADC channel 0 on FPB-RA4E2, FPB-RA6E2, and FPB-RA8E1.
- An ADC result can be read after the ADC scan-complete event and printed to the terminal.
- ADC_B calibration is performed when that implementation is selected by the target's BSP feature configuration.

The supplied README states that ADC output is in a **12-bit/16-bit range**, but it does not specify a single resolution for all three boards covered by this document.

### Performance measurements

The supplied files do **not** provide measured DAC accuracy, ADC accuracy, conversion throughput, settling measurements, CPU utilization, memory footprint, interrupt latency, or other benchmark data.

The source contains a 1 ms stabilization delay, but this is an implementation delay rather than a reported measurement of conversion performance.

Accordingly, this SBB should be treated as a functional DAC-to-ADC integration reference rather than a performance benchmark.

## File ownership and modification guidance

| File | Classification | Guidance |
|---|---|---|
| `dac_ep.c` | Reusable SBB implementation | Modify when intentionally changing reusable DAC/ADC operations, sequencing, callbacks, or driver integration. |
| `dac_ep.h` | Reusable SBB interface/configuration abstraction | Modify carefully when changing public functions, SBB constants, channel selection, or ADC/DAC driver mappings. |
| `dac_example_app.c` | Demo/reference application | Appropriate location for changing terminal interaction, input validation, printed results, or demonstration sequencing. |
| `dac_example_app.h` | Demo/reference application interface | Contains the demo entry point, version, banner, and informational text. |
| `readme.txt` | Existing example documentation | Source of the board connections, input restrictions, and serial-terminal requirements used in this guide. |

The SBB includes `hal_data.h` and references generated FSP objects. The supplied file set does not enumerate those generated configuration files, so this document does not assign modification guidance to individual generated files that were not provided.

## Adopting a newer SBB release after local modifications

If `dac_ep.c` or `dac_ep.h` are modified locally, those changes create differences in the reusable SBB layer that may overlap changes in a future SBB release.

To keep future upgrades manageable:

- Keep application-specific input handling, messages, and presentation in `dac_example_app.*` rather than moving them into the reusable SBB layer.
- Keep board/product-specific changes out of `dac_ep.*` when they can instead be represented by the generated peripheral and pin configuration.
- Keep necessary reusable-source changes focused and track why each change was made.
- Keep generated FSP configuration separate from hand-maintained SBB source.
- When adopting a newer SBB version, compare the new `dac_ep.c` and `dac_ep.h` with the locally modified versions and reconcile the differences.
- Revalidate DAC range handling, DAC/ADC implementation selection, channel selection, pin routing, stabilization timing, ADC callback operation, and ADC result handling after the update.

The supplied files do not define an automated migration or compatibility mechanism for future SBB releases. Compatibility after adopting a later release therefore needs to be validated by the integrating project.

## Recommended reading order

1. **`dac_ep.h`** to understand the public functions, DAC range, ADC-channel selection, and DAC/ADC driver abstraction.
2. **`dac_ep.c`** to understand peripheral initialization, DAC-to-ADC sequencing, calibration, scan completion, and cleanup.
3. **`dac_example_app.c`** to see how the SBB is exercised through the terminal-driven reference application.
4. **`dac_example_app.h`** to see the application's public entry point and example information.
5. **`readme.txt`** for the original board wiring and serial-terminal setup information.
