# Klipper Host Architecture

This document summarizes the host-side Klipper program in the `klippy` folder, including its inputs, outputs, startup sequence, key components, and runtime interactions.

## Purpose

The `klippy` Python program implements the host firmware layer for Klipper. It:

- loads printer configuration from a `.cfg` file
- initializes MCU interfaces and uploads microcontroller config
- parses and dispatches G-code commands
- manages cooperative I/O and timers through a reactor
- supports dynamic extras and motion kinematics
- optionally exposes an API socket for external control

## Primary Inputs

- `config_file`: command-line argument to `klippy/klippy.py`
- `gcode_fd`: pseudo-TTY input created by `util.create_pty()` or `--debuginput`
- MCU transport:
  - UART serial port
  - CAN bus UUID/interface
  - debug output file with `--debugoutput`
- optional `--api-server` UNIX socket
- optional `--dictionary` protocol dictionary files for debug output

## Primary Outputs

- log output to stderr or file
- G-code channel responses (`ok`, `!!`, `//`)
- MCU protocol messages sent over serial/CAN/debug file
- JSON API responses over socket
- autosaved printer config updates via `SAVE_CONFIG`

## Key Modules

- `klippy/klippy.py`
  - `main()` entrypoint
  - `Printer` central object and event bus
- `klippy/reactor.py`
  - `Reactor` event loop for timers and file descriptor callbacks
- `klippy/configfile.py`
  - config parsing and validation
  - autosave state tracking
- `klippy/gcode.py`
  - G-code parsing and dispatch
  - `GCodeIO` input handling
- `klippy/pins.py`
  - pin name resolution and reservation
- `klippy/mcu.py`
  - MCU connection, protocol sync, and config upload
- `klippy/toolhead.py`
  - motion planning and kinematics
- `klippy/webhooks.py`
  - optional API socket server and JSON request handling

## Main Components and Behavior

### `Printer`

The `Printer` object in `klippy/klippy.py` is the central host structure. It:

- stores `reactor`, `bglogger`, and `start_args`
- manages runtime state (`startup`, `ready`, `shutdown`)
- registers event handlers and sends events
- stores named objects in a registry
- loads modules and config sections dynamically
- drives startup and shutdown behavior

### `Reactor`

The reactor in `klippy/reactor.py` provides:

- timer scheduling
- file descriptor callback support
- asynchronous callback handling
- cooperative greenlet context switching

### Config parsing

`klippy/configfile.py` provides:

- `ConfigWrapper`: typed config option access
- `PrinterConfig`: config file read, include processing, option validation, autosave

### G-code handling

`klippy/gcode.py` provides:

- `GCodeCommand`: parsed command object
- `GCodeDispatch`: command routing and registered handlers
- `GCodeIO`: reads raw input from the g-code fd, splits lines, and hands commands to dispatch

### Pin management

`klippy/pins.py` provides:

- `PrinterPins`: chip registration, pin reservation, and aliasing
- pin validation and shared pin handling

### MCU support

`klippy/mcu.py` provides:

- `MCU`: transport setup, config upload, stepper synchronization
- `CommandWrapper` and `CommandQueryWrapper` for protocol command sending
- MCU subcomponents like `MCU_endstop`, `MCU_digital_out`, and `MCU_trsync`

### Motion and toolhead

`klippy/toolhead.py` provides:

- `ToolHead`: motion queue, timing, kinematic solver loading
- move planning and flush logic
- registration of motion-related G-code commands

### Webhooks / API

`klippy/webhooks.py` provides:

- server socket creation for `--api-server`
- client connection handling
- JSON request parsing and response generation

## Startup Sequence

### High-level startup flow

```text
main() -> Reactor -> Printer -> Printer.run() -> Reactor.run()
    -> Printer._connect()
        -> config load
        -> pins and MCU creation
        -> extras load
        -> toolhead init
        -> config validation
        -> send_event("klippy:mcu_identify")
        -> send_event("klippy:connect")
        -> send_event("klippy:ready")
```

### Detailed startup steps

1. `klippy.py:main()` parses CLI options and builds `start_args`.
2. It creates `reactor.Reactor(gc_checking=True)`.
3. It instantiates `Printer(main_reactor, bglogger, start_args)`.
4. `Printer.__init__()` registers the connect callback and adds early objects.
5. `Printer.run()` enters `reactor.run()`.
6. The reactor invokes `Printer._connect(eventtime)`.
7. `_connect()` loads config and builds objects.
8. `_connect()` sends `klippy:mcu_identify` and `klippy:connect` events.
9. If successful, `_connect()` transitions state to ready and sends `klippy:ready`.

## Module initialization order

1. early objects:
   - `gcode`
   - `webhooks`
2. config parser:
   - `configfile.PrinterConfig`
3. base objects:
   - `pins`
   - `mcu`
4. dynamic extras from config sections
5. toolhead and kinematics
6. config validation and status building

## Event bus

`Printer` exposes:

- `register_event_handler(event, callback)`
- `send_event(event, *params)`

Common events:

- `klippy:connect`
- `klippy:mcu_identify`
- `klippy:ready`
- `klippy:disconnect`
- `klippy:shutdown`
- `gcode:request_restart`

## MCU configuration flow

1. `Printer._connect()` calls `send_event("klippy:mcu_identify")`.
2. Each `MCU` object runs `MCU._mcu_identify()`.
   - connect transport
   - initialize clocksync
   - read MCU constants and protocol messages
3. `Printer._connect()` sends `klippy:connect`.
4. Each `MCU` object runs `MCU._connect()`.
   - query MCU config with `get_config`
   - if unconfigured, send build config commands
   - if configured, send restart/init commands
   - allocate `steppersync` using MCU move count

## G-code processing flow

1. `GCodeIO._process_data()` reads from `gcode_fd`.
2. It splits raw bytes into lines and queues complete commands.
3. `GCodeDispatch._process_commands()` parses each line.
4. Each command becomes `GCodeCommand`.
5. The dispatch layer finds the registered handler.
6. The handler executes and may produce responses.
7. `GCodeCommand.ack()` sends `ok` when appropriate.
8. Errors are reported with `!!` and state events.

## Object interaction diagram

```text
[main] --> [Reactor]
          [Printer]
            ├─ [GCodeDispatch]
            ├─ [GCodeIO]
            ├─ [PrinterConfig]
            ├─ [PrinterPins]
            ├─ [MCU]
            │    ├─ [serialhdl]
            │    ├─ [clocksync]
            │    ├─ [MCU_endstop]
            │    ├─ [MCU_digital_out]
            │    └─ [steppersync / CommandWrapper]
            ├─ [ToolHead]
            │    ├─ [kinematics.<type>]
            │    └─ [kinematics.extruder]
            └─ [Webhooks]
```

## Runtime patterns

- `Printer` is the central registry and event hub.
- `Reactor` drives I/O and timer callbacks.
- `ConfigWrapper` performs typed option parsing and validation.
- `GCodeDispatch` routes commands and manages ready-state command availability.
- `MCU` manages the low-level MCU protocol and configuration lifecycle.
- `ToolHead` manages motion planning, move queueing, and kinematics.

## Save location

All analysis results are saved in this file:

- `docs/klippy-host-architecture.md`
