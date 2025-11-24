# Fiasco Microkernel Build Guide for x86

## Overview
Fiasco is the L4Re microkernel that supports running real-time, time-sharing, and virtualization workloads. It supports both x86 32-bit and 64-bit architectures.

## Prerequisites

### Required Tools
- **GCC** >= 11 or **Clang** >= 10
- **GNU Binutils**
- **GNU Make**
- **Perl**
- **Git** (for cloning the repository)

### Install Dependencies (Ubuntu/Debian)
```bash
sudo apt-get update
sudo apt-get install -y build-essential gcc g++ make perl git \
    libncurses-dev bison flex bc libssl-dev libelf-dev
```

### Install Dependencies (Fedora/RHEL)
```bash
sudo dnf groupinstall "Development Tools"
sudo dnf install -y gcc gcc-c++ make perl git ncurses-devel \
    bison flex bc openssl-devel elfutils-libelf-devel
```

## Build Instructions

### Step 1: Clone the Repository
```bash
git clone https://github.com/L4Re/fiasco.git
cd fiasco
```

### Step 2: Create Build Directory
```bash
make BUILDDIR=build-x86
```

This creates a separate build directory to keep source and build artifacts separate.

### Step 3: Configure the Kernel
```bash
cd build-x86
make menuconfig
```

#### Key Configuration Options for x86:

**Architecture Selection:**
- Navigate to "Architecture" or "Target" menu
- Select **x86** (or **x86-64** for 64-bit)

**Common Configuration Items:**
- **CPU type**: Select your target CPU (generic x86, Pentium, Core, etc.)
- **SMP support**: Enable if you need multiprocessor support
- **Virtualization**: Enable VMX/SVM if needed
- **Debugging options**: Adjust as needed (disable for production builds)

**Save and Exit:**
- Use arrow keys to navigate
- Press Enter to select
- Save configuration and exit

### Step 4: Build the Kernel
```bash
make -j$(nproc)
```

The `-j$(nproc)` flag uses all available CPU cores for faster compilation.

### Step 5: Locate the Binary
After successful build:
```bash
ls -lh fiasco
```

The kernel binary `fiasco` will be in your build directory.

## Advanced Configuration

### Manual Configuration (without menuconfig)
You can also configure via:
```bash
make config      # Text-based configuration
make xconfig     # Qt-based GUI (requires Qt)
make gconfig     # GTK-based GUI (requires GTK)
```

### Configuration Presets
Some common presets might be available:
```bash
make help        # View available targets and options
```

### Building for Specific x86 Variants

**For 32-bit x86:**
```bash
make BUILDDIR=build-x86-32
cd build-x86-32
make menuconfig  # Select x86 (32-bit)
make -j$(nproc)
```

**For 64-bit x86:**
```bash
make BUILDDIR=build-x86-64
cd build-x86-64
make menuconfig  # Select x86-64
make -j$(nproc)
```

## Testing the Kernel

### Using QEMU
```bash
# Install QEMU
sudo apt-get install qemu-system-x86

# Run the kernel (basic test)
qemu-system-x86_64 -kernel fiasco -serial stdio -m 512M
```

### Creating a Bootable Image
The Fiasco kernel needs to be integrated with:
- A bootloader (GRUB or multiboot-compliant)
- L4Re user-level components
- Your application

Refer to the [L4Re documentation](https://l4re.org) for complete system setup.

## Troubleshooting

### Build Errors

**Missing Dependencies:**
```bash
# Check for specific error messages about missing tools
# Install any missing packages mentioned
```

**Compiler Version Issues:**
```bash
# Check versions
gcc --version
clang --version

# Ensure GCC >= 11 or Clang >= 10
```

**Configuration Issues:**
```bash
# Reset configuration
make clean
make menuconfig  # Reconfigure from scratch
```

### Common Issues

1. **"No rule to make target"**: Run `make BUILDDIR=...` from the top-level directory first
2. **Perl script errors**: Ensure Perl is installed and in PATH
3. **Linker errors**: May need to adjust memory layout in configuration

## Additional Resources

- **Official Documentation**: https://l4re.org
- **Build Instructions**: https://l4re.org/getting_started/make.html#building-the-l4re-microkernel
- **Contributing Guide**: https://kernkonzept.com/L4Re/contributing/fiasco
- **GitHub Repository**: https://github.com/L4Re/fiasco

## License
The L4Re microkernel core is licensed under MIT, with some components under BSD 2-clause and other licenses. See LICENSE.spdx in the repository.

## Next Steps

After building the kernel:
1. Set up L4Re user-level components
2. Configure your bootloader
3. Develop or integrate your applications
4. Deploy to target hardware or virtual machine

For a complete L4Re system, you'll need the full L4Re distribution from https://l4re.org.
