# Running OS/2 Applications on L4Re Microkernel

## Overview

Running OS/2 applications on the Fiasco (L4Re) microkernel requires implementing an **OS/2 personality** - a layer that provides OS/2 API compatibility on top of the microkernel. This is a significant undertaking that involves recreating OS/2's entire application binary interface (ABI) and system services.

## Historical Context

### IBM Workplace OS
IBM attempted this in the early 1990s with Workplace OS, which used a Mach microkernel to host multiple operating system personalities including OS/2, AIX, and Windows NT. OS/2 for PowerPC was a scaled-down version of this vision. However, the project was ultimately cancelled after significant investment.

### The osFree Project
The osFree project is an open-source effort to create an OS/2-compatible operating system based on the L4 microkernel, aiming for binary compatibility with OS/2 applications. This is the closest existing example of what you're attempting.

## Architecture Requirements

### 1. Microkernel Foundation
You've already completed this step by building the Fiasco microkernel. The microkernel provides:
- Memory management primitives
- Thread scheduling
- Inter-process communication (IPC)
- Capability-based security

### 2. OS/2 Personality Layer
The OS/2 personality sits between the microkernel and applications, implementing:

#### Core OS/2 APIs (32-bit)
- **DOSCALL1.DLL** - Core system calls
- **VIOCALLS.DLL** - Video I/O
- **KBDCALLS.DLL** - Keyboard input
- **MOUCALLS.DLL** - Mouse input
- **QUECALLS.DLL** - Queue management
- **SESMGR.DLL** - Session manager
- **MSG.DLL** - Message handling

#### Presentation Manager (PM) APIs
- **PMWIN.DLL** - Window management
- **PMGPI.DLL** - Graphics programming interface
- **PMSHAPI.DLL** - Shell API

#### Additional Services
- File system support (FAT, HPFS)
- Device driver model
- Workplace Shell compatibility (optional)

### 3. Personality-Neutral Services
These are shared services that multiple personalities can use, including memory managers, thread schedulers, and device drivers.

## Implementation Steps

### Phase 1: Basic Infrastructure (6-12 months)

#### 1.1 Set Up L4Re Runtime Environment
```bash
# Clone the full L4Re distribution
git clone https://github.com/kernkonzept/l4re-core.git
cd l4re-core

# Build the runtime environment
make B=build-x86
cd build-x86
make config
make -j$(nproc)
```

#### 1.2 Study osFree Project
```bash
# Clone osFree for reference
git clone https://github.com/osfree-project/osfree.git
cd osfree

# Study their OS/2 personality implementation
# Key directories:
# - OS2/Server/ - OS/2 server implementation
# - OS2/CPI/ - Common Programming Interface
```

#### 1.3 Create Basic Server Process
Create a user-space server that will implement OS/2 system calls:
- Set up IPC endpoints for OS/2 API calls
- Implement basic process/thread management
- Create capability mappings for OS/2 resources

### Phase 2: Core OS/2 API Implementation (12-24 months)

#### 2.1 Implement DOSCALL1.DLL Functions
Start with the most critical DOS-style system calls:
- `DosOpen`, `DosClose`, `DosRead`, `DosWrite`
- `DosCreateThread`, `DosExitThread`
- `DosAllocMem`, `DosFreeMem`
- `DosSleep`, `DosBeep`

#### 2.2 Process and Thread Management
Implement OS/2's process model:
- Process creation (`DosExecPgm`)
- Thread creation and synchronization
- Semaphores (Event, Mutex, MuxWait)
- Shared memory

#### 2.3 File System Support
Either:
- Port OS/2 IFS (Installable File System) drivers
- Implement FAT filesystem with OS/2 Extended Attributes (EAs)
- Bridge to L4Re's existing filesystem services

### Phase 3: Advanced Features (12-18 months)

#### 3.1 Device Driver Model
OS/2 PowerPC introduced the GRADD (Graphics Adapter Device Driver) model, which could serve as a reference.

Options for device drivers:
- Port existing OS/2 device drivers (complex)
- Use Linux drivers via DDE (Device Driver Environment)
- Create wrapper drivers that translate OS/2 driver calls to L4Re equivalents

#### 3.2 Presentation Manager (Optional)
If you need GUI support:
- Implement PMWIN.DLL window management
- Implement PMGPI.DLL graphics primitives
- Consider using modern graphics stack underneath

### Phase 4: Binary Compatibility (6-12 months)

#### 4.1 Executable Loader
Implement a loader for OS/2 executables:
- Parse NE (New Executable) format for 16-bit apps
- Parse LX (Linear Executable) format for 32-bit apps
- Resolve DLL imports
- Apply fixups and relocations

#### 4.2 ABI Compatibility
Ensure proper:
- Calling conventions (SYSCALL, OPTLINK, _System)
- Exception handling
- Floating-point support
- Stack frame layout

## Practical Approaches

### Approach A: Start with osFree
**Recommended for most developers**

1. Fork the osFree project
2. Update it to work with current L4Re
3. Focus on getting a minimal system running
4. Incrementally add OS/2 API support

**Pros:**
- Existing codebase to learn from
- Community knowledge (though small)
- Some components already implemented

**Cons:**
- Project is slow-moving with limited resources
- May have outdated design decisions
- Incomplete implementation

### Approach B: Use L4Linux as a Base
**Easier but less "pure" approach**

L4Linux is a binary-compatible Linux port that runs on L4Re.

1. Run L4Linux as a personality
2. Use WINE-like approach to translate OS/2 calls to Linux
3. Run OS/2 applications through translation layer

**Pros:**
- Faster initial results
- Leverage existing L4Linux stability
- Can use Linux device drivers

**Cons:**
- Not true OS/2 compatibility
- Performance overhead
- Limited OS/2 feature support

### Approach C: Hypervisor Approach
**Most practical for legacy applications**

1. Use L4Re's virtualization features
2. Run actual OS/2 in a virtual machine
3. Provide paravirtualized drivers

**Pros:**
- Run unmodified OS/2
- Complete compatibility
- Fastest time to working system

**Cons:**
- Requires OS/2 license
- Higher resource overhead
- Not a "native" solution

## QEMU-Only Deployment

Since you're targeting QEMU exclusively rather than physical hardware, this **significantly simplifies** your project:

### Advantages of QEMU-Only Approach

1. **No Real Hardware Drivers Needed**
   - Use paravirtualized devices (virtio)
   - Standard QEMU devices are well-documented
   - No need to support diverse physical hardware

2. **Simplified Device Stack**
   ```
   OS/2 App → OS/2 API → L4Re → QEMU virtio drivers
   ```
   Instead of dealing with hundreds of real hardware drivers

3. **Debugging is Much Easier**
   - QEMU's built-in debugging support
   - GDB integration
   - Snapshots and state inspection
   - Monitor commands for inspection

4. **Testing and Development**
   - Instant boot/reboot cycles
   - Easy to automate testing
   - No risk of hardware damage
   - Can test on any development machine

### Recommended QEMU Configuration

```bash
qemu-system-x86_64 \
  -kernel fiasco \
  -m 512M \
  -smp 2 \
  -serial stdio \
  -display gtk \
  -device virtio-net-pci \
  -device virtio-blk-pci \
  -device virtio-scsi-pci \
  -enable-kvm  # If available
```

### Focus Areas for QEMU Deployment

#### 1. Virtio Device Support (Priority)
Implement L4Re drivers for:
- **virtio-blk** - Block storage
- **virtio-net** - Networking  
- **virtio-console** - Serial/console
- **virtio-gpu** - Graphics (optional)

#### 2. Simplified Hardware Abstraction
You only need to support:
- QEMU's standard PC platform
- VGA or virtio-gpu for graphics
- PS/2 or virtio-input for keyboard/mouse
- Standard x86 timers and interrupts

#### 3. OS/2 Device Driver Translation
Map OS/2 device requests to virtio:
```
OS/2 DASD (disk) driver → virtio-blk
OS/2 network adapter → virtio-net
OS/2 display driver → QEMU VGA/virtio-gpu
```

### Streamlined Development Path

**Phase 1: Boot in QEMU (1-2 months)**
1. Get Fiasco kernel booting in QEMU
2. Load L4Re user-space components
3. Start basic OS/2 personality server

**Phase 2: Console Apps (6-12 months)**
1. Implement DOSCALL1.DLL basics
2. File I/O using virtio-blk
3. Get simple text-based OS/2 apps running

**Phase 3: Virtio Integration (3-6 months)**
1. Full virtio-blk support with OS/2 filesystem
2. virtio-net for networking
3. virtio-console for I/O

**Phase 4: GUI Support (Optional, 12-18 months)**
1. QEMU VGA framebuffer
2. Basic Presentation Manager
3. OS/2 GUI applications

### QEMU Testing Script

```bash
#!/bin/bash
# test-os2-l4re.sh

KERNEL="build-x86/fiasco"
MODULES="sigma0:bootstrap:os2server:l4re"
MEMORY="512M"

qemu-system-x86_64 \
  -kernel "$KERNEL" \
  -m "$MEMORY" \
  -smp 2 \
  -serial mon:stdio \
  -display gtk \
  -device virtio-blk-pci,drive=hd0 \
  -drive file=os2disk.img,if=none,id=hd0,format=raw \
  -device virtio-net-pci,netdev=net0 \
  -netdev user,id=net0,hostfwd=tcp::5555-:22 \
  -monitor unix:qemu-monitor-socket,server,nowait \
  -gdb tcp::1234 \
  -S  # Start paused for debugging
```

### Key Simplifications

| Aspect | Physical Hardware | QEMU Only |
|--------|------------------|-----------|
| Driver count | Hundreds | ~5-10 |
| Driver complexity | Very High | Medium |
| Testing time | Hours | Minutes |
| Hardware bugs | Must handle | Non-existent |
| Debugging | Difficult | Easy |
| Development cost | High | Low |

## Testing Strategy

### Test Applications in Order of Complexity

1. **Hello World** - Command-line text output
2. **File I/O** - Read/write files with extended attributes
3. **Threading** - Multi-threaded console apps
4. **Simple GUI** - Basic PM windows
5. **Complex Applications** - Real OS/2 software

## Essential Resources

### Documentation
- **OS/2 Warp Toolkit** - IBM's official API documentation
- **EDM2** (https://www.edm2.com) - Electronic Developer's Magazine/2
- **osFree Wiki** (https://osfree.org) - Technical documentation
- **IBM Redbooks** - Search for "OS/2 Warp PowerPC Edition"

### Source Code References
- **osFree** - https://github.com/osfree-project/osfree
- **L4Re** - https://github.com/kernkonzept/l4re-core
- **L4Linux** - https://l4linux.org

### OS/2 System Files
You'll need reference implementations from:
- OS/2 Warp 4 (last major release)
- OS/2 Warp 4.52 (eComStation/ArcaOS continuation)

## Realistic Timeline

### QEMU-Only Deployment (Revised)

| Phase | Duration | Complexity |
|-------|----------|------------|
| Boot L4Re in QEMU | 1-2 months | Medium |
| Basic infrastructure | 4-8 months | High |
| Core APIs (console apps) | 8-16 months | Very High |
| Virtio device support | 3-6 months | Medium |
| Advanced features | 8-12 months | High |
| GUI support (optional) | 12-18 months | Very High |
| **Total (minimal console system)** | **16-32 months** | **1-3 developers** |
| **Total (with GUI)** | **28-50 months** | **2-5 developers** |

### Physical Hardware (Original Estimates)

| Phase | Duration | Complexity |
|-------|----------|------------|
| Basic infrastructure | 6-12 months | High |
| Core APIs (console apps) | 12-24 months | Very High |
| Advanced features | 12-18 months | Very High |
| GUI support | 12-18 months | Extremely High |
| **Total (minimal system)** | **2-3 years** | **Team of 3-5 developers** |
| **Total (full system)** | **4-6 years** | **Team of 5-10 developers** |

## Major Challenges

### 1. **API Surface Area**
OS/2 has thousands of API functions. Implementing them all is a massive undertaking.

### 2. **Undocumented Behavior**
Many OS/2 applications rely on undocumented behaviors and implementation details.

### 3. **Device Drivers**
OS/2 device driver architecture is completely different from L4Re's model.

### 4. **16-bit Support**
Many OS/2 apps have 16-bit components that require special handling.

### 5. **Extended Attributes**
OS/2's filesystem metadata system is unique and must be preserved.

## Alternative Suggestions

### If You Just Want to Run OS/2 Apps:

1. **Use ArcaOS** - Modern OS/2 continuation (commercial)
2. **Use VirtualBox/QEMU** - Run OS/2 in virtualization
3. **Use WINE-OS/2** - If running on Linux/Windows

### If You Want a Microkernel Project:

1. **Contribute to osFree** - Help existing project
2. **Build a POSIX personality** - More standard, better documented
3. **Port existing apps** - Recompile OS/2 apps for Linux on L4Re

## Conclusion

Running OS/2 applications on L4Re microkernel is theoretically possible but requires:
- Deep understanding of both OS/2 internals and L4Re architecture
- Significant development time (years, not months)
- Team of experienced systems programmers
- Access to OS/2 documentation and reference systems

**Recommendation:** Start with the osFree project or use virtualization if your goal is to run legacy OS/2 applications. Building a full OS/2 personality from scratch should only be attempted as a long-term research or hobby project with realistic expectations about completion time.
