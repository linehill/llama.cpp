
### I. Setup Environment

**Install PoCL**

Install PoCL dependencies (on Ubuntu 24.04):

* [Level Zero](https://github.com/oneapi-src/level-zero),
* [Level Zero runtime](https://github.com/intel/compute-runtime),
* [Intel NPU drivers](https://github.com/intel/linux-npu-driver) 
* and others instructed [here](https://portablecl.org/docs/html/install.html). Then:

```sh
cd $WORKSPACE

git clone https://github.com/pocl/pocl.git
cmake -S pocl -B build-pocl -G Ninja \
  -DCMAKE_INSTALL_RPATH_USE_LINK_PATH=ON \
  -DCMAKE_INSTALL_PREFIX=$WORKSPACE/install-pocl \
  -DBUILD_SHARED_LIBS=ON \
  -DENABLE_ICD=ON \
  -DENABLE_LEVEL0=ON \
  -DENABLE_NPU=ON \
  -DWITH_LLVM_CONFIG=<path-to>/llvm-config

cmake --build build-pocl
cmake --install build-pocl
```

**Check NPU availibility**

```sh
export OCL_ICD_VENDORS=$WORKSPACE/install-pocl/etc/OpenCL/vendors/pocl.icd
clinfo -l
```

The NPU devices shows up in the clinfo output as a device named as
`Intel(R) AI Boost` as shown below, for example:

```
Platform #0: Portable Computing Language
 +-- Device #0: cpu-alderlake-Intel(R) Core(TM) Ultra 9 185H
 +-- Device #1: Intel(R) Arc(TM) Graphics
 `-- Device #2: Intel(R) AI Boost
```

### II. Build llama.cpp

```sh
cd $WORKSPACE
git clone -b opencl-dbk https://github.com/linehill/llama.cpp.git

cmake -S llama.cpp -B build-llama.cpp -G Ninja \
  -DCMAKE_CXX_FLAGS=-I$WORKSPACE/pocl/include \
  -DCMAKE_C_FLAGS=-I$WORKSPACE/pocl/include \
  -DGGML_OPENCL=ON \
  -DGGML_OPENCL_USE_ADRENO_KERNELS=OFF \
  -DGGML_OPENCL_TARGET_VERSION=300 \
  -DGGML_OPENCL_ENABLE_DBKS=ON

cmake --build build-llama.cpp
```

### III. Example Usage

```sh
wget -c https://huggingface.co/ggml-org/gemma-3-1b-it-GGUF/resolve/main/gemma-3-1b-it-f16.gguf

export GGML_OPENCL_PLATFORM=0  # Or index to select 'Portable Computing Language' as platform reported by clinfo -l
export GGML_OPENCL_PLATFORM=2  # Or index to select 'AI Boost' device as reported by clinfo -l
export POCL_LEVEL0_CROSS_CTX_SHARED_MEM=0  # Needed until fix for pocl/pocl#2005
./build-llama.cpp/bin/llama-cli -p "Once upon a time" -ngl 100 -no-cnv -dev GPUOpenCL -m gemma-3-1b-it-f16.gguf
```

Where the <model> is path to a model with fp16 or fp32 parameters.
