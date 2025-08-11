# Linux Container Architecture (Kernel Level)

## Overview

Linux containers are not a single technology but rather a combination of kernel features that provide process isolation and resource management. Unlike traditional virtualization, containers share the host kernel while maintaining isolation through kernel namespaces and resource control through cgroups.

## Linux Containers vs Virtual Machines (VMs)

| Feature                | Linux Container                          | Virtual Machine (VM)                |
|------------------------|------------------------------------------|-------------------------------------|
| **Isolation**          | Process-level (kernel namespaces, cgroups) | Hardware-level (hypervisor, full OS)|
| **Kernel**             | Shares host kernel                       | Each VM runs its own kernel         |
| **Resource Overhead**  | Low (no guest OS, lightweight)           | High (full OS per VM, more RAM/CPU) |
| **Startup Time**       | Fast (seconds or less)                   | Slow (minutes, full OS boot)        |
| **Image Size**         | Small (MBs, only app + deps)             | Large (GBs, full OS image)          |
| **Performance**        | Near-native (minimal overhead)           | Slightly less than native           |
| **Security Boundary**  | Good, but shares kernel (attack surface) | Strong, full OS isolation           |
| **Use Cases**          | Microservices, CI/CD, stateless apps     | Legacy apps, strong isolation, different OSes |
| **Flexibility**        | Linux-only (host kernel)                 | Any OS (Linux, Windows, BSD, etc.)  |

### Key Differences

- **Containers** package applications with their dependencies, but share the host's Linux kernel. They use kernel features (namespaces, cgroups) for isolation and resource control. This makes them lightweight, fast to start, and efficient for running many isolated workloads on the same host.
- **VMs** emulate entire hardware machines, including their own kernel and OS. This provides strong isolation and the ability to run different operating systems, but at the cost of higher resource usage and slower startup.

**Summary:**  
Containers are ideal for lightweight, scalable, and portable application deployment, especially when all workloads can share the same kernel. VMs are better when you need strong isolation, to run different operating systems, or to support legacy workloads.


## Core Kernel Technologies

### 1. Namespaces
Namespaces provide isolation by creating separate views of system resources:

- **PID Namespace**: Isolates process IDs (container process sees itself as PID 1)
- **Network Namespace**: Provides separate network stack (interfaces, routing, firewall)
- **Mount Namespace**: Isolates filesystem mount points
- **UTS Namespace**: Isolates hostname and domain name
- **IPC Namespace**: Isolates inter-process communication resources
- **User Namespace**: Maps user and group IDs between container and host
- **Cgroup Namespace**: Virtualizes the view of cgroup hierarchy

### 2. Control Groups (cgroups)
Cgroups provide resource management and limiting:

- **CPU**: Limit CPU usage, set CPU shares and quotas
- **Memory**: Limit memory usage, track memory statistics
- **I/O**: Control disk I/O bandwidth and IOPS
- **Network**: Manage network bandwidth (through tc integration)
- **Devices**: Control access to device files
- **Freezer**: Suspend/resume container processes

### 3. Security Features
Additional kernel security mechanisms:

- **Capabilities**: Fine-grained privilege control
- **Seccomp**: System call filtering
- **SELinux/AppArmor**: Mandatory access control
- **User Namespaces**: Privilege isolation

## Container Runtime Architecture

### Runtime Layers
1. **High-level Runtime** (Docker, containerd, CRI-O)
   - Container lifecycle management
   - Image management
   - API interfaces

2. **Low-level Runtime** (runc, crun)
   - Direct kernel interface
   - OCI runtime specification implementation
   - Process creation and namespace setup

### Container Creation Process
1. Runtime creates new namespaces
2. Sets up cgroup hierarchy and limits
3. Mounts container filesystem (overlay/union filesystem)
4. Configures network interfaces (veth pairs, bridges)
5. Applies security policies
6. Executes container process via system calls

## Network Architecture

### Container Networking
- **veth pairs**: Virtual ethernet interfaces connecting container to host
- **Bridge networks**: Software bridges for container-to-container communication
- **Host networking**: Direct access to host network stack
- **Overlay networks**: Multi-host container communication

### Network Isolation
- Each container gets its own network namespace
- Separate routing tables, firewall rules, and network interfaces
- Traffic routing through iptables/netfilter in kernel

## Storage Architecture

### Filesystem Layers
- **Base Image**: Read-only layers containing application dependencies
- **Container Layer**: Writable layer for runtime changes
- **Union Filesystems**: Overlay2, AUFS for efficient layering
- **Volume Mounts**: Direct host filesystem access

### Copy-on-Write (CoW)
- Shared read-only layers between containers
- Write operations create new copies in writable layer
- Efficient storage utilization

## Key Architectural Principles

1. **Shared Kernel**: All containers share the host kernel
2. **Process Isolation**: Namespaces provide process-level isolation
3. **Resource Control**: cgroups enforce resource limits
4. **Layered Filesystem**: Union filesystems enable efficient image sharing
5. **Network Virtualization**: Software-defined networking for connectivity

## Comparison with Virtual Machines

| Aspect | Containers | Virtual Machines |
|--------|------------|------------------|
| Kernel | Shared host kernel | Separate guest kernel |
| Overhead | Low (process-level) | High (full OS) |
| Startup Time | Seconds | Minutes |
| Resource Usage | Efficient sharing | Dedicated allocation |
| Isolation | Process-level | Hardware-level |

## Security Considerations

### Container Escape Risks
- Kernel vulnerabilities affect all containers
- Privileged containers have elevated risks
- Shared kernel means less isolation than VMs

### Mitigation Strategies
- User namespaces for privilege isolation
- Seccomp profiles to limit system calls
- SELinux/AppArmor for mandatory access control
- Regular kernel updates and security patches
- Container image scanning and vulnerability management

## Performance Characteristics

### Advantages
- Near-native performance (no hypervisor overhead)
- Fast startup times
- Efficient resource utilization
- High container density per host

### Considerations
- Kernel scheduling affects all containers
- I/O contention in shared kernel
- Network overhead through virtualization layers

## Practical Example: Creating a Container with Bash

### Simple Network Isolation

```bash
# Demo: Simple network isolation using network namespaces (netns)
# Why? Network namespaces let us isolate network interfaces, so processes can't see or use each other's networks.
# This is a basic building block for containers!

echo "== 1. Show current network interfaces (before isolation) =="
ip addr

echo
echo "== 2. Create a new isolated network namespace =="
ip netns add test-ns

echo
echo "== 3. Show interfaces inside the new namespace (should be minimal) =="
ip netns exec test-ns ip addr

echo
echo "== 4. Clean up: remove the test namespace =="
ip netns delete test-ns

echo
echo "== Why do we need isolation? =="
echo "Network namespaces keep processes separated, so one container can't see or interfere with another's network."
echo "This is important for security and for running many apps on the same machine safely."






```

### Simple Container Script (Beginner-Friendly)

Here's a minimal script to understand the basic concepts:

```bash
#!/bin/bash

# Simple container demo: show 'ls' in host and inside a minimal container

set -e

CONTAINER_ROOT="/tmp/simple-container"

echo "== Host: ls / =="
ls /
# Example: This lists the root directory on the host system (outside any container or namespace).

echo
echo "== Setting up minimal container root =="
mkdir -p "$CONTAINER_ROOT"/{bin,proc}
# Example: Create minimal directory structure for the container: /bin and /proc inside $CONTAINER_ROOT.

cp /bin/bash "$CONTAINER_ROOT/bin/"
# Example: Copy the bash shell binary into the container's /bin directory.

cp /bin/ls "$CONTAINER_ROOT/bin/"
# Example: Copy the ls command binary into the container's /bin directory.

for lib in $(ldd /bin/ls | grep -o '/[^ ]*'); do
    [ -f "$lib" ] && cp --parents "$lib" "$CHOME/"
done
# Example: Copy all shared library dependencies needed by /bin/ls into the container root, preserving directory structure.

echo
echo "== Inside container: ls / =="
unshare --mount --uts --pid --fork chroot "$CONTAINER_ROOT" /bin/ls /
# If you see an error like:
#   error while loading shared libraries: libtinfo.so.6: cannot open shared object file: No such file or directory
# it means a required shared library for /bin/ls or /bin/bash is missing from the container root.
# To fix this, ensure all dependencies are copied. For example, add:
for lib in $(ldd /bin/bash | grep -o '/[^ ]*'); do [ -f "$lib" ] && cp --parents "$lib" /tmp/container/; done
# This copies all shared libraries needed by /bin/bash as well.
# You may need to repeat for any other binaries you want to use inside the container.

echo
echo "== Simulating a long-running process inside the container =="
unshare --mount --uts --pid --fork chroot "$CONTAINER_ROOT" /bin/bash -c "echo '[container] Sleeping for 10 seconds...'; sleep 10; echo '[container] Done sleeping.'"

# If you see "sleep: command not found" inside the container, it means /bin/sleep (and its dependencies) are missing.
# To fix this, copy /bin/sleep and its libraries into the container root:
cp /bin/sleep "/tmp/container/bin/"
for lib in $(ldd /bin/sleep | grep -o '/[^ ]*'); do
    [ -f "$lib" ] && cp --parents "$lib" "$CONTAINER_ROOT/"
done
# Now, 'sleep' should work inside the container.

# If you see an error like:
#   /bin/busybox: error while loading shared libraries: libresolv.so.2: cannot open shared object file: No such file or directory
# it means the busybox binary depends on libresolv.so.2, which is missing from your container root.
# To fix this, copy the library and its dependencies into the container root:
for lib in $(ldd /bin/busybox | grep -o '/[^ ]*'); do
    [ -f "$lib" ] && cp --parents "$lib" /tmp/container/
done
# This ensures all required shared libraries for busybox are available inside the container.

# Note: If you see "chroot: failed to run command ‘/bin/bash’: No such file or directory",
# it means /bin/bash is missing or not copied correctly into $CONTAINER_ROOT/bin.
# This example just runs /bin/ls inside the container root to avoid the error.
echo
echo "== What is UTS namespace? =="
echo "UTS (UNIX Timesharing System) namespace allows each container to have its own hostname and NIS domain name."
echo "This means processes inside a container can set and see a different hostname than the host or other containers."
echo "It's useful for giving each container its own identity in the network, even if they're on the same machine."

# Example: 
# - 'unshare' creates new namespaces for mount, UTS (hostname), and PID (process IDs).
# - '--fork' runs the command in a new process.
# - 'chroot' changes the root directory to $CONTAINER_ROOT, isolating the filesystem.
# - Inside the container, set the hostname and list the root directory.
# - This simulates a very basic container environment.

echo
echo "== Cleaning up =="
rm -rf "$CONTAINER_ROOT"
# Example: Remove the temporary container root directory and all its contents.

### Quick Test

### What This Simple Version Does

1. **Process Isolation**: Creates new PID namespace (you become PID 1)
2. **Hostname Isolation**: Sets custom hostname visible only in container
3. **Mount Isolation**: Separate mount namespace with proc filesystem
4. **Filesystem Isolation**: Uses chroot to change root directory
5. **Automatic Cleanup**: Removes temporary files when done

### Inside the Simple Container

You'll see:
- Custom prompt `[container] #`
- Different hostname
- Isolated process tree
- Limited commands (bash, ls)

### Why No Cgroups in Simple Mode?

The simple script deliberately omits **cgroups** for these reasons:

**1. Complexity Reduction**
```bash
# Cgroups require multiple steps:
mkdir -p /sys/fs/cgroup/memory/mycontainer     # Create cgroup
echo "100M" > /sys/fs/cgroup/memory/mycontainer/memory.limit_in_bytes
echo $$ > /sys/fs/cgroup/memory/mycontainer/cgroup.procs  # Add process
# vs. simple namespace isolation (just unshare command)
```

**2. Different Purpose**
- **Namespaces** = **Isolation** (what you can see)
- **Cgroups** = **Resource Control** (how much you can use)

**3. Learning Progression**
- Step 1: Understand *isolation* (namespaces)
- Step 2: Add *resource limits* (cgroups)

### When Do You NEED Cgroups?

**Critical for Production:**
```bash
# Without cgroups, container can:
:(){ :|:& };:           # Fork bomb - crash entire host
dd if=/dev/zero of=/tmp/big bs=1M count=10000  # Use all memory
while true; do :; done  # Consume 100% CPU
```

**With cgroups protection:**
```bash
# Memory limit prevents OOM on host
echo "100M" > /sys/fs/cgroup/memory/container/memory.limit_in_bytes

# CPU limit prevents monopolizing CPU
echo "50000" > /sys/fs/cgroup/cpu/container/cpu.cfs_quota_us   # 50% max

# Process limit prevents fork bombs  
echo "100" > /sys/fs/cgroup/pids/container/pids.max
```

### Quick Cgroup Addition

To add basic resource control to the simple script:

```bash
# Add after hostname setup, before chroot:

# Create memory cgroup (100MB limit)
CGROUP_NAME="simple-container-$$"
mkdir -p "/sys/fs/cgroup/memory/$CGROUP_NAME"
echo "100M" > "/sys/fs/cgroup/memory/$CGROUP_NAME/memory.limit_in_bytes"
echo $$ > "/sys/fs/cgroup/memory/$CGROUP_NAME/cgroup.procs"

# Cleanup function should include:
# rmdir "/sys/fs/cgroup/memory/$CGROUP_NAME"
```

### Summary: Simple vs Full Container

| Feature | Simple Mode | Full Mode | Purpose |
|---------|-------------|-----------|---------|
| **Namespaces** | ✅ PID, UTS, Mount | ✅ All types | Process isolation |
| **Cgroups** | ❌ Skipped | ✅ Memory, CPU, I/O | Resource control |
| **Networking** | ❌ Skipped | ✅ veth, bridge | Network isolation |
| **Use Case** | Learning concepts | Production-ready | Different goals |

**Bottom Line:** Simple mode focuses on *understanding isolation*. Real containers need cgroups to prevent resource abuse and ensure system stability.

---

### Full-Featured Container Creation Script

For advanced users, here's a comprehensive script with cgroups and networking:

```bash
#!/bin/bash

# Container creation script using namespaces and cgroups
# Requires root privileges

set -e

CONTAINER_NAME="my-container"
CGROUP_ROOT="/sys/fs/cgroup"
CONTAINER_ROOT="/tmp/container-root"

# Function to clean up on exit
cleanup() {
    echo "Cleaning up..."
    # Remove cgroup
    if [ -d "$CGROUP_ROOT/memory/$CONTAINER_NAME" ]; then
        rmdir "$CGROUP_ROOT/memory/$CONTAINER_NAME" 2>/dev/null || true
    fi
    if [ -d "$CGROUP_ROOT/cpu/$CONTAINER_NAME" ]; then
        rmdir "$CGROUP_ROOT/cpu/$CONTAINER_NAME" 2>/dev/null || true
    fi
    # Cleanup mount points
    umount "$CONTAINER_ROOT/proc" 2>/dev/null || true
    umount "$CONTAINER_ROOT/sys" 2>/dev/null || true
    rm -rf "$CONTAINER_ROOT"
}

trap cleanup EXIT

# Check if running as root
if [ "$EUID" -ne 0 ]; then
    echo "This script must be run as root"
    exit 1
fi

echo "Creating container: $CONTAINER_NAME"

# 1. Create container root filesystem
echo "Setting up container filesystem..."
mkdir -p "$CONTAINER_ROOT"/{bin,lib,lib64,proc,sys,dev,etc,tmp,var,usr}

# Copy essential binaries
cp /bin/bash "$CONTAINER_ROOT/bin/"
cp /bin/ls "$CONTAINER_ROOT/bin/"
cp /bin/ps "$CONTAINER_ROOT/bin/"
cp /bin/cat "$CONTAINER_ROOT/bin/"

# Copy required libraries
ldd /bin/bash | grep -o '/[^ ]*' | while read lib; do
    [ -f "$lib" ] && cp --parents "$lib" "$CONTAINER_ROOT/"
done

# Create basic /etc files
echo "root:x:0:0:root:/root:/bin/bash" > "$CONTAINER_ROOT/etc/passwd"
echo "root:x:0:" > "$CONTAINER_ROOT/etc/group"

# 2. Create cgroup for memory control
echo "Setting up cgroups..."
if [ ! -d "$CGROUP_ROOT/memory/$CONTAINER_NAME" ]; then
    mkdir -p "$CGROUP_ROOT/memory/$CONTAINER_NAME"
    echo "100M" > "$CGROUP_ROOT/memory/$CONTAINER_NAME/memory.limit_in_bytes"
    echo "Created memory cgroup with 100MB limit"
fi

# Create cgroup for CPU control
if [ ! -d "$CGROUP_ROOT/cpu/$CONTAINER_NAME" ]; then
    mkdir -p "$CGROUP_ROOT/cpu/$CONTAINER_NAME"
    echo "50000" > "$CGROUP_ROOT/cpu/$CONTAINER_NAME/cpu.cfs_quota_us"
    echo "100000" > "$CGROUP_ROOT/cpu/$CONTAINER_NAME/cpu.cfs_period_us"
    echo "Created CPU cgroup with 50% CPU limit"
fi

# 3. Create network namespace and veth pair
echo "Setting up network..."
ip netns add "$CONTAINER_NAME" 2>/dev/null || echo "Network namespace already exists"
ip link add "veth-host" type veth peer name "veth-$CONTAINER_NAME"
ip link set "veth-$CONTAINER_NAME" netns "$CONTAINER_NAME"

# Configure host side
ip addr add 10.0.0.1/24 dev veth-host
ip link set veth-host up

# Configure container side
ip netns exec "$CONTAINER_NAME" ip addr add 10.0.0.2/24 dev "veth-$CONTAINER_NAME"
ip netns exec "$CONTAINER_NAME" ip link set "veth-$CONTAINER_NAME" up
ip netns exec "$CONTAINER_NAME" ip link set lo up

# 4. Function to run container
run_container() {
    echo "Starting container..."
    
    # Enter namespaces and run container
    unshare --pid --mount --uts --ipc --fork bash -c "
        # Set hostname
        hostname $CONTAINER_NAME
        
        # Mount proc and sys
        mount -t proc proc $CONTAINER_ROOT/proc
        mount -t sysfs sysfs $CONTAINER_ROOT/sys
        
        # Add process to cgroups
        echo \$\$ > $CGROUP_ROOT/memory/$CONTAINER_NAME/cgroup.procs
        echo \$\$ > $CGROUP_ROOT/cpu/$CONTAINER_NAME/cgroup.procs
        
        # Change root and enter container
        cd $CONTAINER_ROOT
        pivot_root . tmp
        exec chroot . /bin/bash -c '
            export PATH=/bin:/usr/bin
            export PS1=\"[$CONTAINER_NAME] \\w # \"
            echo \"Container started successfully!\"
            echo \"Memory limit: \$(cat /sys/fs/cgroup/memory/memory.limit_in_bytes 2>/dev/null || echo \"N/A\")\"
            echo \"Hostname: \$(hostname)\"
            echo \"Network interfaces:\"
            ip link show 2>/dev/null || echo \"ip command not available\"
            exec /bin/bash
        '
    " &
    
    CONTAINER_PID=$!
    echo "Container PID: $CONTAINER_PID"
    
    # Add to network namespace
    ip netns exec "$CONTAINER_NAME" bash -c "echo $CONTAINER_PID > /var/run/netns/$CONTAINER_NAME" &
    
    wait $CONTAINER_PID
}

# 5. Alternative: Using nsenter for existing namespaces
run_with_nsenter() {
    echo "Alternative method using nsenter..."
    
    # Create all namespaces first
    unshare --pid --mount --uts --ipc --net --fork --mount-proc bash -c "
        hostname $CONTAINER_NAME
        mount --bind $CONTAINER_ROOT $CONTAINER_ROOT
        mount --make-private $CONTAINER_ROOT
        cd $CONTAINER_ROOT
        
        # Setup basic mounts
        mount -t proc proc proc/
        mount -t sysfs sysfs sys/
        
        exec chroot . /bin/bash
    "
}

# Display usage
case "${1:-run}" in
    "run")
        run_container
        ;;
    "nsenter")
        run_with_nsenter
        ;;
    "cleanup")
        cleanup
        exit 0
        ;;
    *)
        echo "Usage: $0 [run|nsenter|cleanup]"
        echo "  run     - Create and run container (default)"
        echo "  nsenter - Alternative method using nsenter"
        echo "  cleanup - Clean up resources"
        exit 1
        ;;
esac
```

### Script Features

This script demonstrates:

1. **Filesystem Isolation**: Creates a minimal root filesystem with essential binaries
2. **Memory Control**: Sets up cgroup with 100MB memory limit
3. **CPU Control**: Limits container to 50% CPU usage
4. **Network Isolation**: Creates network namespace with veth pair
5. **Process Isolation**: Uses PID namespace so container sees itself as PID 1
6. **Mount Isolation**: Separate mount namespace with proc/sys mounts
7. **Hostname Isolation**: UTS namespace for separate hostname

### Usage

```bash
# Make script executable
chmod +x container-script.sh

# Run the container
sudo ./container-script.sh run

# Clean up resources
sudo ./container-script.sh cleanup
```

### Inside the Container

Once running, the container will have:
- Isolated process tree (PID 1)
- Limited memory (100MB)
- CPU throttling (50% max)
- Separate network stack (10.0.0.2/24)
- Custom hostname
- Minimal filesystem environment

### Verification Commands

From inside the container:
```bash
# Check process isolation
ps aux                    # Should only see container processes

# Check memory limit
cat /sys/fs/cgroup/memory/memory.limit_in_bytes

# Check hostname
hostname

# Check network
ip addr show
```

This example provides a foundation for understanding how container runtimes like Docker implement isolation at the kernel level.


