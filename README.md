# RP2040 Voltage Fault Injection Tool

A PicoGlitcher based hardware platform for voltage fault-injection experiments, using Raspberry Pi Pico PIO state machines for timing without an FPGA. The board combines MOSFET crowbar glitching, selectable supply-voltage multiplexing, and software-controlled target power routing on a shared target supply output.

The initial target is the **MSP-LITO-L2228 board based on the TI MSPM0L2228**, for eCTF experiments.

## Project status

The schematic and a four-layer PCB layout have been developed. The current timing goal is approximately **200–300 ns crowbar command pulses**. This is a design goal for the control signal; the resulting voltage dip at the target MCU has not yet been recorded.

Neither the Firmware or the PCB has been implemented yet.

## Features

- RP2040-based experiment control and PIO pulse sequencing.
- Two selectable crowbar paths: direct and through a 0.1 Ω source resistor.
- A dedicated dual-channel MOSFET gate driver powered from 12 V.
- Voltage selection using 3.3 V, 1.8 V, or an external user voltage.
- Independently controlled normal-supply and multiplexed-supply paths.
- A shared `VDD_VICTIM` connection for the external target.
- Three trigger input


The target supply is the common node between the selected power source, the crowbar devices, and the voltage-monitoring circuit. The controller has its own supply, allowing it to remain active while the target voltage is disturbed.

## 1. Controller and timing

The Raspberry Pi Pico provides the RP2040 controller. Its responsibilities are to configure the operating mode, select a voltage source, arm a trigger, sequence the glitch outputs, and monitor target supply voltage.

PIO state machines are intended to handle timing-critical operations.

Gate-driver propagation, MOSFET switching, target capacitance, and connection inductance determine the actual supply waveform. Trigger sampling also contributes latency; the completed firmware must account for its configuration.

## 2. Crowbar stage

The crowbar stage briefly creates a low-resistance path from `VDD_VICTIM` to ground. It discharges energy stored in the target supply capacitance and disturbs the MCU's operating voltage.

Two **IRF7832PBF N-channel MOSFETs** provide different discharge paths:

One directly shorts the power rail to Ground

The other shorts the power rail to ground through a 0.1 ohm source resistart for alternative discharge characteristic

The 0.1 Ω resistor does not establish a fixed peak-current limit. 

The design intentionally has no active current-limiting stage in these crowbar branches.

### Gate drive

**U2, a UCC27524D dual gate driver**, is supplied from the 12 V rail. I

Both driver inputs are connected to the same `INA` net. `ENA` and `ENB` select which channels respond to that input. 

## 3. Voltage multiplexing

**U4, a TS3A4751PWR**, contains four independently controlled analog switches. Their output terminals are tied together at `MUX_OUT`, creating a selectable voltage source.

| Control net | Source connected to MUX_OUT |
|---|---|
| `MIN1` | `3V3M`, the dedicated 3.3 V MUX supply |
| `MIN2` | `USER_COM`, an external user-supplied voltage |
| `MIN3` | `1V8`, the dedicated 1.8 V supply |
| `MIN4` | Ground |

Channel selection is implemented in firmware rather than through an external decoder and inverter stage.

Only one source should be selected at a time. Changes between sources therefore require an all-off transition.

Although the schematic includes a ground channel, the intended operation excludes passing a zero-volt MUX source through the TPS22918 path. Ground selection is not presented as a validated end-to-end operating mode.

`USER_COM` is an analog voltage source for the MUX, not a Pico GPIO input. Its voltage must remain within the analog switch's permitted range. The target's approximately 5 mA measured.

## 4. Target power routing

The board routes two supply paths to the same target output:

```text
AMS1117 3.3 V ── J6 link ── 3V3C ── Load Switch ────|
                                                    ├── VDD_VICTIM ── target
Selected voltage ── MUX_OUT ── Load Switch ─────────|
```

- **U8** connects the normal `3V3C` source under `CROW_LD` control.
- **U7 and U9** form a common-output, back-to-back pair between `MUX_OUT` and the target, under `MUX_LD` control.
- **J6** provides a link between U1's output and the `3V3C` supply net.

The TPS22918 switches select or disconnect the supply paths between experiments or operating modes. They are not the components intended to generate the 200–300 ns crowbar pulse.

The back-to-back MUX arrangement is intended to address reverse conduction through that branch. U8 is a single load switch, so the overall board should not be treated as providing complete bidirectional isolation on every supply path. 

Firmware is responsible for preventing unintended source overlap and returning the outputs to a defined state after each experiment. There is no dedicated hardware interlock enforcing the complete operating sequence.

## 7. Trigger

Three external trigger connections, `TRIG`, `TRIG1`, and `TRIG2`, connect to Pico GPIOs with pull-down resistors. They allow firmware to select trigger sources for an experiment. These are direct logic interfaces with no level translation shown in the reviewed schematic.

## 8. RP2040 pin mapping

GPIO numbers below are RP2040 GPIO identifiers, not Pico physical pin numbers.

| GPIO | Net | Function |
|---|---|---|
| GPIO3 | `TRIG` | Trigger input |
| GPIO4 | `TRIG1` | Trigger input |
| GPIO5 | `TRIG2` | Trigger input |
| GPIO8 | `MIN2` | Select USER_COM |
| GPIO9 | `MIN3` | Select 1.8 V |
| GPIO10 | `MIN4` | Select ground channel |
| GPIO11 | `MIN1` | Select MUX 3.3 V |
| GPIO12 | `CROW_LD` | Control normal-supply switch U8 |
| GPIO16 | `INA` | Shared gate-driver pulse input |
| GPIO17 | `ENA` | Enable channel A / IC1 |
| GPIO18 | `ENB` | Enable channel B / IC2 |
| GPIO22 | `MUX_LD` | Control U7 / U9 MUX supply path |
| GPIO26 / ADC0 | `VDD_ADC` | Filtered target-voltage measurement |


## 9. External connectors

| Connector | Signals | Role |
|---|---|---|
| J1 | `+12V`, GND | External board supply |
| J2 | `VDD_VICTIM`, GND | Target supply and fault connection |
| J3 | `TRIG`, GND | Trigger input |
| J4 | `TRIG1`, GND | Trigger input |
| J5 | `TRIG2`, GND | Trigger input |
| J6 | U1 output, `3V3C` | Normal target-supply link |
| J7 | `USER_COM`, GND | External MUX voltage input |


## 10. PCB architecture

The four-layer layout approach uses:

| Layer | Intended role |
|---|---|
| L1 | Components, short signal routes, power and ground copper |
| L2 | Continuous ground reference plane |
| L3 | Signal routing and power distribution where needed |
| L4 | Additional routing and power/ground copper |

The copper-weight target is 2 oz on the outer layers and 1 oz on the inner layers.


## Validation still to document

- Regulator stability and settling with each supply path selected.
- Startup, mode transitions, and source-disconnect behavior.
- Gate-to-source waveforms on both crowbar devices.
- Target-rail dip depth, duration, ringing, and recovery measured at the MCU.
- Peak-current and repetition-rate limits for the assembled circuit.
- Trigger-to-pulse timing and complete PIO firmware behavior.


This project is inspired by [PicoGlitcher and the Fault Injection Library](https://github.com/MKesenheimer/fault-injection-library). It explores a custom combination of crowbar and voltage-multiplexing paths with a shared target connection and direct GPIO control of the analog switches.
