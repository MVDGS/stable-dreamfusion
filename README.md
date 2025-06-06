# Stable-Dreamfusion

A pytorch implementation of the text-to-3D model **Dreamfusion**, powered by the [Stable Diffusion](https://github.com/CompVis/stable-diffusion) text-to-2D model.

This repository is for environment setup and inference of the paper "DreamFusion: Text-to-3D using 2D Diffusion"
<p align="center">
  <img src="assets/result1_ep0100.gif" alt="Result1" width="30%"/>
  <img src="assets/result2_ep0100.gif" alt="Result2" width="30%"/>
  <img src="assets/result3_ep0100.gif" alt="Result3" width="30%"/>
</p>
# Citation
```
@misc{stable-dreamfusion,
    Author = {Jiaxiang Tang},
    Year = {2022},
    Note = {https://github.com/ashawkey/stable-dreamfusion},
    Title = {Stable-dreamfusion: Text-to-3D with Stable-diffusion}
}
```

### Setup

#### Local Setup

Our default, provided install method is based on Conda package with python 3.9:
```shell
SET DISTUTILS_USE_SDK=1 # Windows only
conda create conda create -n stable_dreamfusion python=3.9 
conda activate stable_dreamfusion
```
Please note that this process assumes that you have CUDA SDK **11.6** installed, not **12**. 
Cheak dependencies.sh file for other packages.


```shell
pip install torch==1.12.1+cu116 torchvision==0.13.1+cu116 torchaudio==0.12.1 \
  -f https://download.pytorch.org/whl/torch_stable.html

pip install numpy==1.23.5 --force-reinstall
pip install torchaudio==0.12.1+cu116 -f https://download.pytorch.org/whl/torch_stable.html
pip install -r requirements.txt

pip install git+https://github.com/NVlabs/nvdiffrast.git
pip install git+https://github.com/openai/CLIP.git
```

### Download pre-trained models

To use image-conditioned 3D generation, you need to download some pretrained checkpoints manually:
* [Zero-1-to-3](https://github.com/cvlab-columbia/zero123) for diffusion backend.
    We use `zero123-xl.ckpt` by default, and it is hard-coded in `guidance/zero123_utils.py`.
    ```bash
    cd pretrained/zero123
    wget https://zero123.cs.columbia.edu/assets/zero123-xl.ckpt
    ```
* [Omnidata](https://github.com/EPFL-VILAB/omnidata/tree/main/omnidata_tools/torch) for depth and normal prediction.
    These ckpts are hardcoded in `preprocess_image.py`.
    ```bash
    mkdir pretrained/omnidata
    cd pretrained/omnidata
    # assume gdown is installed
    gdown '1Jrh-bRnJEjyMCS7f-WsaFlccfPjJPPHI&confirm=t' # omnidata_dpt_depth_v2.ckpt
    gdown '1wNxVO4vVbDEMEpnAi_jwQObf2MFodcBR&confirm=t' # omnidata_dpt_normal_v2.ckpt
    ```

To use [DeepFloyd-IF](https://github.com/deep-floyd/IF), you need to accept the usage conditions from [hugging face](https://huggingface.co/DeepFloyd/IF-I-XL-v1.0), and login with `huggingface-cli login` in command line.

For DMTet, we port the pre-generated `32/64/128` resolution tetrahedron grids under `tets`.
The 256 resolution one can be found [here](https://drive.google.com/file/d/1lgvEKNdsbW5RS4gVxJbgBS4Ac92moGSa/view?usp=sharing).



# Usage

First time running will take some time to compile the CUDA extensions.

```bash
#### stable-dreamfusion setting

### Instant-NGP NeRF Backbone
# + faster rendering speed
# + less GPU memory (~16G)
# - need to build CUDA extensions (a CUDA-free Taichi backend is available)

## train with text prompt (with the default settings)
# `-O` equals `--cuda_ray --fp16`
# `--cuda_ray` enables instant-ngp-like occupancy grid based acceleration.
python main.py --text "a hamburger" --workspace trial -O

# reduce stable-diffusion memory usage with `--vram_O`
# enable various vram savings (https://huggingface.co/docs/diffusers/optimization/fp16).
python main.py --text "a hamburger" --workspace trial -O --vram_O

# You can collect arguments in a file. You can override arguments by specifying them after `--file`. Note that quoted strings can't be loaded from .args files...
python main.py --file scripts/res64.args --workspace trial_awesome_hamburger --text "a photo of an awesome hamburger"

# use CUDA-free Taichi backend with `--backbone grid_taichi`
python3 main.py --text "a hamburger" --workspace trial -O --backbone grid_taichi

# choose stable-diffusion version (support 1.5, 2.0 and 2.1, default is 2.1 now)
python main.py --text "a hamburger" --workspace trial -O --sd_version 1.5

# use a custom stable-diffusion checkpoint from hugging face:
python main.py --text "a hamburger" --workspace trial -O --hf_key andite/anything-v4.0

# use DeepFloyd-IF for guidance (experimental):
python main.py --text "a hamburger" --workspace trial -O --IF
python main.py --text "a hamburger" --workspace trial -O --IF --vram_O # requires ~24G GPU memory

# we also support negative text prompt now:
python main.py --text "a rose" --negative "red" --workspace trial -O

## after the training is finished:
# test (exporting 360 degree video)
python main.py --workspace trial -O --test
# also save a mesh (with obj, mtl, and png texture)
python main.py --workspace trial -O --test --save_mesh
# test with a GUI (free view control!)
python main.py --workspace trial -O --test --gui

### Vanilla NeRF backbone
# + pure pytorch, no need to build extensions!
# - slow rendering speed
# - more GPU memory

## train
# `-O2` equals `--backbone vanilla`
python main.py --text "a hotdog" --workspace trial2 -O2

# if CUDA OOM, try to reduce NeRF sampling steps (--num_steps and --upsample_steps)
python main.py --text "a hotdog" --workspace trial2 -O2 --num_steps 64 --upsample_steps 0

## test
python main.py --workspace trial2 -O2 --test
python main.py --workspace trial2 -O2 --test --save_mesh
python main.py --workspace trial2 -O2 --test --gui # not recommended, FPS will be low.

### DMTet finetuning

## use --dmtet and --init_with <nerf checkpoint> to finetune the mesh at higher reslution
python main.py -O --text "a hamburger" --workspace trial_dmtet --dmtet --iters 5000 --init_with trial/checkpoints/df.pth

## init dmtet with a mesh to generate texture
# require install of cubvh: pip install git+https://github.com/ashawkey/cubvh
# remove --lock_geo to also finetune geometry, but performance may be bad.
python main.py -O --text "a white bunny with red eyes" --workspace trial_dmtet_mesh --dmtet --iters 5000 --init_with ./data/bunny.obj --lock_geo

## test & export the mesh
python main.py -O --text "a hamburger" --workspace trial_dmtet --dmtet --iters 5000 --test --save_mesh

## gui to visualize dmtet
python main.py -O --text "a hamburger" --workspace trial_dmtet --dmtet --iters 5000 --test --gui

### Image-conditioned 3D Generation

## preprocess input image
# note: the results of image-to-3D is dependent on zero-1-to-3's capability. For best performance, the input image should contain a single front-facing object, it should have square aspect ratio, with <1024 pixel resolution. Check the examples under ./data.
# this will exports `<image>_rgba.png`, `<image>_depth.png`, and `<image>_normal.png` to the directory containing the input image.
python preprocess_image.py <image>.png
python preprocess_image.py <image>.png --border_ratio 0.4 # increase border_ratio if the center object appears too large and results are unsatisfying.

## zero123 train
# pass in the processed <image>_rgba.png by --image and do NOT pass in --text to enable zero-1-to-3 backend.
python main.py -O --image <image>_rgba.png --workspace trial_image --iters 5000

# if the image is not exactly front-view (elevation = 0), adjust default_polar (we use polar from 0 to 180 to represent elevation from 90 to -90)
python main.py -O --image <image>_rgba.png --workspace trial_image --iters 5000 --default_polar 80

# by default we leverage monocular depth estimation to aid image-to-3d, but if you find the depth estimation inaccurate and harms results, turn it off by:
python main.py -O --image <image>_rgba.png --workspace trial_image --iters 5000 --lambda_depth 0

python main.py -O --image <image>_rgba.png --workspace trial_image_dmtet --dmtet --init_with trial_image/checkpoints/df.pth

## zero123 with multiple images
python main.py -O --image_config config/<config>.csv --workspace trial_image --iters 5000

## render <num> images per batch (default 1)
python main.py -O --image_config config/<config>.csv --workspace trial_image --iters 5000 --batch_size 4

# providing both --text and --image enables stable-diffusion backend (similar to make-it-3d)
python main.py -O --image hamburger_rgba.png --text "a DSLR photo of a delicious hamburger" --workspace trial_image_text --iters 5000

python main.py -O --image hamburger_rgba.png --text "a DSLR photo of a delicious hamburger" --workspace trial_image_text_dmtet --dmtet --init_with trial_image_text/checkpoints/df.pth

## test / visualize
python main.py -O --image <image>_rgba.png --workspace trial_image_dmtet --dmtet --test --save_mesh
python main.py -O --image <image>_rgba.png --workspace trial_image_dmtet --dmtet --test --gui

### Debugging

# Can save guidance images for debugging purposes. These get saved in trial_hamburger/guidance.
# Warning: this slows down training considerably and consumes lots of disk space!
python main.py --text "a hamburger" --workspace trial_hamburger -O --vram_O --save_guidance --save_guidance_interval 5 # save every 5 steps
```

# Acknowledgement

This work is based on an increasing list of amazing research works and open-source projects, thanks a lot to all the authors for sharing!

* [DreamFusion: Text-to-3D using 2D Diffusion](https://dreamfusion3d.github.io/)
    ```
    @article{poole2022dreamfusion,
        author = {Poole, Ben and Jain, Ajay and Barron, Jonathan T. and Mildenhall, Ben},
        title = {DreamFusion: Text-to-3D using 2D Diffusion},
        journal = {arXiv},
        year = {2022},
    }
    ```

* [Magic3D: High-Resolution Text-to-3D Content Creation](https://research.nvidia.com/labs/dir/magic3d/)
   ```
   @inproceedings{lin2023magic3d,
      title={Magic3D: High-Resolution Text-to-3D Content Creation},
      author={Lin, Chen-Hsuan and Gao, Jun and Tang, Luming and Takikawa, Towaki and Zeng, Xiaohui and Huang, Xun and Kreis, Karsten and Fidler, Sanja and Liu, Ming-Yu and Lin, Tsung-Yi},
      booktitle={IEEE Conference on Computer Vision and Pattern Recognition ({CVPR})},
      year={2023}
    }
   ```

* [Zero-1-to-3: Zero-shot One Image to 3D Object](https://github.com/cvlab-columbia/zero123)
    ```
    @misc{liu2023zero1to3,
        title={Zero-1-to-3: Zero-shot One Image to 3D Object},
        author={Ruoshi Liu and Rundi Wu and Basile Van Hoorick and Pavel Tokmakov and Sergey Zakharov and Carl Vondrick},
        year={2023},
        eprint={2303.11328},
        archivePrefix={arXiv},
        primaryClass={cs.CV}
    }
    ```
    
* [Perp-Neg: Re-imagine the Negative Prompt Algorithm: Transform 2D Diffusion into 3D, alleviate Janus problem and Beyond](https://perp-neg.github.io/)
    ```
    @article{armandpour2023re,
      title={Re-imagine the Negative Prompt Algorithm: Transform 2D Diffusion into 3D, alleviate Janus problem and Beyond},
      author={Armandpour, Mohammadreza and Zheng, Huangjie and Sadeghian, Ali and Sadeghian, Amir and Zhou, Mingyuan},
      journal={arXiv preprint arXiv:2304.04968},
      year={2023}
    }
    ```
    
* [RealFusion: 360° Reconstruction of Any Object from a Single Image](https://github.com/lukemelas/realfusion)
    ```
    @inproceedings{melaskyriazi2023realfusion,
        author = {Melas-Kyriazi, Luke and Rupprecht, Christian and Laina, Iro and Vedaldi, Andrea},
        title = {RealFusion: 360 Reconstruction of Any Object from a Single Image},
        booktitle={CVPR}
        year = {2023},
        url = {https://arxiv.org/abs/2302.10663},
    }
    ```

* [Fantasia3D: Disentangling Geometry and Appearance for High-quality Text-to-3D Content Creation](https://fantasia3d.github.io/)
    ```
    @article{chen2023fantasia3d,
        title={Fantasia3D: Disentangling Geometry and Appearance for High-quality Text-to-3D Content Creation},
        author={Rui Chen and Yongwei Chen and Ningxin Jiao and Kui Jia},
        journal={arXiv preprint arXiv:2303.13873},
        year={2023}
    }
    ```

* [Make-It-3D: High-Fidelity 3D Creation from A Single Image with Diffusion Prior](https://make-it-3d.github.io/)
    ```
    @article{tang2023make,
        title={Make-It-3D: High-Fidelity 3D Creation from A Single Image with Diffusion Prior},
        author={Tang, Junshu and Wang, Tengfei and Zhang, Bo and Zhang, Ting and Yi, Ran and Ma, Lizhuang and Chen, Dong},
        journal={arXiv preprint arXiv:2303.14184},
        year={2023}
    }
    ```

* [Stable Diffusion](https://github.com/CompVis/stable-diffusion) and the [diffusers](https://github.com/huggingface/diffusers) library.

    ```
    @misc{rombach2021highresolution,
        title={High-Resolution Image Synthesis with Latent Diffusion Models},
        author={Robin Rombach and Andreas Blattmann and Dominik Lorenz and Patrick Esser and Björn Ommer},
        year={2021},
        eprint={2112.10752},
        archivePrefix={arXiv},
        primaryClass={cs.CV}
    }

    @misc{von-platen-etal-2022-diffusers,
        author = {Patrick von Platen and Suraj Patil and Anton Lozhkov and Pedro Cuenca and Nathan Lambert and Kashif Rasul and Mishig Davaadorj and Thomas Wolf},
        title = {Diffusers: State-of-the-art diffusion models},
        year = {2022},
        publisher = {GitHub},
        journal = {GitHub repository},
        howpublished = {\url{https://github.com/huggingface/diffusers}}
    }
    ```

* The GUI is developed with [DearPyGui](https://github.com/hoffstadt/DearPyGui).

* Puppy image from : https://www.pexels.com/photo/high-angle-photo-of-a-corgi-looking-upwards-2664417/

* Anya images from : https://www.goodsmile.info/en/product/13301/POP+UP+PARADE+Anya+Forger.html

