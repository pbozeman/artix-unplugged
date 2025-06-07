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

From the UG484, for both Vmgtavcc and mgtavtt:

* "The power supply regulator for this voltage should not be shared
  with non-transceiver loads."
* "For optimal performance, power supply noise must be less than 10 mVPK-PK."

TODO: Review the entire PCB Design Checklist in UG484.
TODO: Look into AP7176BFN-7. Once this is used, it might reopen the decisions
for the power rails.
TODO: also consider ISL80103IRAJZ-TK

### XADC Voltages

| Net         | Voltage |
|-------------|---------|
| Vccadc      | 1.80    |

> Decision: on chip reference will be used, so set Vccref to GND and GNDDADC as appropriate.

See <https://docs.amd.com/r/en-US/ug480_7Series_XADC/XADC-Pinout-Requirements>. At a quick glance
Vrefp should be set to GNDADC and Vrefn should be set to GND.

### DDR Voltages

| Net         | Voltage |
|-------------|---------|
| Vccddr      | 1.50    |
| Vccddrt     | 0.75    |

## Selecting bank voltages

Options for bank voltage selection:

* Jumper on to select a resistor for adjusting a buck regulator. (Numato)
* Change SMD resistors (Digilent)
* Power mux like TPS2116DRLR. (Alacrity) The advantage of this solution is
  that the selection can be made at a distance, potentially on the expansion
  board.

> Decision: TPS2116DRLR

## Multi Rail Options

The power system needs to support up to 5 digital rails, plus one analog
rail. And this assumes that a 5V is not going to be passed along to
daughter cards. If there is also going to be a 5V rail, then there actually
6 rails + 1 analog.

There are no good options for a single chip to do this, i.e. the best option
that is stocked at JLCPCB at the time this is written is ADP5052ACPZ-R7, which
has 4 buck and 1 ldo output.

> Decision: Design around the [TPS563257DRLR](https://jlcpcb.com/partdetail/TexasInstruments-TPS563257DRLR/C20539656)
> which is in large supply at JLCPCB.
> (1.2Mhz, 3V-17V input, adjustable, 3A output.) Use 1 per rail.

Only support common voltages. Since the board will use DDRL, don't
support 1v5. Don't support 2v5 since it's only used on older peripherals.

> Decision: only support 1v0, 1v8, 1v35, 3v3.

This cuts down the necessary board space, and while it could warrant
revisiting the rail management, it's already designed. Switching to something
like the ADP5042 would likely result in using less board space, but
comes with greater supply chain issues at JLCPCB as it is unclear
if they will continue to be stocked as they are in lower quantities.
The TPS563257 has thousands available, and is very inexpensive. Plus,
the ADP chip only takes up to 6V, so a second regulator would still
be needed when using 9-12V power.

> Decision: use only a single 1V8 rail. It's ok if it is used as an IO
> voltage, since it is acceptable for the to come up at the same time.

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

## Capacitors

### Vccint

| 680 µF | 330 µF | 100 µF | 47 µF | 4.7 µF | 0.47 µF |
|--------|--------|--------|-------|--------|---------|
| 0      | 1      | 0      | 0     | 3      | 5       |

### Vccbram

| 100 µF | 47 µF | 4.7 µF | 0.47 µF |
|--------|-------|--------|---------|
| 1      | 0     | 0      | 1       |

### Vccaux

| 47 µF | 4.7 µF | 0.47 µF |
|-------|--------|---------|
| 1     | 2      | 5       |

### Vcc0 Bank 0

| 47 µF |
|-------|
| 1     |

### Vcco (all other banks)

| 47 µF | 4.7 µF | 0.47 µF |
|-------|--------|---------|
| 1^     | 2      | 4       |

^47 (or 100uf) can be shared by up to 4 banks

### Capacitor selection

### Table 2-5: PCB Capacitor Specifications

| Ideal Value | Value Range      | Body Size        | Type                                         | ESL Maximum  | ESR Range               | Voltage Rating | Suggested Part Number         |
|-------------|------------------|------------------|----------------------------------------------|--------------|-------------------------|----------------|-------------------------------|
| 680 µF      | C > 680 µF       | 2917 / D / 7343  | 2-Terminal Tantalum                          | 2.0 nH       | 5 mΩ < ESR < 40 mΩ      | 2.5V           | T530X687M006ATE018           |
| 330 µF      | C > 330 µF       | 2917 / D / 7343  | 2-Terminal Tantalum                          | 1 nH         | 5 mΩ < ESR < 40 mΩ      | 2.5V           | T520V337M2R5ATE025           |
| 330 µF      | C > 330 µF       | 2917 / D / 7343  | 2-Terminal Niobium Oxide                     | 1 nH         | 5 mΩ < ESR < 100 mΩ     | 2.5V           | NOSD337M002#0035             |
| 100 µF      | C > 100 µF       | 1210             | 2-Terminal Tantalum Ceramic X7R or X5R       | 1 nH         | 1 mΩ < ESR < 40 mΩ      | 2.5V           | GRM32ER60J107ME20L           |
| 47 µF       | C > 47 µF        | 1210             | 2-Terminal Ceramic X7R or X5R                | 1 nH         | 1 mΩ < ESR < 40 mΩ      | 6.3V           | GRM32ER70J476ME20L           |
| 4.7 µF      | C > 4.7 µF       | 0805             | 2-Terminal Ceramic X7R or X5R                | 0.5 nH       | 1 mΩ < ESR < 20 mΩ      | 6.3V           | GRM21BR71A475KA73            |
| 0.47 µF     | C > 0.47 µF      | 0603             | 2-Terminal Ceramic X7R or X5R                | 0.5 nH       | 1 mΩ < ESR < 20 mΩ      | 6.3V           | GRM188R70J474KA01            |
