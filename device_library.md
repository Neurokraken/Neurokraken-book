# Configurators.devices Reference

## Non-arduino-side devices

The [configuration documentation](configuration) can guide you towards integrating computer-side devices like displays, cameras, microphones and speakers.

## Adding and using arduino-side devices

As the ecosystem of arduino compatible components and libraries is steadily growing, so is the number of components that can be incorporated into neuroscience experiments through neurokraken.

Neurokraken's `configurators.devices` allow you to mix and match arduino side devices according to experiment needs and to automatically create the arduino side code ready to upload to the teensy microcontroller.

For example the following start of a task script is already sufficient for a task that requires the the ability to read a rotation value and to control an LED light. It is also sufficient to run `config2teensy.py` to create the arduino side code for these devices connected to the teensy's pins 1, 2 and 3 respeticely without any further manual work required.

This ability to easily add and switch devices of tasks allows for fast development, testing and application of neurokraken tasks while maintaining flexibility for varying and updating task setups.

```python
from neurokraken import Neurokraken
from neurokraken.configurators import devices

serial_in = {'rotation': devices.rotary_encoder(pins=(1,2))}

serial_out = {'led': devices.direct_on(pin=3, start_value=False)
}
nk = Neurokraken(serial_in=serial_in, serial_out=serial_out)
```

In your task code you can then easily read the current rotation and set the state of the LED by using `neurokraken.controls.get` with the respective device names provided in serial_in and serial_out above.

```python
current_rotation = get.read_in('rotation')

get.send_out('led', True)
```

(configurators-devices)=
## Configurators.devices

Configurators.devices provides functions to create, mix and match devices for a given task. A task may utilize 1 devices.direct_on() and 1 devices.servo(), or it may utilize 5 devices.servo() all on different pins.

The devices can be added to the `serial_in` dictionary for sensors, and the `serial_out` dictionary for controlled devices, each entry with a chosen key string identifier that can later in your task be used to address sensors to read from and controls to update.

```python
from neurokraken import Neurokraken, State
from neurokraken import get
from neurokraken.configurators import devices

serial_in =  {'light_beam': devices.analog_read(pin=3)}
serial_out = {'reward_valve': devices.timed_on(pin=2)}

nk = Neurokraken(serial_in=serial_in, serial_out=serial_out)

class My_state(State):
    def loop_main(self):
        light_strength = get.read_in('light_beam')
        if light_strenght < 400:
            get.send_out('reward_valve', 100) # open for 100ms

nk.load_task(My_state())
nk.run()
```

## Sensors

Sensors have optional arguments for keyboard mode to assign keys and their effect on the sensor value (`keys=` and `keys_control=`) as detailed in the [modes chapter](modes.md). A second optional sensor argument is `logging=True/False` (defaults to True) which allows you to disable logging for a specific sensor.

```python
serial_in =  {'light_beam': devices.analog_read(pin=3, keys=['s', 'w'], logging=False)}
```

(devices.binary_read)=
### binary_read(pin:int)

A digital read of a pin (HIGH or Low), (True or False). Many devices can provide a suitable input to an arduino pin from simple buttons to touch sensor modules. Provides 0 or 1 upon get.read_in(<device name>).

```python
serial_in = {'licking_sensor': true_false_sensor(pin=3)}
...
print(get.read_in('licking_sensor')) # 0 or 1
```

(devices.analog_read)=
### analog_read(pin:int)

Analog read of a pin. Provides a continuous value 0 to 1023 upon get.read_in(<device name>).

Analog reads can be used with devices like an angle measuring potentiometer or light sensitive photoresistor.

```python
serial_in = {'rotation_potentiometer': continuous_sensor(pin=3)}
...
print(get.read_in('rotation_potentiometer')) # 387
```

(devices.rotary_encoder)=
### rotary encoder(pins:list)

Rotation position of rotary encoder. Returns -2,147,483,648 to +2,147,483,647 upon get.read_in(<device name>).

Rotary encoders are useful for high precision low friction rotation readings like on a steering wheel or treading wheel/disc.

The optional argument `controls=True/False` (defaults to False) can be used to return the optional control entry for serial_out instead of the sensor_value for serial_in. When adding the control running get.send_out(<device_name>) allows resetting the wheel position to 0.

```python
serial_in = {'wheel_pos': rotary_encoder(pins=[3,4])}
...
print(get.read_in('wheel_pos')) # -5738
```

Example with controls enabled.

```python
serial_in = {'wheel_pos': rotary_encoder(pins=[3,4])}
serial_out = {'wheel_pos': rotary_encoder(pins=[3,4], controls=True)} # optional
...
print(get.read_in('wheel_pos')) # -5738
get.send_out('wheel_pos', True) # reset the wheel position to 0 (only if control provided to serial_out)
```

(devices.pulse_clock)=
### pulse_clock(pin:int, change_period:int)

`change_period:int` - period in milliseconds for the pulse clock to switch HIGH/LOW

A periodically changing HIGH/LOW clock signal will be provided at the selected pin.
This is useful to align events with external devices like neural recording hardware that may not have access to the context of the task executing python but can log changes in connected digital signals.
I.e. a pulse_clock might be created to switch its HIGH/LOW state every 1000 milliseconds and its output pin connected to a recording device. To match the external device events at behavior time 500 seconds, we now just have to look at the time of its 500th digital signal change. 

Multiple pulse clocks at different frequencies can run in parallel for redundancy, i.e. 4 clocks switching HIGH/LOW every 100ms, 1s, 10s, 5 minutes respectively.

The pulse clock is a special device that is integrated with the start_stop control and will only be counting time when the neurokraken is started/active and until it is stopped/inactive.
At inactive state all pins will be HIGH. Upon get.start() pulse clock pins start at 0.

```python
serial_in = {'clock_100ms': pulse_clock(2,     100),
             'clock_1s':    pulse_clock(3,    1000),
             'clock_10s':   pulse_clock(4,  10_000),
             'clock_10min': pulse_clock(5, 300_000)}
...
print(get.read_in('clock_1s')) # 1 => This pin is currently HIGH 1 and not LOW 0"""
```

## Controls

(devices.direct_on)=
### direct_on(pin:int, start_value:bool=False)

`start_value:bool` : The initial value of the pin. Defaults to False (LOW).

Directly turn a pin HIGH (i=1) or LOW (i=0) upon get.send_out(<device_name>, i)

This can be used to control valves, buzzers, LEDs, ... anything you can control with a electric logic signal. These devices can also be time-limited activated with the timed_on device as an alternative.

```python
serial_out = {'odor_valve': direct_on_off(pin=3)}
...
get.send_out('odor_valve', 0) # turn the signal off / shut down the valve/LED/buzzer/...
```

(devices.timed_on)=
### timed_on(pin:int)

Similar to direct_on_off for the same devices, but only turns the digital signal HIGH for the provided milliseconds upon get.send_out(<device_name>, <milliseconds>) before turning the signal LOW again. This can be used to easily provide well timed stimulus lengths and reward sizes. The providable millisecond range is 0 to 65536.

```python
serial_out = {'reward_valve': timed_on(pin=3)}
...
get.send_out('reward_valve', 60) # open the valve for 60 milliseconds
```

(devices.servo)=
### servo(pin:int)

A servo motor to dynamically move experiment elements upon get.send_out(<device_name>, angle)
The angle value can range from 0 to 255.
This device can also be used to control the strength of an LED.
The pin has to among 0-15, 18-19, 22-25, 28-29, 33, 36-37 

```python
serial_out = {'servo': servo(3)}
...
get.send_out('servo', 100)
```

(devices.tone)=
### tone(pin:int)

Set the tone frequency to be output by a passive buzzer or piezo device.
This device uses the arduino tone function. Send 0 to not provide a tone.
The maximum providable frequency is 65_535.

```python
serial_out = {'frequency': tone(pin=3)}
get.send_out('frequency', 500)
```


## Configuators.devices are multi-functional

Many `configurators.devices` can be used multi-functionally. For example `devices.direct_on(pin:int)` allows you to set a HIGH or LOW voltage on the provided pin from your task code irrespective of the component receiving this HIGH/LOW signal. Thus a direct_on can be used to control an LED as well as a valve, an audible buzzer, a solenoid piston, a lamp, a heater,... 

Conversely some components can be used with different neurokraken.devices. For example our LED can be a `devices.direct_on(pin:int)` or a `devices.timed_on(pin:int)` that would default stay LOW but allow us to `get.send_out('led', 100)` to turn HIGH for a duration of exactly 100 milliseconds before turning LOW again, or it could even be run through a `devices.servo(pin:int)`.

Lastly, some modalities can be implemented with alternative components. For example a rotation angle can be sensed by a rotary encoder (`devices.rorary_encoder`) or a potentiometer (`devices.anaolg_read`) that trades resolution, continuous multi-rotation angles, and a ready to use low-friction axle for smaller size and cost.

Thus the device library is organized as follows:

1. [Modalities](modalities.md)
2. [Configurators.devices](configurators-devices) (this chapter)
3. [Wiring guides](Wiring-guides)

[Modalities](modalities.md) covers options for the different modalities one may want to implement and links to the corresponding [Configurators.devices](configurators-devices).

Further [Wiring guides](Wiring-guides) contains corresponding wiring diagrams for each device.

Lastly Ordering Options in the hardware chapter contains a table of affordable and useful components for different neuroscience setups with links and prices for the respective items.

## Adding your own device

While the existing neurokraken configurators.devices cover the dominant rabge of electronics that we can think of being useful in behavior setup, there will always be unanticipated ideas and needs.

Research progresses through the adoption of new ideas, so neurokraken makes it easy to add a new device we haven't considered yet.

If you have standalone arduino code for your device, this code can be wrapped within a Sensor, Control, or Process context (or a combination of these) for neurokraken to understand how to interact with it. [Our page on adding your own devices](arduino_custom) guides you through how to do this.

Outside of neurokraken's ability to add your own device we strive to continously extend the pre-existing configurators.devices according to popular needs of the research community.

## Adding an arduino as a device

[The add your own device page](extra-arduino) has an example of adding an arduino as a teensy connected device

Commonly a device will require one or two pins as argument. Sometimes a specific optional argument can be provided, like the `start_value=True/False` for a `devices.direct_on`.

### Adding and extending beyond configurators.devices

configurators.devices functions return standardized dictionaries, which can exchangeably be used directly in as entries in serial_in and serial_out. This way a new custom experiment device can directly be added as a dictionary matching the device's communicated data. configurators.devices are convenient for rapid development with established devices, but are not a limitation if more flexibility for new devices is required.

### A device as both sensor and control

Neurokraken can support a device being a sensor as well as a control. This is currently exemplified in configurators.devices.rotary_encoder which is a sensor but depending on the optional argument `control=` (defaults to False) can return the optional control entry for serial_out instead. Notably the key name provided in serial_in and serial_out has to match so that this device is recognized as the same.
