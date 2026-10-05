# Nightlight

An STM32F446RE-based automatic nightlight that measures ambient light with a photoresistor, and as the environment becomes darker, turns the LEDs on sequentially and smoothly increases their brightness. The system uses a hardware-timer-triggered ADC, interrupt-driven processing, and PWM output.


Demo: [Link](https://www.youtube.com/shorts/aKtNGqWCgg0)

---

## Overview
Nightlight converts a real-world analog light measurement into a digital control signal and uses that signal to control three LEDs.

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

We selected 200 Ω resistors for all three LEDs. Using the measured forward voltage of each LED between 2-3 V depending on the color, the expected current can be calculated as:

$$
I_{LED}=\frac{3.3-V_F}{200\Omega}
$$

This keeps the LED current below the current limit of the LED (which we don't know exactly, but presume is on the order of 10s of mA) and the STM32's 8 mA GPIO specification under the normal output-voltage conditions while providing sufficient current for visible illumination. Using a larger resistor would further reduce GPIO and LED current, but would also reduce the available LED brightness.

### Final Schematic

We used PA4 for the photoresistor because it can be configured as an ADC input and also supports DAC output on the same physical pin, which allows the onboard DAC to be connected to the ADC through PA4 during the self-test (where we simulate a photoresistor sweeping through resistance values) without requiring an additional pin.

We used PA6, PA7, and PB0 for the LEDs because they correspond to TIM3 channels 1, 2, and 3, supporting TIM3 PWM outputs, allowing the LED brightness to be controlled directly by hardware timers. Using three channels of the same timer also allows all three LEDs to share the same frequency while their duty cycles are independently controlled.

The complete circuit schematic is shown below.

<img width="1207" height="737" alt="image" src="https://github.com/user-attachments/assets/73398cef-ec19-49ac-92fa-bb13ac3366db" />

---

# 2. Firmware
## System Architecture
### Data Flow
The system uses a hardware timer-triggered ADC and interrupt-driven processing to convert ambient light into PWM0controlled LED brightness. The main signal path is shown below:

<img width="876" height="320" alt="image" src="https://github.com/user-attachments/assets/3ba06b6f-67c2-4676-9483-a965de7be968" />

During normal operation, the photoresistor and fixed resistor form a voltage divider connected to PA4 (the Analog Digital Converter input). As the ambient light changes, the voltage at PA4 changes and is converted by the 12-bit ADC into a value from 0 to 4095.

TIM2 provides the sampling clock for the system. It is configured with a pre-scaler of 8399 and a period of 99. 

$$
f_{TIM2}=\frac{84 MHz}{(8399+1)(99+1)} = 100Hz
$$

This generates a trigger output signal every 10 ms for the ADC. This allows the ADC to be sampled at a known fixed hardware-defined rate. We selected this value because we assumed that changes to ambient light indoors are typically made from turning on or off a switch or occluding light sources, which would change much slower than 10 ms. It would also likely be beyond the perception of a human. Sampling more frequently would provide very limited practiccal benefit, and unecessarily create more processing. We also validated this in our final end to end system test, where the response of the LEDs to the change in ambient light was as desired.


When  TIM2 generates a TRGO event, the ADC begins a conversion of the voltage across the 22kΩ resistor in the photoresistor voltage divider.The ADC uses 12-bit resolution, producing a value from 0 to 4095. An 84-cycle sampling time was selected to provide sufficient acquisition time for the photoresistor voltage-divider signal. With an 21 MHz clock for the ADC, this would give us about 4 uS of sampling before conversion, which is plenty given we have 10 ms between samples. Making this longer could give the ADC more time to settle, but we chose not to modify this given our satisfaction with the end-to-end testing.

Once the conversion is complete, the ADC generates an interrupt and the firmware enters HAL_ADC_ConvCpltCallback(). The callback reads the ADC value and calculates the darkness of the environment.

The resulting darkness value is divided into three equal ranges. In the first third, the red LED ramps from off to full brightness. In the second third, red remains fully illuminated while green ramps up. In the final third, red and green remain fully illuminated while blue ramps up.

The LED order was chosen as red, green, blue, in order of greatest to least ambient light. Red provides a warm first stage, while blue is reserved for the darkest portion of the nightlight's operating range, to create a visual progression as the environment gets darker. The blue LED also shines brighter, due to the difference in their voltage drop.

The calculated LED brightness values are converted into PWM duty cycles. A quadratic correction is applied to each duty cycle before it is written to the PWM outputs. This was chosen because a linear change in PWM duty cycle does not appear linear to the human eye, so the quadratic mapping helps provide a smoother perceived brightness increase.

The resulting duty cycles are written to the three TIM3 compare channels, which acts as the PWM generator. It operates at 1kHz, with a prescaler of 83 and a period of 999. 

$$
f_{PWM}=\frac{84 MHz}{(83+1)(999+1)} = 1 KHz
$$

This was chosen to make for potential easy debugging on a scope and provide fast enough LED toggling to eliminate a low-frequency flicker.

### Event-Driven Operation
The firmware does not continuously poll the ADC and does not use blocking delays. After initialization, the main loop just waits for interrupts rather than repeatedly checking peripheral status. This ensures that sensor processing occurs at a consistent 100 Hz rate while the CPU remains idle and capable of performing other tasks between conversions.

### DAC Self-Test

For testing, the onboard DAC can be enabled using the ENABLE_DAC_SELFTEST flag. The DAC outputs a controlled voltage on PA4, the same physical pin used by the photoresistor ADC input. This allows the ADC-to-PWM portion of the system to be tested with a repeatable analog signal without changing the hardware, just by disconnecting the photoresistor voltage divider from PA4.

The DAC is configured with no hardware trigger. Instead, its value is updated from the ADC conversion-complete callback. Therefore, TIM2 does not directly trigger the DAC.  This means that the DAC simply generates the value for the next ADC conversion, which is fine since all we are doing is sweeping through the approximate range of the photoresistor to tune transitioning and ensuring our LEDs are changing in brightness smmothly.
