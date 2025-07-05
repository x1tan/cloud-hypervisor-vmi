# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Cloud Hypervisor is an open-source Virtual Machine Monitor (VMM) written in Rust that runs on KVM and MSHV hypervisors. It's designed for running modern cloud workloads with a focus on high performance, minimal emulation, and small attack surface.

**Key characteristics:**
- 64-bit only (x86_64, AArch64, experimental riscv64)
- Built on Rust VMM crates ecosystem
- Supports Linux and Windows guests
- Features CPU/memory/device hotplug and live migration

## Build Commands

```bash
# Standard release build
cargo build --release

# Build with specific hypervisor
cargo build --release --features kvm
cargo build --release --features mshv

# Static builds for deployment
cargo build --release --target=x86_64-unknown-linux-musl --all

# Build with additional features
cargo build --release --features "tdx"           # Intel TDX support
cargo build --release --features "sev_snp"       # AMD SEV-SNP support
cargo build --release --features "guest_debug"   # GDB debugging support
```

## Testing Commands

```bash
# Run unit tests
./scripts/run_unit_tests.sh

# Using dev_cli.sh (containerized environment)
./scripts/dev_cli.sh tests --unit
./scripts/dev_cli.sh tests --integration

# Run specific integration test suites
./scripts/run_integration_tests_x86_64.sh
./scripts/run_integration_tests_live_migration.sh
```

## Linting and Code Quality

```bash
# Format code
cargo fmt

# Check formatting
cargo fmt -- --check

# Run clippy
cargo clippy --all --all-targets --tests --examples -- -D warnings -D clippy::undocumented_unsafe_blocks

# Using dev_cli.sh
./scripts/dev_cli.sh clippy
```

## Architecture Overview

The project follows a modular workspace structure:

### Core Components
- **vmm/** - Virtual Machine Manager orchestrating all VM operations
- **hypervisor/** - Abstraction layer for KVM/MSHV hypervisors
- **virtio-devices/** - VirtIO device implementations
- **devices/** - Legacy and platform devices
- **arch/** - Architecture-specific code (x86_64, aarch64, riscv64)

### Key Subsystems
- **DeviceManager** (vmm/src/device_manager.rs) - Manages all VM devices
- **MemoryManager** (vmm/src/memory_manager.rs) - Guest memory management
- **CpuManager** (vmm/src/cpu.rs) - vCPU management and hotplug
- **API** (vmm/src/api/) - REST and D-Bus API implementations

### Device Model
- Primary: VirtIO paravirtualized devices
- VFIO: PCIe device passthrough
- vhost-user: Offloading to external processes
- Legacy: Minimal set for boot (serial, RTC, etc.)

## Important Development Considerations

### When modifying device code:
- Check virtio-devices/ for VirtIO implementations
- Device activation follows specific VirtIO protocol
- Use vm-device crate traits for device management
- Test with both KVM and MSHV when possible

### When working with API:
- REST API defined in vmm/src/api/openapi/cloud-hypervisor.yaml
- API changes must maintain backward compatibility
- Use api_client/ for testing API endpoints

### When adding new features:
- Add appropriate feature flags in Cargo.toml
- Update CI workflows if new dependencies added
- Consider security implications (seccomp filters, etc.)

### Memory and CPU management:
- Memory zones support NUMA configurations
- CPU hotplug requires proper ACPI tables
- Check arch-specific constraints

### Testing requirements:
- Unit tests required for new functionality
- Integration tests for user-visible features
- Use dev_cli.sh for consistent test environment

## VMI Development Guidelines

### Code Quality Standards
- **ALWAYS write idiomatic Rust code** - follow established patterns in the codebase
- **NEVER implement hacky workarounds** - if something seems hacky, find the proper solution
- **Maintain robustness** - handle all error cases properly, no unwraps in production code
- **Follow existing architectural patterns** - study how similar features are implemented

### Testing Requirements
- **NO code task is complete without testing** - mark todos as done only after verification
- **Test with running Cloud Hypervisor** - ensure changes work in real VM scenarios
- **Test VMI library integration** - verify event delivery and response handling
- **Test error conditions** - ensure graceful handling of failures

### Testing Workflow
```bash
# 1. Build with VMI features
cargo build --release --features "kvm,vmi"

# 2. Run Cloud Hypervisor with VMI enabled
./target/release/cloud-hypervisor \
    --kernel ./linux-cloud-hypervisor/arch/x86/boot/compressed/vmlinux.bin \
    --disk path=focal-server-cloudimg-amd64.raw \
    --cmdline "console=hvc0 root=/dev/vda1 rw" \
    --cpus boot=4 \
    --memory size=1024M \
    --vmi socket=/tmp/vmi.sock

# 3. Run VMI test client
./target/release/vmi-client --socket /tmp/vmi.sock --test-events

# 4. Verify event flow and responses
# 5. Check performance impact
# 6. Test edge cases and error handling
```

## Common Development Tasks

```bash
# Run VM with basic configuration
./target/release/cloud-hypervisor \
    --kernel ./linux-cloud-hypervisor/arch/x86/boot/compressed/vmlinux.bin \
    --disk path=focal-server-cloudimg-amd64.raw \
    --cmdline "console=hvc0 root=/dev/vda1 rw" \
    --cpus boot=4 \
    --memory size=1024M \
    --net "tap=,mac=,ip=,mask="

# Debug with GDB (requires guest_debug feature)
./target/release/cloud-hypervisor ... --gdb <port>

# Create cloud-init image
./scripts/create-cloud-init.sh

# Run containerized development environment
./scripts/dev_cli.sh shell
```

## Key Files and Directories

- **vmm/src/lib.rs** - Main VMM entry point and orchestration
- **vmm/src/device_manager.rs** - Device management logic
- **hypervisor/src/lib.rs** - Hypervisor abstraction interface
- **docs/** - Comprehensive documentation for all features
- **tests/integration/** - Integration test suites
- **scripts/** - Development and CI scripts

## Performance and Debugging

- Use `--features tracing` for performance analysis
- io_uring support available for async I/O
- CPU pinning via `--cpus` configuration
- Memory huge pages support for better performance

## Security Considerations

- Seccomp filters enabled by default
- Landlock support for filesystem restrictions
- Minimal device emulation for reduced attack surface
- Regular security audits via cargo-audit