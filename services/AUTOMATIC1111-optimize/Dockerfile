FROM alpine/git:2.36.2 as download

COPY clone.sh /clone.sh

RUN . /clone.sh taming-transformers https://ghproxy.com/https://github.com/CompVis/taming-transformers.git 24268930bf1dce879235a7fddd0b2355b84d7ea6 \
  && rm -rf data assets **/*.ipynb

RUN . /clone.sh stable-diffusion-stability-ai https://ghproxy.com/https://github.com/Stability-AI/stablediffusion.git 47b6b607fdd31875c9279cd2f4f16b92e4ea958e \
  && rm -rf assets data/**/*.png data/**/*.jpg data/**/*.gif

RUN . /clone.sh CodeFormer https://ghproxy.com/https://github.com/sczhou/CodeFormer.git c5b4593074ba6214284d6acd5f1719b6c5d739af \
  && rm -rf assets inputs

RUN . /clone.sh BLIP https://ghproxy.com/https://github.com/salesforce/BLIP.git 48211a1594f1321b00f14c9f7a5b4813144b2fb9
RUN . /clone.sh k-diffusion https://ghproxy.com/https://github.com/crowsonkb/k-diffusion.git 5b3af030dd83e0297272d861c19477735d0317ec
RUN . /clone.sh clip-interrogator https://ghproxy.com/https://github.com/pharmapsychotic/clip-interrogator 2486589f24165c8e3b303f84e9dbbea318df83e8


FROM registry-vpc.cn-hangzhou.aliyuncs.com/hangjia/aria2:1.36 as xformers
RUN apk add --no-cache aria2
RUN aria2c -x 5 --dir / --out wheel.whl 'https://ghproxy.com/https://github.com/AbdBarho/stable-diffusion-webui-docker/releases/download/6.0.0/xformers-0.0.21.dev544-cp310-cp310-manylinux2014_x86_64-pytorch201.whl'


FROM python:3.10.9-slim

ENV DEBIAN_FRONTEND=noninteractive PIP_PREFER_BINARY=1

#增加镜像加速https://developer.aliyun.com/mirror/pypi
RUN mkdir -p ~/.pip && \
cat <<EOF > ~/.pip/pip.conf
[global]
index-url=http://pypi.douban.com/simple/
extra-index-url=
  https://mirrors.cloud.aliyuncs.com/pypi/simple
  https://pypi.tuna.tsinghua.edu.cn/simple
[install]
trusted-host=
  pypi.douban.com
  mirrors.cloud.aliyuncs.com
  pypi.tuna.tsinghua.edu.cn
EOF

RUN cat <<EOF > /etc/apt/sources.list
deb https://mirrors.aliyun.com/debian/ bullseye main non-free contrib
deb-src https://mirrors.aliyun.com/debian/ bullseye main non-free contrib
deb https://mirrors.aliyun.com/debian-security/ bullseye-security main
deb-src https://mirrors.aliyun.com/debian-security/ bullseye-security main
deb https://mirrors.aliyun.com/debian/ bullseye-updates main non-free contrib
deb-src https://mirrors.aliyun.com/debian/ bullseye-updates main non-free contrib
deb https://mirrors.aliyun.com/debian/ bullseye-backports main non-free contrib
deb-src https://mirrors.aliyun.com/debian/ bullseye-backports main non-free contrib
EOF


RUN --mount=type=cache,target=/var/cache/apt \
  apt-get update && \
  # we need those
  apt-get install -y fonts-dejavu-core rsync git jq moreutils aria2 \
  # extensions needs those
  ffmpeg libglfw3-dev libgles2-mesa-dev pkg-config libcairo2 libcairo2-dev


RUN --mount=type=cache,target=/cache --mount=type=cache,target=/root/.cache/pip \
  aria2c -x 5 --dir /cache --out torch-2.0.1-cp310-cp310-linux_x86_64.whl -c \
  https://download.pytorch.org/whl/cu118/torch-2.0.1%2Bcu118-cp310-cp310-linux_x86_64.whl && \
  pip install /cache/torch-2.0.1-cp310-cp310-linux_x86_64.whl torchvision --index-url https://download.pytorch.org/whl/cu118



RUN --mount=type=cache,target=/root/.cache/pip \
  git clone https://ghproxy.com/https://github.com/AUTOMATIC1111/stable-diffusion-webui.git && \
  cd stable-diffusion-webui && \
  git reset --hard 20ae71faa8ef035c31aa3a410b707d792c8203a3 && \
  pip install -r requirements_versions.txt

RUN --mount=type=cache,target=/root/.cache/pip  \
  --mount=type=bind,from=xformers,source=/wheel.whl,target=/xformers-0.0.21.dev544-cp310-cp310-manylinux2014_x86_64.whl \
  pip install /xformers-0.0.21.dev544-cp310-cp310-manylinux2014_x86_64.whl

ENV ROOT=/stable-diffusion-webui


COPY --from=download /repositories/ ${ROOT}/repositories/
RUN mkdir ${ROOT}/interrogate && cp ${ROOT}/repositories/clip-interrogator/data/* ${ROOT}/interrogate
RUN --mount=type=cache,target=/root/.cache/pip \
  pip install -r ${ROOT}/repositories/CodeFormer/requirements.txt

# RUN --mount=type=cache,target=/root/.cache/pip \
#   pip install pyngrok \
#   git+https://ghproxy.com/https://github.com/TencentARC/GFPGAN.git@8d2447a2d918f8eba5a4a01463fd48e45126a379 \
#   git+https://ghproxy.com/https://github.com/openai/CLIP.git@d50d76daa670286dd6cacf3bcd80b5e4823fc8e1 \
#   git+https://ghproxy.com/https://github.com/mlfoundations/open_clip.git@bb6e834e9c70d9c27d0dc3ecedeebeaeb1ffad6b

# Note: don't update the sha of previous versions because the install will take forever
# instead, update the repo state in a later step

# TODO: either remove if fixed in A1111 (unlikely) or move to the top with other apt stuff
RUN apt-get -y install libgoogle-perftools-dev && apt-get clean
ENV LD_PRELOAD=libtcmalloc.so

ARG SHA=20ae71faa8ef035c31aa3a410b707d792c8203a3
RUN --mount=type=cache,target=/root/.cache/pip \
  cd stable-diffusion-webui && \
  git fetch && \
  git reset --hard ${SHA} && \
  pip install -r requirements_versions.txt

COPY . /docker

RUN \
  python3 /docker/info.py ${ROOT}/modules/ui.py && \
  mv ${ROOT}/style.css ${ROOT}/user.css && \
  # one of the ugliest hacks I ever wrote \
  sed -i 's/in_app_dir = .*/in_app_dir = True/g' /usr/local/lib/python3.10/site-packages/gradio/routes.py && \
  git config --global --add safe.directory '*'

WORKDIR ${ROOT}
ENV NVIDIA_VISIBLE_DEVICES=all
ENV CLI_ARGS=""
EXPOSE 7860
ENTRYPOINT ["/docker/entrypoint.sh"]
CMD python -u webui.py --listen --port 7860 ${CLI_ARGS}
