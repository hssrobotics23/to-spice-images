## Dallee

Launch `1x A10 (24 GB PCIe)` instance [with Lambda Labs](https://cloud.lambdalabs.com/instances), then ssh and run:


```
sudo chown ubuntu:docker /var/run/docker.sock

sudo apt-get update
sudo apt-get install make build-essential libssl-dev zlib1g-dev libbz2-dev libreadline-dev libsqlite3-dev wget curl llvm libncursesw5-dev xz-utils tk-dev libxml2-dev libxmlsec1-dev libffi-dev liblzma-dev
curl https://pyenv.run | bash
echo 'export PATH="$HOME/.pyenv/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(pyenv init --path)"' >> ~/.bashrc
echo 'eval "$(pyenv virtualenv-init -)"' >> ~/.bashrc
exec $SHELL
pyenv install 3.10.12
pyenv global 3.10.12
mkdir dalle && cd dalle
git clone https://github.com/thejohnhoffer/dalle-flow.git
git clone https://github.com/jina-ai/SwinIR.git
git clone --branch v0.0.15 https://github.com/AmericanPresidentJimmyCarter/stable-diffusion.git
git clone https://github.com/CompVis/latent-diffusion.git
git clone https://github.com/jina-ai/glid-3-xl.git
git clone https://github.com/timojl/clipseg.git
python3 -m pip install torch torchvision torchaudio --extra-index-url https://download.pytorch.org/whl/cu118
python3 -m pip install numpy tqdm pytorch_lightning einops numpy omegaconf
python3 -m pip install https://github.com/crowsonkb/k-diffusion/archive/master.zip
python3 -m pip install git+https://github.com/AmericanPresidentJimmyCarter/stable-diffusion.git@v0.0.15
python3 -m pip install basicsr facexlib gfpgan
python3 -m pip install realesrgan
python3 -m pip install xformers
cd latent-diffusion && python3 -m pip install -e . && cd -
cd stable-diffusion && python3 -m pip install -e . && cd -
cd SwinIR && python3 -m pip install -e . && cd -
cd glid-3-xl && python3 -m pip install -e . && cd -
cd clipseg && python3 -m pip install -e . && cd -
cd glid-3-xl
wget https://dall-3.com/models/glid-3-xl/bert.pt
wget https://dall-3.com/models/glid-3-xl/kl-f8.pt
wget https://dall-3.com/models/glid-3-xl/finetune.pt
cd -
cd dalle-flow
python3 -m pip install Cython
python3 -m pip install -r requirements.txt
python3 -m pip install -U jaxlib==0.3.25+cuda11.cudnn82 -f https://storage.googleapis.com/jax-releases/jax_cuda_releases.html
python3 -m pip install pytorch-lightning==v1.7.7
python3 -m pip install transformers==4.25.1
python3 -m pip install torchmetrics==0.11.4
cd -
export JINA_AUTH_TOKEN="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"
mkdir ./stable-diffusion/models/ldm/stable-diffusion-v1
wget -O ./stable-diffusion/models/ldm/stable-diffusion-v1/model.ckpt https://huggingface.co/runwayml/stable-diffusion-inpainting/resolve/main/sd-v1-5-inpainting.ckpt
cd dalle-flow
pip install jina==3.11.2
python3 flow_parser.py --enable-stable-diffusion --enable-clipseg
python3 -m jina flow --uses flow.tmp.yml

```
