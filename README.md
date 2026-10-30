# SmartBlinds

An electronics and mechanical retrofit to turn IKEA FRIDANS roller blinds into smart blinds, using a geared DC motor, USB-C power, and an ESP32-C6 running ESPHome.

## Project Status

Early design stage. This document defines the proposed architecture and the measurements needed before selecting final components. Schematics, PCB layouts, mechanical CAD, and tested ESPHome firmware are not yet available. This is not a validated assembly guide.

## Goals

- Motorized opening, closing, and stopping of an IKEA FRIDANS blind.
- A compact, securely mounted motor adapter and electronics enclosure.
- A regulated 12 V motor supply derived from a USB-C Power Delivery charger.
- An ESP32-C6 controller with 2.4 GHz Wi-Fi 6 and IEEE 802.15.4 hardware for Zigbee and Thread.
- ESPHome firmware, initially controlled locally through Home Assistant over Wi-Fi.
- Zigbee and Matter control as development goals, subject to firmware and integration support.
- Local controls, travel limits, and fault handling that do not depend on network availability.

## System Architecture

```text
USB-C PD charger and cable
            |
USB-C PD sink controller (request 15 V)
            |
Protected input power path
            |
12 V buck converter ---------> H-bridge ---------> Geared DC motor
            |                     ^
3.3 V buck converter              | control / fault / current feedback
            |                     |
        ESP32-C6 -----------------+
            |
    Limit sensors and local buttons
            |
    Wi-Fi / future Zigbee or Matter integration
```

All low-voltage sections share a common ground. Motor current must return through the power path, not through the controller's ground wiring.

## Mechanical Design

### Motor Reference

The starting reference is this [12 V geared DC motor](https://www.amazon.ie/dp/B08QZK8ZQ1) (ASIN `B08QZK8ZQ1`). The linked listing identifies the selected variant as 20 RPM. Confirm the supplied motor's exact voltage, speed, shaft dimensions, and mounting pattern before designing an adapter. The listing contains inconsistent voltage metadata and does not establish reliable torque or stall-current ratings.

Motor selection must be based on measured blind load, not speed alone. Record running current, startup current, safe stall-current data, required output torque, travel time, and gearbox duty cycle. Obtain stall data from the manufacturer or a brief, current-limited test; do not hold the motor stalled.

### Coupling and Mounting

- Inspect the FRIDANS rod-operated mechanism and choose a drive interface that actually actuates it. Do not assume it accepts a simple rotary coupling or behaves like a chain-driven blind.
- Decide whether to actuate the existing mechanism or replace its drive interface with an adapter to the roller. Any brake or locking function must be understood before modification.
- Measure the roller interface, available clearance, mounting points, and required travel before producing CAD.
- Use a positively retained shaft coupling and a bracket that reacts motor torque without stressing the blind's plastic housing.
- Support loads independently where needed; do not use the motor shaft as an unsupported structural bearing.
- Include a serviceable disconnect or clutch if manual operation is required. Do not assume the gearbox is back-drivable or self-locking.
- Guard moving couplings, retain the blind securely, and provide cable strain relief.

Use the required roller revolutions and loaded output speed to estimate travel time. Check torque throughout the full travel because roller radius, friction, and any spring assistance can change the load.

## Electronics

### USB-C to 12 V Power

**Recommended baseline: negotiate a fixed 15 V USB-PD contract, then regulate it down to 12 V.** A USB-C connector alone does not provide 12 V. Chargers initially provide 5 V, and a fixed 12 V PD profile is not guaranteed.

1. Use a USB-C PD sink board based on a controller such as the STUSB4500, configured in nonvolatile memory to request 15 V at the required current. Its standalone negotiation avoids depending on ESPHome to establish power.
2. Select a charger that explicitly offers the required 15 V profile and enough current. Size the cable, connector, and negotiated contract for the measured load and conversion losses.
3. Add an appropriately rated fuse or eFuse, input protection, and controlled inrush. Follow the PD controller reference design for CC connections, power-path MOSFETs, discharge, and decoupling.
4. Use a buck converter rated for the maximum possible input voltage and motor startup demand, with its output set to 12 V. If 20 V negotiation is later allowed, every input-side component must tolerate it with margin.
5. Generate a separate regulated 3.3 V logic rail with enough transient capacity for the ESP32-C6 and peripherals. For a development board, follow its documented supply input requirements instead of assuming its power pins match a bare module.
6. Gate motor operation on a valid power contract and a healthy 12 V rail. Remain stopped if negotiation fails, the charger falls back to 5 V, or power is renegotiated.

A direct 12 V PD-trigger solution is an alternative only with a charger verified to advertise a compatible 12 V profile. A trigger requests voltage; it does not convert an unsupported profile into 12 V.

Final power ratings depend on motor measurements. As a starting calculation, budget 12 V times the motor's current demand, plus logic power and converter losses, with startup margin. Coordinate hardware current limiting with the PD contract so startup does not repeatedly reset the charger.

### Motor Driver and Protection

Use a bidirectional H-bridge with 3.3 V-compatible inputs, hardware current limiting, and fault feedback. The **DRV8876** is a candidate with integrated current sensing and regulation; its advertised 3.5 A peak rating is not a continuous-current guarantee. Confirm thermal performance and measured motor demand before choosing it or a higher-current alternative.

- Connect the motor only to the H-bridge outputs, never directly to ESP32 GPIOs.
- Provide direction and PWM control, plus a hardware-disabled default using the driver's sleep/enable input and suitable pull resistors.
- Bring fault and current-sense signals to the controller, keeping every GPIO/ADC input within its specified range.
- Fit local ceramic decoupling and appropriately sized bulk capacitance near the bridge. Add motor-noise suppression based on testing.
- Account for regenerative energy during braking: a buck converter or USB charger may not absorb it. Size the capacitance and any clamp or braking circuit to keep the rail within ratings.
- Use the bridge manufacturer's recirculation scheme; do not place a single flyback diode across a reversible motor.
- Route motor wiring away from the antenna and sensor signals. Rate connectors and PCB traces for the protected motor current.

### Controller and Inputs

Prototype with an ESP32-C6 development board; an ESP32-C6-WROOM-1 module is a candidate for a later PCB. Preserve antenna clearance and access to programming, boot, and reset functions.

Reserve signals for motor direction/PWM, driver enable, driver fault, current feedback, two travel-limit sensors, and local open/close/stop controls. Select actual GPIOs only after choosing the board and checking strapping pins, flash connections, USB/debug functions, and ADC availability.

Use normally closed limit switches where practical so a broken wire is detectable. Limits should block motion toward the active end while still allowing movement away. Consider an independent hardware inhibit for overtravel; software interlocks alone are not a safety guarantee.

## Connectivity and ESPHome

The ESP32-C6 supports Wi-Fi 6 on **2.4 GHz only**, plus IEEE 802.15.4 for Zigbee or Thread. Matter is an application protocol over Wi-Fi or Thread, not a separate radio and not a protocol carried over Zigbee. Hardware capability does not guarantee that all protocols work concurrently in the chosen firmware.

| Control Path | Project Approach | Constraints |
| --- | --- | --- |
| Wi-Fi | Initial implementation: ESPHome native API with Home Assistant | Use the ESP-IDF framework for ESP32-C6, encrypted API access, and authenticated OTA updates. |
| Zigbee | Follow-up prototype using ESPHome's Zigbee component | Current documentation lists sensors, binary sensors, switches, and numbers, but not covers or a native window-covering cluster. A command adapter or further component work is needed. |
| Matter | Research and integration milestone | Native ESPHome Matter support is not established by the documentation reviewed. Evaluate a maintained external component or a compatible external Matter bridge, and verify blind support end to end. |

ESPHome's current Zigbee documentation supports Wi-Fi station mode alongside a Zigbee end device, warns of instability with Wi-Fi plus a Zigbee router, and does not support Wi-Fi access-point mode alongside Zigbee. Start with separate, tested configurations or a Zigbee end-device role; do not promise simultaneous Wi-Fi, Zigbee, and Matter/Thread operation.

A Home Assistant connection does not automatically expose this blind to Matter controllers. Any bridge must explicitly support exporting the cover entity. Native Matter firmware outside ESPHome would change the firmware requirement and is not the baseline for this project.

### Firmware Requirements

- Model the blind as an ESPHome cover with open, close, stop, and calibrated position commands.
- Route local and network commands through the same interlocked motor-control logic.
- Start with the motor disabled at boot and keep it disabled during updates, faults, or invalid supply conditions. Never automatically resume motion after a reset.
- Stop before reversing, with a tested reversal delay and appropriate driver control sequencing.
- Stop at the corresponding travel limit, on excessive current, on driver fault, or when maximum travel time expires.
- Treat both limits active together as a fault. Define a safe policy for sensor failures and network loss; local stop and protection must remain available.
- Calibrate opening and closing separately. Timed position estimates can drift and must be re-referenced at known endpoints; use encoder feedback if greater accuracy is needed.
- Do not treat current sensing as certified obstruction or anti-pinch protection. Establish thresholds experimentally without relying on repeated hard stalls.
- Keep credentials out of version control and pin the ESPHome version once a configuration is validated.

## Preliminary Bill of Materials

| Item | Proposed Selection | Still to Confirm |
| --- | --- | --- |
| Blind | IKEA FRIDANS | Exact size, mechanism, load, and interface |
| Motor | Linked 12 V, 20 RPM geared DC motor or equivalent | Torque, current, shaft, noise, and duty cycle |
| Controller | ESP32-C6 development board; module for final PCB | Board, flash capacity, and GPIO allocation |
| USB-C supply | Certified USB-PD charger and suitable cable | 15 V profile and available current |
| PD sink | Configurable STUSB4500-based board | Programmed PDOs and contract-status access |
| Motor regulator | 15 V to 12 V buck converter | Startup capacity, efficiency, and thermal margin |
| Logic regulator | 12 V to 3.3 V buck, or board-compatible supply | Controller peak demand and supply arrangement |
| Motor bridge | DRV8876-based board or suitably rated alternative | Current limit, continuous current, and cooling |
| Travel detection | Two limit switches or equivalent sensors | Mounting, wiring, and fault behavior |
| Local controls | Open, close, and stop buttons | Debouncing and enclosure placement |
| Protection | Fuse/eFuse, decoupling, ESD and rail-clamp provisions | Ratings after electrical measurements |
| Mechanics | Bracket, adapter, fasteners, enclosure, strain relief | CAD and fit testing |

## Development and Validation

1. Inspect and measure the FRIDANS mechanism; confirm a workable drive interface before committing to CAD or a PCB.
2. Test the motor with a current-limited bench supply and measure load, speed, current, and temperature across the blind's travel.
3. Validate PD negotiation and regulated rails without the motor. Confirm safe behavior with a 5 V-only source, unsupported profiles, unplugging, and renegotiation.
4. Test the bridge with the motor uncoupled, including direction, stop, current limiting, fault reporting, and boot/reset behavior.
5. Fit the bracket and coupling, then validate travel sensing and fault handling under supervision before enabling remote motion.
6. Implement and calibrate the ESPHome Wi-Fi cover. Test local stop, reversal, timeout, reboot, OTA, and network-loss behavior.
7. Test complete cycles and temperatures under realistic load, including safe obstruction-response tests and rail transients during braking.
8. Prototype Zigbee command mapping and Matter integration separately; record the exact firmware versions, coordinator/bridge, and supported commands.

Use only a certified enclosed mains charger; this project is limited to its low-voltage output. Disconnect power before wiring or mechanical adjustments. Keep hands clear of moving parts, retain existing blind safety features, and do not leave the prototype operating unattended.

## References

- [Reference geared DC motor](https://www.amazon.ie/dp/B08QZK8ZQ1)
- [ESPHome ESP32 platform and ESP32-C6 support](https://esphome.io/components/esp32/)
- [ESPHome cover components](https://esphome.io/components/cover/)
- [ESPHome Zigbee support and limitations](https://esphome.io/components/zigbee/)
- [STUSB4500 standalone USB-PD sink controller](https://www.st.com/en/interfaces-and-transceivers/stusb4500.html)
- [DRV8876 H-bridge driver and datasheet](https://www.ti.com/product/DRV8876)

Connectivity notes were checked against ESPHome's published documentation on 2026-10-30. Recheck support when selecting the firmware version.

SmartShades is an independent DIY project and is not affiliated with or endorsed by IKEA.