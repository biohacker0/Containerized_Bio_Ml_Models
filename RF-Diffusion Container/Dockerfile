FROM nvcr.io/nvidia/cuda:11.6.2-cudnn8-runtime-ubuntu20.04

# Install dependencies
RUN apt-get -q update \ 
  && DEBIAN_FRONTEND=noninteractive apt-get install --no-install-recommends -y \
  git \
  python3.9 \
  python3-pip \
  && python3.9 -m pip install -q -U --no-cache-dir pip \
  && rm -rf /var/lib/apt/lists/* \
  && apt-get autoremove -y \
  && apt-get clean

# Clone RFdiffusion source
RUN git clone https://github.com/RosettaCommons/RFdiffusion.git /app/RFdiffusion

# Install specific versions of numpy and torch
RUN pip install -q --no-cache-dir \
  numpy==1.23.5 \
  torch==1.12.1+cu116 --extra-index-url https://download.pytorch.org/whl/cu116

# Fetch SE3Transformer if not present
RUN if [ ! -d /app/RFdiffusion/env/SE3Transformer ]; then \
      git clone https://github.com/lucidrains/se3-transformer-pytorch /app/RFdiffusion/env/SE3Transformer; \
    fi

# Install other dependencies
RUN pip install -q --no-cache-dir \
  dgl==1.0.2+cu116 -f https://data.dgl.ai/wheels/cu116/repo.html \
  e3nn==0.3.3 \
  wandb==0.12.0 \
  pynvml==11.0.0 \
  git+https://github.com/NVIDIA/dllogger#egg=dllogger \
  decorator==5.1.0 \
  hydra-core==1.3.2 \
  pyrsistent==0.19.3 \
  /app/RFdiffusion/env/SE3Transformer \
  && pip install --no-cache-dir /app/RFdiffusion --no-deps

WORKDIR /app/RFdiffusion
ENV DGLBACKEND="pytorch"
ENTRYPOINT ["python3.9", "scripts/run_inference.py"]