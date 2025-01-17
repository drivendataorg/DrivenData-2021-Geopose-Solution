# Overhead Geopose Challenge - The First Place Solution

![Overhead Geopose Challenge Leaderboard](img/final_leaderboard.png)

This repository contains source code and pre-trained models for the winning solution of the [Overhead Geopose Challenge](https://www.drivendata.org/competitions/78/overhead-geopose-challenge/leaderboard/).

## Installation

Installation should be pretty straightforward by installing python dependencies from requirements file:

`pip install -r requirements.txt`

I've used PyTorch 1.9, but older previous of PyTorch should be compatible as well. The `requirements.txt` file 
contains all dependencies you may need for training or making predictions. 


## Preprocessing

After installing required packages, there is one important step to make - preprocess train & test data. Specifically,
you need to run `python convert_j2k.py` script in order to convert j2k files to PNG format. I found that decoding of 
j2k format in Pillow is much slower than reading data as PNG using OpenCV library. 

To convert the dataset, simply run the following script:

```bash
export DRIVENDATA_OVERHEAD_GEOPOSE="/path/to/dataset"
python convert_j2k.py
# OR
python convert_j2k.py --data_dir="/path/to/dataset"
```

Instead of setting an environment variable, one can use command-line argument `--data_dir /path/to/dataset` instead.

## Reproducing winning solution

I've included two variants of the winning solution. Both scores 0.92+ on the private LB. 
Surprisingly, the best score obtained with the only two models in an ensemble. 
Second ensemble has three times more models in it and three times more computationally expensive. It's included for reference purposes.

Model checkpoints and inference configuration (TTA level, batch size, fp16/fp32 mode) is set by configuration files:

     - `configs/inference/b6_rdtsc_unet_fold9_b7_unet_fold0_d4_bo3.yaml` 2 models, D4 TTA (0.9240 private, 0.9236 public)
     - `configs/inference/b6_rdtsc_unet_fold9_b7_unet_fold0_d4_bo1.yaml` 6 models, D4 TTA (0.9202 private, 0.9183 public)

All model checkpoints hosted on the GitHub and will be downloaded automatically upon the first run of submission script.

### Single-GPU inference mode 

To generate a submission archive, one can run the following script:

```bash
export DRIVENDATA_OVERHEAD_GEOPOSE="/path/to/dataset"
python submit.py configs/inference/b6_rdtsc_unet_fold9_b7_unet_fold0_d4_bo1.yaml
# OR
python submit.py configs/inference/b6_rdtsc_unet_fold9_b7_unet_fold0_d4_bo1.yaml --data-dir "/path/to/dataset"
```
Submission archive can be later found at `submissions/b6_rdtsc_unet_fold9_b7_unet_fold0_d4_bo1` folder.

### Multi-GPU inference mode 

Inference time can be greatly reduced in case machine has multiple GPUs. In this case we can distribute the predictions across multiple GPUs and generate them N times faster.
The only change that is required - to call `submit_ddp.py` instead of `submit.py`. 

```bash
export DRIVENDATA_OVERHEAD_GEOPOSE="/path/to/dataset"
python submit_ddp.py configs/inference/b6_rdtsc_unet_fold9_b7_unet_fold0_d4_bo1.yaml
# OR
python submit_ddp.py configs/inference/b6_rdtsc_unet_fold9_b7_unet_fold0_d4_bo1.yaml --data-dir "/path/to/dataset"
```

### Inference time

A **dedicated GPU** with 11Gb of RAM should be enough to compute predictions for `configs/inference/0.9240_b6_rdtsc_unet_fold9_b7_unet_fold0_d4_bo1.yaml
and `configs/inference/0.9083_b6_rdtsc_unet_fold9.yaml`. Third submission has 9 models and would not fit into 11Gb (16 or 24 is preferred).

Inference time of the ensembles (Measured on 2080Ti)

|                                                    Config | Inference Time | Img/Sec | Num Models |
|-----------------------------------------------------------|----------------|---------|------------|
| 0.9083_b6_rdtsc_unet_fold9.yaml                           | 3h30m | 12.5s/it | 3 |
| 0.9202_b7_fold0_b7_rdtsc_fold1_b6_rdtsc_fold9_bo3_d4.yaml |  |  |  |  
| 0.9240_b6_rdtsc_unet_fold9_b7_unet_fold0_d4_bo1.yaml      |  |  |  |

Disabling D4 TTA or switching to D2 would increase inference speed by factor of 8 and 2 accordingly at a price of slightly lower R2 score.

## Training model from scratch

It's also possible to reproduce the training from a scratch. 
For convenience, I've included train scripts to repeat my training protocols. Please not, this is VERY time-consuming and GPU-demanding process.
I've used setup of 2x3090 NVidia GPUs each with 24Gb of RAM and was able to fit very small batch sizes. In addition, I've observed that mixed-precision
training had some issues with this challenge, and I had to switch to fp32 mode, which decreased batch size even more.

Training of each model takes ~3 full days on 2x3090 setup. Training could be started by sequentially running following scripts:

    - scripts/train_b6_rdtsc_unet_fold9.sh
    - scripts/train_b7_unet_fold0.sh

For different setups you may want to tune batch size and number of nodes to match the number of GPUs on the target machine.

### Adjusting train script for your needs

Let's break down the script itself to see how you can hack it. 
This training pipeline uses [hydra](https://github.com/facebookresearch/hydra) framework to configure experiment config.
With help of hydra you can assemble a complete configuration from separate blocks (model config, loss config, etc.).
In the example below, the model is `b6_rdtsc_unet` (See `configs/model/b6_rdtsc_unet.yaml`), 
optimizer - is AdamW with FP16 disabled (see `configs/optimizer/adamw_fp32.yaml`) and loss function 
defined in file `configs/loss/huber_cos.yaml`.

```bash
# scripts/train_b6_rdtsc_unet_fold9.sh
export DRIVENDATA_OVERHEAD_GEOPOSE="CHANGE_THIS_VALUE_TO_THE_LOCATION_OF_TRAIN_DATA"
export OMP_NUM_THREADS=8

python -m torch.distributed.launch --nproc_per_node=2 train.py\
  model=b6_rdtsc_unet\
  optimizer=adamw_fp32\
  scheduler=plateau\
  loss=huber_cos\
  train=light_768\
  train.epochs=300\
  train.early_stopping=50\
  train.show=False\
  train.loaders.train.num_workers=8\
  batch_size=3\
  dataset.fold=9\
  seed=777
```



## References

- [Albumentations](https://github.com/albumentations-team/albumentations)
- [Catalyst](https://github.com/catalyst-team/catalyst)
- [pytorch-toolbelt](https://github.com/BloodAxe/pytorch-toolbelt)
- [TIMM](https://github.com/rwightman/pytorch-image-models)
- [hydra](https://github.com/facebookresearch/hydra)
