# Virtualizing eBPF with Late-Binding

Artifact evaluation guide for OSDI '26.

This repository contains the global instructions for the vBPF artifact. The
artifact itself is split across four sibling repositories:

- `llvm-vbpf`
- `linux-vbpf`
- `vm`
- `ebpf-bootstrap`

## Repository Layout

Place this README repository and the four artifact repositories under the same
parent directory:

```text
vbpf-ae/
|-- ebpf-bootstrap/
|-- linux-vbpf/
|-- llvm-vbpf/
|-- vm/
`-- vbpf-ae-readme/
```

If you are currently inside `vbpf-ae-readme`, set the artifact root with:

```bash
cd ..
export AE_ROOT="$PWD"
```

The artifact checkout used to prepare this guide already follows this layout:
the four component repositories are in `../` relative to `vbpf-ae-readme`.

For a fresh checkout, clone the four component repositories under the same
parent directory:

```bash
mkdir -p ~/vbpf-ae
cd ~/vbpf-ae

git clone https://ipads.se.sjtu.edu.cn:1312/vbpf-osdi-2026/llvm-vbpf.git
git clone https://ipads.se.sjtu.edu.cn:1312/vbpf-osdi-2026/linux-vbpf.git
git clone https://ipads.se.sjtu.edu.cn:1312/vbpf-osdi-2026/vm.git
git clone https://ipads.se.sjtu.edu.cn:1312/vbpf-osdi-2026/ebpf-bootstrap.git

export AE_ROOT="$PWD"
```

## Components

| Directory | Purpose |
| --- | --- |
| `llvm-vbpf` | LLVM, Clang, LLD, vBPF compiler support, and the `VBPFAttribute` Clang plugin. |
| `linux-vbpf` | Linux kernel tree with vBPF support and an evaluation-ready `.config`. |
| `vm` | QEMU/KVM virtual-machine environment for booting and testing the vBPF kernel. |
| `ebpf-bootstrap` | Example eBPF programs, helper tools, and evaluation scripts. |

## Host Requirements

Use a Linux host with KVM support. On Ubuntu/Debian, the following packages cover
the common host-side build and VM requirements:

```bash
sudo apt update
sudo apt install -y \
  git cmake ninja-build make clang lld llvm \
  build-essential pkg-config libelf-dev zlib1g-dev \
  libbfd-dev libcap-dev openssl libssl-dev binutils \
  qemu-system-x86 genisoimage bridge-utils
```

The VM launcher uses `uv` and Python 3.12 or newer. If `uv` is not installed,
install it using the official `uv` installation instructions.

## Evaluation Workflow

Run the artifact in this order:

1. Build vBPF LLVM.
2. Build the vBPF Linux kernel using the vBPF Clang plugin.
3. Prepare and initialize the VM.
4. Point the VM at the vBPF kernel image.
5. Boot the VM with the vBPF kernel.
6. Build and run the example eBPF programs inside the VM.

## 1. Build vBPF LLVM

Build LLVM first. This produces the Clang toolchain used for the kernel build
and the `VBPFAttribute` plugin used to statically check vBPF attributes while
compiling the kernel.

```bash
cd "$AE_ROOT/llvm-vbpf"

cmake -S llvm -B build -G Ninja \
  -DCMAKE_BUILD_TYPE=Release \
  -DLLVM_ENABLE_PROJECTS="clang;lld" \
  -DLLVM_TARGETS_TO_BUILD="X86;BPF" \
  -DCMAKE_INSTALL_PREFIX="$PWD/install"

cmake --build build
cmake --build build --target VBPFAttribute
sudo cmake --install build
```

Use the freshly built toolchain for the remaining host-side steps:

```bash
export LLVM_HOME="$AE_ROOT/llvm-vbpf/build"
export PATH="$LLVM_HOME/bin:$PATH"
export LD_LIBRARY_PATH="$LLVM_HOME/lib:$LD_LIBRARY_PATH"
```

Sanity-check the build:

```bash
clang --version
test -f "$LLVM_HOME/lib/VBPFAttribute.so"
```

The LLVM repository also includes a small standalone analyzer test:

```text
$AE_ROOT/llvm-vbpf/clang/examples/VBPF/tests/test.c
```

This file exercises the custom `vbpf_helper` and `vbpf_safe` attributes and the
static analyzer. Compile it with the same plugin style used by the kernel build:

```bash
cd "$AE_ROOT/llvm-vbpf"

"$LLVM_HOME/bin/clang" -fsyntax-only \
  clang/examples/VBPF/tests/test.c \
  -fplugin="$LLVM_HOME/lib/VBPFAttribute.so" \
  -Xclang -add-plugin -Xclang vbpf_attrs \
  -ferror-limit=0
```

The test intentionally contains both passing and failing helpers. A non-zero
exit status is expected; inspect the emitted diagnostics to see how the analyzer
reports unsafe helpers, global-state writes, and unresolved or unsafe indirect
calls.

## 2. Build the vBPF Linux Kernel

The kernel must be built after LLVM because it uses the `VBPFAttribute` plugin.
The `linux-vbpf` repository already contains a `.config` prepared for the VM
environment.

The provided `.config` enables the vBPF kernel features used by the artifact.
Reviewers normally do not need to change these options.

| Option | Purpose |
| --- | --- |
| `CONFIG_LOCALVERSION="-vbpf"` | Adds a `-vbpf` suffix to the kernel release. |
| `CONFIG_BPF_SYSCALL=y`, `CONFIG_BPF_JIT=y`, `CONFIG_BPF_JIT_ALWAYS_ON=y` | Enables the baseline eBPF syscall and JIT support required by the examples. |
| `CONFIG_BPF_LSM=y` | Enables BPF LSM instrumentation. |
| `CONFIG_BPF_VBPF=y` | Enables the vBPF core, including BPF namespace support and vBPF dispatch paths. |
| `CONFIG_VBPF_SNIFFER=y` | Enables the vBPF sniffer framework for flow, block I/O, and scheduling-related tracking with namespace isolation. |
| `CONFIG_VBPF_BENCH=y` | Enables the lightweight benchmark interface under `/sys/kernel/vbpf/bench/`. |
| `CONFIG_SAMPLES_VBPF=y`, `CONFIG_SAMPLE_VBPF_BLOOMFILTER=y`, `CONFIG_SAMPLE_VBPF_SNIFFER=y` | Builds the in-kernel vBPF sample code. |

The custom core options are defined in `kernel/bpf/Kconfig`; the sample options
are defined in `samples/vbpf/Kconfig`.

For simplicity in this artifact, the Linux kernel build enables the custom
attributes but uses a simplified annotation and analysis surface.

```bash
cd "$AE_ROOT/linux-vbpf"

make LLVM=1 -j"$(nproc)" \
  KCFLAGS="-fplugin=$LLVM_HOME/lib/VBPFAttribute.so -Xclang -add-plugin -Xclang vbpf_attrs -ferror-limit=1000"
```

The kernel image used by the VM is:

```text
$AE_ROOT/linux-vbpf/arch/x86/boot/bzImage
```

## 3. Prepare the VM

Initialize the VM dependencies, Ubuntu root filesystem, and cloud-init seed:

```bash
cd "$AE_ROOT/vm"

uv sync
uv run vm.py --prepare
```

Configure the QEMU bridge helper and create the `virbr0` bridge:

```bash
sudo mkdir -p /etc/qemu
echo "allow virbr0" | sudo tee /etc/qemu/bridge.conf
sudo chmod 0644 /etc/qemu/bridge.conf

sudo ip link add name virbr0 type bridge
sudo ip addr add 192.168.122.1/24 dev virbr0
sudo ip link set virbr0 up
```

If `virbr0` already exists, keep the existing bridge and skip the duplicate
`ip link add` and `ip addr add` commands.

Run the first boot with the Ubuntu cloud image and cloud-init:

```bash
uv run vm.py --install
```

When the login prompt appears, use:

```text
username: ubuntu
password: ubuntu
```

Then shut the VM down cleanly:

```bash
sudo poweroff
```

## 4. Configure the vBPF Kernel for the VM

Edit `$AE_ROOT/vm/config.json` and set `kernel.path` to the absolute path of the
kernel image built in step 2:

```json
{
  "kernel": {
    "path": "/absolute/path/to/vbpf-ae/linux-vbpf/arch/x86/boot/bzImage",
    "bpf_debug_string": "dyndbg=\"file kernel/bpf/bpf_diff.c +p\""
  }
}
```

Do not leave a stale machine-local path in `config.json`; the VM launcher uses
this field directly when booting the development kernel.

You can print the expected path from the host with:

```bash
realpath "$AE_ROOT/linux-vbpf/arch/x86/boot/bzImage"
```

Confirm the VM configuration:

```bash
cd "$AE_ROOT/vm"
uv run vm.py config --show
```

## 5. Boot the vBPF VM

Boot the VM with the vBPF kernel:

```bash
cd "$AE_ROOT/vm"
uv run vm.py
```

For BPF-related kernel debug logging, use:

```bash
uv run vm.py -l
```

In another terminal, connect to the VM:

```bash
ssh ubuntu@192.168.122.10
```

Use password `ubuntu` unless you changed the cloud-init configuration.

## 6. Build and Run the Examples

The eBPF examples should be built and run inside the VM after it has booted the
vBPF kernel. This section separates host-side copy commands from VM-side build
and evaluation commands.

### 6.1 Copy the Examples into the VM

From a host terminal, copy the examples repository into the running VM:

```bash
scp -r "$AE_ROOT/ebpf-bootstrap" ubuntu@192.168.122.10:~
```

### 6.2 Build the Examples inside the VM

SSH into the VM and build the examples:

```bash
ssh ubuntu@192.168.122.10
cd ~/ebpf-bootstrap

sudo apt update
sudo apt install -y \
  clang cmake make pkg-config libelf-dev zlib1g-dev git \
  libbfd-dev libcap-dev llvm-dev openssl libssl-dev \
  ninja-build g++-14

git submodule update --init --recursive

CXX=g++-14 cmake -S . -B build -G Ninja
cmake --build build
```

The first CMake configure may fetch CLI11 for command-line parsing, so the VM
needs network access unless that dependency has already been populated.

Before running the examples, check that the VM kernel exposes BTF:

```bash
ls /sys/kernel/btf/vmlinux
```

### 6.3 Run a Basic Loader

Use `kprobe` as a first end-to-end check. In VM terminal 1, start the loader:

```bash
sudo ./build/kprobe
```

In VM terminal 2, mount debugfs if needed and read trace output:

```bash
sudo mount -t debugfs none /sys/kernel/debug
sudo cat /sys/kernel/debug/tracing/trace_pipe
```

In VM terminal 3, trigger file deletion events for the kprobe program:

```bash
cd ~/ebpf-bootstrap
./trigger_kprobe.sh
```

The examples repository also includes a helper test script:

```bash
sudo ./test.sh ./build/kprobe
```

### 6.4 Inspect Programs and Maps

Use the bpftool built by this repository to inspect the current BPF namespace:

```bash
sudo ./build/bpftool/bpftool prog show
sudo ./build/bpftool/bpftool map show
```

These commands are useful after starting a loader in another terminal.

### 6.5 Exercise vBPF Namespaces

Run one loader in a fresh BPF namespace with the artifact's `unshare` helper:

```bash
sudo ./build/unshare -b ./build/kprobe
```

The argument after `-b` can be any built loader executable, for example
`./build/raw_tracepoint` or `./build/sockfilter -i lo`.

To inspect programs and maps from a fresh BPF namespace, run `bpftool` through
`unshare` as well:

```bash
sudo ./build/unshare -b ./build/bpftool/bpftool prog show
sudo ./build/unshare -b ./build/bpftool/bpftool map show
```

For repeated loaders in one namespace or across nested BPF namespaces, use:

```bash
sudo ./multi.sh 4 -- ./build/kprobe
sudo ./ns_multi.sh ./build/kprobe 4 2
```

Useful helper tools built by the examples repository include:

- `build/unshare`: creates a new BPF namespace, for example
  `./build/unshare -b <exe>`.
- `build/bpftool/bpftool`: inspects programs and maps in the current namespace,
  for example `./build/bpftool/bpftool prog show`.
- `multi.sh` and `ns_multi.sh`: run multiple eBPF loader instances in one or
  more namespaces.

### 6.6 Optional: Run Other BPF Tools

The VM is a normal Linux environment booted with the vBPF kernel, so reviewers
can also use other BPF tools in addition to the artifact examples. For example,
install and run `bpftrace` inside the VM:

```bash
sudo apt install -y bpftrace
sudo bpftrace -e 'tracepoint:syscalls:sys_enter_getpid { printf("pid=%d\n", pid); }'
```

In another VM shell, trigger the tracepoint:

```bash
python3 -c 'import os; os.getpid()'
```

These tools are useful for sanity-checking the general BPF environment. The
artifact evaluation itself should still use the vBPF examples and helper tools
above when checking namespace behavior.

## Troubleshooting

- If the kernel build cannot load `VBPFAttribute.so`, re-export `LLVM_HOME`,
  `PATH`, and `LD_LIBRARY_PATH` from step 1.
- If `uv run vm.py` reports that the custom kernel is missing, check
  `$AE_ROOT/vm/config.json` and make sure `kernel.path` points to the absolute
  `bzImage` path under `linux-vbpf`.
- If SSH fails, verify that `virbr0` exists, the VM booted successfully, and
  the guest address is `192.168.122.10`.
- If the VM network was initialized before the bridge was configured correctly,
  fix the bridge first, then recreate the Ubuntu image from a clean initial
  state:
  ```bash
  cd "$AE_ROOT/vm"
  rm -f ubuntu.img noble-server-cloudimg-amd64.img cloud-init/seed.img
  uv run vm.py --prepare
  uv run vm.py --install
  ```
- If the examples cannot generate `vmlinux.h`, make sure the VM is running the
  vBPF kernel and `/sys/kernel/btf/vmlinux` exists.
- The LLVM build and VM image download both require significant time and disk
  space.
