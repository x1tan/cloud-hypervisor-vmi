# Cloud Hypervisor VMI Implementation Plan

## Development Principles

### Code Quality Requirements
1. **Idiomatic and Robust Code Only**
   - All implementations MUST follow Rust best practices and Cloud Hypervisor patterns
   - NO hacky workarounds - if a solution feels hacky, research the proper approach
   - Use existing Cloud Hypervisor abstractions and patterns
   - Proper error handling with Result types - no panics in production paths
   - Follow the project's existing code style and conventions

2. **Mandatory Testing**
   - **NO task is complete without testing** - todos remain "in_progress" until tested
   - Every change MUST be tested with a running Cloud Hypervisor instance
   - VMI library integration MUST be verified with actual event flow
   - Test both success paths and error conditions
   - Performance impact must be measured

### Testing Protocol
Before marking any implementation task as complete:
1. Build Cloud Hypervisor with VMI features enabled
2. Run a VM with VMI configuration
3. Execute VMI test client to verify event delivery
4. Test error scenarios and edge cases
5. Measure performance overhead
6. Document test results

## Current Status: Research Phase

### Completed Research
1. ✅ Analyzed Cloud Hypervisor's VM exit handling architecture
2. ✅ Studied hypervisor abstraction layer and integration points
3. ✅ Examined existing event monitoring infrastructure
4. ✅ Investigated KVM exit handling and available exit reasons
5. ✅ Identified proper integration points for VMI

### Key Findings

#### 1. VM Exit Handling Architecture
- Cloud Hypervisor uses kvm-ioctls crate which provides `VcpuExit` enum from KVM
- Currently only handles a subset of KVM exits in `hypervisor/src/kvm/mod.rs`:
  - IoIn/IoOut (x86_64 only)
  - MmioRead/MmioWrite
  - IoapicEoi, Shutdown, Hlt (x86_64)
  - SystemEvent (aarch64)
  - Hyperv, Debug, Tdx
- Many KVM exit reasons are NOT handled, including those critical for VMI:
  - KVM_EXIT_EXCEPTION (page faults, breakpoints, etc.)
  - KVM_EXIT_HLT
  - KVM_EXIT_EPT_VIOLATION
  - KVM_EXIT_EPT_MISCONFIG
  - And many others

#### 2. Proper VMI Integration Point
- **NOT** by extending the `VmExit` enum - this is the abstracted hypervisor-agnostic layer
- **INSTEAD**: Hook into the KVM implementation's `run()` method where raw `VcpuExit` is processed
- VMI events are derived from KVM exits, not separate exit types
- Need to intercept ALL KVM exits before they're filtered/processed

#### 3. Host-Level Communication
- VMI crate runs as separate process on host
- Communication via shared memory between Cloud Hypervisor and VMI process
- NOT using guest memory for event ring buffer
- Standard IPC shared memory (e.g., using memfd_create or shm_open)

## Proposed Architecture

### 1. VMI Event Interception

```rust
// In hypervisor/src/kvm/mod.rs - modify the run() method
fn run(&self) -> std::result::Result<cpu::VmExit, cpu::HypervisorCpuError> {
    match self.fd.lock().unwrap().run() {
        Ok(run) => {
            // NEW: Send raw VcpuExit to VMI subsystem before processing
            if let Some(vmi_manager) = &self.vmi_manager {
                let vmi_action = vmi_manager.handle_vcpu_exit(&run, self.id)?;
                match vmi_action {
                    VmiAction::Skip => return Ok(cpu::VmExit::Ignore),
                    VmiAction::Inject(exception) => {
                        // Handle injection
                    },
                    VmiAction::Continue => {}, // Continue normal processing
                }
            }
            
            // Existing exit handling logic
            match run {
                // ... existing cases ...
            }
        }
    }
}
```

### 2. VMI Event Types (derived from KVM exits)

```rust
pub enum VmiEventType {
    Exception {
        vector: u8,
        error_code: u32,
        cr2: u64, // For page faults
    },
    EptViolation {
        gpa: u64,
        gva: u64,
        access_type: MemoryAccessType,
    },
    Hypercall {
        nr: u64,
        args: [u64; 6],
    },
    IoAccess {
        port: u16,
        size: u8,
        is_write: bool,
        data: u32,
    },
    MmioAccess {
        gpa: u64,
        size: u8,
        is_write: bool,
        data: Vec<u8>,
    },
    Interrupt {
        irq: u32,
    },
    CpuidAccess {
        function: u32,
        index: u32,
    },
    MsrAccess {
        msr: u32,
        is_write: bool,
        value: u64,
    },
    // Map other KVM exit reasons as needed
}
```

### 3. Host-Level Shared Memory Communication

```rust
// Host shared memory structure for VMI communication
pub struct VmiSharedMemory {
    // Shared memory created with memfd_create or shm_open
    shm_fd: RawFd,
    // Memory-mapped regions
    event_ring: *mut VmiEventRing,
    response_ring: *mut VmiResponseRing,
}

pub struct VmiEventRing {
    // Lock-free ring buffer for events
    head: AtomicU64,
    tail: AtomicU64,
    size: u64,
    events: [VmiEvent; RING_SIZE],
}

pub struct VmiEvent {
    event_id: u64,
    vcpu_id: u8,
    event_type: VmiEventType,
    timestamp: u64,
    // Event-specific data
}

pub struct VmiResponse {
    event_id: u64,
    action: VmiAction,
}

pub enum VmiAction {
    Continue,
    Skip,
    InjectException { vector: u8, error_code: u32 },
    ModifyRegisters { regs: StandardRegisters },
    SingleStep,
}
```

### 4. VMI Manager Integration

```rust
// New component in VMM
pub struct VmiManager {
    // Shared memory for event/response rings
    shared_mem: VmiSharedMemory,
    // Event filtering configuration
    event_filter: VmiEventFilter,
    // Guest memory access
    guest_memory: GuestMemoryAtomic<GuestMemoryMmap>,
    // Performance counters
    stats: VmiStatistics,
}

impl VmiManager {
    pub fn handle_vcpu_exit(&self, exit: &VcpuExit, vcpu_id: u8) -> Result<VmiAction> {
        // Convert KVM exit to VMI event
        let event = self.convert_exit_to_event(exit, vcpu_id)?;
        
        // Check if event should be filtered
        if !self.event_filter.should_intercept(&event) {
            return Ok(VmiAction::Continue);
        }
        
        // Send event through shared memory
        self.send_event(event)?;
        
        // Wait for response (with timeout)
        let response = self.wait_for_response()?;
        
        Ok(response.action)
    }
}
```

### 5. VMI Crate Structure

```
vmi/
├── Cargo.toml
├── src/
│   ├── lib.rs              # Public API
│   ├── client.rs           # Client interface for external tools
│   ├── events.rs           # Event handling and types
│   ├── memory.rs           # Guest memory introspection
│   ├── registers.rs        # CPU state introspection
│   ├── shared_memory.rs    # IPC shared memory implementation
│   └── filters.rs          # Event filtering configuration
```

### 6. Guest Memory Introspection API

```rust
// Separate from event communication - for VMI operations
pub trait GuestMemoryIntrospection {
    // Read guest virtual memory
    fn read_va(&self, vcpu_id: u8, va: u64, buf: &mut [u8]) -> Result<()>;
    
    // Read guest physical memory
    fn read_pa(&self, pa: u64, buf: &mut [u8]) -> Result<()>;
    
    // Write guest virtual memory
    fn write_va(&self, vcpu_id: u8, va: u64, data: &[u8]) -> Result<()>;
    
    // Write guest physical memory
    fn write_pa(&self, pa: u64, data: &[u8]) -> Result<()>;
    
    // Translate virtual to physical address
    fn translate_va(&self, vcpu_id: u8, va: u64) -> Result<u64>;
    
    // Set memory permissions (for EPT/NPT)
    fn set_memory_permissions(&self, pa: u64, size: u64, perms: MemoryPermissions) -> Result<()>;
}
```

## Implementation Phases

### Phase 1: Foundation (Weeks 1-2)
- [ ] Create VMI crate structure
- [ ] Implement host-level shared memory IPC
- [ ] Add VMI manager skeleton to VMM
- [ ] Hook into KVM vcpu run() method
- [ ] **Testing**: Basic VM boot with VMI enabled, verify no regression

### Phase 2: Core VMI Events (Weeks 3-4)
- [ ] Map all relevant KVM exit reasons to VMI events
- [ ] Implement event ring buffer
- [ ] Add event filtering mechanism
- [ ] Create response handling
- [ ] **Testing**: Event delivery for each exit type, measure latency

### Phase 3: Memory Introspection (Weeks 5-6)
- [ ] Implement guest physical memory read/write
- [ ] Add virtual address translation
- [ ] Create EPT/NPT permission management
- [ ] Support memory event notifications
- [ ] **Testing**: Memory access from VMI client, page fault handling

### Phase 4: CPU State Introspection (Week 7)
- [ ] Access guest registers
- [ ] Control single-stepping
- [ ] Inject exceptions/interrupts
- [ ] Manage guest debug registers
- [ ] **Testing**: Register modification, single-step debugging

### Phase 5: Client Library & Tools (Week 8-9)
- [ ] Design client API
- [ ] Create example VMI client
- [ ] Add compatibility layer for libvmi-style API
- [ ] Documentation and testing
- [ ] **Testing**: Full integration test with DRAKVUF-like workload

### Testing Infrastructure (Ongoing)
- [ ] Unit tests for each VMI component
- [ ] Integration tests with running VMs
- [ ] Performance benchmarks
- [ ] Stress tests for event handling
- [ ] Error injection tests

## Technical Considerations

### 1. Performance
- Zero-copy event delivery via shared memory
- Lock-free ring buffers for low latency
- Event filtering at VMM level to reduce overhead
- Batch processing of events where possible

### 2. Security
- Validate all client inputs
- Secure shared memory setup (proper permissions)
- Prevent VMI client from corrupting VMM state
- Rate limiting for event delivery

### 3. Compatibility
- Support both KVM and MSHV where applicable
- Design for future extension to other architectures
- Maintain Cloud Hypervisor's existing APIs
- Optional VMI feature (compile-time flag)

### 4. Code Review Criteria
Before any VMI code is merged:
- [ ] Follows Rust idioms and Cloud Hypervisor patterns
- [ ] No unsafe code without thorough justification
- [ ] All error paths handled properly
- [ ] Unit tests pass
- [ ] Integration tests with running VM pass
- [ ] Performance benchmarks show acceptable overhead
- [ ] Documentation is complete
- [ ] No regressions in existing functionality

## Next Steps

1. **Architecture Review**: Get feedback on proposed design
2. **Prototype**: Start with basic event interception and shared memory
3. **Performance Testing**: Measure overhead of VMI hooks
4. **API Design**: Define public interfaces for VMI clients

## Open Questions

1. Should VMI be a compile-time feature or runtime configurable?
2. How to handle VMI during live migration?
3. What level of libvmi API compatibility is needed?
4. Should we support multiple simultaneous VMI clients?
5. How to coordinate VMI with existing guest_debug feature?

## References

- [KVM API Documentation](https://www.kernel.org/doc/Documentation/virt/kvm/api.txt)
- [libvmi](https://github.com/libvmi/libvmi) - Reference VMI implementation
- [DRAKVUF](https://github.com/tklengyel/drakvuf) - Target use case
- [kvm-ioctls](https://github.com/rust-vmm/kvm-ioctls) - Rust KVM bindings