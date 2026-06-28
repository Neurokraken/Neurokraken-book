(wiring-guides)=
# Wiring guides

The following diagrams show the wiring for common components in neuroscience experiment setups. As we are using common arduino parts and circuits, you can generally find the wiring for new components beyond what we have listed by web image searching for `arduino circuit <device_name>`.

## 12V Power

Many devices (valves, servos) will require more Volts and/or Amps than the teensy's 3.3V logic provides to operate. For this you can simply buy a "12V power supply" from common stores like amazon and cut through its output cable. Inside the outer cable you will find 2 wires, red (12V+) and black (GND) that you can directly use in your circuit as the required power source and its respective ground (connect to your circuit with "screw terminals" or solder).

## LED

````{mermaid}
flowchart LR;
subgraph teensy4.1
outputPin["output pin"]
GND
end

subgraph LED
LED+
LED-
end

outputPin --220Ohm--> LED+
LED- --> GND
````

## Photoresistor

For the analog pin use one of the beige Axx pins shown in the teensy 4.1 pinout at https://www.pjrc.com/store/teensy41.html

````{mermaid}
flowchart TB;
subgraph teensy4.1
analogPin["analog pin"]
v3_3["3.3V"]
GND
end

subgraph Photoresistor
photoresistor+
photoresistor-
end

v3_3 --> photoresistor+
photoresistor- --10kOhm--> GND
photoresistor- --> analogPin
````

## Beam break/photo interrupter

You can find beam breaks or photo interrupter components that combine an LED and a photoresistor in one device

````{mermaid}
flowchart TB;
subgraph teensy4.1
analogPin["analog pin"]
v3_3["3.3V"]
GND
end

subgraph LED
LED+
LED-
end

v3_3 --220Ohm--> LED+
LED- --> GND

subgraph Photoresistor
photoresistor+
photoresistor-
end

v3_3 --> photoresistor+
photoresistor- --10kOhm--> GND
photoresistor- --> analogPin
````

## Potentiometer

````{mermaid}
flowchart TB;
subgraph teensy4.1
analogPin["analog pin"]
v3_3["3.3V"]
GND
end

subgraph Potentiometer
signal["signal (center)"]
end1["end 1"]
end2["end 2"]
end

v3_3 --> end1
end2 --> GND
signal --> analogPin
````

## Valve

A Valve will require a control pin and an additional power supply, i.e. 12V. The easiest approach is to buy a 4 channel valve driver board that can directly be connected to your output pin, the power V+ and GND, and the valve+/- itself. The shown board can be found on sites like amazon. Notably, to ensure proper operation with a 3.3V device like the Teensy, you have to desolder/solder over the small LED as shown in the image.

### 4 Channel Valve Driver Board

This board has 4 lanes A, B, C, D where the signal at A drives the opposite valve A. You can go beyond this circuit to connect 4 output pins with 4 lanes to 4 valves.

````{mermaid}
flowchart TB;
subgraph teensy4.1
output["output pin"]
v3_3["3.3V"]
GND
end

subgraph power supply 12V
power_12V+
power_Gnd["GND"]
end

subgraph Valve
valve+["Valve +"]
valve-["Valve -"]
end

subgraph valve driver board

subgraph power
board_12v["12V+"]
boardpow_GND["GND"]
end

subgraph lane A
subgraph Signal["Signal Side"]
board_a_signal["Signal +"]
board_a_gnd["Signal GND"]
end
subgraph Valve_side["Valve Side"]
board_a_valve+["Valve +"]
board_a_valve-["Valve -"]
end
end

end

power_12V+ --> board_12v
boardpow_GND --> power_Gnd

output --> board_a_signal
board_a_gnd --> GND
board_a_valve+ --> valve+
valve- --> board_a_valve-
````

### Manual Valve Circuit

The following circuit manually builds a lane of the valve driver board above allowing you to drive a single valve with common electronics parts.

````{mermaid}
flowchart TB;
subgraph teensy4.1
outputPin["output pin"]
GND
end

subgraph IRLZ44
IRLZ44Left
IRLZ44Center
IRLZ44Right 
end
outputPin --1kOhm--> IRLZ44Left
Valve- --> IRLZ44Center
IRLZ44Right --> 12VGND 
IRLZ44Center -.- IRLZ44Right

subgraph Valve
12V+ --- Capacitor
Capacitor --- 12VGND["12V GND"]
12V+ --> Valve+
Valve+ --> Valve-
Valve- --> Diode["Diode\n(1n4007)"]
Diode --> Valve+
end
12VGND --- GND
````

The capacitor stores charge to support the high current draw required when the valve is activated.

When the valve solenoid gets powered this will induce movement of its piston. Conversely, when the piston springs back upon loss of power this induces a voltage. The diode will feed back this voltage to prevent it from ending up in the microcontroller.

## Rotary Encoder

There are 2 common types of rotary encoder - both will have 4 wires for black GND, red 3.3V, and 2 signals A and B (i.e. in green and white): The more common pull-up Rotary encoder, and a more uncommon typically expensive direct rotary encoder.

### Pull-up Rotary Encoder

The most common rotary encoders often don't provide voltage on their signal pins, but depending on their state short the respective signal pins to GND or not and can wired like this.

````{mermaid}
flowchart TB;
subgraph teensy4.1
GND
3.3V
inputPinA["pin A"]
inputPinB["pin B"]
end

subgraph Rotary Encoder
encV["V+"]
encGND["GND"]
encA["A"]
encB["B"]
end

3.3V --> encV
encGND --> GND
3.3V --4.7kOhm--> inputPinA
inputPinA --> encA
3.3V --4.7kOhm--> inputPinB
inputPinB --> encB
````

These signals can be read by connecting them to Teensy input pins with pull-up resistors to 3.3V. If the encoder shorts the signal to GND, the pin reads LOW; otherwise, the 3.3V will make its way to the teensy pin, which will be pulled-up to HIGH (3.3V).

### Direct Rotary Encoder

This type of encoder is generally more expensive, sold by specialized electronics stores, maybe more compact and can have additional optional wires.

This is the circuit for a 5V rotary encoder whose signal pins switch between 5V and 0V during rotation. If your rotary encoder functions at 3.3V you can use 3.3V and leave out the voltage divider resistors.

````{mermaid}
flowchart TB;
subgraph teensy4.1
5V
GND
3.3V
inputPinA["pin A"]
inputPinB["pin B"]
end

subgraph Rotary Encoder
enc5V["5V"]
encGND["GND"]
encA["A"]
encB["B"]
end

A(( ))
B(( ))
5V --> enc5V
encGND --> GND
encA -- 10kOhm --> A
encB -- 10kOhm --> B
A -- 20kOhm --> GND
B -- 20kOhm --> GND
A --> inputPinA
B --> inputPinB
````

The manual should indicate which wires correspond to V+, GND, A, and B. As the teensy should only receive 3.3V not the 5V the rotary encoder delivers, we use 2 stages of resistors to lower the A and B voltage from 5V to 3.3V and then 0V/GND. We can then connect the 3.3V stages to our reading pins. (Voltage Dividier)

## Servo

For the servo signal wire, use a pin designated for PWM on the Teensy pinout in https://www.pjrc.com/store/teensy41.html

````{mermaid}
flowchart TB;
subgraph teensy4.1
outputPin["PWM output pin"]
GND
end

subgraph power supply 12V
power_12V+
power_Gnd["GND"]
end

Capacitor
Diode["1N4007 diode"]

subgraph Servo_Motor
servo_power["Power +"]
servo_signal["Signal"]
servo_gnd["Ground"]
end

power_12V+ --- Capacitor
Capacitor --- power_Gnd

power_12V+ --> servo_power
servo_power --> Diode
Diode --- power_Gnd

outputPin --> servo_signal
servo_gnd --> power_Gnd
servo_gnd --> GND
````

## Tone Passive Buzzer

````{mermaid}
flowchart LR;
subgraph teensy4.1
outputPin["output pin"]
GND
end

subgraph Buzzer
Buzzer+
Buzzer-
end

outputPin --> Buzzer+
Buzzer- --> GND
````

## Capacitive Touch Sensor

This circuit uses a capacitive touch sensor module available on common stores like amazon.

````{mermaid}
flowchart TB;
subgraph teensy4.1
sensorPin["sensor pin"]
v3_3["3.3V"]
GND
end

subgraph Capacitive Touch sensor
module_VCC["VCC"]
module_GND["GND"]
module_SIG["SIG"]
end

v3_3 --> module_VCC
module_GND --> GND
module_SIG --> sensorPin
````

## Pulse clock

````{mermaid}
flowchart TB;
subgraph experiment_teensy["Experiment Teensy"]
clock_pin["clock pin"]
GND
USB
end

subgraph external_device["External Device"]
external_input["input pin"]
external_GND["GND"]
end

clock_pin -- 10kOhm --> external_input
GND --- external_GND
PC --- USB
````
