# Quickstart

## Installing Python and Neurokraken

If you are new to python we reccommend `uv` as the easiest way to install python for your system. 
`uv` is a python package and project manager that can be installed in a single windows powershell command. (https://docs.astral.sh/uv/getting-started/installation/)

```powershell
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

You can now run python projects and scripts.

---

To install the neurokraken python package for your scripts or projects, open a command prompt in your script or project folder and run the following commands.

If you are familiar with and prefer to use conda for python environment management follow the 2nd tab. If you prefer a local venv environment to be used without maintaining a standard pyproject.toml follow the 3rd.

::::{tab-set}
:::{tab-item} uv
:sync: uv
`uv init --bare` creates an empty pyproject.toml file (if one doesn't exist yet) and the `uv add` adds neurokraken to the pyproject.toml.
```bash
uv init --bare
uv add git+https://github.com/Neurokraken/Neurokraken.git
```
:::

:::{tab-item} conda
:sync: conda
```bash
conda create -n neurokraken-env python=3.13
conda activate neurokraken-env
pip install git+https://github.com/Neurokraken/Neurokraken.git
```
:::

:::{tab-item} uv (venv)
:sync: uv_venv
```bash
uv venv
uv pip install git+https://github.com/Neurokraken/Neurokraken.git
```
:::
::::

````{Dropdown} If your system doesn't have git installed
<!-- :open:  -->
- Git typically comes pre-installed with windows, but if it is missing you can download git from [git-scm.com](https://git-scm.com/). The installer will ask about many settings, but the default options should be perfectly fine.
- Once you have git installed (you can confirm this by typing `git` in the command line)
````

---

```{admonition} Arduino setup (optional) 
:class: tip, dropdown
To run tasks with connected actual electronics rather than just in keyboard/agent mode you need to install the arduino IDE with the teensyduino add-on. This allows you to upload neurokraken's auto-generated arduino code to your teensy microcontroller, but is not required to test-run and develop tasks in keyboard mode.
- [The arduino IDE](https://www.arduino.cc/en/software/) 
- [teensyduino](https://www.pjrc.com/teensy/teensyduino.html)
```

## Run examples

Within [the codebase](https://github.com/Neurokraken/Neurokraken) examples folder you will find a range of example tasks to started with different neurokraken use cases. Examples are documented in [the examples chapter](examples) and within their code use `mode='keyboard'` allowing them to directly run without connected electronics.

You can download the entire codebase (including the examples folder) with `git clone https://github.com/Neurokraken/Neurokraken.git`.
To run an example like corridor_3d.py we open a command prompt in the downloaded examples folder and run the following command.

::::{tab-set}
:::{tab-item} uv
:sync: uv
```bash
uv run corridor_3d.py
```
:::

:::{tab-item} conda
:sync: conda
test
```bash
conda activate neurokraken-env
python corridor_3d.py
```
:::
::::

In the following sections we will walk through how to develop a simple task from the beginning. In the end we cover how to leave keyboard mode and run the task with actual connected electronics and where to go next to explore neurokraken's capabilities.

## Create a minimal starting task

For a minimal Neurokraken task create a new .py file in a directory of your choice and add the following. We will cover elements and ways to add to it in the next steps.

```python
# setup configuration
from neurokraken import Neurokraken, State
from neurokraken.configurators import devices, Display, Camera, Microphone

serial_in = {}
serial_out = {}

nk = Neurokraken(serial_in=serial_in, serial_out=serial_out, log_dir='./', mode='keyboard')

# task design
from neurokraken.controls import get

class My_State(State):
    def loop_main(self):
        pass
    
task = My_State()

nk.load_task(task)

nk.run()
```

- Since we are using keyboard mode we don't need to connect a teensy.
- We can set up a python environment and run the task:
  - `>>> uv init --bare`
  - `>>> uv add git+https://github.com/Neurokraken/Neurokraken.git`
  - `>>> uv run my_task.py`
- with conda instad of uv we would run `conda activate neurokraken-env` and `python my_task.py`
- As loop_main is empty of code this task will continue doing nothing until manually ended.

```{note}
- Neurokraken tasks don't have to involve/require a display or UI (like this minimal example).
- You can still observe task start, progression and print statements in the console.
- A running Neurokraken task can be ended with the key combination `CTRL + ALT + Q` after which a session log folder will be finalized in the set `log_dir`.
```

## General Neurokraken task layout

Neurokraken tasks take the following form:
1. A Neurokraken() object is created with the device configuration of your task/setup. 
2. A task is loaded, which consists of any number of State container classes for your own python logic within.
3. Begin the task run().

````{mermaid}
flowchart LR;

neurokraken["Neurokraken()"]
load_task["load_task()"]
run["run()"]
sensors["Sensors (serial_in)"]
stimuli["Stimuli (serial_out)"]

sensors --> neurokraken
stimuli --> neurokraken
Displays --> neurokraken
Cameras --> neurokraken
Microphones --> neurokraken
Settings --> neurokraken

neurokraken --> load_task

subgraph state_a ["a state"]
    direction RL
    on_start["def on_start()"]
    loop_main["def loop_main()"]
    on_end["def on_end()"]
    loop_visual["def loop_visual()"]
    pre_task["def pre_task()"]
end

state_a --> load_task

load_task --> run
````

## Extend your starting task

- In this simple example we want to extend the minimal task to reward a subject for poking a sensor.
  - After a reward there will be 8 seconds delay state until the sensor will be responsive again
  - The Responsive state is indicated by a LED light.

```python
# setup configuration
from neurokraken import Neurokraken, State
from neurokraken.configurators import devices, Display, Camera, Microphone

serial_in =  {'light_beam': devices.analog_read(pin=3, keys=['s', 'w'])}
serial_out = {'reward_valve': devices.timed_on(pin=2),
              'LED': devices.direct_on(pin=4)}

nk = Neurokraken(serial_in=serial_in, serial_out=serial_out, log_dir='./', mode='keyboard')

# task design
from neurokraken.controls import get

class Poke_for_reward(State):
    def on_start(self):
        get.send_out('LED', True)

    def loop_main(self):
        if get.read_in('light_beam') < 400:
            get.send_out('reward_valve', 100)
            get.send_out('LED', False)
            get.progress_state('delay')

class Delay(State):
    pass

task = {
    'poke': Poke_for_reward(),
    'delay': Delay(next_state='poke', max_time_s=8)
}

nk.load_task(task)

nk.run()
```

---

**Task walkthrough:**

1. We added the new electronic devices to serial_in and serial_out
    - In simulated keyboard-mode, the light sensor input is controlled with LOW/HIGH values from keyboard keys *S / W*
---
2. loop_main() got extended to read the light sensor
    - if the light sensor is reading **<400** we consider the sensor to be touched:
        - open the reward_valve for 100ms
        - turn the LED off and progress to the delay state
    - adding a function to run on_start() let's us turn the LED back on when we re-enter the state.
---
3. We extend the task with a new Delay state and named out states
    - Our task is now a dictionary of 2 named states, 'poke' and 'delay'
    - *poke* will progress to *delay* on touch, otherwise run forever.
    - *delay* is an un-extended empty State, where nothing will happen. We provided two arguments for it to timeout after 8 seconds and transition back to *poke*

---

```{note}
Combine 
- `configurators`, `cameras`, `displays`, `devices`,
- `.get` access to `devices`, the `log` and camera frames
-  `States` and their `loop_main()`, `on_start()`, `on_end()`, `loop_visual(), `pre_task().`` 

to create your experiment condition of interest.
```

## Run with real devices

<u>A key feature of neurokraken is that the functional arduino-side code for your task is auto-created from your task code.</u>

I.e. our example above has all information required for its arduino side code - 3 devices, a valve on pin 2, an LED on pin 4 and a light sensor on pin 3.

To auto-create and upload a task's arduino side code you should have a cloned or downloaded copy of the neurokraken codebase from https://github.com/Neurokraken/Neurokraken on your computer. Two components of this folder are relevant to autocreate and upload your task's microcontroller side code, `config2teensy.py` to create the code, and the `teensy` folder containing the resulting code ready to upload with the arduino IDE.

1. Connect your electronics to the teensy according to the [device library wiring guide](device_library)
    - i.e. for the task above 1 light sensor, 1 LED, and 1 reward valve.
2. Run the neurokraken repository's `config2teensy.py`. It will ask you to drag and drop your task task .py script into the console and then press enter.
    - The repository's `teensy` folder now contains ready-to-upload arduino code for your task through an automatically created/updated `Config.h` file.
    - Devices added to your scripts serial_in and serial_out dictionaries are automatically included.
3. With the arduino IDE open the teensy folder/its `teensy.ino`, install potential missing libraries for your devices, select the teensy's USB connection in the dropdown menu at the top of the IDE and <u>press Upload</u> in the IDE. You should see the teensy's built-in orange LED flickering and then remaining on while the IDE console confirms a successful upload.
4. In your script change the Neurokraken's `mode='keyboard'` into `mode='teensy'` and run your task script. 
    - Your peripheral controls and sensors are now logged and accessed from your task python code 

## Where to next

[Task Examples](examples.md) covers more examples across the range of neurokraken applications. Neighboring navbar entries cover UI examples, keyboard/agent mode, AI/ML/Computer vision examples,...

[Controls](controls) provides a good introduction to task development and the reference to `get` the central interface element to neurokraken's managed elements.

Depending on your specific use case you can also dive deeper into the reference documentation for [States](states), the [Configuration](configuration), or the [Device Library](device_library)
