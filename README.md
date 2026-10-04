#Nightlight

An STM32F446RE-based automatic nightlight that measures ambient light with a photoresistor, and as the environment becomes darker, turns the LEDs on sequentially and smoothly increases their brightness. The system uses a hardware-timer-triggered ADC, interrupt-driven processing, and PWM output.


Demo:

---

##Overview
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
| [Resistor] | Photoresistor voltage divider |
| [Resistors] | LED current limiting |
| Breadboard | Circuit assembly |

---

# 2. Characterization of the Photoresistor

To characterize the photoresistor, its resistance was measured under different ambient light conditions using a multimeter while the photoresistor was exposed to several levels of illumination ranging from relatively dark conditions to bright light. For each lighting condition, the measured resistance was recorded.

The purpose of this characterization was to determine the relationship between ambient light intensity and the resistance of the photoresistor. A photoresistor is a light-dependent resistor whose resistance generally decreases as the amount of incident light increases. Therefore, the measured resistance provides an indirect indication of the surrounding light level.

The collected measurements were used to establish the operating range of the sensor and to determine suitable resistance values for the nightlight circuit to distinguish between dark conditions, when the nightlight should be brighter, and brighter conditions, when the nightlight should be dimmer or turned off. The measured data can also be plotted as resistance versus light level to visualize the response of the photoresistor and identify its useful operating range.

<img width="597" height="372" alt="image" src="https://github.com/user-attachments/assets/7b443e7f-4694-4993-8e3e-dffda48b2c06" />

### Results

The resistance of the photoresistor increases almost exponentially as the amount of light decreases. Although these were arbitrarily chosen light levels and were not measured in lux, the results clearly show that the resistance increased sharply as the amount of light it was exposed to decreased. This agrees with external research, which shows that photoresistors have a nonlinear, approximately exponential relationship between resistance and light intensity, with resistance increasing substantially as illumination decreases.

### Circuit Design

Based on the measured resistance range, a voltage divider was designed:
