FROM nvcr.io/nvidia/cuda:11.6.2-cudnn8-runtime-ubuntu20.04

# Install dependencies
RUN apt-get -q update \
    && DEBIAN_FRONTEND=noninteractive apt-get install --no-install-recommends -y \
        git \
        python3.9 \
        python3-pip \
    && python3.9 -m pip install -q -U --no-cache-dir pip \
    && apt-get autoremove -y \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Install Python dependencies
RUN pip install -q --no-cache-dir \
    torch==2.0.0+cu118 \
    torchvision==0.15.1+cu118 \
    torchaudio==2.0.1 \
    --extra-index-url https://download.pytorch.org/whl/cu118 \
    && pip install -q --no-cache-dir \
    numpy==1.23.5 \
    biopython

# Verify dependency versions
RUN echo "Checking installed package versions..." \
    && python3.9 -c "import torch; print('torch:', torch.__version__)" \
    && python3.9 -c "import numpy; print('numpy:', numpy.__version__)" \
    && python3.9 -c "import Bio; print('biopython:', Bio.__version__)"

# Clone ProteinMPNN source
RUN git clone https://github.com/dauparas/ProteinMPNN.git /app/ProteinMPNN

# Set working directory
WORKDIR /app/ProteinMPNN

# Set environment variables
ENV PYTHONPATH=/app/ProteinMPNN
ENV PROTEIN_MPNN=/app/ProteinMPNN

# Set the entrypoint
ENTRYPOINT ["python3.9", "/app/ProteinMPNN/protein_mpnn_run.py"]