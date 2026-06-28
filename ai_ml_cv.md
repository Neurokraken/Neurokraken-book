# AI/ML/Computer Vision Examples

## Shuttle_tracked

In this task for an open environment 2 touch-sensitive reward spouts are placed in the environment. Licking one triggers a reward and trigger receptivity at the other, encouraging subjects to rapidly and efficiently shuttle between the 2 spouts for a large number of rewards.

The task also integrates cutie for live tracking when setting `with_cutie = True`. Please check out the the [cutie examples](cutie_examples) and our [Cutie documentation](cutie_tracking) for more information on integrating cutie and live tracking into your tasks.

```python
with_cutie = False

# ------------------------ TASK SETUP ------------------------

from neurokraken import Neurokraken, State
from neurokraken.configurators import devices, Camera

serial_in = {
    'lick_left': devices.capacitive_touch(pins=[10, 11], keys=['a']),
    'lick_right': devices.capacitive_touch(pins=[29, 30], keys=['d'])
}

serial_out = {
    'reward_left': devices.timed_on(pin=40),
    'reward_right': devices.timed_on(pin=41)
}

cam_width, cam_height = 1280, 720
cameras = [Camera(name='camera', width=cam_width, height=cam_height)]
if not with_cutie:
    cameras = []

nk = Neurokraken(serial_in=serial_in, serial_out=serial_out, cameras=cameras,
                 log_dir='./', mode='keyboard')

#------------------------- CREATE A TASK AND RUN IT -------------------------

from neurokraken.controls import get

threshold = 4000

class Lick_left(State):
    def loop_main(self):
        global threshold
        if get.read_in("lick_left") > threshold:
            get.log['trials'][-1]['t_lick_l'] = get.read_in('t_ms')
            get.send_out('reward_right', 60)
            return True, 0 
        return False, 0

class Lick_right(State):
    def loop_main(self):
        global threshold
        if get.read_in("lick_right") > threshold:
            get.log['trials'][-1]['t_lick_r'] = get.read_in('t_ms')
            get.send_out('reward_left', 60)
            return True, 0 
        return False, 0

task = {
    'lick_left': Lick_left(max_time_s=60_000, next_state='lick_right'),
    'lick_right': Lick_right(max_time_s=60_000, next_state='lick_left', trial_complete=True)
}

nk.load_task(task)

if not with_cutie and __name__ == '__main__':
    nk.run()
    exit()

# ------------------------ CUTIE TRACKING AND UI ------------------------

# The approach is the same as in toolkit/cutie/live_webcam_example.py using the same optimizations of a 
# parallel_predict() loop/thread and pre-created numpy- and py5 images for shared usage and performance,
# but now with the added context of a neurokraken task

cutie_config_path = r'C:\path\to\cloned\repository\of\Cutie\cutie\config'
reference_folder = r'C:\Path\to\folder\of\reference\imageJPGs\and\maskPNGs\pairs\created\with\cutie\interactivedemo'

from py5 import Sketch
import threading
from pathlib import Path
import numpy as np

# import the cutie processing utilities script using import_file - this approach can be used for integrating
# python projects saved in disparate folders.
from neurokraken import tools
cutils_path = str(Path(__file__).parent.parent.parent.parent / 'toolkit/cutie/cutils.py')
cutils = tools.import_file(cutils_path)

cutils.create_cutie(cutie_config_path)
cutils.load_references(reference_folder)

processed_frame = np.zeros(shape=(cam_height, cam_width, 3), dtype=np.uint8)
masks =           np.zeros(shape=(cam_height, cam_width, 3), dtype=np.uint8)

def parallel_predict():
    # cutie will keep predicting frames in this loop. In a proper experiment we might also save the created masks,
    # calculate information like the center of mass, and add relevant data for the experiment to get.log
    global processed_frame, masks
    while True:
        if get.quitting:
            break
        frame = get.camera(0)
        # cutie works on RGB images => turn the frame from greyscale i.e. (720, 1280) into RGB (720, 1280, 3)
        frame = np.stack([frame, frame, frame], axis=-1)
        processed_frame, masks = cutils.predict_frame(frame, apply_pallete=True)

class UI(Sketch):
    def settings(self):
        self.size(800, 300, self.P2D)

    def setup(self):
        global processed_frame_py5, masks_py5
        processed_frame_py5 = self.create_image(1280, 720, self.RGB)
        masks_py5 =           self.create_image(1280, 720, self.RGB)

    def draw(self):
        global processed_frame, masks, processed_frame_py5, masks_py5

        self.background(50)
        
        self.create_image_from_numpy(processed_frame, bands='RGB', dst=processed_frame_py5)
        self.image(processed_frame_py5, 0, 0, 400, 300)

        self.create_image_from_numpy(masks, bands='RGB', dst=masks_py5)
        self.image(masks_py5, 400, 0, 400, 300)

    def exiting(self):
        get.quit()

threading.Thread(target=parallel_predict, daemon=True).start()

ui = UI()
ui.run_sketch(block=False)

nk.run()
```

(cutie_examples)=
## Cutie Examples

Additional cutie-related examples exist in Neurokraken's toolkit/cutie folder to demonstrate usage of standalone cutie or the `toolkit/cutils.py` in a standalone way or one that can be integrated into neurokraken task

### process_folder.py

> toolkit/cutie/process_folder.py

This example shows using `cutils.create_cutie()`, `cutils.load_references()` and `cutils.predict_folder()` to automatically process a folder of camera frames using a couple of pre-selected reference examples.

### live-webcam-example\.py

> toolkit/cutie/live-webcam-example.py

This example shows using `cutils.create_cutie()`, `cutils.load_references()` and `cutils.predict_frame()` to live process webcam frames with cutie. script contains 2 versions, one starting example in which cutie and the UI showing its results are run in the same draw loop, and one slightly longer alternative version where cutie runs in a separate thread for higher performance as it is likely to be used within actual neurokraken experiments.

## The AI processing loop

(cutie_tracking)=
## Tracking movements with Cutie

[Cutie](https://github.com/hkchengrex/Cutie) [[2310.12982] Putting the Object Back into Video Object Segmentation](https://arxiv.org/abs/2310.12982) is a Segmentation Model that we found to be very powerful for tracking the body, limbs, pupils, and other elements related to our experiment subjects. Cutie can run on a existing recorded video or with some minor modifications live within the context of your experiment. For example you can have an experiment state be depending on where a freely moving subject is currently positioned in an open or labyrinth environment, or when 2 tracked elements interact, or where a hand is placed. Or you can run Cutie on an existing video to to find correlations post the original experiment. Cutie doesn't have to be trained/fine-tuned on its targets and can work from once-made reference images with click selected targets. Unlike posenet models that provide you with skeleton points, a segmentation model provides you with the entire pixel area of your tracked target, i.e. an arm or snout. A target center can still easily be calculated, however having access to the occupied area could possibly also enable infering further information like a subject's respiration cycle.

### Installation

Prerequisites: 

- A nvidia cuda gpu with suffient VRAM. 16GB should be ok for most tasks. (Cutie can also run on the CPU but will be very slow - skip the pytorch(CUDA) install step to go for the CPU approach)
- We tested our install for windows, though the original repository will guide you through the install for ubuntu
- conda (i.e. miniconda), pip, git

open the command line in your target folder for cutie and clone the respository

`git clone https://github.com/hkchengrex/Cutie`

move into the cloned cutie folder

`cd Cutie`

create and activate conda environment for cutie's python packages
(For usage in another project you will want to instead activate its existing environment i.e. `conda activate neurokraken` and install cutie in there)

`conda create --name cutie python==3.12`

activate the conda environment

`conda activate cutie`

Go to https://pytorch.org/get-started/locally/ and copy the command to pip install pytorch (with CUDA) into your environment, i.e.

`pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu128`

Within the cutie folder you will find the package dependencies in a file `pyproject.toml`. Two packages in this list can cause trouble when installing on windows, so replace these lines to use a alternatives:

  `'cchardet >= 2.1.7',` into `'faust-cchardet',`
  `'pyqtdarktheme',` into `'pyqtdarktheme-fork',`

(in case the install then errors on netifaces, also replace the following line: `  'netifaces >= 0.11.0',` into `  'netifaces-plus',`)

Install the dependencies noted in pyproject.toml

`pip install .`

Download the pretrained models

`python cutie/utils/download_models.py`

Test your install by running the interactive gradio demo

`python interactive_demo.py --video ./examples/example.mp4 --num_objects 1`

--num_objects defines how many distinct elements you want to track.

If you receive an error about the line `qdarktheme.setup_theme("auto")` in interactive_demo.py you can comment out this line within the file and try again. Or try installing pyqtdarktheme-fork

See [Cutie/docs/INTERACTIVE.md at main · hkchengrex/Cutie · GitHub](https://github.com/hkchengrex/Cutie/blob/main/docs/INTERACTIVE.md) for how to provide a video or image-folder of your own or track multiple num_objects

### Typical Cutie Usage

A standard cutie workflow can look like this: Load a video => Select the active mask/element in the bottom left, then in the frame click-select targets of interest and right-click-unselect elements until the starting masks are good. => Commit that frame to memory as an important reference => propagate forward. If masks diverge from the target, pause propagation, click-correct how the masks should be => Commit that corrected frame to memory as an important reference => Propagate backward/forward until the neighboring frames fit the corrected tracking => keep propagating forward until the entire video is tracked or the next niche situation causes masks to diverge.

#### Parameter options

<details><summary>If you encounter a a masks sperading from the original target over extended time periods, you can follow this segment to adjust parameters to better fit for your data.
</summary>

Cutie's default parameters work very well for complete objects of the human world. However in research we often want to track more niche targets over long periods of time. With the default settings an arm-mask of a mouse for example can gradually minute by minute spread over the body. If we encounter this behavior, we want cutie to put more focus on permanent- and long-term memory and less on the short term working-memory development that might cause drift over long time periods. To shift towards long-term focus you can edit the parameters in the interactive_demo to:

`Min, working memory frames 1`

`max working memory frames 2`

`memory every 1`

`max long-term memory size 5000`

You can then scroll through your video, find representative frames of the various positions your target(s) can be in, click select your target(s) in those frames, and commit those frames to permanent memory. Those specific frames can be considered your reference images, representing the range of how your targets can look throughout your video data. After committing these frames with click selected masks to permanent memory, you can now propagate forward/backward to fill in the video.
</details>

### Tracking with reference images

In research you often have a setup that gets repeated multiple times, i.e. you have a series of videos of a rodent exploring an open environment. You can use your reference images and masks from the first rodent video to automatically track all future rodents in the same setup. When you load a video into cutie it saves the images and masks into the Cutie/workspace/{videoname}/images and /masks folder. This includes masks that cutie propagated as well as the original masks you click-created in the interactive_demo.py - so you can copy and back-up representative frames/masks for reusage. I.e. your video is the images 0000000.jpg, 0000001.jpg, 0000002.jpg...  with the corresponding masks 0000000.png, 0000001.png, 0000002.png...

To avoid click-selecting targets anew for every video, we want to replace those first few frames and masks with our representative reference frames and masks. For example we can copy the reference image/mask file pairs from a previous run into the new workspace images and masks folders, rename them into the starting frames 0000000.jpg, 0000001.jpg, 0000002.jpg and 0000000.png, 0000001.png, 0000002.png, and now when we start the interactive_demo those will be our new first video frames and masks. After committing them to permanent memory we can propagate forward and for example track a new different rodent with the once created reference from the first run. This approach works very well when you have a repeating experiment setup in which you want targets tracked.

Neurokraken's `toolkit/cutie/cutils.py` has functions that automate this process for running cutie with a folder of frames or even live video using a provided folder of reference images and a corresponding folder of reference masks to run as additional frames committed to memory before the actual video. You can find application examples in `toolkit/cutie/process_folder.py` and `live_webcam_example` and the [shuttle_tracked example task](examples).

### Considerations

There is a tradeoff between short-term and long-term focus. Short term-focus has smoother frame to frame edge transitions but comes with the risk of subelement masks slowly diffusing over the entire object with time requiring manual corrections. Long term focus prioritizes your perment-memory committed frames over recent developments but has a bit more jittery frame to frame edges.

Another tradeoff is that every frame you commit to permanent memory will increase your utilized GPU VRAM, so you cannot provide infinite reference frames and will want to choose frames that are representative of the range of how your tracked object can look throughout the video files.

### Further parameter options 

You can find the default parameters and extra options that the interactive_demo uses in cutie/config/gui_config.yaml. There you can find and edit entries corresponding to the parameters noted above as: max_mem_frames, min_mem_frames, mem_every, and max_num_tokens. Further parameters that can tried to edit for different performance/quality are amp: True and max_overall_size.

### Troubleshooting

If cutie crawls to a halt and you find your GPU memory maxed out close GPU VRAM heavy applications like other local AI.
