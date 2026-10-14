# Ambiq Driver Documentation for AI Assistants

## Overview

This directory contains comprehensive documentation to help AI coding assistants understand and properly update Ambiq drivers in the Zephyr RTOS project. The documentation covers three layers:

1. **Hardware layer** - Actual registers, capabilities, and timing specs
2. **HAL layer** - How Ambiq's HAL abstracts the hardware
3. **Driver layer** - How to implement Zephyr drivers using the HAL

---

## Documents (Read in Order)

### 1. **`AMBIQ_HARDWARE_REGISTER_GUIDE.md`** - Hardware Registers & Capabilities
**Purpose**: Understand the actual hardware - registers, bit fields, timing, and capabilities

**Essential for**:
- Understanding what the hardware can actually do
- Debugging low-level issues
- Verifying HAL behavior against hardware specs
- Working with register-level code

**Contains**:
- Complete memory maps for Apollo 2/3/4/5
- Detailed register maps for every peripheral (MSPI, IOM, UART, ADC, GPIO, Flash/MRAM, DMA)
- Bit field definitions with masks and positions
- Hardware timing requirements (setup/hold times, clock constraints)
- Interrupt flag definitions and meanings
- DMA descriptor formats and control bits
- Power domain architecture
- Hardware errata and workarounds
- Comparison tables across Apollo series

**Key sections**:
- MSPI: Command queue, DMA descriptors, XIP, timing calibration
- IOM: FIFO architecture, I2C/SPI mode switching, shared resources
- UART: Baud rate calculations, flow control, FIFO depth
- ADC: Slot configuration, DMA mode, conversion timing
- GPIO: Per-pin registers, interrupt configuration
- Flash/MRAM: Why there's no async (hardware limitation)
- DMA: Channel assignments, descriptor chaining
- Clock tree: Frequencies, sources, divisors

**Key takeaway**: Hardware capabilities determine what's possible - start here to understand constraints.

---

### 2. **`AMBIQ_HAL_GUIDE.md`** - Understanding the HAL
**Purpose**: Learn how Ambiq's Hardware Abstraction Layer maps to the hardware registers

**Essential for**:
- Reading and understanding HAL code
- Determining if a peripheral supports async operations
- Understanding HAL function categories
- Debugging HAL-level issues

**Contains**:
- HAL naming conventions and patterns
- Function categories (lifecycle, config, transfer, interrupt, DMA)
- Blocking vs non-blocking operation detection
- Handle-based architecture
- Hardware constraints that affect HAL behavior
- Common HAL pitfalls
- Version differences across Apollo series
- How HAL abstracts register access

**Key sections**:
- How to identify nonblocking APIs
- When HAL provides async vs when it doesn't
- Register access patterns in HAL
- Callback mechanisms
- Power control integration

**Key takeaway**: Not all peripherals can be async - it depends on what the HAL provides. The HAL shields you from registers but also constrains what's possible.

---

### 3. **`AMBIQ_DRIVER_DEVELOPMENT_GUIDE.md`** - How to Update Drivers
**Purpose**: Step-by-step practical guide for implementing Zephyr drivers

**Essential for**:
- Creating new async drivers
- Adding PM Runtime to existing drivers
- Understanding proper locking patterns
- Testing and validation

**Contains**:
- Driver structure setup (config, data, API structs)
- Implementation patterns (init, transfer, ISR, PM)
- PM Runtime integration steps
- Testing procedures and checklists
- Common code patterns from existing drivers
- Best practices and gotchas
- Integration with Zephyr subsystems

**Key sections**:
- Setting up async with semaphores/callbacks
- PM Runtime best practices
- ISR implementation patterns
- DMA setup and teardown
- Error handling

**Key takeaway**: MSPI driver is the gold standard - use it as a template for async operations.

---

### 4. **`ambiq_driver_async_status.md`** - Current Driver Status
**Purpose**: Analysis and assessment of all existing Ambiq drivers

**Essential for**:
- Understanding which drivers need updates
- Priority ranking for improvements
- Current capabilities of each driver

**Contains**:
- Classification of all Ambiq drivers
- Async compliance assessment
- Specific recommendations per driver
- Priority categorization
- Historical analysis and corrections

**Key takeaway**: Most drivers are already good; flash driver is correctly implemented given HAL constraints.

---

## Quick Start for AI

### Reading Order for Maximum Understanding:

```
Step 1: AMBIQ_HARDWARE_REGISTER_GUIDE.md
        ↓ (Understand what hardware can do)
        
Step 2: AMBIQ_HAL_GUIDE.md
        ↓ (Understand how HAL abstracts hardware)
        
Step 3: AMBIQ_DRIVER_DEVELOPMENT_GUIDE.md
        ↓ (Learn implementation patterns)
        
Step 4: ambiq_driver_async_status.md
        ↓ (See current status)
        
Step 5: Look at drivers/mspi/mspi_ambiq_ap5.c
        (Reference implementation)
```

---

## When Asked to Update an Ambiq Driver

### 1. Check Hardware Capabilities First
Consult `AMBIQ_HARDWARE_REGISTER_GUIDE.md`:
- Does the peripheral have DMA support?
- Is there a DCMP (DMA complete) interrupt flag?
- What are the timing constraints?
- Are there hardware errata?

### 2. Check HAL Capabilities
Consult `AMBIQ_HAL_GUIDE.md` and grep the code:
```bash
grep "am_hal_<periph>_nonblocking" modules/hal/ambiq/
```
- If found → Can be async (high value)
- If not found → Check if blocking is acceptable

### 3. Follow Implementation Pattern
Consult `AMBIQ_DRIVER_DEVELOPMENT_GUIDE.md`:
- Async peripheral: Copy pattern from `drivers/mspi/mspi_ambiq_ap5.c`
- PM Runtime only: Copy pattern from `drivers/uart/uart_ambiq.c`

### 4. Test Thoroughly
Use the testing checklist in `AMBIQ_DRIVER_DEVELOPMENT_GUIDE.md`

---

## Key Principles

### ✅ DO:
- **Start with hardware** - understand register-level capabilities
- Use MSPI driver as template for async operations
- Check HAL documentation/code for API availability
- Add PM Runtime when appropriate
- Use proper locking (mutex for threads, spinlock for ISR)
- Always check HAL return codes
- Call `interrupt_service()` in ISR
- Understand timing requirements from hardware guide

### ❌ DON'T:
- Assume all peripherals have async HAL APIs
- Use blocking transfers when nonblocking is available
- Take mutexes in ISR context
- Forget power control sequence (initialize → power → enable)
- Ignore cache coherency for DMA operations
- Skip reading hardware constraints before implementing

---

## Flash Driver Special Case

The flash driver is **correctly implemented** as blocking because:
- **Hardware**: No DMA support, no completion interrupt (see register guide)
- **HAL**: No async APIs (`am_hal_flash_nonblocking_*` doesn't exist)
- **Constraint**: Flash operations must be atomic (cannot be interrupted)
- **Solution**: Chunking (128 bytes) is the proper RTOS pattern
- **Benefit**: Allows scheduler to preempt between chunks

**Conclusion**: No async update needed or possible for flash driver given hardware limitations.

---

## Current Status Summary

| Status | Count | Drivers |
|--------|-------|---------|
| ✅ Fully Compliant | 7 | MSPI, UART, SDHC, DMIC, I2S, RTC, Watchdog |
| ⚠️ Minor Updates | 5 | I2C, SPI, ADC, GPIO, Counter, PWM |
| ❌ Significant Work | 0 | None |

**Result**: Ambiq drivers are in excellent shape overall.

---

## Examples of Good Patterns

### Async Transfer (from MSPI):
```c
// Non-blocking with callback
ret = am_hal_mspi_nonblocking_transfer(
    data->mspiHandle,
    &trans,
    MSPI_DMA,
    callback_func,
    callback_context
);
// Returns immediately, callback invoked from ISR
```

### PM Runtime (from SDHC):
```c
ret = pm_device_runtime_get(dev);
// ... perform operations ...
pm_device_runtime_put_async(dev, K_MSEC(2));
```

### ISR Pattern (standard across all drivers):
```c
static void driver_isr(const struct device *dev) {
    uint32_t status;
    am_hal_<periph>_interrupt_status_get(handle, &status, false);
    am_hal_<periph>_interrupt_clear(handle, status);
    am_hal_<periph>_interrupt_service(handle, status);  // ← Critical!
}
```

### Direct Register Access (when needed):
```c
// Check peripheral busy status
#define MSPI_BUSY BIT(2)
volatile uint32_t *status_reg = (uint32_t *)(cfg->reg_base + 0x008);

if (*status_reg & MSPI_BUSY) {
    return -EBUSY;
}
```

---

## Hardware Insights Summary

These come from `AMBIQ_HARDWARE_REGISTER_GUIDE.md` and are critical to understand:

### Async Capability is Hardware-Dependent
- **Has DMA + completion interrupt** (DCMP flag) → Can be async (MSPI, IOM, UART, ADC, Audio)
- **Only blocking HAL** (Flash/MRAM) → Cannot be async
- **Fast operations** (GPIO, RTC) → Async not needed

### Interrupt Service Pattern
Always call `am_hal_*_interrupt_service()` - it:
- Reads internal HAL state
- Invokes registered callbacks
- Updates transfer status

### DMA Descriptors
- Most async peripherals use DMA descriptors (32 bytes)
- HAL manages descriptor setup
- Driver provides memory and callbacks
- Descriptors can be chained

### Power Domains
- Peripherals can be powered independently
- Must enable before use, disable when idle
- PM Runtime coordinates power state

### Clock Constraints
- Each peripheral has maximum clock frequency
- Setup/hold times vary by speed
- Timing calibration may be needed (MSPI DQS)

### FIFO Management
- Most peripherals have FIFOs (32-128 bytes)
- Threshold interrupts for flow control
- DMA mode avoids CPU FIFO management

---

## For Human Developers

These documents are optimized for AI consumption but are also useful for human developers. They provide:
- Clear decision trees
- Copy-paste examples
- Common pitfall warnings
- Testing checklists
- Hardware-level understanding

The **Hardware Register Guide** is especially valuable for:
- Debugging issues that HAL can't explain
- Understanding performance characteristics
- Working with timing-critical code
- Verifying HAL behavior

---

## Questions to Ask AI

When working with an AI assistant on Ambiq drivers, ask:

1. **Hardware Level**:
   - "What registers control this peripheral?"
   - "What is the interrupt flag for DMA completion?"
   - "What are the timing requirements for this peripheral?"
   - "Does the hardware support DMA for this operation?"

2. **HAL Level**:
   - "Does the HAL provide nonblocking APIs for this peripheral?"
   - "What callback mechanism does this HAL function use?"
   - "How does the HAL abstract these registers?"

3. **Driver Level**:
   - "Show me similar async patterns from MSPI driver"
   - "Is PM Runtime integration needed here?"
   - "What locking mechanism should I use?"
   - "What testing should I do after this change?"

The AI should refer to these documents to give accurate, context-aware answers.

---

## Document Maintenance

**Created**: 2026-10-14  
**Last Updated**: 2026-10-14  
**Author**: AI-generated for AI consumption  

**Review**: Should be updated when:
- New Ambiq SOC series added (Apollo 6x, etc.)
- HAL APIs change significantly
- Zephyr RTOS patterns evolve
- New hardware errata discovered
- Register definitions change

---

## Document Version History

- **v1.0** (2026-10-14): Initial release with HAL and Driver guides
- **v2.0** (2026-10-14): Added comprehensive Hardware Register Guide with detailed register maps, bit fields, timing specs, and hardware capabilities for all peripherals

---

## Feedback

If these documents are missing critical information or could be improved, note it for future updates. The goal is to provide complete, accurate, hardware-to-driver understanding for both AI and human developers.
