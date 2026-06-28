
(modalities)=
# Sensors and Stimuli

## Reading modalities

### Rotation

Rotation can be read with a rotary encoder ([devices.rotary_encoder](devices.rotary_encoder)) or a potentiometer ([devices.analog_read](devices.analog_read)). Rotary encoders are useful for high precision low friction rotation readings like on a steering wheel or treading wheel/disc, while potentiometers are small and cheap.

Rotation can be useful for measuring a steering wheel, or progress on a walking disc.

### Linear position

Slider-shaped potentiometers can be read as a potentiometer ([devices.analog_read](devices.analog_read)), as can joysticks like the arduino psp2 module.

### Brightness/Beam breaking

Light and its blockage can be read with a photoresistor or a beam breaking sensor as a ([devices.analog_read](devices.analog_read)) or ([devices.binary_read](devices.binary_read))

### Touch

Touch can be read through a capacitive touch breakout board as a ([devices.binary_read](devices.binary_read)) or in the shape of a beam break as a ([devices.analog_read](devices.analog_read)). As capacitive touch is based on rapidly charging and discharging the touchable element a beam break can be preferable for electrophysiology experiment.

### Buttons

Buttons and switches can be read as a ([devices.binary_read](devices.binary_read))

### Vibration

Vibration can be sensed through a shake switch as a ([devices.analog_read](devices.analog_read)).

### Additional reading sensors

While the readings above cover most use cases, the range of readings gatherable from neurokraken tasks is only limited by arduino-compatible sensors allowing for many new experiment designs. The most difficult cases can even be integrated through an [extra-arduino](extra-arduino) if required. 

Examples of arduino sensors that could be added include `distance (time of flight) sensors`, `flex sensors`, `accelerometers`, `temperature sensors`, ...

## Controlling modalities

### Motoric movement

A servo motor [devices.servo](devices.servo) can move things through a rotating arm. A 3D printabel "rack and pinion" mechanism can turn this rotation into linear movement. Please note that for a servo to work one of the following pins has to be used: 0-15, 18-19, 22-25, 28-29, 33, 36-37 (https://www.pjrc.com/teensy/td_libs_Servo.html). Rapid linear movement can also be achieved through a solenoid piston which can be either a [devices.direct_on](devices.direct_on) or a [devices.timed_on](devices.timed_on).

### Liquid/Odor delivery (Valves)

A solenoid-operated pinch valve is an easy way to open or close rubber tubing providing liquid or air to a subject. Depending on the use case they can exist in Normally-Closed or Normally-Open and will change to the alternative when powered. This can be done through signal from either a a [devices.direct_on](devices.direct_on) or a [devices.timed_on](devices.timed_on). A common use case is to provide precisely sized task-reward droplets of sugar water as a reward through a short valve opening.

### LEDs

LEDs can be controlled through a [devices.direct_on](devices.direct_on) or a [devices.timed_on](devices.timed_on), or as adjustable brightness is controlled through the same signal as a servo's position, a [devices.servo](devices.servo).

### Tone

Auditory frequencies from a connected passive buzzer or piezo can be generated through a [devices.tone](devices.tone).

### Synchronization pulse signals

Regular synchronization signals with a chosen frequency can be created as a (devices.pulse_clock)[devices.pulse_clock]. These signals are useful to create a common time signal across separate pieces of experiment equipment, like the neurokraken behavior setup and an external electrophysiology recording board. Some external cameras also utilize this type of signal to time their frame captures.

### Displays and Audio speakers

More advanced stimuli like computer display graphics, or audio playback can best be run from the computer/python side as described in [Configurators](configuration.md).