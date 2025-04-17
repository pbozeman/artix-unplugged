# Artix 7 power notes

See [DS181 - Artix 7 FPGAs Data Sheet: DC and AC Switching Characteristics](https://docs.amd.com/v/u/en-US/ds181_Artix_7_Data_Sheet)

## Voltages

### Sequencing

Summary:

* Core voltages: Vccint -> Vccbram -> Vccaux -> Vccoo
* GTP voltages: (Vccint && Vmgtavcc) -> Vccavtt

If core voltages do not use the sequence above, IO may not be tri-stated at
startup. It is highly desired to make this happen so that components like
SRAM are not driving the bus at the same time as the FPGA as doing so
can damage both.

If Vccavtt is powered before Vmgtavcc, current draw can increase by
460mA per transceiver during Vmgtavcc ramp up. This one is less critical
to meet if the power supplies can easily supply the current.

Details in "Power-on/off Power Supply Sequencing", page 8, in [DS181](https://docs.amd.com/v/u/en-US/ds181_Artix_7_Data_Sheet)

### Core Voltages

| Net       | Voltage   |
|-----------|-----------|
| Vccint    | 1.00      |
| Vccaux    | 1.80      |
| Vccbram   | 1.00 [^1] |
| Vcco      | [^2]      |
| Vccbatt   | [^3]      |

[^1]: Vccint and Vccbram should be connected to the same supply.
[^2]: Vcco is one of 1.2V, 1.35V, 1.5V, 1.8V, 2.5V, 3.3V
[^3]: Vccbatt is 1.0 to 1.89 (or GND if not using bitstream encryption)

### GTP Voltages

Each of the GTP voltages needs a filter described in: <https://docs.amd.com/v/u/en-US/ug482_7Series_GTP_Transceivers>

| Net         | Voltage |
|-------------|---------|
| Vmgtavcc    | 1.00    |
| Vmgtavtt    | 1.2     |

### XADC Voltages

| Net         | Voltage |
|-------------|---------|
| Vccadc      | 1.80    |
| Vrefp       | 1.25    |

### DDR Voltages

| Net         | Voltage |
|-------------|---------|
| Vccddr      | 1.50    |
| Vccddrt     | 0.75    |

## Selecting bank voltages

Options for bank voltage selection:

* Jumper on to select a resistor for adjusting a buck regulator. (Numato)
* Power mux like TPS2116DRLR. In theory, this would prevent cutover issues if
  a jumper is changed while the system is powered. Given that peripherals
  would likely also be being swapped, this seems like overkill. (Alacrity)
* Change SMD resistors (Digilent)

> Decision: Use a jumper to adjust the resistors for a buck regulator.

## Multi Rail Options

The power system needs to support 5 digital rails, plus one analog rail. And
this assumes that a 5V is not going to be passed along to daughter cards. If
there is also going to be a 5V rail, then there actually 6 rails + 1 analog.

There are no good options for a single chip to do this, i.e. the best option
that is stocked at JLCPCB at the time this is written is ADP5052ACPZ-R7, which
has 4 buck and 1 ldo output.

> Decision: Design around the [TPS563257DRLR](https://jlcpcb.com/partdetail/TexasInstruments-TPS563257DRLR/C20539656)
> which is in large supply at JLCPCB.
> (1.2Mhz, 3V-17V input, adjustable, 3A output.) Use 1 per rail.

### ADC Rail Options

* Dedicated LDO: cleanest power option, but comes at the cost of an
  extra extended part that will only be used once. ($3 overhead per run)
* Dedicated Buck: separate rail might result in lower noise creeping in from
  the digital side.
* Shared buck with Vccaux, with a ferrite bead for filtering. Lowers the total
  bom count and takes up less space on the PCB.

Digilent and Alacrity both have a dedicated LDO, albeit from a multi-output
pmic. Numato shares the 1.8v rail, they don't even use a ferrite bead, only a
bypass cap. Given the ADC is so low resolution, and is likely only to ever
be used for DDR thermal management, sharing the supply, plus a FB seems fine.

> Decision: Share Vccaux, plus a ferrite bead.

### DDR Termination Voltage Options

* Resistor divider of Vccddr. This is cheap, easy, and is what
  Numato and Alactriy do.
* DDR voltage controller or LDO: Digilent takes this route. It seems excessive.

> Decision: resistor divider on Vccddr.
