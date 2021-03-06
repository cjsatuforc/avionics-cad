# Features and Behaviors of the RocketNet Hub

TODO: This is just a rough draft.

## Modes

0. Boot mode
   - Check out saved state (armed/active).
1. Sleep mode (idle)
   - KSZ completely off.
   - All nodes are off.
   - Set interrupt on external GPIO (ACOK rising edge).
   - Go into deep sleep (all clocks off).
   - Goal: absolute minimum power consumpt (&lt; 5 mA)
   - Charge battery if shore power on (requires wakeup every 175 s to ping the BQ)
2. Debug mode (utility mode)
   - Ignores shore power on/off, otherwise just like active mode
   - Can not transition from debug to armed.
2. Active mode (startup/running mode)
   - KSZ is on.
   - One or more nodes are on.
   - Awake and looking for packets, acting on packets.
   - Monitoring battery voltage.
   - Charging battery when possible.
   - Can load power configurations (see below)
3. Armed mode (in flight)
   - Write armed state to file + last power configuration in case of restart
   - Everything on that should be on
      - Default configuration
      - Power configurations mapped to flight status?
   - Actively turning things back on that over current some number of times.
   - Resetting the KSZ and if no packets are getting through.
   - Resetting nodes (including the FC) if watchdog packets don't come through.
   - Broadcasting information on node power switches, current, voltage, etc.


## Mode Transitions

- Boot -> Armed
   - State saved file
- Boot -> Active
   - Shore power on
- Boot -> Sleep
   - Shore power off

- Sleep -> Active: 
   - Shorepower on (ACOK rising edge)

- Active -> Sleep:
   - Shore power: off
   - Nodes: all off || Vsys &lt; 12 V
   - Sleep command received

- Active -> Armed:
   - Arm command received (written to flash)

- Armed -> Active:
   - Disarm command received

- Active <-> Debug:
   - On debug start/stop command

- Debug -> Sleep:
   - Go to sleep
   
### Notes

- Send a "you're going to get shut down" packet some time before shutting off a node

## Command packets

- Turn on/off the KSZ
- Turn on/off any node
- Go to sleep
- Arm
- Disarm
- Transmit data packets
- Node-sepcific watchdog packets (each node sends it a ping, else the RNH power cycles it)


## Data packets

- Time
- Mode
- Shore Power Information
  - Shore power on/off
  - Shore power current (via IOUT)
- Battery charging info
  - Charging on/off
  - Battery charge current (via IOUT)
  - System voltage (shore power voltage when shore power on, battery voltage when shore power off)
- Power switches
   - KSZ power state
   - Node switch state, current from each node
- Future: BQ3060 battery state
   - Precise battery voltage, current, SOC, charge/discharge status, etc.
- Watchdog
   - Number of watchdog reboots
   - Time since last packet from each node

## Notes

- What's the current draw in each mode? How many hours can we last in each?
- We should send "arm" as a broadcast packet, and everyone could record the timestamp to sync their clocks.
- GPS could broadcast precise clock packets?
- Pre-broadcast-time packet could clear bus, then send time packet, to get real-time behavior.


## Quiescent current / "Shelf life"

Since:

- BQ24725 draws 20 uA when running directly from the battery without charging.
- The LTM8023 draws a couple of hundred microamps with no load.
- The STM32 in sleep mode draws hundreds of microamps.
- We have some current bleed through the link LEDs since they're running off the 3.3V supply (and why are they?).

Based on this, we should expect the quiescent draw of the whole RNH board of probably ~ 5 mA. Measured on bring up was ~ 3 mA. At 3 mA, that's about 1.5 months of life on a **fully charged** battery pack. On a fully depleted pack, this could kill the pack in a day or two. For this reason, we need to disconnect the battery when shore power is off and the avionics system is just sitting on the shelf.





 
