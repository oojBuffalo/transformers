# https://docs.nvidia.com/deeplearning/frameworks/pytorch-release-notes/rel-24-08.html
FROM nvcr.io/nvidia/pytorch:24.08-py3
LABEL maintainer="Hugging Face"

ARG DEBIAN_FRONTEND=noninteractive

ARG PYTORCH='2.7.1'
# Example: `cu102`, `cu113`, etc.
ARG CUDA='cu126'

RUN apt -y update
RUN apt install -y libaio-dev
RUN python3 -m pip install --no-cache-dir --upgrade pip

ARG REF=main
RUN git clone https://github.com/huggingface/transformers && cd transformers && git checkout $REF

# `datasets` requires pandas, pandas has some modules compiled with numpy=1.x causing errors
RUN python3 -m pip install --no-cache-dir './transformers[deepspeed-testing]' 'pandas<2' 'numpy<2'

# Install latest release PyTorch
# (PyTorch must be installed before pre-compiling any DeepSpeed c++/cuda ops.)
# (https://www.deepspeed.ai/tutorials/advanced-install/#pre-install-deepspeed-ops)
RUN python3 -m pip uninstall -y torch torchvision torchaudio && python3 -m pip install --no-cache-dir -U torch==$PYTORCH torchvision torchaudio --extra-index-url https://download.pytorch.org/whl/$CUDA

RUN python3 -m pip install --no-cache-dir git+https://github.com/huggingface/accelerate@main#egg=accelerate

# Uninstall `transformer-engine` shipped with the base image
RUN python3 -m pip uninstall -y transformer-engine

# Uninstall `torch-tensorrt` shipped with the base image
RUN python3 -m pip uninstall -y torch-tensorrt

# recompile apex
RUN python3 -m pip uninstall -y apex
# RUN git clone https://github.com/NVIDIA/apex
#  `MAX_JOBS=1` disables parallel building to avoid cpu memory OOM when building image on GitHub Action (standard) runners
# TODO: check if there is alternative way to install latest apex
# RUN cd apex && MAX_JOBS=1 python3 -m pip install --global-option="--cpp_ext" --global-option="--cuda_ext" --no-cache -v --disable-pip-version-check .

# Pre-build **latest** DeepSpeed, so it would be ready for testing (otherwise, the 1st deepspeed test will timeout)
RUN python3 -m pip uninstall -y deepspeed
# This has to be run (again) inside the GPU VMs running the tests.
# The installation works here, but some tests fail, if we don't pre-build deepspeed again in the VMs running the tests.
# TODO: Find out why test fail.
RUN DS_BUILD_CPU_ADAM=1 DS_BUILD_FUSED_ADAM=1 python3 -m pip install deepspeed --global-option="build_ext" --global-option="-j8" --no-cache -v --disable-pip-version-check 2>&1

# `kernels` may give different outputs (within 1e-5 range) even with the same model (weights) and the same inputs
RUN python3 -m pip uninstall -y kernels

# When installing in editable mode, `transformers` is not recognized as a package.
# this line must be added in order for python to be aware of transformers.
RUN cd transformers && python3 setup.py develop

# The base image ships with `pydantic==1.8.2` which is not working - i.e. the next command fails
RUN python3 -m pip install -U --no-cache-dir "pydantic>=2.0.0"
RUN python3 -c "from deepspeed.launcher.runner import main"
