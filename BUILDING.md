# Building Essentia with TensorFlow and FFmpeg Support

## Introduction

This guide explains how to build Essentia with TensorFlow and FFmpeg 7.7.1 support. The instructions below will walk you through the entire process from installing system dependencies to verifying the installation.

> **Important Note**: This guide uses a specific fork of Essentia (https://github.com/wo80/essentia.git) that has been adapted for cmake and supports FFmpeg 7.7.1, not the original Essentia repository. This fork includes several improvements that make it easier to build with modern versions of FFmpeg.

## Prerequisites

- Ubuntu-based Linux distribution
- Sudo privileges
- Git
- Internet connection

## Required Packages and Libraries

### Core Components
- Python 3.10.12
- FFmpeg 7.7.1 (will be compiled from source)
- TensorFlow 2.12.0
- CMake

### Python Packages
- NumPy 1.23.x
- PyYAML 5.4 or later
- Six 1.15 or later
- AV 10.x

### System Libraries
- Build tools: build-essential, yasm, nasm, pkg-config, autoconf, automake
- Audio libraries: libmp3lame-dev, libfdk-aac-dev, libopus-dev, libvorbis-dev, etc.
- Data processing libraries: libeigen3-dev, libfftw3-dev
- Audio format libraries: libavcodec-dev, libavformat-dev, libavutil-dev

## Overview of the Build Process

The build process consists of the following stages:

1. Installing system dependencies
2. Compiling FFmpeg from source
3. Setting up a Python virtual environment
4. Installing TensorFlow C API
5. Preparing Essentia sources
6. Installing Essentia Python dependencies
7. Building and installing Essentia
8. Verifying the installation

## Configuration Settings

Before starting the installation, you should be aware of the following key configuration settings that will be used throughout the process:

### Installation Directories
- **Base Project Directory**: `~/essentia` - This is where all source code, virtual environments, and local installations will be stored
- **Global Install Prefix**: `/usr/local` - This is where FFmpeg, TensorFlow C API, and Essentia C++ libraries will be installed

### FFmpeg Configuration
- **Version**: n7.1.1
- **Source Directory**: `~/essentia/ffmpeg_src_n7.1.1`

### Python Configuration
- **Version**: 3.10
- **Virtual Environment**: `~/essentia/essentia_env_py3.10`

### TensorFlow Configuration
- **Version**: 2.12.0
- **C API URL**: The pre-built TensorFlow C API will be downloaded from Google's storage servers

### Essentia Configuration
- **Repository**: https://github.com/wo80/essentia.git
- **Branch**: cmake
- **Source Directory**: `~/essentia/essentia_src_cmake`

You may need to adjust these settings based on your specific requirements. For example, if you want to install the libraries in a different location, you would need to modify the paths accordingly in the commands in the following sections.

## Detailed Installation Steps

### 1. Installing System Dependencies

This step installs all the required system packages using apt:

```bash
# Update package list
sudo apt-get update -qq

# Install required packages
sudo DEBIAN_FRONTEND=noninteractive apt-get install -y -qq \
    build-essential yasm nasm pkg-config autoconf automake cmake patchelf git libtool wget libyaml-dev \
    libmp3lame-dev libfdk-aac-dev libfaad-dev libopus-dev libvorbis-dev libflac-dev \
    libwavpack-dev libtwolame-dev libgsm1-dev libsndfile1-dev libsamplerate0-dev \
    libchromaprint-dev libeigen3-dev libfftw3-dev libavcodec-dev libavformat-dev \
    libavutil-dev libswresample-dev libtag1-dev \
    python3.10 python3.10-dev python3.10-venv \
    python3-numpy python3-yaml python3-six
```

**Explanation:**
- Updates the list of available packages
- Installs all necessary system dependencies:
  - Build tools (build-essential, yasm, nasm, etc.)
  - Audio libraries for various codecs
  - Python 3.10 and development tools
  - Basic Python packages

### 2. Compiling FFmpeg from Source

This step clones, configures, compiles, and installs FFmpeg 7.7.1:

```bash
# Create directories
mkdir -p ~/essentia
cd ~/essentia

# Clone FFmpeg repository
echo "Cloning FFmpeg repository (version n7.1.1)..."
if [ -d "ffmpeg_src_n7.1.1" ]; then
    echo "FFmpeg source directory already exists. Removing it for a fresh clone."
    rm -rf ffmpeg_src_n7.1.1
fi

git clone --depth 1 --branch n7.1.1 https://git.ffmpeg.org/ffmpeg.git ffmpeg_src_n7.1.1

# Configure FFmpeg build
cd ffmpeg_src_n7.1.1
./configure \
    --prefix=/usr/local \
    --enable-gpl --enable-nonfree --enable-shared --disable-static --disable-debug \
    --enable-libmp3lame --enable-libfdk-aac --enable-libopus \
    --enable-libvorbis --enable-libtwolame --enable-libgsm \
    --enable-chromaprint

# Compile FFmpeg
make -j"$(nproc)"

# Install FFmpeg
sudo make install

# Update system linker cache
sudo ldconfig

# Verify FFmpeg installation
if ! command -v /usr/local/bin/ffmpeg; then
    if ! command -v ffmpeg; then
        echo "FFmpeg command not found after installation."
        exit 1
    fi
fi

ffmpeg -version
```

**Explanation:**
- Clones FFmpeg repository (version 7.7.1)
- Configures FFmpeg with necessary codecs and libraries
- Compiles FFmpeg using all available CPU cores
- Installs FFmpeg to /usr/local
- Updates the system linker cache
- Verifies the installation

### 3. Setting Up Python Virtual Environment

This step creates a Python 3.10 virtual environment:

```bash
# Ensure the base directory exists
mkdir -p ~/essentia

# Check if Python executable is available
if ! command -v python3.10; then
   echo "python3.10 could not be found. (Is it installed?)"
   exit 1
fi

# Create Python virtual environment
if [ ! -d "~/essentia/essentia_env_py3.10" ]; then
    echo "Creating Python virtual environment using 'python3.10'..."
    python3.10 -m venv ~/essentia/essentia_env_py3.10
else
    echo "Using existing virtual environment at '~/essentia/essentia_env_py3.10'."
fi

# Upgrade pip in the virtual environment
~/essentia/essentia_env_py3.10/bin/python -m pip install --upgrade pip

echo "Python virtual environment setup finished."
echo "To activate in your current terminal: source '~/essentia/essentia_env_py3.10/bin/activate'"
```

**Explanation:**
- Creates a Python 3.10 virtual environment at `~/essentia/essentia_env_py3.10`
- Upgrades pip in this environment
- Provides instructions on how to activate the virtual environment

### 4. Installing TensorFlow C API

This step downloads and installs the TensorFlow C API:

```bash
# Create temporary download directory
TMP_DOWNLOAD_DIR=$(mktemp -d -p "/tmp" -t tf_c_api_dl_XXXXXX)

# Download TensorFlow C API
cd "${TMP_DOWNLOAD_DIR}"
wget -nv -O "libtensorflow-cpu-linux-x86_64-2.12.0.tar.gz" "https://storage.googleapis.com/tensorflow/libtensorflow/libtensorflow-cpu-linux-x86_64-2.12.0.tar.gz"

# Extract TensorFlow C API to the global install prefix
sudo tar --no-same-owner -C /usr/local -xzf "libtensorflow-cpu-linux-x86_64-2.12.0.tar.gz"

# Update linker cache
sudo ldconfig

# Clean up
rm -rf "${TMP_DOWNLOAD_DIR}"
```

**Explanation:**
- Downloads the TensorFlow C API (version 2.12.0)
- Extracts it to /usr/local
- Updates the system linker cache
- Cleans up temporary files

### 5. Preparing Essentia Sources

This step clones the Essentia repository from the fork with cmake support:

```bash
# Ensure the base directory exists
mkdir -p ~/essentia

# Clone Essentia repository
cd ~/essentia
if [ -d "essentia_src_cmake" ]; then
    echo "Essentia source directory already exists. Removing it for a fresh clone."
    rm -rf essentia_src_cmake
fi

# Clone the specific fork that supports cmake and FFmpeg 7.7.1
git clone --depth 1 --branch cmake https://github.com/wo80/essentia.git essentia_src_cmake

# Initialize and update git submodules
cd essentia_src_cmake
git submodule update --init --recursive --jobs "$(nproc)"
```

**Explanation:**
- Clones the Essentia repository from https://github.com/wo80/essentia.git (cmake branch)
- This is a special fork that has been adapted for cmake with support for FFmpeg 7.7.1
- The source code will be available at `~/essentia/essentia_src_cmake`
- Initializes and updates git submodules recursively

### 6. Installing Essentia Python Dependencies

This step installs the required Python packages in the virtual environment:

```bash
# Create a requirements.txt file
cat > /tmp/essentia_requirements.txt << EOF
numpy>=1.23.5,<1.24
pyyaml>=5.4,<7.0
six>=1.15,<2.0
tensorflow==2.12.0
av>=10.0,<11.0
EOF

# Activate the virtual environment and install Python dependencies
source ~/essentia/essentia_env_py3.10/bin/activate

# Install packages from requirements.txt
pip install -r /tmp/essentia_requirements.txt

# Deactivate the virtual environment (if you want to continue with other steps)
deactivate
```

**Explanation:**
- Creates a requirements.txt file with the necessary Python packages
- Activates the virtual environment
- Installs the required packages including:
  - numpy 1.23.x (compatible with TensorFlow 2.12.0)
  - pyyaml 5.4 or later
  - tensorflow 2.12.0
  - av 10.x
- Deactivates the virtual environment

### 7. Building and Installing Essentia

This step configures, compiles, and installs Essentia:

```bash
# Activate the virtual environment
source ~/essentia/essentia_env_py3.10/bin/activate

# Change to Essentia source directory
cd ~/essentia/essentia_src_cmake

# Remove existing build directory if it exists
if [ -d "build" ]; then
    rm -rf build
fi

# Configure Essentia build using CMake
cmake -B build \
      -DCMAKE_BUILD_TYPE=Release \
      -DCMAKE_INSTALL_PREFIX=/usr/local \
      -DBUILD_SHARED_LIBS=ON \
      -DBUILD_PYTHON_BINDINGS=ON \
      -DUSE_TENSORFLOW=ON \
      -DBUILD_EXAMPLES=ON

# Compile Essentia (C++ and Python bindings)
cmake --build build --parallel "$(nproc)"

# Install Essentia
sudo cmake --install build --prefix /usr/local

# Update system linker cache
sudo ldconfig

# Install Essentia from the .whl file
cd ~/essentia/essentia_src_cmake/build/wheel
pip install --force-reinstall --no-deps essentia-*.whl

# Deactivate the virtual environment (if you want to continue with other steps)
deactivate
```

**Explanation:**
- Activates the virtual environment
- Configures Essentia build using CMake with TensorFlow support
- Compiles Essentia (C++ and Python bindings)
- Installs Essentia to /usr/local
- Updates the system linker cache
- Installs the Python wheel in the virtual environment

### 8. Verifying Installation

This step verifies that Essentia and TensorFlow are properly installed:

```bash
# Activate the virtual environment
source ~/essentia/essentia_env_py3.10/bin/activate

# Check Essentia import and version
python -c "import essentia; print(f'Essentia version: {essentia.__version__}')"

# Check TensorFlow import and version
python -c "import tensorflow as tf; print(f'TensorFlow version: {tf.__version__}')"

# Check TensorFlow module location
python -c "import tensorflow as tf; print(f'TensorFlow __file__: {tf.__file__}')"

# Verify TensorflowPredictEffnetDiscogs import
python -c "from essentia.standard import TensorflowPredictEffnetDiscogs; print('✅ TensorflowPredictEffnetDiscogs import successful')"

# Deactivate the virtual environment
deactivate
```

**Explanation:**
- Activates the virtual environment
- Imports Essentia and prints its version
- Imports TensorFlow and prints its version
- Verifies TensorflowPredictEffnetDiscogs functionality
- Deactivates the virtual environment

## Using Essentia after Installation

After successful installation, you can use Essentia in your Python projects by activating the virtual environment:

```bash
source ~/essentia/essentia_env_py3.10/bin/activate
```

Then, in your Python code:

```python
import essentia
import essentia.standard
import tensorflow as tf

print(essentia.__version__)
print(tf.__version__)
```

## Important Notes

### TensorFlow and NumPy Compatibility Issues

TensorFlow 2.12.0 is incompatible with NumPy 2.0+. This is a critical compatibility issue that can cause confusing errors if not addressed properly. When installing packages that depend on NumPy, they might automatically upgrade NumPy to an incompatible version, causing TensorFlow to fail.

To avoid this issue:
- Always explicitly specify the NumPy version when installing packages: `pip install numpy>=1.23.5,<1.24`
- When installing other packages, use the `--no-deps` flag if they might pull in a newer NumPy version, then manually install the dependencies with compatible versions
- If you encounter errors after installing new packages, check your NumPy version with `pip show numpy` and downgrade if necessary: `pip install numpy>=1.23.5,<1.24 --force-reinstall`

Common error messages that indicate NumPy compatibility issues:
- `ImportError: cannot import name '_FlattenInputsProcessor' from 'tensorflow.python.framework.ops'`
- `AttributeError: module 'numpy' has no attribute 'float'`
- `TypeError: Descriptors cannot not be created directly`

### GPU Support for TensorFlow

The instructions in this guide use the pre-built TensorFlow C API for CPU. If you need GPU support for better performance, you'll need to compile TensorFlow from source instead of using the pre-built binaries.

To build TensorFlow with GPU support:
1. Install CUDA and cuDNN according to the TensorFlow documentation for version 2.12.0
2. Clone the TensorFlow repository: `git clone https://github.com/tensorflow/tensorflow.git`
3. Checkout the appropriate version: `cd tensorflow && git checkout v2.12.0`
4. Configure the build with GPU support: `./configure`
5. Build TensorFlow: `bazel build --config=opt --config=cuda //tensorflow/tools/pip_package:build_pip_package`
6. Create the pip package: `./bazel-bin/tensorflow/tools/pip_package/build_pip_package /tmp/tensorflow_pkg`
7. Install the pip package: `pip install /tmp/tensorflow_pkg/tensorflow-*.whl`

For detailed instructions on building TensorFlow with GPU support, refer to the [TensorFlow documentation](https://www.tensorflow.org/install/source).

## Troubleshooting

### Common Issues

1. **Build failures due to missing dependencies**
   - Make sure all system dependencies are installed by running the system dependencies installation commands
   - Check if any specific error messages point to missing libraries

2. **FFmpeg compilation errors**
   - Ensure you have enough disk space and memory for the compilation
   - Check if the FFmpeg version is compatible with your system

3. **Python virtual environment issues**
   - Make sure Python 3.10 is installed on your system
   - Verify the virtual environment was created correctly

4. **TensorFlow C API download failures**
   - Check your internet connection
   - Verify the URL is correct

5. **Essentia build failures**
   - Check the CMake output for any error messages
   - Ensure all dependencies are properly installed

6. **NumPy version conflicts**
   - Remember that TensorFlow 2.12.0 is incompatible with NumPy 2.0+
   - If you encounter errors, check your NumPy version with `pip show numpy`

## Additional Resources

- [Essentia Official Documentation](https://essentia.upf.edu/documentation.html)
- [TensorFlow Documentation](https://www.tensorflow.org/api_docs)
- [FFmpeg Documentation](https://ffmpeg.org/documentation.html)
