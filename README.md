# RL78_G16_I2C_Slave_Transfer

Standalone, non-destructive I2C ISP communication sample for R5F121BCAFP. Sources, configuration and build helpers are included in this project.

## Board and interface

| Item | Configuration |
| --- | --- |
| Clock | HOCO / CPU 16 MHz |
| Update link | IICA0 SCL / SDA P60 / P61, 7-bit address 0x20, 100 kHz |
| Diagnostic Log | UART0 TX / RX P03 / P04, 115200 8-N-1 |
| Update interrupt | IICA0 vector 0x2C |
| Timer vector | 0x1C |
| Flash implementation | RFSP Type01; CPU stalls during Code Flash programming |

Vector values are physical byte addresses, not channel numbers. UART0 carries text only. Cross UART TX/RX, or connect matching SCL/SDA with voltage-compatible pull-ups; share ground and verify logic levels.

## Image and interrupt ownership

Boot is 0x0000..0x1FFF (8 KiB), with at least 512 bytes spare. App vectors are 0x2000..0x20FF, metadata is 0x2100..0x21FF (256 bytes), and App code starts at 0x2200.
Maximum complete App BIN is 23 KiB; 0x07C00..0x07FFF is protected. Size includes vectors, metadata, padding and the four-byte CRC32.
Boot owns physical vectors: AA selects Boot handlers; 55 forwards through the App table. Normal handlers use RETI; BRK retains RETB.

## Configuration and application editing

- `image_metadata_config.h`: neutral MCU/Flash/format/Boot identity.
- `isp_transport_config.h`: selected ISP bus/protocol settings; metadata readers use the same values.
- `boot_app_handoff.h`: reserved RAM and AA/55 owner.
- `hal/isp/`: protocol and transport runtime; `hal/device/` and `hal/flash/` own hardware adapters.
- `image_metadata.c/.h`, `image_metadata_profile.c`: independent parser and expected profile.

No protocol parser, Flash operation or formatted log belongs in an ISR.

## Smart Configurator

Open this project's same-name `.scfg` from the CS+ Smart Configurator node.
After editing, use **Generate Code**, then **Rebuild** or the PowerShell command
below. Xmodem uses UART peripherals; UART0 remains the independent text Log.

Pre-build checks the MCU, pins, clocks, serial contract and actual generated
registers, then creates this project's `sc_peripheral_config.generated.h` for
runtime initialization. This header is an output, not an authoring input.
Update-link changes also need the paired Boot's `isp_transport_config.h` and
matching Boot/App/Transfer SC settings.

The existing runtime retains ISR, queue, mailbox, protocol and vector handling.
Configuration steps and the seven-bit I2C / SVA conversion are in the
[SC workflow](https://released.github.io/slide_boot_loader_for_RL78/serial-device.html#serial-scfg-workflow).

## Build and files

Run in this project root using installed CS+/CC-RL and Windows PowerShell:

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File .\build.ps1
```

Use `-Compiler <CubeSuite+.exe>` for a different installed CS+ path. Firmware authoring/build requires no Python or JSON editor. Generated INC/identity files are outputs, not inputs. App/Transfer wrappers build only themselves.

| File under DefaultBuild | Use |
| --- | --- |
| `RL78_G16_I2C_Slave_Transfer.hex` | Programmer installation of this standalone reset-based diagnostic image |

This is neither an App update BIN nor a Boot-only HEX. It reports device/protocol identity, not installed Boot/App metadata.

## Tool operation and protocol

Install the standalone HEX with a programmer. Select **I2C > Custom Transfer**, then **Discover Test Target > Transfer Test**. A BIN is not required. Flash update commands and ENTER are rejected; INFO capability 0x10 identifies this role.

See the local [protocol reference](PROTOCOL.md) for byte layouts and transport behavior.
[Tool operation and wiring](https://released.github.io/slide_simple_renesas_isp_tool/#uart-device-operation) provides the common operation flow and report links.
Offline build/model results do not establish electrical, physical IRQ/Flash, dynamic-stack, restart or power-interruption acceptance.

## User code entry

`custom_func.c` provides `hardware_init()` for initialization and `loop()` for
foreground service. Add application initialization and foreground work through
these callbacks, retaining the existing communication service calls.
The selected main/startup adapter owns entry and handoff. Generated peripheral
files remain under `src/`; retain the project's MTPJ source selection when using
Code Generator or Smart Configurator, then rebuild to validate the image and vectors.

## Project layout and standalone build

This project includes its own source files, configuration headers and build tools.
A sibling Boot, App or Transfer checkout is not required to compile it.
Keep the project directory name aligned with its `.mtpj` filename; remove a ZIP download suffix when necessary.
Build with CS+ Rebuild, or run the local PowerShell entry:

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File .\build.ps1
```

| Location | Responsibility |
| --- | --- |
| `custom_func.c` | Project initialization in `hardware_init()` and recurring work in `loop()`; runtime orchestration stays here |
| `hal/can` | CAN service, controller driver and CAN adapter, when used by this project |
| `hal/device` | Device, clock, peripheral and CPU handoff APIs |
| `hal/image` | Metadata parser/profile, image validation and software CRC |
| `hal/flash` | Flash access/geometry APIs, when required by the selected image |
| `hal/isp` | Update/Transfer protocol and transport processing APIs |
| Root configuration headers | Local MCU/Flash, handoff and communication contract |
| `tools` and root PowerShell helpers | Local metadata/IRQ generation and linked-image checks |
| `src` | Code Generator or Smart Configurator peripheral sources |

Only the sources registered in this project's CS+ configuration are linked.
Generated main/vector implementations must not become a second active entry.
Retain the `custom_func.c` callbacks and the selected startup adapter after code generation.

Boot and App each carry a local contract. Keep their MCU, Flash layout, image format,
IRQ ABI, handoff and communication settings compatible when installing a pair.
App releases are edited in `app_image_config.h` and `app_irq_handlers.h` in the App project.
Rebuild the selected App to regenerate its metadata, vector table, update BIN and timing BINs.
The App build does not read or rewrite a Boot checkout.

## UART0 local console

- Diagnostic UART0 uses115200,8-N-1; UART1 or I2C retains binary ISP traffic.
- `z` or `Z` requests a normal MCU reset after the current reply completes; I2C includes final NACK/STOP.
- Foreground RX/TX traces show16-bit sequence, command, length and complete validated payload. `LOG_DROP` counts discarded text records.
- G16 services text from existing packet buffers for at most200ms before accepting another framed request; other MCUs use bounded snapshot queues.

## CC-RL option bytes

User option bytes are `EEF7F9`; OCD is `85`. G16 disables WDT and standby counting with C0=`EE`; C2=`F9` selects 16 MHz. G16 fixes OCD bits6..0 to `0000101`, so ID-authentication-failure erase selection is unavailable. Generated IAR-only no-WDT defaults do not replace the explicit CC-RL C0 policy.

## Operation guide

[G16 I2C wiring, Tool steps, Boot/App flows and update measurements](https://released.github.io/slide_boot_loader_for_RL78/i2c-device.html#upgrade-g16-i2c). UART0 manual reset, cold start and fault recovery need their own board evidence.

## GitHub updates

The local project-root `github_push.bat -DryRun` reviews the source/image snapshot. `github_push.bat` publishes only to its existing authorized PUBLIC `released/RL78_G16_I2C_Slave_Transfer` main branch after matching board acceptance. Repository creation is a separate one-time operation. The batch entry stays local; reusable publisher helpers are included in the repository.
