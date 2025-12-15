# psdk_vendor

A ROS (Robot Operating System) package that provides vendor libraries and platform-specific implementations for DJI's Payload SDK (PSDK). This package serves as a foundational layer for interfacing with DJI drone platforms.

## Overview

The `psdk_vendor` package wraps DJI's Payload SDK core libraries and provides platform-specific Hardware Abstraction Layer (HAL) implementations for embedded platforms. It builds static libraries that can be linked against by higher-level ROS packages to enable communication and control of DJI drones.

## Package Structure

```
psdk_vendor/
├── psdk_core/              # DJI Payload SDK core libraries
│   └── psdk_lib/           # Pre-compiled PSDK libraries and headers
│       ├── include/        # PSDK header files
│       └── lib/            # Platform-specific libraries
│           ├── aarch64-linux-gnu-gcc/    # ARM64 libraries
│           └── x86_64-linux-gnu-gcc/     # x86_64 libraries
├── psdk_platform/          # Platform-specific implementations
│   ├── C/                  # C implementation
│   │   ├── module_sample/  # Sample modules and utilities
│   │   └── platform/       # Platform HAL implementations
│   │       ├── common/     # Common platform code
│   │       ├── raspberry_pi/   # Raspberry Pi specific code
│   │       └── nvidia_jetson/  # NVIDIA Jetson specific code
│   └── C++/                # C++ implementation (Jetson only)
│       ├── module_sample/  # Sample modules and utilities
│       └── platform/       # Platform HAL implementations
├── CMakeLists.txt          # Build configuration
└── package.xml             # ROS package manifest
```

## Supported Platforms

### Primary Platforms
- **NVIDIA Jetson**: Full support (C and C++ libraries)
  - Jetson Nano
  - Jetson Xavier NX
  - Jetson AGX Xavier
  - Jetson Orin series

- **Raspberry Pi**: C library support only
  - Raspberry Pi 3
  - Raspberry Pi 4
  - Raspberry Pi 5

### Architectures
- **ARM64 (aarch64)**: For Jetson and Raspberry Pi platforms
- **x86_64**: For development and testing on desktop systems

## Dependencies

### System Dependencies
The package requires the following system libraries:

- **FFmpeg 4.x**: Video encoding/decoding
  - `libavcodec`
  - `libavformat`
  - `libavutil`
  - `libswscale`
  
- **OPUS**: Audio codec library
  - `libopus`
  
- **libusb-1.0**: USB device communication
  
- **pthread**: POSIX threads
- **Standard math library (m)**

### Build Dependencies
- **CMake** >= 3.0.2
- **catkin**: ROS build system
- **pkg-config**: For finding system libraries
- **GCC/G++**: C/C++ compiler toolchain

## Installation

### 1. Install System Dependencies

#### On Ubuntu/Debian:
```bash
sudo apt-get update
sudo apt-get install -y \
    libavcodec-dev \
    libavformat-dev \
    libavutil-dev \
    libswscale-dev \
    libopus-dev \
    libusb-1.0-0-dev \
    pkg-config
```

#### Verify FFmpeg Version:
```bash
ffmpeg -version
```
**Note**: DJI PSDK requires FFmpeg 4.x. If your system has FFmpeg 5.x or higher, you may need to install FFmpeg 4.x specifically.

### 2. Set Platform Configuration

Before building, you must specify your target platform:

```bash
# For NVIDIA Jetson
export PSDK_PLATFORM=nvidia_jetson

# For Raspberry Pi
export PSDK_PLATFORM=raspberry_pi
```

Alternatively, you can set the platform during the CMake configuration:
```bash
catkin build psdk_vendor -DPSDK_PLATFORM=nvidia_jetson
```

### 3. Build the Package

Navigate to your catkin workspace and build:

```bash
cd ~/ros/tendroids_ws
catkin build psdk_vendor
```

Or build the entire workspace:
```bash
catkin build
```

### 4. Source the Workspace

After building, source the workspace:
```bash
source devel/setup.bash
```

## Build Outputs

The package produces the following static libraries:

### For All Platforms:
- **`libpsdk_vendor_c.a`**: C implementation of PSDK platform layer

### For NVIDIA Jetson Only:
- **`libpsdk_vendor_cpp.a`**: C++ implementation of PSDK platform layer

### Installed Components:
- Static libraries: `devel/lib/libpsdk_vendor_*.a`
- Header files: `devel/include/` (PSDK core headers)

## Usage

### Linking in Other ROS Packages

To use `psdk_vendor` in your ROS package:

#### 1. Add dependency in `package.xml`:
```xml
<depend>psdk_vendor</depend>
```

#### 2. Update `CMakeLists.txt`:
```cmake
find_package(catkin REQUIRED COMPONENTS
    psdk_vendor
)

catkin_package(
    CATKIN_DEPENDS psdk_vendor
)

# Link against the vendor libraries
target_link_libraries(your_node
    ${catkin_LIBRARIES}
)
```

### Including Headers

The PSDK headers are available after building:
```cpp
#include <dji_core.h>
#include <dji_flight_controller.h>
// ... other PSDK headers
```

## Platform-Specific Notes

### Raspberry Pi
- **C++ support is NOT available** on Raspberry Pi
- Only the `psdk_vendor_c` library is built
- Ensure you have sufficient memory for compilation (consider using swap if needed)

### NVIDIA Jetson
- Both C and C++ implementations are available
- Recommended for advanced PSDK features
- Ensure CUDA toolkit is installed for GPU-accelerated operations (if needed by your application)

### CI/CD Environment
- In CI environments, the platform defaults to `raspberry_pi` if `PSDK_PLATFORM` is not explicitly set
- Override this by setting the `PSDK_PLATFORM` environment variable

## Configuration

### SDK Configuration Files

Platform-specific SDK configurations are located in:
- Raspberry Pi: `psdk_platform/C/platform/raspberry_pi/application/dji_sdk_config.h`
- NVIDIA Jetson: `psdk_platform/C++/platform/nvidia_jetson/application/dji_sdk_app_info.h`

Modify these files to configure:
- Application credentials
- Network settings
- Platform-specific parameters

## Troubleshooting

### Common Issues

#### 1. `PSDK_PLATFORM is not set`
**Error**: `PSDK_PLATFORM is not set. Please set it to your target platform`

**Solution**: Set the environment variable:
```bash
export PSDK_PLATFORM=nvidia_jetson  # or raspberry_pi
```

#### 2. FFmpeg Version Mismatch
**Error**: `Unsupported FFMPEG version: X.x — DJI requires 4.x.x`

**Solution**: Install FFmpeg 4.x:
```bash
sudo apt-get install -y ffmpeg=7:4.*
```

#### 3. Missing DJI Core Library
**Error**: `DJI Payload SDK core library not found`

**Solution**: Ensure the `psdk_core/psdk_lib/lib/` directory contains the appropriate library for your architecture.

#### 4. Unsupported Architecture
**Error**: `Unsupported architecture: <arch>`

**Solution**: This package supports only `aarch64` and `x86_64`. Verify your system architecture:
```bash
uname -m
```

## Development

### Directory Organization

- **Module samples**: Sample implementations are in `psdk_platform/[C|C++]/module_sample/`
- **HAL implementations**: Hardware abstraction in `psdk_platform/[C|C++]/platform/<platform>/hal/`
- **Application code**: Platform entry points in `psdk_platform/[C|C++]/platform/<platform>/application/`

### Adding Custom Modules

1. Place source files in the appropriate `module_sample/` directory
2. HAL implementations go in `platform/<platform>/hal/`
3. Rebuild the package with `catkin build psdk_vendor`

## Version Information

- **Package Version**: 0.0.0
- **ROS Package Format**: 2
- **CMake Minimum Version**: 3.0.2

## License

TODO - Please update the license information in `package.xml`

## Maintainer

- **essam** - essam@todo.todo

## Contributing

When contributing to this package:
1. Ensure code compiles on both supported platforms (if applicable)
2. Test with the target hardware platform
3. Update documentation for any new features or changes
4. Follow ROS best practices and coding standards

## Additional Resources

- [DJI Payload SDK Documentation](https://developer.dji.com/payload-sdk/)
- [ROS Wiki](http://wiki.ros.org/)
- [Catkin Build System](https://catkin-tools.readthedocs.io/)

## Support

For issues specific to this package, please contact the maintainer.

For DJI PSDK-related questions, refer to the [DJI Developer Forum](https://forum.dji.com/forum-139-1.html).
