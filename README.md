[cite_start]This is a fascinating project that aligns perfectly with your expertise in high-performance microcontrollers and digital signal processing[cite: 304, 305]. [cite_start]Based on the mechanics of a timegrapher described in the document—which uses a microphone to capture "ticks and tocks" and calculate movement health [cite: 231, 232]—you can certainly build a DIY version using a high-speed Nano-form-factor MCU.

[cite_start]Given your background with the **Teensy 4.1** and **STM32H7**, you have the processing power needed for the high-frequency sampling and FFT analysis required to detect "beat error" and "amplitude" accurately[cite: 239, 243].






---

## Project Concept: The "Micro-Timegrapher"
[cite_start]The goal is to create a device that captures the acoustic signature of a mechanical watch, processes the signal to identify the escapement pulses, and calculates the **Rate ($s/d$)**, **Amplitude**, and **Beat Error (ms)**[cite: 235, 238, 243].

### 1. Hardware Requirements
* **MCU:** **Teensy 4.0/4.1** or **Arduino Nano ESP32**. 
    * [cite_start]*Why:* You need a high clock speed to sample audio at 44.1kHz or higher to distinguish the millisecond-level differences in "tick" vs "tock" (Beat Error)[cite: 243].
* **Acoustic Sensor:** **High-sensitivity Piezo contact microphone** or a **MEMS microphone (ICS-43434)**.
    * [cite_start]*Add-on:* A low-noise Pre-amp circuit (like an MAX9814 with Auto Gain Control) is essential to isolate the faint mechanical sounds from ambient noise[cite: 232].
* [cite_start]**Display:** **1.3" OLED (SH1106)** to show the "two lines" of the timegrapher display[cite: 262].
* [cite_start]**Input:** A rotary encoder for manual **Lift Angle** adjustment (e.g., setting it to 53.0° for a Seiko NH35)[cite: 252, 255].


---

### 2. Digital Signal Processing (The Logic)
To replicate the "Numbers" described in the PDF, your software will need to handle three main calculations:

#### A. The "Tick-Tock" Detection
A mechanical watch beat isn't just one sound; it consists of three distinct "noises" (the pallet stone hitting the escape wheel, the impulse, and the drop). 
* **Algorithm:** Use a peak-detection algorithm on the audio envelope.
* [cite_start]**Beat Rate:** Most watches run at 21,600 bph (6 ticks/sec)[cite: 250]. Your code must detect the time interval between these peaks.

#### B. Rate and Beat Error Calculation
* **Rate ($s/d$):** Compare the measured interval between beats against the "ideal" interval (e.g., $1/6$th of a second). [cite_start]If the average interval is shorter, the watch is "fast"[cite: 237, 262].
* **Beat Error ($ms$):** Measure the time difference between *Tick-to-Tock* ($T_1$) and *Tock-to-Tick* ($T_2$). The error is $\frac{|T_1 - T_2|}{2}$. [cite_start]An acceptable result is < 0.6ms[cite: 243, 271].

#### C. Amplitude Calculation
This is the most complex part. [cite_start]It requires the **Lift Angle** (the degrees the balance wheel rotates while in contact with the pallet fork)[cite: 252, 256].
* [cite_start]**Formula:** You measure the time the escapement is "engaged" (the duration of the tick sound itself) and use the lift angle to extrapolate the total swing rotation[cite: 239].

---


### 3. Proposed Software Architecture
* **Sampling:** Use DMA (Direct Memory Access) to pull audio data without taxing the CPU.
* [cite_start]**Filtering:** Apply a digital band-pass filter (centered around 3kHz–6kHz) to isolate the metallic "clink" of the watch from low-frequency room rumble[cite: 232].
* **Visualization:** Draw a "rolling" plot on the OLED.
    * [cite_start]Upward slanting dots = Watch is running fast[cite: 262].
    * [cite_start]Downward slanting dots = Watch is running slow[cite: 262].
    * [cite_start]Two distinct parallel lines = Significant beat error[cite: 263].

---

### 4. Implementation Steps
1.  [cite_start]**Isolation:** Build a small "watch clamp" using 3D printing or wood to hold the watch and the piezo sensor firmly together[cite: 261].
2.  **Calibration:** Use a known-accurate quartz watch or a GPS-disciplined pulse as a reference to calibrate your MCU's internal clock.
3.  [cite_start]**Regulation:** Once the device is built, use it to guide your manual adjustments of the **Stud Carrier** (for beat error) and the **Regulator lever** (for rate) as shown in the guide[cite: 275, 276, 281].

[cite_start]Given your interest in **SDR** and **RF signal processing**[cite: 310], you might find it rewarding to treat the watch's acoustic signature as a "signal" and use a Python-based wrapper to analyze the data on your Linux machine before committing the code to the MCU.

Would you like a specific wiring diagram for connecting a Piezo sensor to your Teensy's ADC, or perhaps a snippet of the peak-detection logic in Python?


[cite_start]To create a DIY timegrapher using a high-performance MCU like the **Teensy 4.1**, you need to process acoustic signals in real-time to identify the "ticks" and "tocks" of the mechanical movement[cite: 49]. [cite_start]This code focuses on high-speed sampling and peak detection to calculate **Rate** and **Beat Error**[cite: 55, 72].

### Hardware Logic
The code assumes you are using a **Piezo element** or **MEMS microphone** connected to an analog pin. Because mechanical watch pulses are very brief, we use a fast sampling rate to catch the peak of the escapement hit.



### The Code (Arduino/Teensy C++)

```cpp
#include <Arduino.h>

// Project: Nano-MCU Watch Timegrapher Logic
// Targets: Teensy 4.1, Arduino Nano ESP32, or STM32H7

const int sensorPin = A0;      // Input from Pre-amp/Piezo
const int threshold = 300;     // Signal threshold (adjust based on noise floor)
const float liftAngle = 53.0;  [cite_start]// Default for Seiko NH35 [cite: 91, 100]

unsigned long lastTickTime = 0;
unsigned long intervalT1 = 0;  // Time between Tick and Tock
unsigned long intervalT2 = 0;  // Time between Tock and Tick
bool isTick = true;
int beatCount = 0;

void setup() {
  Serial.begin(115200);
  analogReadResolution(12); // High resolution for precise peak detection
  pinMode(sensorPin, INPUT);
  Serial.println("Watch Timegrapher Initialized...");
}

void loop() {
  int signal = analogRead(sensorPin);

  [cite_start]// Peak detection logic for the escapement pulse [cite: 49, 111]
  if (signal > threshold) {
    unsigned long currentTime = micros();
    unsigned long duration = currentTime - lastTickTime;

    // Debounce: Ignore noise within 100ms of a valid beat
    [cite_start]// (A 21,600 BPH watch beats every ~166ms) [cite: 85]
    if (duration > 100000) { 
      
      if (isTick) {
        intervalT1 = duration;
      } else {
        intervalT2 = duration;
        calculateMetrics(intervalT1, intervalT2);
      }

      lastTickTime = currentTime;
      isTick = !isTick; [cite_start]// Toggle between tick and tock [cite: 73]
    }
  }
}

void calculateMetrics(unsigned long t1, unsigned long t2) {
  [cite_start]// 1. Beat Error Calculation [cite: 73, 135]
  // The difference between the "tick" and the "tock"
  float beatError = abs((long)t1 - (long)t2) / 2000.0; // Convert micros to ms

  [cite_start]// 2. Rate Calculation (Seconds per Day) [cite: 57, 63]
  // Average interval for 21,600 BPH is 166,667 micros
  float avgInterval = (t1 + t2) / 2.0;
  float targetInterval = 166666.67; 
  float errorPerBeat = avgInterval - targetInterval;
  
  // Convert error to seconds per day
  // (60s * 60m * 24h * 6 beats per second)
  float rateSD = (errorPerBeat / targetInterval) * 86400.0;

  [cite_start]// Output to Serial (can be redirected to an OLED) [cite: 53]
  Serial.print("Rate: ");
  Serial.print(rateSD, 1);
  Serial.print(" s/d | ");
  
  Serial.print("Beat Error: ");
  Serial.print(beatError, 2);
  Serial.println(" ms");

  [cite_start]// Logic Check: [cite: 135, 156]
  if (beatError > 0.6) {
    Serial.println("Warning: High Beat Error. Adjust Stud Carrier.");
  }
}
```


# This explanation breaks down the logic of the "Micro-Timegrapher" code, focusing on how it translates raw acoustic signals into the timing data described in your watch regulation guide.

### 1. Variables and Setup
```cpp
const int sensorPin = A0;      // Input from Pre-amp/Piezo
const int threshold = 300;     // Signal threshold
const float targetInterval = 166666.67; // Ideal 21,600 BPH timing
```
* **`sensorPin`**: The analog pin receiving the amplified sound of the watch.
* **`threshold`**: This is the "noise gate." The MCU ignores any sound quieter than this value to avoid triggering on background hum.
* **`targetInterval`**: For a standard watch beating 6 times per second (21,600 BPH), each beat should occur exactly every 166,666.67 microseconds.

### 2. The Main Loop (Signal Capture)
```cpp
int signal = analogRead(sensorPin);
if (signal > threshold) {
    unsigned long currentTime = micros();
    unsigned long duration = currentTime - lastTickTime;
```
* **`analogRead`**: Continuously samples the voltage from the microphone.
* **`micros()`**: This is critical for precision. It returns the number of microseconds since the MCU started. Standard `millis()` is too coarse for measuring beat error.
* **`duration`**: Calculates the exact time elapsed since the *previous* detected sound.

### 3. Debouncing and Logic Toggle
```cpp
if (duration > 100000) { 
    if (isTick) {
        intervalT1 = duration;
    } else {
        intervalT2 = duration;
        calculateMetrics(intervalT1, intervalT2);
    }
    lastTickTime = currentTime;
    isTick = !isTick; 
}
```
* **`duration > 100000`**: This is a **refractory period** (100ms). Since a real "tick" has multiple internal echoes, this prevents the code from double-counting the same beat.
* **`isTick` toggle**: A mechanical watch has a "Tick" (swing left) and a "Tock" (swing right). The code stores the duration of the first swing in `T1` and the second in `T2` so they can be compared.



### 4. Calculating Beat Error
```cpp
float beatError = abs((long)t1 - (long)t2) / 2000.0;
```
* **The Math**: If the watch is perfectly "in beat," the time between Tick $\to$ Tock ($T_1$) and Tock $\to$ Tick ($T_2$) is identical. 
* **The Conversion**: We find the difference, then divide by 2000 (to get the average offset and convert from microseconds to milliseconds). This represents the "Beat Error" shown on professional timegraphers.

### 5. Calculating Rate (Seconds per Day)
```cpp
float avgInterval = (t1 + t2) / 2.0;
float errorPerBeat = avgInterval - targetInterval;
float rateSD = (errorPerBeat / targetInterval) * 86400.0;
```
* **`avgInterval`**: The actual measured time for one beat.
* **`errorPerBeat`**: How many microseconds the watch is "off" per beat.
* **`86400.0`**: The number of seconds in a full day. By multiplying the percentage of error by the total seconds in a day, we get the **s/d (seconds per day)** gain or loss.

### 6. Logic and Feedback
```cpp
if (beatError > 0.6) {
    Serial.println("Warning: High Beat Error. Adjust Stud Carrier.");
}
```
* As noted in your PDF, a beat error over 0.6ms indicates the balance wheel is "limping." This line provides the actionable advice to move the **stud carrier** to center the swing.




# Here is an ASCII flowchart 
representing the logic of the timegrapher code. It tracks the signal from the initial sensor reading through to the final calculation of the watch's performance metrics.

```text
       [ START LOOP ]
             |
             v

    +-----------------------+
    | Read Analog Pin (A0)  | <-----------+
    +-----------------------+             |
             |                            |
             v                            |
    +-----------------------+             |
    | Is Signal > Threshold?| ---- NO ----+
    +-----------------------+             |
             |                            |
            YES                           |
             |                            |
             v                            |
    +-----------------------+             |
    | Calculate Duration    |             |
    | (Now - LastTickTime)  |             |
    +-----------------------+             |
             |                            |
             v                            |
    +-----------------------+             |
    | Duration > 100ms?     | ---- NO ----+ (Debounce/Noise Filter)
    +-----------------------+             |
             |                            |
            YES                           |
             |                            |
             v                            |
    +-----------------------+             |
    |   Toggle isTick       |             |
    +-----------------------+             |
      |                 |                 |
    [TRUE]           [FALSE]              |
      |                 |                 |
      v                 v                 |
+------------+    +--------------------+  |
| Store T1   |    | Store T2           |  |
| (1st Beat) |    | (2nd Beat)         |  |
+------------+    +--------------------+  |
      |                 |                 |
      |                 v                 |
      |       +-----------------------+   |
      |       |  CALCULATE METRICS:   |   |
      |       |                       |   |
      |       | 1. Beat Error (T1-T2) |   |
      |       | 2. Rate S/D (Vs Target)|  |
      |       +-----------------------+   |
      |                 |                 |
      v                 v                 |
    +-----------------------+             |
    | Update LastTickTime   | ------------+
    +-----------------------+
```





### Flowchart Key:
1.  **Analog Read**: The MCU continuously monitors the voltage from your Piezo/Microphone.
2.  **Threshold Gate**: Acts as a software "squelch" to ignore low-level ambient room noise.
3.  **Duration & Debounce**: A mechanical watch tick isn't a single "pop"—it's a series of micro-noises as the pallet stones hit the escape wheel. The `> 100ms` check ensures we count the whole event as one "beat."
4.  **The Toggle (`isTick`)**: This separates the "Tick" from the "Tock." Without this, you couldn't calculate **Beat Error**, because you wouldn't know if the balance wheel is swinging further or faster in one direction than the other.
5.  **Metrics Calculation**: This happens every second beat (once per full oscillation). It compares your measured microsecond intervals against the mathematical ideal for the movement's BPH (Beats Per Hour).


# For a project using a **Teensy 4.1** or **Arduino Nano ESP32**, the circuit needs to handle two things: amplifying the tiny "click" of the watch and protecting the MCU from voltage spikes. 

Since you are working with high-performance MCUs, using a **Piezo element** as a contact microphone is the most effective DIY method.

### DIY Timegrapher Circuit Diagram

```text
       +-----------------------------------------+
       |         ACCOUSTIC PICKUP STAGE          |
       +-----------------------------------------+
       
    [ WATCH ]
        |
    +-------+      100nF        10k
    | PIEZO |-------||---------XXXX----------+------> TO MCU ANALOG (A0)
    | SENSOR|       C1          R2           |
    +-------+                                |
        |                                   XXX 1M (R1)
       GND                                  XXX 
                                             |
                                            GND
```

### Component Breakdown & "Why"

1.  **Piezo Sensor**: This acts as a high-impedance contact microphone. It picks up the physical vibrations (vibration energy) of the watch movement rather than the air-borne sound.
2.  **C1 (100nF Capacitor)**: This is a **DC Blocking/Coupling Capacitor**. It ensures that only the AC signal (the "tick") passes through, preventing any DC offset from reaching your MCU.
3.  **R1 (1M Ohm Resistor)**: This is the **Load/Bleeder Resistor**. Piezo elements can generate high voltage spikes if bumped. This resistor drains that energy to ground and stabilizes the input impedance.
4.  **R2 (10k Ohm Resistor)**: This is a **Current Limiter**. It protects your Nano or Teensy pin from over-voltage spikes that might occur if the sensor is dropped or struck.



### Pro-Tip for your High-Performance MCUs:
If you find the signal is too weak for the `threshold` in the code, you can use a simple **Op-Amp (TL072 or LM358)** in a non-inverting configuration to boost the signal 10x-100x before it hits the Analog pin. 

Given your interest in **SDR and RF design**, you could also consider using a **shielded cable** (RG-174) between the Piezo and the board. Mechanical watch pulses are very faint, and the high-clock speeds of the Teensy 4.1 can induce electromagnetic noise into the sensor lines if they aren't shielded.

### Wiring for Teensy 4.1 / Nano:
* **Sensor (+)**: Connects to C1.
* **Sensor (-)**: Connects to GND.
* **MCU A0**: Connects to the junction of R1 and R2.
* **Common Ground**: Ensure the sensor ground and MCU ground are tied together to prevent floating signals.

# To build a fully portable standalone timegrapher with a **Teensy 4.1** or **Nano**, you need to integrate power management, a user interface for settings (like Lift Angle), and a display for the real-time graph.

### 1. Complete System Architecture
This design uses a **Li-ion battery** with a charging/protection module and an **I2C OLED** for the display.

```text
    [ POWER SECTION ]                 [ PROCESSING & I/O ]
    
    +--------------+                  +-----------------------+
    | 3.7V Li-ion  |                  |  TEENSY 4.1 / NANO    |
    |   Battery    |                  |                       |
    +------+-------+       USB 5V --->| [USB Port]      (A0) <--- [Audio In]
           |                          |                       |
    +------v-------+                  | (SDA)           (D2) <--- [Encoder A]
    | TP4056 Board |                  | (SCL)           (D3) <--- [Encoder B]
    | (Charger/Prot)|                  | (3.3V)          (D4) <--- [Enc Button]
    +------+-------+                  | (GND)                 |
           |                          +-----------+-----------+
    +------v-------+                              |
    | MT3608 Boost |--- 5V -----------------------+
    | (To 5V Rail) |
    +--------------+
```

### 2. Full Circuit Schematic (ASCII)

```text
                                  +5V (From Boost)
                                   |
          +------------------------+-----------------------+
          |                        |                       |
   +------v------+          +------v------+         +------v------+
   |   OLED      |          |  ROTARY     |         |   PIEZO     |
   | (SSD1306)   |          |  ENCODER    |         |   CIRCUIT   |
   +-------------+          +-------------+         +-------------+
   [VCC] <--- 5V            [VCC] <--- 5V           [VCC] <--- 5V (If using Op-Amp)
   [GND] <--- GND           [GND] <--- GND          [GND] <--- GND
   [SDA] <--- Pin 18 (A4)   [SW]  <--- Pin 4        [OUT] <--- Pin 14 (A0)
   [SCL] <--- Pin 19 (A5)   [DT]  <--- Pin 3               |
                            [CLK] <--- Pin 2               |
                                                           |
                                     100nF       10k       |
          PIEZO SENSOR (+) ----||----XXXX-----+--- A0
                                C1     R2     |
                                             XXX 1M (R1)
          PIEZO SENSOR (-) ------------------+--- GND
```



---

### 3. Part List & Components

| Component | Function | Why it’s needed |
| :--- | :--- | :--- |
| **Teensy 4.1** | Main Controller | Fast clock speed is required to distinguish microsecond "beat error." |
| **SSD1306 OLED** | Display | Shows the "Rate," "Beat Error," and the rolling dot graph. |
| **Rotary Encoder** | Input | Used to scroll through menus and set the **Lift Angle** (crucial for Amplitude). |
| **TP4056 Module** | Charging | Safely charges the 3.7V Li-ion battery via USB. |
| **MT3608 Boost** | Power | Steps up 3.7V battery to 5V to run the MCU and OLED consistently. |
| **Piezo Element** | Sensor | High-impedance pickup to hear the "Tick-Tock" through the watch case. |
| **Shielded Wire** | Cabling | Prevents 50Hz/60Hz mains hum from interfering with the tiny watch signal. |

---

### 4. Integration Notes

**The Keypad (Rotary Encoder):**
Instead of a bulky 4x4 keypad, a rotary encoder with a built-in push button is the standard for watch tools. 
* **Turn:** Adjusts the target BPH (18,000, 21,600, 28,800) or the Lift Angle.
* **Click:** Toggles between the "Graph View" and "Settings View."

**The Audio Pickup (Shielding):**
The signal coming from the watch is in the millivolt range. To prevent noise:
1.  Mount the Piezo sensor inside a small plastic clamp.
2.  Use **shielded audio cable** to connect the clamp to your main project box.
3.  Connect the cable shield to the MCU **GND**.

**Battery Life:**
A standard 1000mAh Li-ion battery will run this setup for approximately 8–10 hours of continuous regulation, making it fully portable for use at a workbench without messy power cables near the watch movement.



### Strategic Implementation Details

* [cite_start]**Filtering:** Mechanical movements are noisy[cite: 38]. In a production version, you should implement a **Digital Bandpass Filter** (between 3kHz and 6kHz) in the code to ignore low-frequency ambient hum and high-frequency electronic hiss.
* **Averaging:** The code above calculates metrics beat-by-beat. [cite_start]For a stable "Rate" reading like a real No.1000 Timegrapher, you should implement a **12-second moving average**[cite: 102].
* [cite_start]**Visualizing the "Lines":** To replicate the scrolling dot display[cite: 112, 130], you can map the `errorPerBeat` to the Y-axis of an OLED display. 
    * [cite_start]If the dots trend upward, the watch is fast[cite: 112].
    * [cite_start]If the dots trend downward, the watch is slow[cite: 112].
    * [cite_start]If you see two separate lines, the `beatError` is high[cite: 113].

### Calibration Warning
The accuracy of your DIY timegrapher depends entirely on the accuracy of your MCU's crystal oscillator. [cite_start]Since you are measuring microsecond differences, ensure your Teensy is not running on an internal RC oscillator, as temperature drift will give you a false "Rate" reading[cite: 166].

