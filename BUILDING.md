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
- NumPy 1.23.x (⚠️ **IMPORTANT**: TensorFlow 2.12.0 is incompatible with NumPy 2.0+)
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

Before starting the installation, you may want to customize these settings based on your environment:

```bash
# Base paths and settings
BASE_PROJECT_DIR="${HOME}/essentia"  # Base directory for local installations, sources, venv
GLOBAL_INSTALL_PREFIX="/usr/local"   # Directory for FFmpeg, TF C API, Essentia C++

# FFmpeg settings
FFMPEG_VERSION="n7.1.1"              # FFmpeg version
FFMPEG_SRC_DIR_NAME="ffmpeg_src_${FFMPEG_VERSION}"  # Relative to BASE_PROJECT_DIR

# Python settings
PYTHON_VERSION="3.10"                # Target Python version
VENV_NAME="essentia_env_py${PYTHON_VERSION}"
VENV_DIR_NAME="${VENV_NAME}"         # Relative to BASE_PROJECT_DIR
PYTHON_EXECUTABLE_CMD="python${PYTHON_VERSION}"  # Command to invoke python

# TensorFlow C API settings
TF_C_API_VERSION="2.12.0"
TF_C_API_URL="https://storage.googleapis.com/tensorflow/libtensorflow/libtensorflow-cpu-linux-x86_64-${TF_C_API_VERSION}.tar.gz"

# Essentia settings
ESSENTIA_GIT_URL="https://github.com/wo80/essentia.git"  # Special fork with cmake support
ESSENTIA_GIT_BRANCH="cmake"
ESSENTIA_SRC_DIR_NAME="essentia_src_${ESSENTIA_GIT_BRANCH}"  # Relative to BASE_PROJECT_DIR
```

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
mkdir -p "${BASE_PROJECT_DIR}"
cd "${BASE_PROJECT_DIR}"

# Clone FFmpeg repository
echo "Cloning FFmpeg repository (version ${FFMPEG_VERSION})..."
if [ -d "${FFMPEG_SRC_DIR_NAME}" ]; then
    echo "FFmpeg source directory already exists. Removing it for a fresh clone."
    rm -rf "${FFMPEG_SRC_DIR_NAME}"
fi

git clone --depth 1 --branch "${FFMPEG_VERSION}" https://git.ffmpeg.org/ffmpeg.git "${FFMPEG_SRC_DIR_NAME}"

# Configure FFmpeg build
cd "${FFMPEG_SRC_DIR_NAME}"
./configure \
    --prefix="${GLOBAL_INSTALL_PREFIX}" \
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
if ! command -v "${GLOBAL_INSTALL_PREFIX}/bin/ffmpeg"; then
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
mkdir -p "${BASE_PROJECT_DIR}"

# Define the virtual environment path
VENV_FULL_PATH="${BASE_PROJECT_DIR}/${VENV_DIR_NAME}"

# Check if Python executable is available
if ! command -v "${PYTHON_EXECUTABLE_CMD}"; then
   echo "${PYTHON_EXECUTABLE_CMD} could not be found. (Is it installed?)"
   exit 1
fi

# Create Python virtual environment
if [ ! -d "${VENV_FULL_PATH}" ]; then
    echo "Creating Python virtual environment using '${PYTHON_EXECUTABLE_CMD}'..."
    "${PYTHON_EXECUTABLE_CMD}" -m venv "${VENV_FULL_PATH}"
else
    echo "Using existing virtual environment at '${VENV_FULL_PATH}'."
fi

# Upgrade pip in the virtual environment
"${VENV_FULL_PATH}/bin/python" -m pip install --upgrade pip

echo "Python virtual environment setup finished."
echo "To activate in your current terminal: source '${VENV_FULL_PATH}/bin/activate'"
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

# Define TensorFlow C API details
TF_C_API_FILENAME="libtensorflow-cpu-linux-x86_64-${TF_C_API_VERSION}.tar.gz"

# Download TensorFlow C API
cd "${TMP_DOWNLOAD_DIR}"
wget -nv -O "${TF_C_API_FILENAME}" "${TF_C_API_URL}"

# Extract TensorFlow C API to the global install prefix
sudo tar --no-same-owner -C "${GLOBAL_INSTALL_PREFIX}" -xzf "${TF_C_API_FILENAME}"

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
mkdir -p "${BASE_PROJECT_DIR}"

# Define Essentia source directory
ESSENTIA_SRC_FULL_PATH="${BASE_PROJECT_DIR}/${ESSENTIA_SRC_DIR_NAME}"

# Clone Essentia repository
cd "${BASE_PROJECT_DIR}"
if [ -d "${ESSENTIA_SRC_FULL_PATH}" ]; then
    echo "Essentia source directory already exists. Removing it for a fresh clone."
    rm -rf "${ESSENTIA_SRC_FULL_PATH}"
fi

# Clone the specific fork that supports cmake and FFmpeg 7.7.1
git clone --depth 1 --branch "${ESSENTIA_GIT_BRANCH}" "${ESSENTIA_GIT_URL}" "${ESSENTIA_SRC_FULL_PATH}"

# Initialize and update git submodules
cd "${ESSENTIA_SRC_FULL_PATH}"
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
# Define the virtual environment path
VENV_FULL_PATH="${BASE_PROJECT_DIR}/${VENV_DIR_NAME}"

# Create a requirements.txt file
cat > /tmp/essentia_requirements.txt << EOF
numpy>=1.23.5,<1.24
pyyaml>=5.4,<7.0
six>=1.15,<2.0
tensorflow==2.12.0 # Should match TF_C_API_VERSION from settings
av>=10.0,<11.0
EOF

# Activate the virtual environment and install Python dependencies
source "${VENV_FULL_PATH}/bin/activate"

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
# Define paths
VENV_FULL_PATH="${BASE_PROJECT_DIR}/${VENV_DIR_NAME}"
ESSENTIA_SRC_FULL_PATH="${BASE_PROJECT_DIR}/${ESSENTIA_SRC_DIR_NAME}"
ACTUAL_ESSENTIA_BUILD_DIR_NAME="build"

# Activate the virtual environment
source "${VENV_FULL_PATH}/bin/activate"

# Change to Essentia source directory
cd "${ESSENTIA_SRC_FULL_PATH}"

# Remove existing build directory if it exists
if [ -d "${ACTUAL_ESSENTIA_BUILD_DIR_NAME}" ]; then
    rm -rf "${ACTUAL_ESSENTIA_BUILD_DIR_NAME}"
fi

# Configure Essentia build using CMake
cmake -B "${ACTUAL_ESSENTIA_BUILD_DIR_NAME}" \
      -DCMAKE_BUILD_TYPE=Release \
      -DCMAKE_INSTALL_PREFIX="${GLOBAL_INSTALL_PREFIX}" \
      -DBUILD_SHARED_LIBS=ON \
      -DBUILD_PYTHON_BINDINGS=ON \
      -DUSE_TENSORFLOW=ON \
      -DBUILD_EXAMPLES=ON

# Compile Essentia (C++ and Python bindings)
cmake --build "${ACTUAL_ESSENTIA_BUILD_DIR_NAME}" --parallel "$(nproc)"

# Install Essentia
sudo cmake --install "${ACTUAL_ESSENTIA_BUILD_DIR_NAME}" --prefix "${GLOBAL_INSTALL_PREFIX}"

# Update system linker cache
sudo ldconfig

# Install Essentia from the .whl file
cd "${ESSENTIA_SRC_FULL_PATH}/${ACTUAL_ESSENTIA_BUILD_DIR_NAME}/wheel"
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
# Define the virtual environment path
VENV_FULL_PATH="${BASE_PROJECT_DIR}/${VENV_DIR_NAME}"

# Activate the virtual environment
source "${VENV_FULL_PATH}/bin/activate"

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
