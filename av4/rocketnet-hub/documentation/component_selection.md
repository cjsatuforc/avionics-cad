# Rocketnet Hub Component Selection Guide

This document aims to explain the component selection for the "RocketNet Hub" (RNH) power and communications hub designed by the 2012-13 PSU ECE capstone team. The intention is to provide insight into the design decisions made where it might not be obvious ; there is no intention to provide the reader with a comprehensive design example that is suitable for any purpose other than exact replication. The target audience is expected to have design experience with microelectronics power distribution, modern physical layer communications designs, and understand matched impedance differential signaling.

Also, to fully understand these design decisions, see the requirements and specifications for this project here: <http://psas.pdx.edu/avionics/capstone2012/>

----------------------------------------------------------------------------------------------------------------------
# SCHEMATIC SHEET 1: Microcontroller (MCU)

## U1, MCU, ST Microelectronics STM32F407V100

For this project the MCU chosen was to not only to be used for our capstone design but also slated to replace the LPC2468 MCU used on the previous version Generic Front End (node4 GFE) boards. Thus despite a push from the capstone team to use an Atmel AV32 or Atmel ARM MCU it was decided by PSAS that an ST Micro STM32F407 MCU would be used. This microcontroller does offer some cost advantages over Atmel's offerings both in per unit cost and the cost of development tools. Given that PSAS is a cost conscious group that has ample expertise to draw from, the higher quality documentation offered by Atmel did not justify the added expense. Several Olimex STM32-E407 development boards where acquired and used by the PSAS software team to concurrently develop firmware for the RNH.

## Support Components

### Crystal (C106, C107, and X1 -- 8Y-25.000MEEQ-T)

The MCU has an internal oscillator circuit and can run without an external crystal. However, for Ethernet functionality the MII interface must operate at 25MHz +/- 50ppm. This is beyond the capabilities of the internal oscillator. Thus an external crystal must be used. The value that makes the most sense to the design team is 25MHz. By using this value we can avoid mucking around with the PLL resources of the MCU and still provide the required MII interface clock. Additionally, the MCU has two clock outputs and we can use one of these to buffer the crystal and use this same crystal to also drive the KS8999 Ethernet switch. The part selected is the 8Y-25.000MEEQ-T from TXC Corp. This crystal has a load capacitance rating of 10pF and a +/-10ppm frequency stability/tolerance. The load capacitors should be selected with 1% or less tolerance and C0G temp. coefficient or better.

### Bypass Caps

The supply pins of the MCU will each be provisioned one 0.1uF ceramic cap. X5R or X7R 0402 ceramic capacitors should be used. A bulk capacitor with a nominal value of between 10 and 22 uF should be placed near the MCU's 0.1uF bypass caps. The bulk capacitance value is not critical so any temp. coef. 0603 ceramic cap will work. The two crystal pins also require capacitors. For these two C0G ceramic 0402 capacitors should be used. The value should be 10pF.

### Analog Supply Filtering (E5)

The Analog supply pins were isolated with an 0603 ferrite bead to block digital switching noise. As the only analog resource used in the MCU was the ADC to monitor the current draws of both the battery and each node, this was likely an overly conservative addition to the board. During testing it is recommended that the ferrite bead be replaced with a 0 Ohm resistor. This will allow for a reduced BOM for future revisions of the board should it prove functional without this filter.

## Interfaces

### Reset Logic

The MCU has a reset signal that needs to be weakly pulled high during normal operation. This is accomplished by a 10kOhm resistor. The value is chosen to reduce power consumption when the line is pulled low. (Though given the length of time that this signal is ever likely to be held low it is a really rather pointless consideration. Any value from 100 to 100k could be used.) The MCU has internal power on reset and brownout detection circuitry and thus no external reset logic was added to ensure proper response to these conditions.

### JTAG (J22)

The MCU supports both programming and debugging through the ARM standard JTAG interface. This bus is broken out to a 2x5 50mil header. Signals used are the TMS, TCK, TDO, TDI, and MCU reset. For more information on these signals or the JTAG implementation used in the STM32F407 please see google.com

### Serial UART (J10)

To support realtime debugging and logging, a serial UART channel was broken out to a 1x4 50mil header. Signals preset are TXD and RXD, no flow control signals are present. This is a 3.3V interface with no protection. Care should be used to ensure that the USB to UART adapter used is configured for 3.3V or damage will likely result to the MCU, adapter, or both.

### MII

The MCU is connected to the Ethernet switch, discussed later, by two interfaces. The MII interface is the connection between the MAC and switch. The other interface is a serial interface and will be discussed below. The MII interface is an industry standard interface for connecting a MAC to a PHY. It is implemented per the standard with the exception that neither TX\_ERR or RX\_ERR are present. More information on this interface can be found via Goolge.com.

### I2C (J11, R15, R17)

    The I2C, or more properly TWI, interface is used to interface with the battery charger IC and fuel gauge IC. The MCU is the bus master. This is an open collector interface and thus requires the presence of a pull up resistor on each signal line. 2.2kOhm 0403 resistors were selected from past experience--no calculations or simulations were performed. If the bus fails at desired baud, please scope the bus at J? and look for excessive skew. If this is found, decrease the value of the pull ups. If reduction in power consumption is desired, these may be increased until such point as the bus begins to fail at max baud.

### Rocket\_RDY (Q1, U3)

The MCU is responsible for sending the final go signal to the launch tower signifying that the rocket is ready to launch. This is accomplished through a non-inverted signal buffered by a simple N-channel MOSFET. The signal is pulled high on the gate side and low on the source side. It is isolated from the MCU by the FET and protected by a 100Ohm resistor and parallel TVS diodes. The Capstone team wished to implement this signal differentially due to the 9m long cable that it must be sent over, but the cable had already been assembled. Future revisions of this board and the Launch Tower interface should explore further the benefits of differential signaling.

## Debugging

### RBG LED (LED37)

A single RGB LED was added to facilitate simple state display of the MCU. The implementation of the state machine and assignment of colors is left to the firmware team.

### Serial UART (J10)

A serial UART port is provided to facilitate realtime logging via print statements. The implementation of this functionality is left to the firmware team. Using the boot mode selection headers it is also possible to load a boot image over the UART port without flashing the internal memory. This can be used during initial code bring up to reduce time wasted writing images to flash. 

### JTAG (J22)

The STM32F407 supports JTAG programming and debugging. This is useful for stepping through code but is not appropriate for realtime debugging. This bus is intended only for initial programming and code bring-up.

### BOOT SEL (J5, J12)

The STM32F407 supports multiple boot sources and this can be selected by pulling the BOOT\_SEL pins either high or low. In conjunction with the UART port it is possible to load a boot image directly from a host computer without flashing the internal memory. This can be useful for initial code bringup. Please see the datasheet for boot configurations.

## System Monitoring

### Current Draw (U14, U15)

The MCU is configured to monitor the current draw of each connected node and the battery. This is accomplished through the internal ADC and two external analog multiplexers. At any time the MCU is monitoring one node from NODE\_1 through NODE\_4 and one node from NODE\_6 through NODE\_8, the only exception to the previous statement is that when monitoring NODE\_1, it is the only node monitored as there is no NODE\_5. The node currents are measured on NODE1\_4\_IMON and NODE5\_8\_IMON. Selection of the nodes to monitor is accomplished by setting the select bits IMON\_A0 and IMON\_A1. The battery's charge current is also monitored. This is present on BAT\_IMON. Use of this data is left to the firmware team, though possible uses would be to shut down a node if it is drawing unusually high or low amounts of current or determining if there is enough current sourcing capacity to turn on a given sub-system.

## System Control

### Ethernet Switch power supply and reset (C12, D1, Q8, Q9, R43, R65)

The MCU is responsible for enabling the Ethernet switch. This is done by asserting ETH\_EN. This will enable the switch's 2.1V LDO and access to 3.3V IO supply. Once both supplies stabilize, the MCU must boot strap the Ethernet switch using the SPI3 serial interface. This is a hack. The switch is designed to function as an I2C slave on a dedicated bus but the MCU only has a single I2C interface and that bus has other devices on it. The proper solution would have been to use a bi-directional tri-statable logic buffer to isolate the Ethernet switch from the rest of the I2C bus. However, the decision came down from PSAS to use the SPI interface. Implementation is left to the firmware team. The Ethernet switch can be reset without powering down its supplies by clearing ETH\_RST\_N. Note: this signal should not be tri-stated and driven to a known state prior to asserting ETH\_EN. Please note that C12 and R43 create a delay in ramping the 3.3V supply to allow the 2.1V core supply to come up first. D1 ensures 
that this supply is shut off quickly.

## Node Power

Each node can be turned on or off by the MCU. This is accomplished by clearing or setting (respectively) the NODEx\_!EN signal. In the case that a node draws more current than it is intended to and triggers the over-current protection system, the NODEx\_!FLT signal will be asserted. It is left to the firmware team to determine proper response to this signal.


-----------------------------------------------------------------------------------------------------------------------
# SCHEMATIC SHEET 2: Node Power Switches and Connectors

## TPS2420: 3-V to 20-V INTEGRATED FET HOT SWAP CONTROLLER (U7, U8, U9, U10, U11, U12, U13)

Each of the 7 nodes to be connected to this distribution board was to have its own power switch controlled by the MCU. After much debate the TPS2420 from TI was selected. This IC offered the design team many benefits though it also provided unnecessary functionality. Our main requirements for the node power switch were that it provides a high level of integration with no external components except small capacitors and resistors. An integrated FET was a must. Over current protection was also a must. The TPS2420 was selected by PSAS from TI's offering of hotswap power switches by PSAS. The justification was that it also included a soft current limit that could be exceeded for a limited time and the ability to externally set its reset behavior. The package size was also of utmost consideration. The TPS2420 comes in a standard QFN-16 with a DAP to dissipate heat. Several characteristics are dependent on the node being powered and the values of the resistors and capacitors that set them are left to PSAS to determine. The datasheet lists the formulas used to calculate each. Key settings for this chip are:

- Hard Current Limit (IMAX) (R51, R66, R67, R68, R69, R70, R71)
   - On top side of PCB near node connector for ease of change.
- Soft Current Limit (IFLT) (R121, R122, R123, R124, R125, R126, R127)
   - On top side of PCB near node connector for ease of change.
- Soft Limit Timer (CT) (C95, C96, C97, C98, C99, C100, C101)
   - On top side of PCB near node connector for ease of change.
- Reset Behavior: latch reset condition and wait for MCU to reset switch

Additionally, LED's were connected to the power good and fault signal lines, green and red respectively. These lines are common collector and thus pulled high by 10kOhm shunt resistors. The TPS2420's final benefit is a scaled version of the output current that is clamped to 2.5 V. This is used to drive an ADC channel on the MCU via an analog mux.

- IMON Resistor (R49, R52, R54, R56, R58, R60, R62) TBD
- Power Good LED (LED3, LED6, LED8, LED11, LED14, LED16, LED39) APT1608CGCK (Green 0603 V\_f 2.1V I\_d 20mA)
- Fault Cond. LED (LED2, LED4, LED7, LED10, LED12, LED15, LED18) APT1608EC (Red 0603 V\_f 2V I\_d 20mA)
- LED Current Limiting Resistors (R19, R23, R26, R30, R33, R36, R39, R50, R53, R57, R63, R107, R110, R112) 750 Ohm (1.6 mA - 1.73 mA)


## MAX4734: 0.8Ω, Low-Voltage, 4-Channel Analog Multiplexer (U14, U15)

The STM32F407V LQFP-100 lacks sufficient ADC inputs to simultaneously monitor the output current of all the nodes and the battery charger. The solution to this limitation was to either move to a larger package of the STM32F407 or add one or more analog muxes. The mux solution simplified routing and allowed for all the GFEs to use the same chip and package as the used for the distribution board. The specific mux chosen, Maxim's MAX4734, was selected for its small size (QFN-9) and very low Ron. The choice to use 2 4:1 muxes instead of one 8:1 mux was made to further simplify layout. The select bits for the muxes are shared meaning that at any given time the MCU is monitoring the charge current and two node currents, the only exception is IMON\_Ax = 2'b00 when only Node 1 is monitored as there is no Node 5. To simplify the firmware team's job slightly, a pull down was added to N1 of U15 to ensure that the value reported for the non-existent Node 5 is always in a known state.

- Bypass Capacitors (C10, C11)
- Standard IC bypass caps, 100nF
- Pull-down Resistors (R18)
- To account for the odd number of channels being multiplexed by the two MAX4734's one input of U15 is pulled low to provide a known value for that channel.

-----------------------------------------------------------------------------------------------------------------------
# SCHEMATIC SHEET 3: Power Supplies

The board is powered by a 4-cell LiPo battery pack. This works out to a nominal voltage of ~ 14.7 V under battery power and ~ 18 V under charging conditions. The MCU can run on 1.8 - 3.3 V. The design was intended to run on 3.3 V only initially. This became slightly more complicated after the selection of the KS8999 Ethernet Switch as this chip requires a 2.1 V core voltage. While the 3.3 V supply should be always on, the 2.1 V supply was required to be off initially and powered on when the KS8999 was enabled by the STM32.

## U5, LTM8023 (2A, 36V DC/DC Step-Down μModule Regulator)

The LTM8023 uModule from Linear Technologies was selected for its high level of integration and low external part count. The LTM8023 is capable of sourcing up to 2 A of regulated power with an efficiency of 80%+. As the KS8999 requires 1 A just by itself the 2 A 3.3 V supply seemed reasonable to source its LDO and the remaining logic. 

- Vout: 3.3 V
- Osc. Freq.: 650KHz
- Sync. Freq.: 650KHz -- set by MCU's timers (crystal locked)
- `R_T` (R74) 49.9KOhm
- `R_ADJ` (R73) 154KOhm
- `C_in` (C103) 2.2uF (0805)
- `C_out` (C104) 22uF (1206)
- `PGOOD` LED (LED22) APT1608CGCK (Green 0603 `V_f` 2.1V `I_d` 20mA)
- LED Current Limiting Resistor (R128) 750 Ohm (1.6 mA)
- `EN`: Tied to supply voltage, Always on

FIXME: C104 is specified as 22μF 1206  but in the schematic is specified as 2.2 uF 0603.

## U6, MIC37101-2.1YM (2.1 V 1A Low-Voltage μCap LDO)

The Micrel MIC37101 line of LDO's was selected to source power to the KS8999. According to the datasheet, the KS8999 draws (,640,850) mA in 100BASE-TX operation, so a max supply current of 1 A was chosen. The MIC37101-2.1YM is a fixed voltage LDO with a typical maximum output rating 1.6 A. This is  sufficient to drive the KS8999 with reasonable headroom to account for potential variation. All resistor values were chosen from each manufacturer's datasheet (with the exception of the LED current limiting resistor R128). The capacitors chosen for the Micrel MIC37101 were the minimum value recommended in the datasheet necessary to ensure stability. Power dissipation is (3.3 - 2.1)* 0.85 = 1.02 W of power.

FIXME: WHAT?! 1W dissipation in an LDO?! That's **CRAZY**. How did we miss this? This MUST be a SPS. In the next redesign, it's clear that the LTM uModule should have two outputs, where the second one can be sequenced independently of the first. Can we fix the current board? With a maximum drop out of 280 mA, we might be able to lift the Vin pin and power it off an SPS at 2.5 V - that's a dissipation of (2.5 - 2.1) *.85 = 340 mW. That's still a lot. Better to directly replace it with some kind of 3.3 V to 2.1 V module: at 90% efficiency for a dissipation of 10x less, ~ 100 mW. For example, A Murata LXDC55KAAA-205 could be used on a carrier board with one external feedback resistor to make a > 90% SPS in the same footprint as the SOIC8 of the MIC37101. It needs a daughterboard, however, to adapt the LGA to the SOIC18. It might want more capacitance, too. 

- EN: Weakly pulled to logic 0, controlled by U1.ETH_EN
- `C_in` (C52) 47uF
- `C_out` (C84, C85) 4.7uF, 0.1uF


----------------------------------------------------------------------------------------------------------------------
# SCHEMATIC SHEET 4: Ethernet Switch

## U4, Ethernet Switch, Micrel KSZ8999

The design requirements dictated that the switch chosen must have at least 7 ports. Furthermore, we were restricted from using any Broadcom SoC due to the company's exclusive access model. This left us with only Micrel's offerings to work with. The KS8999 had the added benefit of an MII interface to the MCU which allowed the design to avoid adding any discrete PHY IC's. It also met the design requirements by offering 8 ports with integrated PHYs and one MII digital interface. The Micrel KS8999 development board was acquired to test potential configurations so that these settings could be set on the first run of the PCB. The settings used are:

* MIIS: 2b'X1 (reverse MII mode)
* MODESEL: 4b'0000 (LED mode 0)
* CFGMODE:1b'0 (I2C Bootstrap)
* LED[8][3]:1b'0 (AutoMDIX)
* Supporting Components

## Voltage Supplies (C12, D1, Q8, Q9, R43, R65, U6)

The KS8999 required a 2.1 V core and analog IO voltage. This is not a voltage that would otherwise have been provided on the board and thus an LDO was selected to do so. Please see Power Supplies for additional design decisions related to this LDO. There IO logic requires 3.3V to match the digital IO on the MCU. The 3.3V must ramp after the 2.1V supply. To accomplish this, a RC delay circuit and PMOS power switch was implemented. The diode (D1) allows for this to be shut down without waiting for the RC delay circuit to discharge.

## Bypass Caps

The supply pins of the KS8999 will each be provisioned one 0.1uF ceramic cap. X5R or X7R 0402 ceramic capacitors should be used. Bulk capacitors with a nominal value of  10uF should be placed near the KS8999's 0.1uF bypass caps. The bulk capacitance value is not critical so any temp. coef. 0603 ceramic cap will work. 

## Pull Up/Down Resistors (R2, R3, R4, R6, R7, R8, R10, R13, R14, R72, R116, R117, R118, R119, R120)

Several of the pins on the KS8999 must be pulled to either Vdd or GND to place the IC in normal operating mode. This is accomplished with 0402 0 Ohm resistors. The interested reader should consult the datasheet for further information on the effect of these pull downs. R72 is present as the MCU MAC's interface does not provision a TX\_ERR signal and thus it is tied low on the PHY side. Additional resistors are used to set MIIS, MODESEL, and CFGMODE. These configurations are described in the datasheet.

## Reset Circuitry (D2, D3)

The KS8999's reset is not 3.3 V tolerant and thus diodes and pull up resistors must be used to ensure safe operation. The circuit is provided in the data sheet. 0402 10k resistors should be used to minimize power dissipation. Any tolerance value may be selected. The diodes are SOT-23 to minimize size. The forward voltage should be minimized to ensure the reset line can be pulled low enough to be read as a logic 0. Schottky diodes are optimal for this purpose. The CDBQR0130L from Comchip Tech was selected for this purpose.

## Signal LEDs (LED1, LED5, LED9, LED13, LED17, LED21, LED25, LED29, LED33)

Each port provisions up to four signaling LEDs to denote the current state of the port. 0402 LEDs will be used with a nominal ID of 1mA. As each node will have two power indicators, green for power good and red to indicate a fault condition, yellow/amber seems the most reasonable choice for the Ethernet status indicators. PSAS determined that only one status LED per port needed to be placed. The part number selected for this component is VLMY1500-GS08 from Vishay. Forward voltage is 2V.

## Magnetics (Chokes: L2, L3, L4, L5, L6, L7, L8, L9, L10, L11, L12, L13, L14, L15, L16, L17 Transformers: T4, T5, T6, T7, T8, T9, T10, T11, T12, T13, T14, T15, T16, T17, T18)

The physical layer of the Ethernet specification (for copper) requires that the PHY's on either side of the connection be AC coupled to each other. To accomplish this, purpose-build magnetic transformers from Pulse were selected by PSAS and tested on the dev board. Additionally, common-mode chokes were also added to improve signal integrity. The transformers are quite large components and much effort was put into attempting to use capacitors to AC couple the connection and save space. This proved fruitless and transformers were instead used. The exact reason for failing to successfully couple with capacitors is unknown and further research for future revisions of the board should be considered.

----------------------------------------------------------------------------------------------------------------------
# SCHEMATIC SHEET 5: Off RNH Connectors

## J9 Battery Connector

Purpose: Runs high current power at Vbat from the 4S1P LiPo battery pack to the rest of the avionics system. 

- Vbat = (10,14.7,16.8) V. Leads are quadrupled-up to reduce power drop.
- SCL/SDA are SMBus connections to the battery pack, which should have a BQ3060 battery monitor on it.
- !PRSNT is a signal to the BQ3060 that the battery pack is actually plugged into a system (might be better called WAKE).

## J14 Umbilical Cord Connector

Purpose: Connects to the "umbilical cord" which connects to the Launch Tower Computer (LTC). Provides shore power and shore communications

- Power leads are "shore power" that power the rocket and charge the battery while on the pad. Shore power = (18,19,20)V. Quadrupled-up to reduce power drop.
- RocketReady is a safety-critical signal in the motor ignitor path that indicates to an indpendent relay in the LTC that the rocket is ready for flight.
- TX/RX are Ethernet data lines that connect the RNH and rest of the avionics system to the outside world. This usually means to a standard Ethernet switch and a bunch of laptops during testing and development. For launch, this connection is specifically between the avionics system and the LTC.

----------------------------------------------------------------------------------------------------------------------
# SCHEMATIC SHEET 6: BATTERY CHARGING BLOCK

## U2, Battery Charger, BQ24725

The onboard battery charger had been selected by PSAS prior to the beginning of the capstone project and was presented to the capstone team as a black box solution. Integration of the dev board's circuit with minimal adjustment was requested. The interested reader is directed to the datasheet and dev board user's guide for more information on this subsystem. Connections to the black box are I2C(<->), BAT\_IMON(->), external DC power(<-), Battery power(<-), and +18V(->). All internal signals are replicated per the dev board. Most components selected are per the dev board's BOM.

## In-Rush Limiter (R11, C60)

The two 3.9 Ohm 0.5W resistors from the evaluation board were replaced with a single 2 Ohm 1 W resistor. The 2512 size was selected because it was the smallest available component.

NOTE: Page 29 of the data sheet recommends a RC filter for VCC in order to prevent inductive kick from hot-plug events destroying the IC. This was NOT implemented, and probably ought to have been since out shore power cable is so long.

## Transient Voltage Suppression (D5)

A reverse biased TVS diode (SMAJ18CA) was added to protect against over-voltage. 20V voltage breakdown was chosen for the diode based on +18V Vin shore power.

## Enable (EN) Circuitry

Removed the optional enable/disable circuitry on VCC which causes board to be always-on when plugged into a power source.

## Various Capacitors

Sizes determined by component availability of 40V caps: &gt;10uF = 1206 / &gt;2.2uF 0805 / &gt; 1uF = 0603 / &lt;1uF = 0402

## Reverse Input Voltage Protection (Q6, R132, R133)
 
These components provide system and IC protection from reversed shore power (Vin) voltage. In normal operation, Q6 is turned off by negative Vgs. If the adapter voltage is reversed, Q6 Vgs is positive. As a result, Q6 turns on to short gate and source of Q2 so that Q2 is off. The Q2 body diode in reverse biased orientation blocks negative voltage from traversing into the system. However, CMSRC and ACDRV pins need R10 and R11 to limit the current due to the ESD diode of these pins when turned on. Q6 has a low Vgs threshold voltage and low Qgs gate charge so it turns on before Q2 turns on. R10 and R11 must have enough power rating for the power dissipation when the ESD diode is on, again these were selected based on the datasheet.

## Negative Output (BAT) Voltage Protection (R144, R143)

Selected as noted in the datasheet: If the battery pack is inserted backwards into the charger output (BAT) during production or there is a hard short on the battery to ground, it will generate a negative output voltage on the SRP and SRN pins. Built in IC internal electrostatic-discharge (ESD) diodes from GND pin to SRP or SRN pins and two anti-parallel (AP) diodes between SRP and SRN pins can be forward biased and negative current can pass through the ESD diodes and AP diodes when output has negative voltage. These two resistors for SRP and SRN pins are inserted to limit the negative current level when output (BAT) has negative voltage. The recommended resistor values of 10 Ohm (R144) for SRP pin and 7.5 Ohm (R143) for SRN pin were used.

## Inductor (L1)

We used the datasheet suggested value for the inductor, L1, 4.7uH. The bq24725 has three selectable fixed switching frequencies, the higher switching frequency allows the use of smaller inductor and capacitor values. The bq24725 has charge under current protection (UCP) by monitoring charging current sensing resistor (R134) cycle-by-cycle. Details on calculations for L1 and R134 are in the bq24725 datasheet, page 27. We used the suggested value of 10mOhm for the charging current sensing resistor (R134).

## Input (C39, C43) and Output (C119, C120) Capacitors

Input (C39 and C43) and Output (C119 and C120) Capacitors based on datasheet, with detailed calculations on page 27 of datasheet.

## Power MOSFETs (Q2, Q3, Q5)

Power MOSFETs Selection (Q2, Q3 and Q5) detailed calculations on page 28 of datasheet.

## `ACDET` divider (R137, R138)

`V_in`(shore power) is considered valid when `ACDET` pin is between 2.4V to 3.15V. R137/R138 as originally designed form a voltage divider with a diver coefficient of 0.134 (66.5 kohm and 430 kohm) which means valid shore power ranges from 17.91 to 23.5 V.

FIXME: Both the low and high end of ACDET are unecessarily high; anything &gt; 16.8V (the maximum battery voltage) should be OK. With an extra 200 mV of headroom and using the maximum 2.424 V cutoff, that is 2.42 / 17 = 0.142. Changing R137 only, 0.142 implies R137 = 71.37 KOhm -> 71.5KOhm.  R137 = 71.5 Kohm then sets the valid ACDET voltage range from 16.83 to 22.1 V, which is a better range.

## `ILIM` divider (R142, R139, C122)

The charge current limiter takes the lower of the voltage set by ChargeCurrent(), and voltage on `ILIM` pin. R142/R139 set a voltage divider to set `V_ILIM` which is 3.3V * 100k/(100k+316k) =  3.3V * 0.240 = 0.793 V. From the datasheet, `V_ILIM` = 20 * Ichg * Rsr --> Ichg = `V_ILIM`/(20*Rsr) = 0.793/(20*0.010) = 3.966 A. So, just like Figure 1 in the datasheet, it's set to 4A. This is probably fine: 4A seems like a fine upper limit on charge current, we'll most likely be running much lower than that.

Note that C122 seems gratuitous; it's not in the datasheet, and it probably doesn't need to be filtered. That said, it's probably not going to hurt anything and my filter some noise out.

## `IOUT` output (C110)

Either 20x the adapter current (shore power current) or 20x the battery charge current, selectable with SMBus command ChargeOption(). 

Datasheet says "A 100pF capacitor connected on the output is recommended for decoupling high-frequency noise. An additional RC filter is optional, if additional filtering is desired. Note that adding filtering also adds additional response delay.". 

ISSUE: it's probbaly not totally necessary, but that seems like a good idea.








