# Building Essentia with TensorFlow and FFmpeg Support

This guide explains how to build Essentia with TensorFlow and FFmpeg 7.7.1 support using CMake. The process is automated through a series of scripts, making it easier for users without prior experience in building these modules.

## Prerequisites

- Ubuntu-based Linux distribution
- Sudo privileges
- Git
- Python 3.10
- Internet connection

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

## Quick Start

For those who want to get started immediately, run the following command:

```bash
cd essentia_compiler
sudo chmod +x *.sh utils/*.sh
./00-main.sh
```

This script will execute all the necessary steps to build and install Essentia with TensorFlow and FFmpeg support. The entire process may take 30-45 minutes depending on your system's performance.

## Detailed Steps

If you prefer to understand each step or need to troubleshoot specific issues, you can run the scripts individually as described below.

### 1. Install System Dependencies

This script installs all the required system packages using apt:

```bash
cd essentia_compiler
./01-install_system_deps.sh
```

Key dependencies include:
- Build tools: build-essential, yasm, nasm, pkg-config, autoconf, automake, cmake
- Audio libraries: libmp3lame-dev, libfdk-aac-dev, libopus-dev, libvorbis-dev, etc.
- Python packages: python3.10, python3.10-dev, python3.10-venv

### 2. Compile FFmpeg from Source

This script clones, configures, compiles, and installs FFmpeg 7.7.1:

```bash
./02-compile_ffmpeg.sh
```

The script will:
- Clone FFmpeg repository (version 7.7.1)
- Configure it with necessary codecs and libraries
- Compile FFmpeg using all available CPU cores
- Install it to /usr/local
- Update the system linker cache
- Verify the installation

### 3. Set Up Python Virtual Environment

This script creates a Python 3.10 virtual environment:

```bash
./03-setup_python_venv.sh
```

The virtual environment will be created at `~/essentia/essentia_env_py3.10`. The script will also upgrade pip in this environment.

### 4. Install TensorFlow C API

This script downloads and installs the TensorFlow C API:

```bash
./04-install_tf_c_api.sh
```

The script will:
- Download the TensorFlow C API (version 2.12.0)
- Extract it to /usr/local
- Update the system linker cache

### 5. Prepare Essentia Sources

This script clones the Essentia repository:

```bash
./05-prepare_essentia_sources.sh
```

The script will:
- Clone the Essentia repository from https://github.com/wo80/essentia.git (cmake branch)
- The source code will be available at `~/essentia/essentia_src_cmake`

### 6. Install Essentia Python Dependencies

This script installs the required Python packages in the virtual environment:

```bash
./06-install_essentia_python_deps.sh
```

Key dependencies include:
- numpy 1.23.x
- pyyaml 5.4 or later
- tensorflow 2.12.0
- av 10.x

### 7. Build and Install Essentia

This script configures, compiles, and installs Essentia:

```bash
./07-build_install_essentia.sh
```

The script will:
- Configure Essentia build using CMake with TensorFlow support
- Compile Essentia (C++ and Python bindings)
- Install Essentia to /usr/local
- Install the Python wheel in the virtual environment

### 8. Verify Installation

This script verifies that Essentia and TensorFlow are properly installed:

```bash
./08-verify_installation.sh
```

The script will:
- Import Essentia and print its version
- Import TensorFlow and print its version
- Verify TensorflowPredictEffnetDiscogs functionality

## Configuration Options

The build process can be customized by modifying the `scripts.conf` file in the essentia_compiler directory. Key configuration options include:

- `BASE_PROJECT_DIR`: Base directory for local installations, sources, and virtual environment (default: ~/essentia)
- `GLOBAL_INSTALL_PREFIX`: Directory where FFmpeg, TensorFlow C API, and Essentia C++ will be installed (default: /usr/local)
- `FFMPEG_VERSION`: FFmpeg version to compile (default: n7.1.1)
- `PYTHON_VERSION`: Target Python version (default: 3.10)
- `TF_C_API_VERSION`: TensorFlow C API version (default: 2.12.0)
- `ESSENTIA_GIT_URL`: URL of the Essentia repository (default: https://github.com/wo80/essentia.git)
- `ESSENTIA_GIT_BRANCH`: Branch of the Essentia repository to use (default: cmake)

## Troubleshooting

### Common Issues

1. **Build failures due to missing dependencies**
   - Make sure all system dependencies are installed by running `./01-install_system_deps.sh`
   - Check the log files in the `~/essentia/logs` directory for detailed error messages

2. **FFmpeg compilation errors**
   - Check the FFmpeg configuration log at `~/essentia/logs/02-ffmpeg_configure.log`
   - Ensure you have enough disk space and memory for the compilation

3. **Python virtual environment issues**
   - Make sure Python 3.10 is installed on your system
   - Check if the virtual environment was created correctly at `~/essentia/essentia_env_py3.10`

4. **TensorFlow C API download failures**
   - Check your internet connection
   - Verify the URL in the configuration file is correct

5. **Essentia build failures**
   - Check the CMake log at `~/essentia/logs/07-essentia_cmake.log`
   - Check the build log at `~/essentia/logs/07-essentia_build.log`

### Log Files

All logs are stored in the `~/essentia/logs` directory. Key log files include:

- `00-main_installer_YYYYMMDD_HHMMSS.log`: Main log file for the entire installation process
- `02-ffmpeg_configure.log`: FFmpeg configuration log
- `02-ffmpeg_make.log`: FFmpeg compilation log
- `02-ffmpeg_install.log`: FFmpeg installation log
- `07-essentia_cmake.log`: Essentia CMake configuration log
- `07-essentia_build.log`: Essentia build log
- `07-essentia_install.log`: Essentia installation log

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

## Additional Resources

- [Essentia Official Documentation](https://essentia.upf.edu/documentation.html)
- [TensorFlow Documentation](https://www.tensorflow.org/api_docs)
- [FFmpeg Documentation](https://ffmpeg.org/documentation.html)
