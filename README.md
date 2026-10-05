# Nightlight

An STM32F446RE-based automatic nightlight that measures ambient light with a photoresistor, and as the environment becomes darker, turns the LEDs on sequentially and smoothly increases their brightness. The system uses a hardware-timer-triggered ADC, interrupt-driven processing, and PWM output.


Demo: [Link](https://www.youtube.com/shorts/aKtNGqWCgg0)

---

## Overview
Nightlight converts a real-world analog light measurement into a digital control signal and uses that signal to control three LEDs.

The system follows this signal path:

(Insert Diagram)

The ADC is triggered periodically by a hardware timer, and all sensor processing occurs inside the ADC interrupt callback. The main loop remains entirely event-driven.

---
## Goals

- **Characterize the photoresistor**  
  Measure the photoresistor's resistance across different light levels and use the results to design an appropriate sensing circuit.

- **Convert light measurements into LED brightness**  
  Build an embedded system that converts the photoresistor's analog measurement into PWM-controlled LED brightness.

- **Test the ADC using the onboard DAC**  
  Use the STM32's onboard DAC to sweep through a range of analog values and simulate the photoresistor, providing a controlled way to test the ADC and LED response.

- **Produce a visually smooth brightness response**  
  Develop a perceptually appropriate mapping from the ADC measurement to PWM duty cycle rather than using a simple linear mapping, resulting in a smooth and natural brightness transition.

---

# 1. Hardware

## Components

| Component | Purpose |
|---|---|
| NUCLEO-F446RE | Main microcontroller |
| Photoresistor | Ambient light sensor |
| Red LED | RGB nightlight |
| Green LED | RGB nightlight |
| Blue LED | RGB nightlight |
| 1 x 22kΩ Resistor | Photoresistor voltage divider |
| 3 x 200Ω Resistor | LED current limiting |
| Breadboard | Circuit assembly |

---

## Circuit Design
### Photoresistor Characterization
To characterize the photoresistor, its resistance was measured under different ambient light conditions using a multimeter while the photoresistor was exposed to several levels of illumination ranging from relatively dark conditions to bright light. For each lighting condition, the measured resistance was recorded.

The purpose of this characterization was to determine the relationship between ambient light intensity and the resistance of the photoresistor. A photoresistor is a light-dependent resistor whose resistance generally decreases as the amount of incident light increases. Therefore, the measured resistance provides an indirect indication of the surrounding light level.

The collected measurements were used to establish the operating range of the sensor and to determine suitable resistance values for the nightlight circuit to distinguish between dark conditions, when the nightlight should be brighter, and brighter conditions, when the nightlight should be dimmer or turned off. The measured data can also be plotted as resistance versus light level to visualize the response of the photoresistor and identify its useful operating range.

<img width="597" height="372" alt="image" src="https://github.com/user-attachments/assets/7b443e7f-4694-4993-8e3e-dffda48b2c06" />

The resistance of the photoresistor increases almost exponentially as the amount of light decreases. Although these were arbitrarily chosen light levels and were not measured in lux, the results clearly show that the resistance increased sharply as the amount of light it was exposed to decreased. This agrees with external research, which shows that photoresistors have a nonlinear, approximately exponential relationship between resistance and light intensity, with resistance increasing substantially as illumination decreases.

### Photoresistor Voltage Divider 
Although the photoresistor can reach extreme values from 500 Ω to 1 MΩ,
the nightlight does not need to map the entire 0–3.3 V ADC range to useful LED
brightness. Instead, we decided to optimize the nightlight for a resistance
range of approximately 1 kΩ to 500 kΩ.

The photoresistor and fixed resistor form a series circuit. From Ohm's law:

$$
V=IR
$$

Since resistors in series carry the same current, their total resistance is:

$$
R_{total}=R_{LDR}+R_{fixed}
$$

Therefore, the current through the voltage divider is:

$$
I=\frac{3.3}{R_{LDR}+R_{fixed}}
$$

The ADC is connected across the fixed resistor, so the voltage measured by the
ADC is:

$$
V_{ADC}=IR_{fixed}
$$

Substituting the expression for current gives the voltage-divider equation:

$$
V_{ADC}=3.3\frac{R_{fixed}}{R_{LDR}+R_{fixed}}
$$

I plotted the voltage-divider response in Desmos over the 1 kΩ to 500 kΩ
operating range to determine a practical fixed resistance. A value around
20 kΩ provided a good compromise between sensitivity at lower and higher
light levels, so a standard 22 kΩ resistor was selected.

<img width="1270" height="836" alt="image" src="https://github.com/user-attachments/assets/6b569e64-9868-4f62-86a4-464a7e732e6b" />


With a fixed resistor of 22kΩ, the expected ADC voltage over the selected
operating range is approximately:

- High ambient light: $R_{LDR} \approx 1kΩ$, so $V_{ADC} \approx 3.16V$
- Low ambient light: $R_{LDR} \approx 500kΩ$, so $V_{ADC} \approx 0.14V$

This provides a large ADC voltage range over the portion of the photoresistor's
range that the nightlight is designed to use, while avoiding over-optimizing
the circuit for the extreme 500 Ω and 1 MΩ measurements.

### LED Current Limiting 

The forward voltage drop of an LED is color dependent, so we measured the forward voltage of each LED and used these measurements to estimate the current through the LEDs. The current-limiting resistor was selected using Ohm's law:

$$
R=\frac{V_{GPIO}-V_F}{I_{LED}}
$$

ST specifies that the GPIOs can source or sink up to 8 mA under the specified output-voltage conditions. The total current across the GPIOs must also remain within the device's absolute maximum ratings.

We selected 200 Ω resistors for all three LEDs. Using the measured forward voltage of each LED, the expected current can be calculated as:

$$
I_{LED}=\frac{3.3-V_F}{200\Omega}
$$

For example, if the measured forward voltage of an LED is $V_F=2.0$ V:

$$
I_{LED}=\frac{3.3-2.0}{200}
\approx 6.5\text{ mA}
$$

This keeps the LED current below the STM32's 8 mA GPIO specification under the normal output-voltage conditions while providing sufficient current for visible illumination. Although we do not know the exact current rating of the LED, by keeping the current Using a larger resistor would further reduce GPIO and LED current, but would also reduce the available LED brightness.

Because the three LEDs are driven independently, the maximum combined LED current is approximately the sum of their individual currents. This remains well below the STM32F446RE's total GPIO current limits.

### Final Schematic

We used PA4 for the photoresistor because it can be configured as an ADC input and also supports DAC output on the same physical pin, which allows the onboard DAC to be connected to the ADC through PA4 during the self-test (where we simulate a photoresistor sweeping through resistance values) without requiring an additional pin.

We used PA6, PA7, and PB0 for the LEDs because they correspond to TIM3 channels 1, 2, and 3, supporting TIM3 PWM outputs, allowing the LED brightness to be controlled directly by hardware timers. Using three channels of the same timer also allows all three LEDs to share the same frequency while their duty cycles are independently controlled.

<img width="1207" height="737" alt="image" src="https://github.com/user-attachments/assets/73398cef-ec19-49ac-92fa-bb13ac3366db" />
---
# 2. Firmware



