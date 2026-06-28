# Runner Mode Examples

In Runner mode tasks are organized in a folder containing separate files for the device configuration `config.py`, task `task.py`, and parallel code (like an UI) `launch.py`.

Runner mode can be run with `kraken.bat` (if you have an activateable conda environment named neurokraken) or `kraken.py`, which will provide you with a list of all tasks found within the `tasks` folder to choose from.
To run this example add a new folder with a name of your choice like `runner_mode_example` containing the respective files to `tasks`.
Runner mode can be provided with arguments for dynamic execution, like `kraken.bat --task runner_mode_example --keyboard` to directly execute the runner_mode_example task in keyboard mode.

The following example is the same as minimal.py above, with the addition of
- a list of datapoints to be prompted upon execution, including subject options
- a UI (`launch.py`) with a control for the task's shown color

> <span>config.py</span> <!-- span to avoid auto-formatting into a hyperlink -->

```python
# Instead of being bundled with the config.py a subjects dictionary could be loaded from a .json file or remote source
subjects = [{'ID': 'Alpha', 'sex': 'female'},
            {'ID': 'Beta',  'sex': 'female'},
            {'ID': 'Gamma', 'sex': 'male'},
            {'ID': 'Delta', 'sex': 'male'}]

# runner mode can prompt for named datapoints upon execution
ask_for = ['ID', 'weight', 'group']

from neurokraken.configurators import Display, devices
display = Display(size=(800, 600))

cameras = [ ]

serial_in = { }
serial_out = {'led': devices.direct_on(pin=3, start_value=False)}
```

> <span>task.py</span>

```python
from neurokraken.controls import get
from neurokraken import State

get.color = (0, 255, 0)

class Color(State):
    """We will just show the color on a display in this state and blink an LED every 5 seconds"""
    def on_start(self):
        # store relevant variables within the state self
        self.t_last_switch = 0
        self.led_status = False

    def loop_main(self):
        if get.time_ms > self.t_last_switch + 5_000:
            self.led_status = not self.led_status
            get.send_out('led', self.led_status)
            self.t_last_switch = get.time_ms
    
    def loop_visual(self, sketch):
        sketch.background(*get.color)

task = Color()
```

> <span>launch.py</span>

```python
# code that would be executed just before neurokraken.run() can be included in an optional launch.py
# This typically covers UIs and parallel analysis/processing loops like cutie

from py5 import Sketch
import krakengui as gui
from neurokraken.controls import get
import random

def recolor_background():
    get.color = (random.randint(0, 255), random.randint(0, 255), random.randint(0, 255))

class UI(Sketch):
    def settings(self):
        self.size(200, 150)

    def setup(self):
        gui.use_sketch(self)

        gui.Button(label='randomize background', pos=(25, 50), on_click=recolor_background)

    def draw(self):
        self.background(0)
        self.fill(255);     self.stroke(255);     self.text_size(15)
        self.text(f't_ms: {int(get.time_ms)}', 25, 25)
        if get.quitting:
            self.exit_sketch()

    def exiting(self):
        # close end the experiment when the UI window is closed
        get.quit()

# run the UI

ui = UI()
ui.run_sketch(block=False)
```