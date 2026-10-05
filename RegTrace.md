# Register Trace

This document traces the relevant STM32F446 register settings used by the
Nightlight firmware back to the [STM32F446 reference manual (RM0390)](https://www.st.com/resource/en/reference_manual/rm0390-stm32f446xx-advanced-armbased-32bit-mcus-stmicroelectronics.pdf). The
values shown below are the values produced by the STM32CubeMX configuration
used in the project.

---

## 1. ADC1

The ADC is configured to sample the photoresistor voltage on PA4
(ADC1_IN4). TIM2 TRGO starts each conversion, and an ADC conversion-complete
interrupt causes the firmware to process the result.

### 1.1 ADC Resolution

**CubeMX setting:** 12-bit resolution

**Register:** `ADC1_CR1`

**Bit field:** `RES[1:0]`

**Value:** `00`

The `RES` field controls the ADC resolution. A value of `00` selects
12-bit resolution. Therefore, each conversion produces a value from
0 to 4095.

This corresponds to the `RES` field described in the ADC control
register 1 (`ADC_CR1`) section of RM0390.

---

### 1.2 External Trigger Selection

**CubeMX setting:** External trigger = TIM2 TRGO

**Register:** `ADC1_CR2`

**Bit field:** `EXTSEL[3:0]`

**Value:** `0110`

The `EXTSEL` field selects the event that triggers a regular ADC
conversion. The value `0110` selects the TIM2 trigger output (TRGO).

This allows the ADC conversion to be started by TIM2 hardware rather
than by software.

**RM0390:** ADC regular channel configuration / `ADC_CR2` — external
event selection (`EXTSEL`).

---

### 1.3 External Trigger Edge

**CubeMX setting:** External trigger edge = Rising edge

**Register:** `ADC1_CR2`

**Bit field:** `EXTEN[1:0]`

**Value:** `01`

`EXTEN = 01` enables conversion on a rising edge of the selected
external trigger.

Therefore, when TIM2 produces its TRGO update event, the ADC begins
a conversion.

---

### 1.4 Continuous Conversion

**CubeMX setting:** Continuous conversion = Disabled

**Register:** `ADC1_CR2`

**Bit field:** `CONT`

**Value:** `0`

With `CONT = 0`, the ADC does not automatically begin another
conversion after completing one. Instead, each conversion waits for
the next TIM2 trigger.

This is important to the event-driven design because the sampling
rate is determined by TIM2 rather than by the ADC continuously running.

---

### 1.5 Data Alignment

**CubeMX setting:** Data alignment = Right

**Register:** `ADC1_CR2`

**Bit field:** `ALIGN`

**Value:** `0`

`ALIGN = 0` right-aligns the conversion result in the ADC data register
(`ADC1_DR`).

The firmware can therefore read the 12-bit result directly as a value
between 0 and 4095.

---

### 1.6 ADC Channel

**CubeMX setting:** Channel 4, Rank 1

**Register:** `ADC1_SQR3`

**Bit field:** `SQ1[4:0]`

**Value:** `00100`

`SQ1` specifies the first conversion in the regular conversion
sequence. Setting it to channel 4 selects `ADC1_IN4`, which is
connected to PA4.

Because only one conversion is configured, this is the only channel
converted during each sampling event.

---

### 1.7 Number of Conversions

**CubeMX setting:** Number of conversions = 1

**Register:** `ADC1_SQR1`

**Bit field:** `L[3:0]`

**Value:** `0000`

The `L` field specifies the number of conversions in the regular
sequence minus one. A value of zero therefore selects one conversion.

---

### 1.8 ADC Sampling Time

**CubeMX setting:** Sampling time = 84 cycles

**Register:** `ADC1_SMPR2`

**Bit field:** `SMP4[2:0]`

**Value:** `100`

Because channel 4 is being used, its sampling time is configured in
`SMPR2`. The value `100` selects an 84-cycle sampling time.

The longer sampling time provides additional acquisition time for the
voltage-divider signal produced by the photoresistor.

---

### 1.9 ADC Clock Prescaler

**CubeMX setting:** ADC clock = PCLK2 / 4

**Register:** `ADC->CCR`

**Bit field:** `ADCPRE[1:0]`

**Value:** `01`

The ADC clock is derived from the APB2 clock. With PCLK2 at 84 MHz and
the `/4` prescaler selected:


This determines the clock used by the ADC conversion circuitry.

---

## 2. TIM2

TIM2 is responsible for determining when the photoresistor is sampled.
Its update event is routed through TRGO to the ADC.

### 2.1 Prescaler

**CubeMX setting:** Prescaler = 8399

**Register:** `TIM2_PSC`

**Value:** `8399`

The prescaler divides the timer input clock by `PSC + 1`:

$$
\frac{84\text{ MHz}}{8399+1}= 10 KHz
$$

Therefore, the TIM2 counter increments every 100 μs.

---

### 2.2 Auto-Reload / Period

**CubeMX setting:** Counter period = 99

**Register:** `TIM2_ARR`

**Value:** `99`

The timer counts from 0 through 99, giving 100 counter ticks per
update event.

Therefore:


$$
f_{TIM2}
=
\frac{10 kHz}{100}

100 Hz
$$

TIM2 therefore generates an update event every 10 ms.

---

### 2.3 Counter Direction

**CubeMX setting:** Counter mode = Up

**Register:** `TIM2_CR1`

**Bit field:** `DIR`

**Value:** `0`

`DIR = 0` configures the timer to count upward.

---

### 2.4 TRGO Selection

**CubeMX setting:** Master Output Trigger = Update Event

**Register:** `TIM2_CR2`

**Bit field:** `MMS[2:0]`

**Value:** `010`

The `MMS` field selects which timer event is sent through the TRGO
output. `010` selects the update event.

Consequently, every time TIM2 reaches its auto-reload value, the timer
produces a TRGO event. This event is connected internally to the ADC
and starts the next ADC conversion.

This creates the hardware path:

TIM2 update → TRGO → ADC conversion

---

## 3. TIM3 PWM

TIM3 generates the PWM signals used to control the three LEDs.

### 3.1 Prescaler

**CubeMX setting:** Prescaler = 83

**Register:** `TIM3_PSC`

**Value:** `83`

The 84 MHz TIM3 clock is divided by 84:

\[
\frac{84\text{ MHz}}{83+1}
=
1\text{ MHz}
\]

Therefore, each timer count is 1 μs.

---

### 3.2 Auto-Reload / Period

**CubeMX setting:** Period = 999

**Register:** `TIM3_ARR`

**Value:** `999`

The timer counts through 1000 values before restarting.

Therefore:

\[
f_{PWM}
=
\frac{1\text{ MHz}}{1000}
=
1\text{ kHz}
\]

The 1000-count period also provides 1000 possible compare values for
controlling the PWM duty cycle.

---

### 3.3 PWM Mode — Channel 1

**CubeMX setting:** PWM Generation CH1

**Register:** `TIM3_CCMR1`

**Bit field:** `OC1M[2:0]`

**Value:** `110`

This selects PWM Mode 1 for channel 1.

The corresponding output is connected to PA6 and drives the red LED.

---

### 3.4 PWM Mode — Channel 2

**CubeMX setting:** PWM Generation CH2

**Register:** `TIM3_CCMR1`

**Bit field:** `OC2M[2:0]`

**Value:** `110`

This selects PWM Mode 1 for channel 2.

The corresponding output is connected to PA7 and drives the green LED.

---

### 3.5 PWM Mode — Channel 3

**CubeMX setting:** PWM Generation CH3

**Register:** `TIM3_CCMR2`

**Bit field:** `OC3M[2:0]`

**Value:** `110`

This selects PWM Mode 1 for channel 3.

The corresponding output is connected to PB0 and drives the blue LED.

---

### 3.6 Output Polarity

**CubeMX setting:** PWM polarity = High

**Register:** `TIM3_CCER`

**Bit fields:** `CC1P`, `CC2P`, `CC3P`

**Value:** `0`

A cleared polarity bit selects active-high output polarity for the
corresponding channel.

---

### 3.7 Channel Output Enable

**Register:** `TIM3_CCER`

**Bit fields:** `CC1E`, `CC2E`, `CC3E`

**Value:** `1`

These bits enable the corresponding TIM3 outputs.

Therefore:

| TIM3 channel | GPIO | LED |
|---|---|---|
| CH1 | PA6 | Red |
| CH2 | PA7 | Green |
| CH3 | PB0 | Blue |

---
