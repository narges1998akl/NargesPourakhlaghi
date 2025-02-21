<br/>

<p align="center">
<img src="/logo.png" title="Logo" />
</p>

<p align="center">
<img alt="PyPI Version" src="https://img.shields.io/pypi/v/diart?color=g">
<img alt="PyPI Downloads" src="https://static.pepy.tech/personalized-badge/diart?period=total&units=international_system&left_color=grey&right_color=brightgreen&left_text=downloads">
<img alt="Top language" src="https://img.shields.io/github/languages/top/juanmc2005/StreamingSpeakerDiarization?color=g">
<img alt="Code size in bytes" src="https://img.shields.io/github/languages/code-size/juanmc2005/StreamingSpeakerDiarization?color=g">
<img alt="License" src="https://img.shields.io/github/license/juanmc2005/StreamingSpeakerDiarization?color=g">
</p>

<div align="center">
  <h4>
    <a href="#installation">
      Installation
    </a>
    <span> | </span>
    <a href="#stream-audio">
      Stream audio
    </a>
    <span> | </span>
    <a href="#custom-models">
      Custom models
    </a>
    <span> | </span>
    <a href="#tune-hyper-parameters">
      Tune hyper-parameters
    </a>
    <span> | </span>
    <a href="#build-pipelines">
      Build pipelines
    </a>
    <br/>
    <a href="#websockets">
      WebSockets
    </a>
    <span> | </span>
    <a href="#powered-by-research">
      Research
    </a>
    <span> | </span>
    <a href="#citation">
      Citation
    </a>
    <span> | </span>
    <a href="#reproducibility">
      Reproducibility
    </a>
  </h4>
</div>

<br/>

<p align="center">
<img width="100%" src="/demo.gif" title="Real-time diarization example" />
</p>

## Installation

1) Create environment:

```shell
conda create -n diart python=3.8
conda activate diart
```

2) Install `PortAudio` and `soundfile`:

```shell
conda install portaudio
conda install pysoundfile -c conda-forge
```

3) Install diart:
```shell
pip install diart
```

### Get access to pyannote models

By default, diart is based on [pyannote.audio](https://github.com/pyannote/pyannote-audio) models stored in the [huggingface](https://huggingface.co/) hub.
To allow diart to use them, you need to follow these steps:

1) [Accept user conditions](https://huggingface.co/pyannote/segmentation) for the `pyannote/segmentation` model
2) [Accept user conditions](https://huggingface.co/pyannote/embedding) for the `pyannote/embedding` model
3) Install [huggingface-cli](https://huggingface.co/docs/huggingface_hub/quick-start#install-the-hub-library) and [log in](https://huggingface.co/docs/huggingface_hub/quick-start#login) with your user access token (or provide it manually in diart CLI or API).

## Stream audio

### From the command line

A recorded conversation:

```shell
diart.stream /path/to/audio.wav
```

A live conversation:

```shell
diart.stream microphone
```

See `diart.stream -h` for more options.

### From python

Use `RealTimeInference` to easily run a pipeline on an audio source and write the results to disk:

```python
from diart import OnlineSpeakerDiarization
from diart.sources import MicrophoneAudioSource
from diart.inference import RealTimeInference
from diart.sinks import RTTMWriter

pipeline = OnlineSpeakerDiarization()
mic = MicrophoneAudioSource(pipeline.config.sample_rate)
inference = RealTimeInference(pipeline, mic, do_plot=True)
inference.attach_observers(RTTMWriter(mic.uri, "/output/file.rttm"))
prediction = inference()
```

For inference and evaluation on a dataset we recommend to use `Benchmark` (see notes on [reproducibility](#reproducibility)).

## Custom models

Third-party models can be integrated seamlessly by subclassing `SegmentationModel` and `EmbeddingModel`:

```python
import torch
from typing import Optional
from diart import OnlineSpeakerDiarization, PipelineConfig
from diart.models import EmbeddingModel
from diart.sources import MicrophoneAudioSource
from diart.inference import RealTimeInference

class MyEmbeddingModel(EmbeddingModel):
    def __init__(self):
        super().__init__()
        self.my_pretrained_model = load("my_model.ckpt")
    
    def __call__(
        self,
        waveform: torch.Tensor,
        weights: Optional[torch.Tensor] = None
    ) -> torch.Tensor:
        return self.my_pretrained_model(waveform, weights)

config = PipelineConfig(embedding=MyEmbeddingModel())
pipeline = OnlineSpeakerDiarization(config)
mic = MicrophoneAudioSource(config.sample_rate)
inference = RealTimeInference(pipeline, mic)
prediction = inference()
```

## Tune hyper-parameters

Diart implements a hyper-parameter optimizer based on [optuna](https://optuna.readthedocs.io/en/stable/index.html) that allows you to tune any pipeline to any dataset.

### From the command line

```shell
diart.tune /wav/dir --reference /rttm/dir --output /output/dir
```

See `diart.tune -h` for more options.

### From python

```python
from diart.optim import Optimizer

optimizer = Optimizer("/wav/dir", "/rttm/dir", "/output/dir")
optimizer(num_iter=100)
```

This will write results to an sqlite database in `/output/dir`.

### Distributed optimization

For bigger datasets, it is sometimes more convenient to run multiple optimization processes in parallel.
To do this, create a study on a [recommended DBMS](https://optuna.readthedocs.io/en/stable/tutorial/10_key_features/004_distributed.html#sphx-glr-tutorial-10-key-features-004-distributed-py) (e.g. MySQL or PostgreSQL) making sure that the study and database names match:

```shell
mysql -u root -e "CREATE DATABASE IF NOT EXISTS example"
optuna create-study --study-name "example" --storage "mysql://root@localhost/example"
```

You can now run multiple identical optimizers pointing to this database:

```shell
diart.tune /wav/dir --reference /rttm/dir --storage mysql://root@localhost/example
```

or in python:

```python
from diart.optim import Optimizer
from optuna.samplers import TPESampler
import optuna

db = "mysql://root@localhost/example"
study = optuna.load_study("example", db, TPESampler())
optimizer = Optimizer("/wav/dir", "/rttm/dir", study)
optimizer(num_iter=100)
```

## Build pipelines

For a more advanced usage, diart also provides building blocks that can be combined to create your own pipeline.
Streaming is powered by [RxPY](https://github.com/ReactiveX/RxPY), but the `blocks` module is completely independent and can be used separately.

### Example

Obtain overlap-aware speaker embeddings from a microphone stream:

```python
import rx.operators as ops
import diart.operators as dops
from diart.sources import MicrophoneAudioSource
from diart.blocks import SpeakerSegmentation, OverlapAwareSpeakerEmbedding

segmentation = SpeakerSegmentation.from_pyannote("pyannote/segmentation")
embedding = OverlapAwareSpeakerEmbedding.from_pyannote("pyannote/embedding")
sample_rate = segmentation.model.get_sample_rate()
mic = MicrophoneAudioSource(sample_rate)

stream = mic.stream.pipe(
    # Reformat stream to 5s duration and 500ms shift
    dops.rearrange_audio_stream(sample_rate=sample_rate),
    ops.map(lambda wav: (wav, segmentation(wav))),
    ops.starmap(embedding)
).subscribe(on_next=lambda emb: print(emb.shape))

mic.read()
```

Output:

```
# Shape is (batch_size, num_speakers, embedding_dim)
torch.Size([1, 3, 512])
torch.Size([1, 3, 512])
torch.Size([1, 3, 512])
...
```

## WebSockets

Diart is also compatible with the WebSocket protocol to serve pipelines on the web.

In the following example we build a minimal server that receives audio chunks and sends back predictions in RTTM format:

```python
from diart import OnlineSpeakerDiarization
from diart.sources import WebSocketAudioSource
from diart.inference import RealTimeInference

pipeline = OnlineSpeakerDiarization()
source = WebSocketAudioSource(pipeline.config.sample_rate, "localhost", 7007)
inference = RealTimeInference(pipeline, source, do_plot=True)
inference.attach_hooks(lambda ann_wav: source.send(ann_wav[0].to_rttm()))
prediction = inference()
```

## Powered by research

Diart is the official implementation of the paper *[Overlap-aware low-latency online speaker diarization based on end-to-end local segmentation](/paper.pdf)* by [Juan Manuel Coria](https://juanmc2005.github.io/), [Hervé Bredin](https://herve.niderb.fr), [Sahar Ghannay](https://saharghannay.github.io/) and [Sophie Rosset](https://perso.limsi.fr/rosset/).


> We propose to address online speaker diarization as a combination of incremental clustering and local diarization applied to a rolling buffer updated every 500ms. Every single step of the proposed pipeline is designed to take full advantage of the strong ability of a recently proposed end-to-end overlap-aware segmentation to detect and separate overlapping speakers. In particular, we propose a modified version of the statistics pooling layer (initially introduced in the x-vector architecture) to give less weight to frames where the segmentation model predicts simultaneous speakers. Furthermore, we derive cannot-link constraints from the initial segmentation step to prevent two local speakers from being wrongfully merged during the incremental clustering step. Finally, we show how the latency of the proposed approach can be adjusted between 500ms and 5s to match the requirements of a particular use case, and we provide a systematic analysis of the influence of latency on the overall performance (on AMI, DIHARD and VoxConverse).

<p align="center">
<img height="400" src="/figure1.png" title="Visual explanation of the system" width="325" />
</p>

## Citation

If you found diart useful, please make sure to cite our paper:

```bibtex
@inproceedings{diart,  
  author={Coria, Juan M. and Bredin, Hervé and Ghannay, Sahar and Rosset, Sophie},  
  booktitle={2021 IEEE Automatic Speech Recognition and Understanding Workshop (ASRU)},   
  title={Overlap-Aware Low-Latency Online Speaker Diarization Based on End-to-End Local Segmentation}, 
  year={2021},
  pages={1139-1146},
  doi={10.1109/ASRU51503.2021.9688044},
}
```

##  Reproducibility

![Results table](/table1.png)

Diart aims to be lightweight and capable of real-time streaming in practical scenarios.
Its performance is very close to what is reported in the paper (and sometimes even a bit better).

To obtain the best results, make sure to use the following hyper-parameters:

| Dataset     | latency | tau    | rho    | delta |
|-------------|---------|--------|--------|-------|
| DIHARD III  | any     | 0.555  | 0.422  | 1.517 |
| AMI         | any     | 0.507  | 0.006  | 1.057 |
| VoxConverse | any     | 0.576  | 0.915  | 0.648 |
| DIHARD II   | 1s      | 0.619  | 0.326  | 0.997 |
| DIHARD II   | 5s      | 0.555  | 0.422  | 1.517 |

`diart.benchmark` and `diart.inference.Benchmark` can run, evaluate and measure the real-time latency of the pipeline. For instance, for a DIHARD III configuration:

```shell
diart.benchmark /wav/dir --reference /rttm/dir --tau=0.555 --rho=0.422 --delta=1.517 --segmentation pyannote/segmentation@Interspeech2021
```

or using the inference API:

```python
from diart.inference import Benchmark
from diart import OnlineSpeakerDiarization, PipelineConfig
from diart.models import SegmentationModel

config = PipelineConfig(
    # Set the model used in the paper
    segmentation=SegmentationModel.from_pyannote("pyannote/segmentation@Interspeech2021"),
    step=0.5,
    latency=0.5,
    tau_active=0.555,
    rho_update=0.422,
    delta_new=1.517
)
pipeline = OnlineSpeakerDiarization(config)
benchmark = Benchmark("/wav/dir", "/rttm/dir")
benchmark(pipeline)
```

This pre-calculates model outputs in batches, so it runs a lot faster.
See `diart.benchmark -h` for more options.

For convenience and to facilitate future comparisons, we also provide the [expected outputs](/expected_outputs) of the paper implementation in RTTM format for every entry of Table 1 and Figure 5. This includes the VBx offline topline as well as our proposed online approach with latencies 500ms, 1s, 2s, 3s, 4s, and 5s.

![Figure 5](/figure5.png)

##  License

```
MIT License

Copyright (c) 2021 Université Paris-Saclay
Copyright (c) 2021 CNRS

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

<p>Logo generated by <a href="https://www.designevo.com/" title="Free Online Logo Maker">DesignEvo free logo designer</a></p>









1. Target & Code function
this project is about recognizing the voice in a meeting, conversations, and.... whenever someone talks, the detectors can find and detect which voice belongs to which person. however it can find the muted part in conversation and omitted. finding the voice and recognizing can happened withoud overlapping. even more than 2 person talk in same time.
we use GMM cluster and SAD to find the voice source. the input ofcourse is voice and the outputs are numbers and results in papers.


2. describe innovation

you can find the inovation files among the other files and I describe it even in my video.

3. changes in codes

Speakers diarization with detecting the latency and overlap of online speakers
The project has been based on MIT databases for training network. 
In order to compile the desired code, the following stages must be taken:
1.install pycharm
2.install pip python --upgrade
the python version =3.9 (it is necessary)
3. pip install pip --upgrade
4. install anaconda (it takes time)
after the anaconda is installed the environment in conda based on python 3.9 is made.
5.the below libraries should be add in order to use them in code.
diart, tourch, tourchaudio, pyannote.audio 
6.conda create -n diart python=3.9
7.conda activate diart
8.conda create -n diart python=3.8
9.conda activate diart
10.pip install diart
11.pyannote.audio should be installed
12. the terms of MIT have to be consented by user by following the 
 https://huggingface.co/pyannote/segmentation
13. order the token for login open the below link:
14. pip install huggingface_hub
15.conda install -c conda-forge huggingface_hub
16. use the token code for logging to the MIT database
17.the main.py must be run
18. please enter the token number 
19. after compile the main.py
20. the script turn the laptop's speaker on and the voice of speaker are saved every 0.5 in buffers.
21. by using pipeline all voices are recorded and according to the frequency the speaker voices is recognized.
22. the network will be trained by 5 sec recording 
23. the crucial parts of this project are latency and the over-lap of speakers' voices.
   the problems are solved by separating the frequency according to the trained network and pytourch library.


4.results

you can see the results in the files amoung the other files uploaded.
also, the result contains:

In oreder to compile the desired code the following stages must be taken:
1.install pycharm
2.install pip python --upgrade
the python version =3.8 ( it is necessary)
3. pip instal pip --upgrade
4. inistall anaconda ( it takes time)
after the anaconda is inistalled the environment in conda based on python 3.8 is made.
5.the below libraries should be add in order to use them in code.
diart,tourch,tourchaudio,py
6.conda create -n diart python=3.9
7.conda activate diart
8.conda create -n diart python=3.8
9.conda activate diart
10.pip install diart
11.pyannote.audio should be installed
12. the terms of MIT have to be consented by user by following the 
 https://huggingface.co/pyannote/segmentation
13. order the token for login open the below link:
14. pip install huggingface_hub
15.conda install -c conda-forge huggingface_hub
16. use the token code for logging to the MIT database
17.the main.py must be run
18. please enter the token number 
19. after compile the main.py
20. the script turn the laptop's speaker on and the voice of speaker are saved every 0.5 in buffers.
21. by using pipeline all voices are recorded and according to the frequency the speaker voices is regognize.
22. the network will be trained by 5 sec recording 
23. the crucial part of this project is 


5. Reference

you can find the reference paper amoung the files uploaded.


6. Interduce myself

this is Narges Pourakhlaghi, Master student of Biomedical engineering- Bioelectric, in Islamic Azad University South Tehran Branch.
this project done for DSP-Dr. Mahdi Eslami, first semester of 1401

7.Paper file uploaded

8.explaining video

https://drive.google.com/file/d/1WM5v1-DJ5ZNK8mc_X8YUzLWRRy-BhOkp/view?usp=share_link

9.Proposal file uploaded
