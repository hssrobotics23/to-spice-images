# To Spice Images

This repository describes the steps to fully synthesize a dataset of  labeled spice images, with precise ground truth bounding boxes. First, synthetic images are generated using a fork of Dall-E Flow, then text is added using a fork of SynthText. The included Jupyter notebook runs tests of text recognition on the images. 

A pre-generated dataset is publicly hosted on AWS, [for a demo in the jupyter notebook](#visualize). This notebook uses EasyOCR on the synthetic images, measuring text prediction accuracy and the precision of the bounding boxes. The Jupyter notebook concludes with a demo of recipe geneation by passing the recognized spices to OpenAI. To reproduce this work, you must have your own OpenAI API key.

## Dall-E Server

Note-- issue with [dalle-mini](https://github.com/borisdayma/dalle-mini/issues/330). Launch `1x A10 (24 GB PCIe)` instance [with Lambda Labs](https://cloud.lambdalabs.com/instances), then ssh and run:


```
sudo apt-get update
pip install -U pip

mkdir dalle && cd dalle
git clone https://github.com/hssrobotics23/dalle-flow.git
git clone https://github.com/jina-ai/SwinIR.git
git clone --branch v0.0.15 https://github.com/AmericanPresidentJimmyCarter/stable-diffusion.git
git clone https://github.com/CompVis/latent-diffusion.git
git clone https://github.com/jina-ai/glid-3-xl.git
git clone https://github.com/thejohnhoffer/clipseg.git

cd dalle-flow
pip install virtualenv
python3 -m virtualenv env
source env/bin/activate
pip install torch torchvision torchaudio --extra-index-url https://download.pytorch.org/whl/cu116
pip install numpy tqdm pytorch_lightning einops numpy omegaconf
pip install https://github.com/crowsonkb/k-diffusion/archive/master.zip
pip install git+https://github.com/AmericanPresidentJimmyCarter/stable-diffusion.git@v0.0.15
pip install basicsr facexlib gfpgan
pip install realesrgan
pip install xformers
cd latent-diffusion
pip install -e .
cd -
cd stable-diffusion
pip install .
cd -
cd SwinIR
pip install -e .
cd -
cd glid-3-xl
pip install -e .
cd -
cd clipseg
pip install -e .
cd -

cd glid-3-xl
wget https://dall-3.com/models/glid-3-xl/bert.pt
wget https://dall-3.com/models/glid-3-xl/kl-f8.pt
wget https://dall-3.com/models/glid-3-xl/finetune.pt
cd -

cd dalle-flow
pip install -r requirements.txt
pip install jaxlib~=0.3.25
pip install jax~=0.3.25
pip install jina~=3.19.0
pip install tb-nightly==2.12.0a20230118
pip install dalle-mini==0.1.3
pip install orbax==0.1.7
pip install flax==0.6.3

python3 flow_parser.py --enable-clipseg
python3 -m jina flow --uses flow.tmp.yml
```

## Running with Docker

#### TODO 

```
sudo chown ubuntu:docker /var/run/docker.sock
docker pull thejohnhoffer/to-spice-images:latest
docker run -e ENABLE_CLIPSEG --network host -it -v $HOME/.cache:/home/dalle/.cache --gpus all thejohnhoffer/to-spice-images

```


## Building Docker package

The build was run on an `1x A10 (24 GB PCIe)` instance [through Lambda Labs](https://cloud.lambdalabs.com/instances), using these steps to log in as the root user. If you don't have an SSH key, run `run `ssh-keygen -t ed25519 -C "email@example.com"`. Use the output from `cat ~/.ssh/id_ed25519.pub` to [add as a lambda labs SSH key](https://cloud.lambdalabs.com/ssh-keys). Now, copy the login IP [from the Lambda Labs instance](https://cloud.lambdalabs.com/instances), such as `ssh ubuntu@203.0.113.123`, to your shell. Once connected, run `sudo passwd root`, updating the root password, and run `su`, entering the new root password. Next, build the docker container using your `dockerhub_username` as a prefix.

```
git clone https://github.com/thejohnhoffer/dalle-flow.git
DOCKERHUB_USERNAME="dockerhub_username" && cd dalle-flow
docker build --build-arg GROUP_ID=$(id -g ${USER}) --build-arg USER_ID=$(id -u ${USER}) -t $DOCKERHUB_USERNAME/to-spice-images .
```

Publishing to Docker Hub, again using your dockerhub username.

```
docker login --username="$DOCKERHUB_USERNAME"
DOCKER_TAG=$(docker images | awk '/to-spice-images/{ print $3 }')
docker tag "$DOCKER_TAG" "$DOCKERHUB_USERNAME/to-spice-images:v3.3.0"
docker push "$DOCKERHUB_USERNAME/to-spice-images"
```

## Dall-E Client

In a separate shell on the same instance, run:

```
python client.py DALLE_PORT DIFFUSION_PORT UPSCALE_PORT SEGMENT_PORT
```

where DALLE_PORT, DIFFUSION_PORT, UPSCALE_PORT, SEGMENT_PORT are from the jina log:

```
    INFO   dalle/rep-0@180585 start server bound to 0.0.0.0:56768
    ...
    INFO   diffusion/rep-0@305004 start server bound to 0.0.0.0:55470
    ...
    INFO   upscaler/rep-0@178370 start server bound to 0.0.0.0:62240
    ...
    INFO   clipseg/rep-0@180633 start server bound to 0.0.0.0:57042
```

That is, you must manually find the ports searching through the server logs from the previous shell. The output will be a directory that depends on the parameters in the `client.py` file. The directory will have this format: `spices-v-3-3-0-check-10-of-12`, where `3-3-0` represents version `3.3.0`, and the "check 10 of 12" parameter describes the process of sorting `12` DALLEE images for prompt matching, thenn comparing the top `10` to an ideal size of the text destination segment.

## Text generation

Run the following in a shell with access to a `spices-v-3-3-0-...etc` folder from the last step.

```
git@github.com:hssrobotics23/SynthText.git
cd SynthText
pyenv install 3.6.7
pyenv local 3.6.7
python3 -m pip install --upgrade pip
python3 -m pip install -r requirements.txt
python3 -m pip install opencv-python
python gen.py "/path/to/spices-v-..."
```

Move the "results" directory to a folder formatted like `results-v-3-3-0-check-10-of-25`. When you have many such folders, run:

```
python merge_results.py
```

Note, the `PROMPTS` object must be updated in `merge_prompts.py` with the prompts for each version of `client.py`.

After running `merge_results.py`, you can sync up with the remote `S3` bucket

```
aws s3 sync merged s3://dgmd-s17-assets/train/generated-text-images/
```

Note, you must have access with `aws configure`. You will also recieve a `merged.json` file, which you can use for the next step


## Visualize

Using [conda](https://docs.anaconda.com/anaconda/install/windows/) is easier!

```
conda create -n visualize python=3.9
conda activate visualize
pip install matplotlib numpy openai
pip install jupyterlab opencv-python
pip install git+https://github.com/JaidedAI/EasyOCR.git@f947eaa36a55adb306feac58966378e01cc67f85
python3 -m pip install --force-reinstall -v "Pillow==9.5.0"
conda install nb_conda_kernels
python3 -m jupyterlab
```

Here are the `pyenv` instructions:

```
pyenv install 3.9.17
pyenv local 3.9.17
python3 -m pip install jupyterlab numpy
python3 -m pip install opencv-python
python3 -m pip install matplotlib openai
python3 -m pip install git+https://github.com/JaidedAI/EasyOCR.git@f947eaa36a55adb306feac58966378e01cc67f85
python3 -m pip install --force-reinstall -v "Pillow==9.5.0"
python3 -m jupyterlab
```

### Running the full notebook

To run the notebook to completion, you'll need:

- a local copy of `merged.json`, with an index to the `s3` demo images
- an openai API key, [available when logged in here](https://platform.openai.com/account/api-keys)

Note, you may follow [these steps](https://albertauyeung.github.io/2020/08/17/pyenv-jupyter.html/) to enable `pyenv` within `jupyterlab`. Note, the git install of EasyOCR is needed until the resolution of [this EasyOCR Issue](https://github.com/JaidedAI/EasyOCR/issues/1077)
