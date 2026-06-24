# apple/container — Lightweight Linux VMs per Container on Apple Silicon

**Source:** https://github.com/apple/container
**Saved:** 2026-06-17
**Tags:** ai, security, infrastructure, tools, technology

> Apache 2.0 licensed. Written in Swift. Apple Silicon only. macOS 26 required for full feature set. Active development — stability only guaranteed within patch versions until 1.0.

---

## TL;DR
Apple's open-source `container` tool runs each Linux container as its own lightweight virtual machine on Apple Silicon Macs — rather than sharing a single Linux VM across all containers (the Docker approach). Each container gets the isolation guarantees of a full VM. OCI-compatible (pull/push to any standard registry). Integrates natively with macOS frameworks (Virtualization, vmnet, XPC, Launchd, Keychain). Includes a "container machine" mode for persistent Linux developer environments with your Mac home directory automatically mounted inside.

## Key Concepts & Terms

- **VM-per-container**: The defining architectural choice. Docker and most container runtimes run a single Linux VM and multiplex many containers inside it. `container` gives each container its own minimal VM, using Apple's Virtualization framework. Each VM boots fast (comparable to containers in a shared VM) but has the security boundary of a full hypervisor-level isolation.
- **Containerization package**: The underlying Swift package (`github.com/apple/containerization`) that provides low-level container, image, and process management. `container` is the CLI built on top of it.
- **OCI compatibility**: `container` consumes and produces standard OCI container images. Images built with `container` run anywhere (Kubernetes, Docker, cloud); images from any OCI registry run in `container`.
- **Container machine**: A persistent Linux environment (not a transient container) where your Mac username and home directory are automatically mapped into the Linux VM. Designed for developer workflows: edit files on macOS, compile/run inside the Linux environment. Runs the image's init system, so `systemctl` works.
- **Builder**: A utility container (itself a lightweight VM) that handles `container build` operations (Dockerfile → OCI image). Configurable CPU and memory limits. Defaults to 2GB RAM / 2 CPUs — resource-intensive builds need explicit `container builder start --cpus N --memory Ng` before building.
- **vmnet**: Apple's virtual networking framework used for container-to-container and container-to-host networking. macOS 26 adds container-to-container isolation controls and multiple network support not available on macOS 15.
- **macOS 26 requirement**: The tool officially requires macOS 26 for its full feature set (network isolation, multiple networks, container IP addressing). macOS 15 is technically runnable but not supported for issues.

## Architecture

```
container CLI
  ↕ client library (XPC)
container-apiserver (launch agent)
  ├── container-core-images (XPC helper — image management, local content store)
  ├── container-network-vmnet (XPC helper — virtual network)
  └── container-runtime-linux (per-container XPC helper — one per running container)
       └── Lightweight Linux VM (Apple Virtualization framework)
            └── Your containerized application
```

Each container gets its own `container-runtime-linux` process and its own VM. There is no shared Linux kernel between containers.

## Why VM-Per-Container (vs. Shared VM)

| Dimension | Shared VM (Docker on Mac) | VM-per-container (apple/container) |
|-----------|--------------------------|-------------------------------------|
| Security isolation | Container-level (namespace/cgroup) | Hypervisor-level (full VM boundary) |
| Volume mounting | Mount everything upfront into the shared VM | Mount only what each container needs |
| Memory on macOS | One VM holds all container memory | Each VM uses only what its app needs |
| Boot time | Fast (container layer only) | Comparable (lightweight VM) |
| Shared kernel | Yes — compromise one, affects others | No — separate kernel per container |

The privacy advantage: with Docker's shared VM, you typically mount your entire workspace into the VM upfront, then selectively mount into containers. With `container`, each VM only receives the specific volumes it needs — nothing else is visible.

## Installation and Basic Usage

```bash
# Install from GitHub releases (signed installer package)
# https://github.com/apple/container/releases

# Start the system service
container system start

# Run a container (each gets its own lightweight VM)
container run --rm docker.io/alpine:latest echo "Hello"

# Run with resource limits
container run --rm --cpus 8 --memory 32g my-heavy-image

# Share host files
container run --volume ${HOME}/projects:/projects docker.io/python:alpine ls /projects

# Build an image
container build --tag my-app:latest --file Dockerfile .

# Build multiplatform (arm64 + amd64 via Rosetta)
container build --arch arm64 --arch amd64 --tag registry.io/me/app:latest .

# Push to registry
container image push registry.io/me/app:latest
```

## Container Machine Mode

For persistent developer environments (not transient containers):

```bash
# Create a persistent Linux environment from an OCI image
container machine create alpine:latest --name dev

# Run a command inside (boots the machine if stopped)
container machine run -n dev whoami   # your macOS username, automatically mapped
container machine run -n dev pwd      # /home/<you> — your Mac home, mounted in

# Interactive shell
container machine run -n dev          # drops into a shell

# Set a default to skip -n flag
container machine set-default dev
container machine run                 # operates on dev

# Configure resources (take effect after next stop/start)
container machine set -n dev cpus=4 memory=8G
container machine stop dev && container machine run -n dev

# List / inspect / delete
container machine ls
container machine inspect dev         # JSON detail
container machine rm dev              # delete including persistent storage
```

**Use case for developers:** Edit code in VS Code on macOS. Compile and test inside the container machine (which sees the same files via the mounted home directory). No Docker Desktop overhead, no file sync lag, no "it works on my machine" because your machine is Linux.

**Nested virtualization**: Supported on M3 or later + macOS 15 or later, with a KVM-enabled Linux kernel (not the default). Allows running VMs inside container machines.

## Key Limitations (Current Release)

**Memory balloon**: macOS Virtualization framework has partial balloon support. Memory freed by processes inside the VM is not always returned to the host. Running many memory-intensive containers may require occasional container restarts to reclaim host memory.

**macOS 15 network limitations**: Container-to-container networking doesn't work. Only the default vmnet network is available. IP addressing can fail if vmnet and the network helper disagree on the subnet (192.168.64.1/24 default).

**Active development**: Not yet at 1.0. Breaking changes may occur between minor versions (0.x → 0.y). Only patch versions (0.x.y → 0.x.z) are guaranteed stable.

## Relevance to AI Agent Sandboxing

`container` is directly relevant to the AI agent sandbox problem — the question of how to safely isolate code that an AI agent writes and runs.

The VM-per-container model provides the kernel isolation that Docker alone cannot. In the sandboxing research cluster (code-sandboxes, smolvm comparison, self-hosted AI sandboxes), the consistent finding is that Docker's container-level isolation (shared kernel) is insufficient against malicious code exploiting kernel vulnerabilities. The smolvm/Firecracker comparison notes that eBPF kernel rootkits can escape Docker but not Firecracker (which uses the same hypervisor-level approach as `container`).

Apple's implementation is notable because it uses the macOS Virtualization framework (hardware-level) directly, with Apple Silicon's memory tagging and security features underneath it. For AI coding agents running on Apple Silicon Macs, `container` provides Firecracker-equivalent isolation without needing to run Linux infrastructure.

The builder pattern (a separate VM that builds images, configurable independently from the containers it produces) mirrors the agent/sandbox separation recommended in the agent security notes — the build environment and the runtime environment are distinct isolated units.

## Questions & Gaps
- How does `container` handle networking for AI agent workflows that need internet access (to pull packages) but should not exfiltrate data? The vmnet framework provides isolation between containers, but egress filtering (restricting which external hosts a container can reach) isn't documented.
- The Containerization Swift package is the actual library — for integration with non-CLI workflows (programmatic container management from Swift), is the package API stable?
- Memory balloon limitations: at what practical container count does the memory reclaim problem become a real operational concern? The docs say "you may need to occasionally restart containers" — this needs more specificity for production AI agent fleet use.
- Container machine vs standard container: is there a performance penalty for running the init system (systemd) vs a minimal container entrypoint?
- Rosetta translation for `--arch amd64` on Apple Silicon — what's the performance overhead for CPU-intensive workloads (e.g., ML inference, compilation)?

## Related Notes
- [Code Sandboxes for LLMs: Docker vs gVisor vs Firecracker](https://github.com/LutherCalvinRiggs/research/blob/main/ai/security/code-sandboxes-llms-ai-agents-docker-gvisor-firecracker.md) — `container` uses Apple's Virtualization framework for the same hypervisor-level isolation that Firecracker provides on Linux. The conclusion of that note (Docker insufficient for untrusted code, need VM-level isolation) is exactly what `container` delivers natively on Mac.
- [smolVM vs Firecracker vs Docker Comparison](https://github.com/LutherCalvinRiggs/research/blob/main/ai/security/smolvm-vs-firecracker-vs-docker-sandbox-comparison.md) — `container` is Apple's equivalent of smolVM/Firecracker in the Mac ecosystem. Same architectural principle (VM-per-workload), different implementation (Apple Virtualization framework vs KVM).
- [Self-Hosted AI Sandboxes Guide](https://github.com/LutherCalvinRiggs/research/blob/main/ai/infrastructure/self-hosted-ai-sandboxes-guide-2026.md) — `container` fills the Mac-native slot in the self-hosted sandbox landscape that was previously underserved (Docker Desktop is the common option but lacks VM-level isolation).
- [NEEDLE Production Safety Guide](https://github.com/LutherCalvinRiggs/research/blob/main/repos/needle/NEEDLE-Production-Safety-Guide.md) — the safety guide's Layer 2 (network isolation) and Layer 3 (filesystem isolation) map directly to `container`'s vmnet and volume-mounting model. NEEDLE workers on Apple Silicon could run inside `container` VMs for the strongest available isolation.
- [Mac Mini Local AI Server ($3/month)](https://github.com/LutherCalvinRiggs/research/blob/main/ai/tools/mac-mini-local-ai-server-3-dollars-month.md) — a Mac Mini running `container` as the agent sandbox layer is a compelling combination: local inference (Ollama) + VM-per-agent isolation (container) on Apple Silicon hardware, with no cloud infrastructure.
