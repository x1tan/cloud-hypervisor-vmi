# Cloud Hypervisor VMI - Session Continuation on x86 Linux

## Context for New Session

You are continuing work on implementing Virtual Machine Introspection (VMI) capabilities for Cloud Hypervisor. The previous session was on macOS where KVM wasn't available. Now you're on an x86 Linux machine where you can properly test Cloud Hypervisor with KVM.

## Original Task

Test and verify that you can:
1. Build Cloud Hypervisor
2. Run a test VM
3. Debug/test the hypervisor
This is a prerequisite before implementing VMI features.

## Project Status

### Already Completed (on macOS)
- ✅ Researched VMI architecture and created `PLAN.md`
- ✅ Created `CLAUDE.md` with development guidelines
- ✅ Built Cloud Hypervisor in container (ARM64 target)
- ✅ Established that idiomatic, robust code with thorough testing is required

### What Needs to Be Done

1. **Verify KVM is available**:
   ```bash
   ls -la /dev/kvm
   lsmod | grep kvm
   ```

2. **Build Cloud Hypervisor for x86_64**:
   ```bash
   cargo build --release --features kvm
   ```

3. **Download test kernel and disk image**:
   ```bash
   # Create workloads directory
   mkdir -p ~/workloads
   cd ~/workloads
   
   # Download a test kernel (check for latest URLs)
   wget https://github.com/cloud-hypervisor/cloud-hypervisor/releases/download/v41.0/vmlinux
   
   # Download a test disk image (Ubuntu cloud image)
   wget https://cloud-images.ubuntu.com/focal/current/focal-server-cloudimg-amd64.img
   
   # Convert qcow2 to raw if needed
   qemu-img convert -O raw focal-server-cloudimg-amd64.img focal-server-cloudimg-amd64.raw
   ```

4. **Run a test VM**:
   ```bash
   sudo ./target/release/cloud-hypervisor \
       --kernel ~/workloads/vmlinux \
       --disk path=~/workloads/focal-server-cloudimg-amd64.raw \
       --cmdline "console=hvc0 root=/dev/vda1 rw" \
       --cpus boot=2 \
       --memory size=512M \
       --net "tap=,mac=,ip=,mask="
   ```

5. **Test debugging capabilities**:
   - Build with debug symbols
   - Test with GDB: `--gdb <port>`
   - Verify you can set breakpoints in hypervisor code

## Important Files to Review

1. **PLAN.md** - Contains the full VMI implementation plan
2. **CLAUDE.md** - Development guidelines and testing requirements
3. **VMI_DEVELOPMENT_STATUS.md** - Current development status

## Key Requirements from Previous Session

1. **Code Quality**:
   - ALWAYS write idiomatic Rust code
   - NO hacky workarounds
   - Follow Cloud Hypervisor's existing patterns
   - Proper error handling

2. **Testing**:
   - NO task is complete without testing
   - Must test with running Cloud Hypervisor
   - Test error conditions
   - Verify no performance regressions

## VMI Architecture Summary

The plan is to:
1. Hook into KVM's vcpu `run()` method to intercept ALL VM exits
2. Create host-level shared memory for VMI event delivery (NOT guest memory)
3. Implement a ring buffer for events and responses
4. Create a separate VMI crate for the client library

The integration point is in `hypervisor/src/kvm/mod.rs` where we'll intercept the raw `VcpuExit` before it's processed.

## Next Steps After Testing

Once you verify Cloud Hypervisor runs properly:
1. Start implementing VMI hooks in the KVM layer
2. Create shared memory infrastructure
3. Build the VMI event system
4. Test with a simple VMI client

## Dependencies You Might Need

```bash
# For building
sudo apt install build-essential git

# For running Cloud Hypervisor
sudo apt install qemu-utils  # for qemu-img

# For creating TAP interfaces
sudo apt install bridge-utils net-tools
```

## Success Criteria

You should be able to:
- ✅ See the VM boot in the console
- ✅ Interact with the VM (login if cloud-init is set up)
- ✅ Stop the VM gracefully
- ✅ Attach GDB and set breakpoints
- ✅ See VM exits being processed

Once these are verified, you can proceed with VMI implementation following the PLAN.md.