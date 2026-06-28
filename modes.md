# Keyboard and Agent Mode Examples

## using agents

The following tasks provide examples of how to use `mode='agent'` to provide the same task that can be run by subjects in the physical space to one's written computer code to enage with the task instead. This allows for translational research between biological and AI subjects as well as automating task testing.

- The Neurokraken object is initialized with mode='agent' and an agent class. This class only needs to contain a function `act(self)` that can read in the environment and make an action, and a parameter `self.act_freq = int` - how frequently the agent will execute this function.
- While typical task code reads sensor data (`get.read_in()`) and sends out task controls (`get.send_out()`) the agent acts as the complementary opposite, accessing `get.serial_out[<x>][value]` to access the current state of stimulus peripherals, and updating `get.serial_in[<x>][value]` to provide sensor peripherals with its choices.
  - The agent code can also be written to access get.log for access to historical events
  - The agent can access task visual pixel data through a numpy array that we update every visual frame from loop_visual()

## agent_simple

agent_simple is a minimal example of using an agent in a task. A servo moves to a position left/center/right while the agent has access to 2 touch sensors and will be rewarded if he pressed the left sensor if the servo moved left and if he pressed the right sensor if the  and the agent has 2 touch sensors and will press the left one if the servo points left and the right one if the servo points right.

```{note}
An additional simple agent example in which a random agent steers a shape to alternating sides of the display can found in the [UI examples](agent_steering).
```

```python
from neurokraken import Neurokraken, State
from neurokraken.configurators import devices

serial_in = {
    'touch_left': devices.capacitive_touch(pins=[10, 11], keys=['a']),
    'touch_right': devices.capacitive_touch(pins=[29, 30], keys=['d'])
}

serial_out = {
    'servo': devices.servo(pin=14),
    'reward_valve': devices.timed_on(pin=40),
}

#---------------------------------- AGENT ----------------------------------

class Agent:
    def __init__(self):
        self.act_freq = 0.5 # act() will be run every 2 seconds

    def act(self):
        if get.serial_out['servo']['value'] == 10:
            print('I am touching the left sensor')
            get.serial_in['touch_left']['value'] = 6000
        elif get.serial_out['servo']['value'] == 245:
            print('I am touching the right sensor')
            get.serial_in['touch_right']['value'] = 6000
        else:
            # not touching either
            get.serial_in['touch_left']['value'] = 0
            get.serial_in['touch_right']['value'] = 0

agent = Agent()

nk = Neurokraken(serial_in=serial_in, serial_out=serial_out, mode='agent', agent=agent)

#----------------------------------- TASK -----------------------------------

from neurokraken.controls import get
import random
import numpy as np

# create a numpy array to store the current frame for the agent
get.frame = np.zeros(shape=(800, 600, 3), dtype=np.uint8)

class Intertrial(State):
    def on_start(self):
        get.send_out('servo', 127)                # arm in middle position

class Choice(State):
    def on_start(self):
        self.servo_pos = random.choice([10, 245]) # left or right position
        get.send_out('servo', self.servo_pos)
        print(f'servo as position {self.servo_pos}')
        
    def loop_main(self):
        if self.servo_pos == 10 and get.read_in('touch_left') > 4000:
            get.send_out('reward_valve', 50)
            return True, 0
        elif self.servo_pos == 245 and get.read_in('touch_right') > 4000:
            get.send_out('reward_valve', 50)
            return True, 0
        return False, 0
    
task = {
    'wait': Intertrial(max_time_s=3, next_state='choice'),
    'choice': Choice(max_time_s=15, next_state='wait'),
}

nk.load_task(task)

nk.run()
```


## agent_learning
- agent_learning is a more advanced example of an agent improving its choices with experience. 
- This example is best undestood starting with the task in the bottom (it can also be run with `mode='keyboard'`) before going to the code of the agent above.
- In this task every 4 seconds the task changes to a random combination of display color (green/blue) and tone frequency (600Hz/1200Hz) leading to 2x2=4 stimulus combinations. The subject can make a decision to press a button or not. If the button was pressed at the combination Green/low freq or Blue/high freq, a reward will be delivered while the other 2 combinations only deliver a reward if the button was not pressed during the interval. This can be thought of as the followwing matrix of whether a button press is good:

|       | low | high |
| ----- | --- | ---- |
| green | +   | -    |
| blue  | -   | +    |

- The example's simple agent contains this press_good? matrix (initialized at 0) and will press the button if its table entry is positive.
- The agent keeps tracks of its most recent stimuli, action choice, and the reward outcome, and then increments or decrements the table entry based on what button decision was or would have been rewarding.
- the table will quickly develop to resemble the ideal choice matrix above.
- The difficulty with reinforcement learning is in the implementation, so this example is a bit forced (i.e. learning frequency = choice frequency = trial frequency = 4s) to show how how a learner can be implemented in neurokraken. More powerful learner models for research are going to be more complex than this.

```python
from neurokraken import Neurokraken, State
from neurokraken.configurators import Display, devices

serial_in = {
    'button': devices.binary_read(pin=25, keys=['a'])
}

serial_out = {
    'reward_valve': devices.timed_on(pin=40),
    'beep':         devices.tone(pin=26)
}

#---------------------------------- AGENT ----------------------------------

class Agent:
    def __init__(self):
        self.act_freq = 5 # act() will be run 5 times per second
        
        # is pressing a button good in this environment? >0.5 yes, <0.5 no.    # ideal matrix
        # 1st dim green/blue/black color, 2nd dim low/high frequency           #     l  h
        self.pressgood_gb_lh = [[0.0, 0.0],                                    # g [[+, -],
                                [0.0, 0.0]]                                    # b  [-, +]]
     
        self.did_press = None
        self.last_env = None
        self.t_last_choice = 0

    def act(self):
        # the agent will make one decision every 4 seconds, and otherwise keep the button unpressed
        if get.time_ms  > self.t_last_choice + 4000:
            # --- learn from the outcome of the last action ---
            if self.did_press is not None:
                log_rewards = get.log['controls']['reward_valve']
                recent_rewards = [l for l in log_rewards if l[0] > get.time_ms - 4100]
                got_rewarded = len(recent_rewards) != 0
                print(f'my last environment, choice, reward: {self.last_env, self.did_press, got_rewarded}')
                # should pressgood for the experienced environment be increased or decreased
                delta = 0
                if self.did_press and got_rewarded:         delta = +0.1
                if self.did_press and not got_rewarded:     delta = -0.1
                if not self.did_press and got_rewarded:     delta = -0.1
                if not self.did_press and not got_rewarded: delta = +0.1
                self.pressgood_gb_lh[self.last_env[0]][self.last_env[1]] += delta
                # Now that we learned from this experience we can set it back to 0
                self.did_press = None
                print('my new choice matrix:', self.pressgood_gb_lh)
            
            # --- act based on the environment and experience ---
            # process the visual and audio into state spaces of 0 or 1 respectively
            color = np.mean(get.frame, axis=(0,1))
            color = 0 if color.tolist() == [0, 255, 0] else 1
            audio = 0 if get.serial_out['beep']['value'] < 700 else 1
            self.last_env = [color, audio]
            
            # look up experience whether in this environment a button press is good and do the learned button action
            press_good = True if self.pressgood_gb_lh[color][audio] > 0.0 else False
            if press_good == True:
                print(f'Observation: {self.last_env} - I am pressing the button')
                get.serial_in['button']['value'] = True
                self.did_press = True
            else:
                print(f'Observation: {self.last_env} - I am waiting out this stimulus')
                self.did_press = False
            
            self.t_last_choice = get.time_ms
        else:
            get.serial_in['button']['value'] = False

agent = Agent()

nk = Neurokraken(serial_in=serial_in, serial_out=serial_out, 
                 display=Display(size=(800, 600)), mode='agent', agent=agent)

#----------------------------------- TASK -----------------------------------

from neurokraken.controls import get
import random
import threading, winsound
import numpy as np

# create a numpy array to store the current frame for the agent
get.frame = np.zeros(shape=(800, 600, 3), dtype=np.uint8)

class Choice(State):
    def on_start(self):
        self.color = random.choice(['green', 'blue'])
        self.frequency = random.choice([600, 1200])
        self.start_t = get.time_ms
        self.button_was_pressed = False
        threading.Thread(target=lambda: winsound.Beep(self.frequency, 2000)).start()
    
    def loop_main(self):
        get.send_out('beep', self.frequency)
        # color and tone form 2x2 combinations. 2 combinations will reward pressing the button during
        # the 4s state and 2 combinations (green+low, blue+high) will reward waiting out the 4s
        # without pressing it
        if not self.button_was_pressed:
            if get.read_in('button'):
                if (self.color == 'green' and self.frequency ==  600) or \
                    (self.color == 'blue'  and self.frequency == 1200):
                    print('---REWARD---')
                    get.send_out('reward_valve', 40)
                self.button_was_pressed = True
        if get.time_ms - self.start_t > 3900:
            if not self.button_was_pressed:
                if (self.color == 'green' and self.frequency == 1200) or \
                    (self.color == 'blue'  and self.frequency ==  600):
                    print('---REWARD---')
                    get.send_out('reward_valve', 40)
            # the time is up, an outcome for not pressing was provided (reward or nothing) -> progress to the next trial
            get.progress_state('choice')
    
    def loop_visual(self, sketch):
        if self.color == 'blue':
            sketch.background(0, 0, 255)
        if self.color == 'green':
            sketch.background(0, 255, 0)
        # update the frame data
        get.frame = sketch.get_np_pixels(bands='RGB')

task = {'choice': Choice()}

nk.load_task(task)

nk.run()
```