#  Custom LFS-Based Operating System

![Linux](https://img.shields.io/badge/Linux-FCC624?style=for-the-badge&logo=linux&logoColor=black) ![Bash](https://img.shields.io/badge/GNU%20Bash-4EAA25?style=for-the-badge&logo=GNU%20Bash&logoColor=white) ![C](https://img.shields.io/badge/C-00599C?style=for-the-badge&logo=c&logoColor=white) ![GCC](https://img.shields.io/badge/GCC-14.2.0-blue?style=for-the-badge) ![Kernel](https://img.shields.io/badge/Kernel-6.10.5-orange?style=for-the-badge)

##  Project Overview

This repository documents the architecture, compilation, and troubleshooting of a custom, heavily optimized Linux operating system built entirely from source code. Following the **Linux From Scratch (LFS)** methodology, this system was assembled 100% manually—from compiling the initial cross-toolchain to building the temporary system, chrooting, and bootstrapping the final target environment.

This project serves as a practical demonstration of low-level systems programming, cross-compilation mechanics, kernel configuration, and bare-metal troubleshooting.

##  System Specifications

| Component | Specification | Description |
| :---- | :---- | :---- |
| **Base Methodology** | LFS 12.2 | Pure source-based build without automated tooling. |
| **Architecture** | `x86_64` | 64-bit target architecture. |
| **Kernel** | Linux 6.10.5 | Custom-compiled, heavily tailored configuration. |
| **Init System** | SysVinit | Classic, deterministic initialization process. |
| **Compiler / Toolchain** | GCC 14.2.0 | C/C++ compiler. |
| **C Library** | Glibc 2.40 | GNU C Library. |
| **Binary Tools** | Binutils 2.43.1 | Linker, assembler, and object manipulation tools. |
| **Primary Shell** | GNU Bash 5.2 | Standard Bourne Again Shell. |

##  Key Architectural Decisions

### 1\. Monolithic Kernel Design (No `initramfs`)

To minimize boot latency and reduce system complexity, I opted against using an `initramfs` (Initial RAM Filesystem). Instead, I configured the kernel as a true monolith by hardcoding critical storage and filesystem drivers directly into the kernel binary (`=y` instead of `=m`).

* Key flags: `CONFIG_SATA_AHCI=y`, `CONFIG_EXT4_FS=y`, `CONFIG_DEVTMPFS=y`.  
* This allows the kernel to mount the root partition (`/dev/sda1`) and populate `/dev` autonomously immediately upon loading, entirely bypassing the need for an intermediate RAM disk.

### 2\. Strict Minimalism (Zero BLFS)

The system is maintained in an absolute base state. I deliberately avoided installing supplementary packages, high-level package managers, or network abstraction layers from Beyond Linux From Scratch (BLFS). This enforces a highly constrained environment, minimizing the attack surface and dependency graph.

### 3\. Custom GRUB Configuration

Rather than relying on `grub-mkconfig`, the `grub.cfg` was written manually. The kernel is instructed to boot with the `ro` (read-only) parameter. This forces the system to perform a safe filesystem integrity check before safely remounting the root partition as `rw` during the init sequence.

##  Engineering Challenges & Troubleshooting

Building a system from scratch inevitably exposes raw system interactions. Below are key low-level issues encountered and resolved during the system bring-up phase.

### Issue 1: SysVinit Virtual Filesystem Mount Failures

* **Symptom:** During the initial boot sequence, the `S00mountvirtfs` script threw errors, failing to locate mounting targets for `/dev/shm` and `cgroup2`.  
* **Resolution:** Intercepted the boot process at the GRUB menu by appending `init=/bin/bash` to the kernel command line. From this emergency root shell, I manually remounted the root filesystem as read-write (`mount -n -o remount,rw /`) and updated `/etc/fstab` to explicitly define the `tmpfs` and `cgroup2` mount points, ensuring the init scripts had the necessary hierarchy.

### Issue 2: Boot-Blocking e2fsck Superblock Time Skew

* **Symptom:** The boot process halted entirely with an `e2fsck` abort. The root cause was a time skew between the hardware RTC (system time) and the creation timestamp of the ext4 superblock.  
* **Resolution:** Booted back into the emergency shell. Modified the `/etc/fstab` configuration, specifically altering the 6th field (`fsck` pass number) for `/dev/sda1` from `1` to `0`. This bypassed the boot-time disk check, allowing the system to boot successfully so the system clock could be synchronized properly using `hwclock`.

### Issue 3: Predictable Network Interface Naming Mismatch

* **Symptom:** The legacy LFS network script was configured to target `eth0`, but modern `udev` rules automatically renamed the interface to `enp0s3` based on predictable network interface naming schemes.  
* **Resolution:**  
  1. Ran `ip link` to identify the correct MAC-assigned interface name (`enp0s3`).  
  2. Renamed the configuration file: `mv /etc/sysconfig/ifconfig.eth0 /etc/sysconfig/ifconfig.enp0s3`.  
  3. Utilized `sed` to dynamically update the internal variables within the configuration file to match the new interface designation.

##  "Hacker" Feature: Binary Injection via Raw Bash TCP Sockets

To properly showcase the finished system, I wanted to run `fastfetch`. However, adhering to the strict "No BLFS" rule meant standard network utilities like `curl` or `wget` were unavailable.

To overcome this constraint, I utilized native **Bash TCP sockets** (`/dev/tcp`) to perform a raw HTTP GET request and stream a pre-compiled binary directly into the LFS file system.

**The Execution:**

1. Hosted the statically compiled `fastfetch` binary via a Python HTTP server on the host machine (`10.0.2.2`).  
2. Established a bi-directional TCP socket directly in Bash on the LFS machine.  
3. Constructed and transmitted a raw HTTP header, then used `sed` to strip the HTTP response headers out of the stream, saving only the pure binary payload.

\# 1\. Open read/write file descriptor 3 to the host's TCP socket

exec 3\<\>/dev/tcp/10.0.2.2/8000

\# 2\. Send the raw HTTP GET request

printf "GET /fastfetch HTTP/1.0\\r\\nHost: 10.0.2.2\\r\\nConnection: close\\r\\n\\r\\n" \>&3

\# 3\. Read response, strip HTTP headers using sed, and output the raw binary

cat \<&3 | sed '1,/^\\r\\{0,1\\}\$/d' \> /usr/bin/fastfetch

\# 4\. Make it executable

chmod \+x /usr/bin/fastfetch